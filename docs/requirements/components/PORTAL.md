# Component Requirements: PORTAL

> Web gateway with content filtering, epoch gating, and request quotas.
> Crate: `kingdom-portal`
> Design reference: [../design/07-PORTAL.md](../../design/07-PORTAL.md)

---

## 1. Purpose

PORTAL is the Kingdom's controlled window to the outside world. It allows agents to fetch documentation, access public APIs, and observe real-world software ecosystems for inspiration -- but never to import code directly. All fetched content passes through a code-stripping filter that ensures agents learn concepts, not copy implementations.

PORTAL uses PostgreSQL for request logging, response caching, and domain rule management. Outbound HTTP requests use `reqwest` with `rustls`.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-portal"

[dependencies]
# Workspace
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Database
sqlx = { workspace = true }

# HTTP client
reqwest = { workspace = true }

# Cryptography
sha2 = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
chrono = { workspace = true }
regex = { workspace = true }
```

---

## 3. Data Models

### 3.1 Epoch Gating

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AccessLevel {
    /// Epochs 0-2: no web access at all.
    Closed,
    /// Epoch 3: can fetch pages, cannot interact with APIs.
    ReadOnly,
    /// Epoch 4+: can use APIs, submit queries.
    ReadWrite,
}

impl AccessLevel {
    pub fn from_epoch(epoch: u32) -> Self {
        match epoch {
            0..=2 => AccessLevel::Closed,
            3 => AccessLevel::ReadOnly,
            _ => AccessLevel::ReadWrite,
        }
    }
}

/// Compute per-cycle request quota based on epoch.
pub fn requests_per_cycle(epoch: u32) -> u32 {
    match epoch {
        0..=2 => 0,
        3 => 5,
        e => 5 + (e - 3) * 2,
    }
}
```

### 3.2 HTTP Method

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum HttpMethod {
    Get,
    Post,
    Head,
}
```

### 3.3 Content Filter

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ContentFormat {
    /// Return raw filtered content.
    Raw,
    /// Return structured extraction (headings, paragraphs, lists).
    Structured,
    /// Return a brief summary.
    Summary,
}

/// Default max response size: 64 KB.
pub const DEFAULT_MAX_SIZE: u64 = 65_536;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentFilter {
    /// Strip fenced code blocks from content.
    pub strip_code_blocks: bool,
    /// Strip inline code (backtick spans) -- configurable per request.
    pub strip_inline_code: bool,
    /// Convert API documentation to abstract interface descriptions.
    pub transform_apis: bool,
    /// Convert code examples to pseudocode descriptions.
    pub transform_examples: bool,
    /// Maximum response size in bytes.
    pub max_size: u64,
    /// Output format.
    pub format: ContentFormat,
}

impl Default for ContentFilter {
    fn default() -> Self {
        Self {
            strip_code_blocks: true,
            strip_inline_code: true,
            transform_apis: false,
            transform_examples: false,
            max_size: DEFAULT_MAX_SIZE,
            format: ContentFormat::Raw,
        }
    }
}
```

### 3.4 Portal Request

```rust
use kingdom_core::{Hash256, AgentId};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PortalRequest {
    pub id: Hash256,
    pub agent: AgentId,
    pub url: Vec<u8>,                                               // URL as bytes
    pub method: HttpMethod,
    pub headers: std::collections::HashMap<Vec<u8>, Vec<u8>>,       // restricted headers only
    pub body: Option<Vec<u8>>,                                      // for POST only
    pub filter: ContentFilter,
    pub purpose: Vec<u8>,                                           // structured justification
    pub signature: Vec<u8>,                                         // ed25519 from agent
}
```

### 3.5 Portal Response

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PortalResponse {
    pub request_id: Hash256,
    pub status: u16,                    // HTTP status code
    pub content: Vec<u8>,              // filtered content
    pub content_type: Vec<u8>,
    pub filtered: FilterReport,
    pub cached: bool,
    pub cost: u64,                     // Spark charged
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FilterReport {
    pub code_blocks_removed: u32,
    pub bytes_stripped: u64,
    pub transformations: u32,
    pub warnings: Vec<Vec<u8>>,        // e.g., "high code density detected"
}
```

### 3.6 Cache Policy

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CachePolicy {
    pub ttl: u64,                      // cache duration in cycles
    pub key: Hash256,                  // sha256(url + method + body)
    pub shared: bool,                  // can other agents reuse this cached response?
}
```

### 3.7 API Interaction (Epoch 4+)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ApiInteractionKind {
    RestGet,
    RestPost,
    GraphQL,
    WebsocketSnap,     // one-shot websocket connection snapshot
}
```

### 3.8 Domain Rules

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum DomainRuleAction {
    Allow,
    Block,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DomainRule {
    pub pattern: String,               // glob or regex pattern for domain matching
    pub action: DomainRuleAction,
    pub category: String,              // human-readable category
    pub reason: String,                // why this rule exists
}
```

### 3.9 Portal Report

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PortalReport {
    pub cycle: u64,
    pub total_requests: u32,
    pub cache_hit_rate: f32,
    pub domains_accessed: Vec<Vec<u8>>,
    pub top_users: Vec<(AgentId, u32)>,
    pub code_filter_hits: u32,
    pub blocked_requests: u32,
    pub total_cost: u64,
}
```

### 3.10 Request Costs

```rust
pub mod costs {
    /// (tick_cost, spark_cost)
    pub const GET: (u64, u64) = (3, 2);
    pub const POST: (u64, u64) = (3, 3);
    pub const HEAD: (u64, u64) = (2, 1);
    pub const CACHE_HIT: (u64, u64) = (1, 0);
    pub const FAILED: (u64, u64) = (1, 1);
}
```

---

## 4. PostgreSQL Schema

All tables are in the `portal` schema.

```sql
CREATE SCHEMA IF NOT EXISTS portal;

-- Request log (every request, successful or not)
CREATE TABLE portal.request_log (
    id              BYTEA PRIMARY KEY,           -- 32-byte Hash256 (request id)
    agent           BYTEA NOT NULL,              -- 32-byte AgentId
    url             TEXT NOT NULL,
    method          SMALLINT NOT NULL,           -- 0=GET, 1=POST, 2=HEAD
    purpose         BYTEA,
    status_code     SMALLINT,                    -- HTTP status code (NULL if blocked/failed before request)
    content_size    BIGINT,                      -- response size in bytes after filtering
    cached          BOOLEAN NOT NULL DEFAULT FALSE,
    blocked         BOOLEAN NOT NULL DEFAULT FALSE,
    block_reason    TEXT,
    tick_cost       BIGINT NOT NULL,
    spark_cost      BIGINT NOT NULL,
    code_blocks_removed  INTEGER NOT NULL DEFAULT 0,
    bytes_stripped  BIGINT NOT NULL DEFAULT 0,
    cycle           BIGINT NOT NULL,
    created_at      BIGINT NOT NULL              -- tick
);

CREATE INDEX idx_request_log_agent ON portal.request_log(agent, created_at);
CREATE INDEX idx_request_log_cycle ON portal.request_log(cycle);
CREATE INDEX idx_request_log_url ON portal.request_log(url);

-- Response cache
CREATE TABLE portal.cache (
    cache_key       BYTEA PRIMARY KEY,           -- 32-byte Hash256: sha256(url + method + body)
    url             TEXT NOT NULL,
    method          SMALLINT NOT NULL,
    status_code     SMALLINT NOT NULL,
    content         BYTEA NOT NULL,              -- filtered content
    content_type    TEXT NOT NULL,
    filter_report   BYTEA NOT NULL,              -- MessagePack-encoded FilterReport
    shared          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at_cycle BIGINT NOT NULL,
    expires_at_cycle BIGINT NOT NULL
);

CREATE INDEX idx_cache_expires ON portal.cache(expires_at_cycle);
CREATE INDEX idx_cache_shared ON portal.cache(shared) WHERE shared = TRUE;

-- Domain allowlist/blocklist rules
CREATE TABLE portal.domain_rules (
    id              SERIAL PRIMARY KEY,
    pattern         TEXT NOT NULL UNIQUE,         -- glob or regex domain pattern
    action          SMALLINT NOT NULL,            -- 0=ALLOW, 1=BLOCK
    category        TEXT NOT NULL,
    reason          TEXT NOT NULL,
    created_at      BIGINT NOT NULL,
    updated_at      BIGINT
);

-- Seed default domain rules
INSERT INTO portal.domain_rules (pattern, action, category, reason, created_at) VALUES
    -- Allowed categories
    ('docs.rs',            0, 'documentation', 'Rust documentation', 0),
    ('doc.rust-lang.org',  0, 'documentation', 'Rust standard library docs', 0),
    ('en.wikipedia.org',   0, 'reference',     'General knowledge reference', 0),
    ('developer.mozilla.org', 0, 'documentation', 'Web standards documentation', 0),
    ('www.rfc-editor.org', 0, 'standards',     'IETF RFCs', 0),
    ('www.w3.org',         0, 'standards',     'W3C specifications', 0),
    ('arxiv.org',          0, 'papers',        'Research papers', 0),
    -- Blocked categories
    ('github.com',         1, 'code_repo',     'Prevent direct code copying', 0),
    ('gitlab.com',         1, 'code_repo',     'Prevent direct code copying', 0),
    ('bitbucket.org',      1, 'code_repo',     'Prevent direct code copying', 0),
    ('npmjs.com',          1, 'package_mgr',   'Agents must build their own', 0),
    ('pypi.org',           1, 'package_mgr',   'Agents must build their own', 0),
    ('crates.io',          1, 'package_mgr',   'Agents must build their own', 0),
    ('api.openai.com',     1, 'ai_api',        'No external AI access', 0),
    ('api.anthropic.com',  1, 'ai_api',        'No external AI access', 0),
    ('twitter.com',        1, 'social_media',  'Irrelevant to experiment', 0),
    ('x.com',              1, 'social_media',  'Irrelevant to experiment', 0),
    ('facebook.com',       1, 'social_media',  'Irrelevant to experiment', 0),
    ('reddit.com',         1, 'social_media',  'Irrelevant to experiment', 0);
```

---

## 5. Public Trait

```rust
use kingdom_core::{Hash256, AgentId, EventBus};

#[async_trait::async_trait]
pub trait PortalGateway: Send + Sync {
    /// Process a web request: validate access, check domain rules, fetch, filter, cache, charge.
    async fn handle_request(&self, req: PortalRequest) -> Result<PortalResponse, PortalError>;

    /// Check if a URL is allowed under current domain rules.
    async fn check_domain(&self, url: &[u8]) -> Result<DomainRuleAction, PortalError>;

    /// Get remaining request quota for an agent in the current cycle.
    async fn remaining_quota(&self, agent: AgentId, cycle: u64) -> Result<u32, PortalError>;

    /// Generate the cycle-end portal usage report.
    async fn generate_report(&self, cycle: u64) -> Result<PortalReport, PortalError>;

    /// Evict expired cache entries.
    async fn evict_cache(&self, current_cycle: u64) -> Result<u32, PortalError>;

    /// Add or update a domain rule (governance action).
    async fn update_domain_rule(&self, rule: DomainRule) -> Result<(), PortalError>;

    /// Get the current access level based on epoch.
    fn access_level(&self) -> AccessLevel;
}
```

---

## 6. Inbound Messages

| Code | Name | Payload | Response | Sender |
|------|------|---------|----------|--------|
| `0x0700` | `WEB_REQUEST` | `PortalRequest { id, agent, url, method, headers, body, filter, purpose, signature }` | `WEB_RESPONSE` | Agent |
| `0x0701` | `WEB_RESPONSE` | _(internal: not sent by external agents)_ | — | — |
| `0x0710` | `PORTAL_REPORT` | _(trigger: cycle number)_ | `PortalReport` | Nexus (cycle boundary) |

---

## 7. Outbound Messages / Events

### 7.1 Response Messages

| Code | Name | Payload | Trigger |
|------|------|---------|---------|
| `0x0701` | `WEB_RESPONSE` | `PortalResponse { request_id, status, content, content_type, filtered, cached, cost }` | Web request completed |

### 7.2 Event Kinds (Bus, System = Portal, range 0x6000-0x6FFF)

| Kind | Name | Payload | Emitted When |
|------|------|---------|--------------|
| `0x6000` | `WEB_REQUEST_SUBMITTED` | `{ request_id, agent, url, method, purpose }` | Request received and validated |
| `0x6001` | `WEB_REQUEST_COMPLETED` | `{ request_id, agent, url, status, cached, tick_cost, spark_cost }` | Response delivered to agent |
| `0x6002` | `WEB_REQUEST_BLOCKED` | `{ request_id, agent, url, reason: bytes }` | Request blocked by domain rules or quota |
| `0x6003` | `WEB_REQUEST_FAILED` | `{ request_id, agent, url, error: bytes }` | HTTP request failed (timeout, DNS, etc.) |
| `0x6010` | `CONTENT_FILTERED` | `{ request_id, code_blocks_removed, bytes_stripped, warnings }` | Content filter applied with notable results |
| `0x6020` | `CACHE_HIT` | `{ request_id, agent, cache_key }` | Response served from cache |
| `0x6021` | `CACHE_STORED` | `{ cache_key, url, ttl_cycles, shared }` | New response cached |
| `0x6022` | `CACHE_EVICTED` | `{ cache_key, reason: bytes }` | Cache entry evicted (TTL or space) |
| `0x6030` | `DOMAIN_RULE_UPDATED` | `DomainRule` | Domain rule added or modified |
| `0x6040` | `QUOTA_EXHAUSTED` | `{ agent, cycle, requests_used: u32 }` | Agent hit cycle request quota |
| `0x6050` | `PORTAL_REPORT_PUBLISHED` | `PortalReport` | Cycle-end usage report |

---

## 8. Performance Targets

| Metric | Target |
|--------|--------|
| Request validation + domain check | < 2 ms |
| Cache lookup | < 1 ms |
| Content filtering (64 KB page) | < 50 ms |
| End-to-end latency (cache hit) | < 5 ms |
| End-to-end latency (cache miss, external fetch) | < 5 s (dominated by external HTTP) |
| External HTTP timeout | 10 s max |
| Redirect hop limit | 2 hops max |
| Report generation | < 200 ms |
| Cache eviction scan | < 100 ms |
| Concurrent outbound requests | <= 8 (semaphore-bounded) |

---

## 9. Component Dependencies

| Dependency | Direction | Purpose |
|------------|-----------|---------|
| `kingdom-core` | compile | Hash256, AgentId, Envelope, EventBus, Signature, msg_types, errors |
| PostgreSQL | runtime | Request logging, response cache, domain rules (schema `portal`) |
| Mint (runtime) | outbound `TRANSFER` | Charge Spark cost for each request |
| Nexus (runtime) | inbound | Provides current epoch (for access-level gating) and cycle boundaries (for reports/quota resets) |
| Event Bus | publish/subscribe | Emits all request lifecycle events |
| `reqwest` + `rustls` | compile | Outbound HTTP client with TLS |

---

## 10. Key Algorithms

### 10.1 Request Processing Pipeline

```
fn handle_request(req: PortalRequest) -> Result<PortalResponse>:
    // 1. Epoch gating
    if access_level() == Closed:
        return Err(PortalClosed)
    if access_level() == ReadOnly && req.method == Post:
        return Err(ReadOnlyEpoch)

    // 2. Quota check
    used = count_requests_this_cycle(req.agent)
    max = requests_per_cycle(current_epoch)
    if used >= max:
        emit QUOTA_EXHAUSTED
        return Err(QuotaExceeded)

    // 3. Domain check
    domain = extract_domain(req.url)
    if is_blocked(domain):
        log_request(req, blocked=true, reason)
        emit WEB_REQUEST_BLOCKED
        return Err(DomainBlocked)

    // 4. Cache lookup
    cache_key = sha256(req.url + req.method + req.body)
    if let Some(cached) = lookup_cache(cache_key, req.agent):
        emit CACHE_HIT
        charge_cost(req.agent, costs::CACHE_HIT)
        return Ok(PortalResponse { cached: true, cost: 0, .. })

    // 5. External fetch
    emit WEB_REQUEST_SUBMITTED
    http_response = reqwest_fetch(req.url, req.method, req.headers, req.body, timeout=10s)
    if http_response.is_err():
        charge_cost(req.agent, costs::FAILED)
        emit WEB_REQUEST_FAILED
        return Err(FetchFailed)

    // 6. Content filtering
    (filtered_content, filter_report) = apply_filter(http_response.body, req.filter)
    if filter_report.code_blocks_removed > 0:
        emit CONTENT_FILTERED

    // 7. Cache store
    store_cache(cache_key, filtered_content, filter_report, shared=true)
    emit CACHE_STORED

    // 8. Charge cost
    cost = match req.method:
        Get  => costs::GET
        Post => costs::POST
        Head => costs::HEAD
    charge_cost(req.agent, cost)

    // 9. Log and respond
    log_request(req, http_response.status, filter_report)
    emit WEB_REQUEST_COMPLETED
    return Ok(PortalResponse { ... })
```

### 10.2 Content Filtering

```
fn apply_filter(raw: bytes, filter: ContentFilter) -> (bytes, FilterReport):
    report = FilterReport::default()
    content = raw

    // Truncate to max size first
    if content.len() > filter.max_size:
        content = content[..filter.max_size]

    // Strip fenced code blocks: ```...```
    if filter.strip_code_blocks:
        (content, count) = regex_remove_all(content, r"```[\s\S]*?```")
        report.code_blocks_removed += count
        report.bytes_stripped += (raw.len() - content.len())

    // Strip inline code: `...`
    if filter.strip_inline_code:
        (content, count) = regex_remove_all(content, r"`[^`]+`")
        report.code_blocks_removed += count
        report.bytes_stripped += delta

    // Transform APIs to abstract descriptions
    if filter.transform_apis:
        content = transform_api_docs(content)
        report.transformations += 1

    // Transform examples to pseudocode
    if filter.transform_examples:
        content = transform_examples(content)
        report.transformations += 1

    // High code density warning
    if code_density_score(content) > 0.5:
        report.warnings.push("high code density detected")

    return (content, report)
```

### 10.3 Rate Limiting

```
fn rate_limit_check(agent: AgentId) -> bool:
    // Max 1 request per tick -- use per-agent timestamp tracking
    last_tick = agent_last_request_tick.get(agent)
    if last_tick == current_tick:
        return false  // too fast

    // Auto backoff: if last response was 429 or 503, enforce cooldown
    if agent_backoff.get(agent) > current_tick:
        return false  // in backoff period

    return true
```

### 10.4 Cache Key Computation

```
fn cache_key(url: &[u8], method: HttpMethod, body: Option<&[u8]>) -> Hash256:
    let mut hasher = Sha256::new()
    hasher.update(url)
    hasher.update(&[method as u8])
    if let Some(b) = body:
        hasher.update(b)
    Hash256(hasher.finalize().into())
```

---

## 11. Error Types

```rust
#[derive(Debug, thiserror::Error)]
pub enum PortalError {
    #[error("portal is closed in current epoch")]
    PortalClosed,

    #[error("read-only access: POST/API not allowed in epoch 3")]
    ReadOnlyEpoch,

    #[error("quota exceeded: {used}/{max} requests used this cycle")]
    QuotaExceeded { used: u32, max: u32 },

    #[error("domain blocked: {domain} ({reason})")]
    DomainBlocked { domain: String, reason: String },

    #[error("rate limited: max 1 request per tick")]
    RateLimited,

    #[error("fetch failed: {0}")]
    FetchFailed(String),

    #[error("request timeout after {0}ms")]
    Timeout(u64),

    #[error("too many redirects (max 2 hops)")]
    TooManyRedirects,

    #[error("invalid URL: {0}")]
    InvalidUrl(String),

    #[error("invalid request: {0}")]
    InvalidRequest(String),

    #[error("permission denied: {0}")]
    PermissionDenied(String),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("http client error: {0}")]
    HttpClient(#[from] reqwest::Error),

    #[error("internal error: {0}")]
    Internal(String),
}

impl From<PortalError> for kingdom_core::KingdomError {
    fn from(e: PortalError) -> Self {
        use kingdom_core::ErrorCategory;
        let (category, code) = match &e {
            PortalError::PortalClosed | PortalError::ReadOnlyEpoch => (ErrorCategory::Unauthorized, 0x0700),
            PortalError::QuotaExceeded { .. } | PortalError::RateLimited => (ErrorCategory::QuotaExceeded, 0x0701),
            PortalError::DomainBlocked { .. } => (ErrorCategory::Unauthorized, 0x0702),
            PortalError::FetchFailed(_) | PortalError::Timeout(_) | PortalError::TooManyRedirects => (ErrorCategory::Internal, 0x0703),
            PortalError::InvalidUrl(_) | PortalError::InvalidRequest(_) => (ErrorCategory::InvalidRequest, 0x0704),
            PortalError::PermissionDenied(_) => (ErrorCategory::Unauthorized, 0x0705),
            PortalError::Database(_) | PortalError::HttpClient(_) | PortalError::Internal(_) => (ErrorCategory::Internal, 0x07FF),
        };
        kingdom_core::KingdomError {
            code,
            category,
            message: e.to_string().into_bytes(),
            context: std::collections::HashMap::new(),
        }
    }
}
```

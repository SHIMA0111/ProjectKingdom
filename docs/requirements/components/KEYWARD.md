# Technology Requirements: Keyward

> API key security system. Separate binary with process isolation.
> Crate: `kingdom-keyward`
> Binary: `kingdom-keyward`
> Design reference: [14-KEYWARD.md](../../design/14-KEYWARD.md)

---

## 1. Purpose

Keyward is the Kingdom's outer wall: a process-isolated, memory-locked security component that manages API keys, sponsor identities, LLM proxying, and cost tracking. It is the sole path between the Kingdom and LLM providers. API key plaintext never exists outside Keyward's process boundary. Keyward is a separate binary (`kingdom-keyward`) communicating with the Kingdom via an encrypted Unix domain socket.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-keyward"

[dependencies]
# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# Cryptography
aes-gcm = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }
zeroize = { workspace = true }

# HTTP client (for LLM provider calls)
reqwest = { workspace = true }

# Logging (filtered — no println!/eprintln!)
tracing = { workspace = true }
tracing-subscriber = { workspace = true }

# Utilities
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
regex = { workspace = true }
bytes = { workspace = true }

# Unix
libc = "0.2"
```

### 2.1 Implementation Constraints

| Constraint | Rationale |
|-----------|-----------|
| Rust only, no GC | Direct memory safety control; GC may retain key plaintext |
| Minimal external dependencies | Reduce attack surface |
| TLS via `rustls` | Avoid OpenSSL version management risks |
| `println!`/`eprintln!` banned | Compile-time lint via macro; all output goes through audit log |
| `Debug`/`Display` traits must never contain key values | Prevent `derive(Debug)` accidents; custom impls required |
| No `kingdom-core` dependency | Process isolation; Keyward is fully standalone |

---

## 3. Data Models

### 3.1 Sponsor

```rust
use uuid::Uuid;
use chrono::{DateTime, Utc};

/// A human sponsor who provides API keys and budget.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Sponsor {
    /// Time-sortable UUID v7. Authentication = knowing this UUID.
    pub id: Uuid,
    /// When this sponsor was created.
    pub created_at: DateTime<Utc>,

    // --- Budget ---
    /// Total budget deposited (USD).
    pub budget_total_usd: f64,
    /// Total budget spent (USD).
    pub budget_spent_usd: f64,
    /// Remaining budget (USD).
    pub budget_remaining_usd: f64,

    // --- Statistics (never contains key values) ---
    /// Provider references (fingerprint only, no key material).
    pub providers: Vec<ProviderRef>,
    /// Agent IDs powered by this sponsor's keys.
    pub agents_powered: Vec<[u8; 32]>,

    /// Lifecycle status.
    pub status: SponsorStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum SponsorStatus {
    Active,
    Paused,
    Exhausted,
}
```

### 3.2 ProviderRef

```rust
/// Reference to a registered API key provider (no key material).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProviderRef {
    /// Provider type.
    pub provider_type: ProviderType,
    /// First 16 hex characters of sha256(plaintext_key). For identification only.
    pub key_fingerprint: String,
    /// When the key was registered.
    pub registered_at: DateTime<Utc>,
    /// When the key was last used for an API call.
    pub last_used: Option<DateTime<Utc>>,
    /// Current key status.
    pub status: KeyStatus,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ProviderType {
    Anthropic,
    OpenAi,
    Google,
    OpenAiCompatible { base_url: String },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum KeyStatus {
    Registered,
    Probing,
    Valid,
    Invalid,
    RateLimited,
    Revoked,
}
```

### 3.3 SecureKeyVault

```rust
use aes_gcm::{Aes256Gcm, Key, Nonce};
use zeroize::Zeroize;

/// In-memory encrypted key store. Master key never written to disk.
pub struct SecureKeyVault {
    /// AES-256-GCM master key. Generated from OS CSPRNG at startup.
    /// Zeroed on shutdown. NEVER persisted.
    master_key: Key<Aes256Gcm>,  // [u8; 32], implements Zeroize

    /// key_id -> EncryptedKey
    keys: std::collections::HashMap<Uuid, EncryptedKey>,
}

/// An API key encrypted at rest with the master key.
/// The Debug impl MUST NOT print ciphertext, nonce, or tag.
pub struct EncryptedKey {
    /// Unique identifier for this key.
    pub key_id: Uuid,
    /// Owning sponsor.
    pub sponsor_id: Uuid,
    /// Provider type.
    pub provider: ProviderType,
    /// AES-256-GCM encrypted API key.
    pub ciphertext: Vec<u8>,
    /// GCM nonce (12 bytes).
    pub nonce: [u8; 12],
    /// GCM authentication tag (16 bytes).
    pub tag: [u8; 16],
    /// sha256(plaintext_key)[0..16] hex for display/identification.
    pub fingerprint: String,
    /// When the key was registered.
    pub created_at: DateTime<Utc>,
}

// Custom Debug that never prints key material.
impl std::fmt::Debug for EncryptedKey {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("EncryptedKey")
            .field("key_id", &self.key_id)
            .field("sponsor_id", &self.sponsor_id)
            .field("provider", &self.provider)
            .field("fingerprint", &self.fingerprint)
            .field("created_at", &self.created_at)
            .finish_non_exhaustive()
    }
}

// Custom Display that never prints key material.
impl std::fmt::Display for EncryptedKey {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Key({}, fingerprint={})", self.key_id, self.fingerprint)
    }
}
```

### 3.4 LLM Request/Response

```rust
/// LLM call request from Kingdom (via Unix socket).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMRequest {
    /// Unique request identifier for tracing.
    pub request_id: Uuid,
    /// Cost attribution target.
    pub sponsor_id: Uuid,
    /// Key to use (NOT the key value).
    pub key_id: Uuid,
    /// Model identifier (e.g., "claude-sonnet-4-20250514").
    pub model: String,
    /// Conversation messages (system/user/assistant).
    pub messages: Vec<LLMMessage>,
    /// Maximum tokens in the response.
    pub max_tokens: u32,
    /// Temperature for sampling.
    pub temperature: f32,
    /// Whether to request structured (JSON) output.
    pub structured: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMMessage {
    pub role: LLMRole,
    pub content: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum LLMRole {
    System,
    User,
    Assistant,
}

/// LLM call response returned to Kingdom.
/// API key is NEVER included.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMResponse {
    /// Matching request identifier.
    pub request_id: Uuid,
    /// Model response content.
    pub content: String,
    /// Token usage.
    pub usage: TokenUsage,
    /// Computed cost in USD.
    pub cost_usd: f64,
    /// Round-trip latency in milliseconds.
    pub latency_ms: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
}
```

### 3.5 Audit Log

```rust
/// A single audit log entry.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuditEntry {
    pub timestamp: DateTime<Utc>,
    pub event: AuditEvent,
    pub sponsor_id: Option<Uuid>,
    /// First 16 hex chars only. Key value is NEVER recorded.
    pub key_fingerprint: Option<String>,
    /// Additional context. NEVER contains: API key plaintext/ciphertext,
    /// master key, or prompt content.
    pub details: std::collections::HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AuditEvent {
    KeyRegistered,
    KeyUsed,
    KeyRevoked,
    KeyProbe,
    KeyDecryption,
    SponsorCreated,
    SponsorFunded,
    BudgetWarning,
    BudgetExhausted,
    AuthFailure,
    RateLimitHit,
    AnomalyDetected,
}
```

### 3.6 Anomaly Detection

```rust
/// Anomaly detection rule.
#[derive(Debug, Clone)]
pub struct AnomalyRule {
    pub condition: AnomalyCondition,
    pub action: AnomalyAction,
}

#[derive(Debug, Clone)]
pub enum AnomalyCondition {
    /// More than N requests per minute for a single key.
    HighVolumeRequests { threshold_per_minute: u32 },
    /// Cost in current minute exceeds N times average.
    CostSpike { multiplier: f64 },
    /// Reference to a non-existent key_id.
    InvalidKeyReference,
    /// Request from a sponsor whose budget is exhausted.
    PostExhaustionRequest,
}

#[derive(Debug, Clone, Copy)]
pub enum AnomalyAction {
    Alert,
    Throttle,
    PauseKey,
    Reject,
    Log,
}
```

### 3.7 Model and Rate Limit Info (Exposed to Kingdom)

```rust
/// Model information discovered during probing.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ModelInfo {
    pub model_id: String,
    pub max_context: u32,
    pub input_cost_per_million: f64,
    pub output_cost_per_million: f64,
    pub capability_tier: CapabilityTier,
    pub supports_structured: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum CapabilityTier {
    Tier1,
    Tier2,
    Tier3,
}

/// Rate limit information for a provider.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimits {
    pub requests_per_minute: u32,
    pub tokens_per_minute: u32,
    pub tokens_per_day: u32,
}

/// Probe result for a provider key.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProbeResult {
    pub provider: ProviderType,
    pub available_models: Vec<ModelInfo>,
    pub rate_limits: RateLimits,
    pub status: ProbeStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ProbeStatus {
    Ok,
    RateLimited,
    InvalidKey,
    Error,
}
```

---

## 4. Public Trait

```rust
/// Keyward-specific errors.
#[derive(Debug, thiserror::Error)]
pub enum KeywardError {
    #[error("sponsor not found: {0}")]
    SponsorNotFound(Uuid),

    #[error("key not found: {0}")]
    KeyNotFound(Uuid),

    #[error("key does not belong to sponsor")]
    KeySponsorMismatch,

    #[error("budget exhausted for sponsor: {0}")]
    BudgetExhausted(Uuid),

    #[error("rate limit exceeded for key: {0}")]
    RateLimited(String),

    #[error("key is invalid or revoked: {0}")]
    KeyInvalid(String),

    #[error("LLM provider error: {0}")]
    ProviderError(String),

    #[error("anomaly detected: {0}")]
    AnomalyDetected(String),

    #[error("socket communication error: {0}")]
    SocketError(String),

    #[error("memory protection setup failed: {0}")]
    MemoryProtectionFailed(String),
}

/// The Keyward service interface (runs in the separate keyward process).
#[async_trait::async_trait]
pub trait KeywardService: Send + Sync {
    // --- Sponsor Management ---

    /// Create a new sponsor. Returns the sponsor UUID.
    async fn create_sponsor(&self) -> Result<Uuid, KeywardError>;

    /// Register an API key for a sponsor. Key is read, encrypted, and the
    /// plaintext is immediately zeroed. Returns key_id and fingerprint.
    async fn register_key(
        &self,
        sponsor_id: Uuid,
        provider: ProviderType,
        plaintext_key: zeroize::Zeroizing<String>,
    ) -> Result<(Uuid, String), KeywardError>;

    /// Add budget to a sponsor.
    async fn fund_sponsor(
        &self,
        sponsor_id: Uuid,
        amount_usd: f64,
    ) -> Result<(), KeywardError>;

    /// Get sponsor info (no key material).
    async fn get_sponsor(&self, sponsor_id: Uuid) -> Result<Sponsor, KeywardError>;

    /// List all sponsors (no key material).
    async fn list_sponsors(&self) -> Result<Vec<Sponsor>, KeywardError>;

    // --- LLM Proxy ---

    /// Proxy an LLM request. The key is decrypted within this function scope,
    /// used for the HTTP call, and immediately zeroed.
    async fn call_llm(&self, request: LLMRequest) -> Result<LLMResponse, KeywardError>;

    // --- Key Management ---

    /// Probe a key's validity (lightweight API call, e.g., models.list).
    async fn probe_key(&self, key_id: Uuid) -> Result<ProbeResult, KeywardError>;

    /// Revoke a key.
    async fn revoke_key(
        &self,
        sponsor_id: Uuid,
        key_id: Uuid,
    ) -> Result<(), KeywardError>;

    /// Get available models across all valid keys.
    async fn available_models(&self) -> Result<Vec<ModelInfo>, KeywardError>;

    /// Get rate limits for a specific key.
    async fn rate_limits(&self, key_id: Uuid) -> Result<RateLimits, KeywardError>;

    // --- Health ---

    /// Run health checks on all keys (called every 10 cycles by Kingdom).
    async fn health_check(&self) -> Result<Vec<(Uuid, KeyStatus)>, KeywardError>;

    /// Get audit log entries (filtered).
    async fn audit_log(
        &self,
        since: Option<DateTime<Utc>>,
        event_filter: Option<Vec<AuditEvent>>,
        limit: u32,
    ) -> Result<Vec<AuditEntry>, KeywardError>;
}
```

---

## 5. Inbound Messages

Keyward communicates via Unix domain socket at `/tmp/kingdom-keyward.sock` using encrypted MessagePack frames.

| Source | Message | Payload | Description |
|--------|---------|---------|-------------|
| Summoner CLI | `SponsorCreate` | `{}` | Create a new sponsor. |
| Summoner CLI | `KeyRegister` | `{ sponsor_id, provider, key }` | Register an API key (key read from stdin). |
| Summoner CLI | `SponsorFund` | `{ sponsor_id, amount_usd }` | Add budget to a sponsor. |
| Nexus | `LLMRequest` | `LLMRequest` (see 3.4) | Proxy an LLM call. |
| Nexus | `ProbeKey` | `{ key_id }` | Probe key validity. |
| Nexus | `HealthCheck` | `{}` | Trigger health checks on all keys. |
| Nexus | `ListModels` | `{}` | List available models. |
| Nexus | `GetRateLimits` | `{ key_id }` | Get rate limits for a key. |

### 5.1 Socket Frame Format

```
[u32 length (big-endian)][MessagePack payload][u32 CRC32]
```

---

## 6. Outbound Messages / Events

| Target | Message | Payload | Description |
|--------|---------|---------|-------------|
| Summoner CLI | `SponsorCreated` | `{ sponsor_id }` | Sponsor UUID returned. |
| Summoner CLI | `KeyRegistered` | `{ key_id, fingerprint }` | Key registered confirmation. |
| Summoner CLI | `SponsorFunded` | `{ sponsor_id, budget_remaining }` | Budget updated. |
| Nexus | `LLMResponse` | `LLMResponse` (see 3.4) | LLM call result. |
| Nexus | `ProbeResult` | `ProbeResult` (see 3.7) | Key probe result. |
| Nexus | `HealthResult` | `Vec<(Uuid, KeyStatus)>` | Health check results. |
| Nexus | `ModelList` | `Vec<ModelInfo>` | Available models. |
| Nexus | `RateLimitInfo` | `RateLimits` | Rate limits for a key. |

### 6.1 What Keyward Exposes to the Kingdom

| Information | Format | Purpose |
|-------------|--------|---------|
| Available model list | `Vec<ModelInfo>` | Nexus resource planning |
| Key IDs | `Vec<Uuid>` | Nexus specifies in LLM requests |
| Rate limit info | `RateLimits` | Nexus scheduling |
| Sponsor UUID | `Uuid` | Observer's Sponsor View |
| Key fingerprint | `String` (16 hex) | Observer display |
| Cost information | `CostTracker` | Nexus budget management |
| LLM response | `LLMResponse` | Agent thought results |

**Never exposed**: API key plaintext, API key ciphertext, master key, provider auth headers.

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Key encryption (register) | < 1 ms | AES-256-GCM encrypt |
| Key decryption (per call) | < 0.5 ms | AES-256-GCM decrypt |
| LLM proxy overhead (excl. provider latency) | < 5 ms | Decrypt + HTTP build + sanitize |
| Response sanitization | < 2 ms | Regex pattern matching |
| Health check (per key) | < 5 s | Lightweight API probe |
| Anomaly detection evaluation | < 0.1 ms | In-memory rule check |
| Socket message round-trip (excl. provider) | < 10 ms | Encode + transmit + decode |
| Memory zeroing on shutdown | < 1 ms | explicit_bzero all key regions |
| Audit log append | < 0.1 ms | In-memory append |
| Startup (memory protection setup) | < 100 ms | mlock, mlockall, RLIMIT_CORE |

---

## 8. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| None (standalone) | - | Keyward has NO Kingdom crate dependencies |
| LLM Providers | External | HTTP(S) API calls via reqwest + rustls |
| Unix domain socket | IPC | Communication with Kingdom process |
| OS kernel | System | mlock, mlockall, RLIMIT_CORE, getrandom |

---

## 9. Key Algorithms

### 9.1 Memory Protection Initialization

```
fn init_key_vault():
    1. setrlimit(RLIMIT_CORE, 0)           // Disable core dumps
    2. prctl(PR_SET_DUMPABLE, 0)            // Prevent ptrace attach
    3. mlockall(MCL_CURRENT | MCL_FUTURE)   // Lock all memory pages
    4. master_key = getrandom(32)           // OS CSPRNG
    5. Verify all protections are active
```

### 9.2 Key Registration

```
fn register_key(sponsor_id, provider, plaintext_key):
    1. Validate sponsor_id exists and is ACTIVE.
    2. Compute fingerprint = hex(sha256(plaintext_key))[0..16].
    3. Generate key_id = uuid_v7().
    4. Generate nonce = getrandom(12).
    5. Encrypt: (ciphertext, tag) = aes_256_gcm_encrypt(master_key, nonce, plaintext_key).
    6. explicit_bzero(plaintext_key).
    7. Store EncryptedKey { key_id, sponsor_id, provider, ciphertext, nonce, tag, fingerprint }.
    8. Initiate async probe (transition to PROBING -> VALID or INVALID).
    9. Audit log: KEY_REGISTERED.
   10. Return (key_id, fingerprint).
```

### 9.3 LLM Proxy Call

```
fn call_llm(request: LLMRequest) -> LLMResponse:
    1. Validate: key_id belongs to request.sponsor_id.
    2. Validate: sponsor budget_remaining > 0.
    3. Validate: key status is VALID.
    4. Check anomaly rules (volume, cost spike).

    5. let plaintext_key = decrypt(vault.get(request.key_id))  // only here
    6. let start = Instant::now()
    7. let provider_response = http_post(
           provider_url(request.model),
           headers: { "Authorization": format!("Bearer {}", plaintext_key) },
           body: build_provider_body(request)
       )
    8. explicit_bzero(&plaintext_key)                           // immediately zeroed

    9. let sanitized = sanitize(provider_response)
   10. let cost = compute_cost(request.model, usage)
   11. Deduct cost from sponsor budget.
   12. Audit log: KEY_USED.
   13. Return LLMResponse { request_id, content, usage, cost_usd, latency_ms }
```

### 9.4 Response Sanitization

```
fn sanitize(response_body: String, known_keys: &[EncryptedKey]) -> String:
    1. For each known key:
       a. Decrypt to plaintext.
       b. If response_body.contains(plaintext):
          - Audit log: ANOMALY_DETECTED("key_in_response").
          - Replace plaintext with "[REDACTED]".
       c. explicit_bzero(plaintext).

    2. Pattern match for known API key formats:
       - r"sk-ant-api\d{2}-[A-Za-z0-9_-]{80,}"    (Anthropic)
       - r"sk-[A-Za-z0-9]{48,}"                     (OpenAI)
       - r"AIzaSy[A-Za-z0-9_-]{33}"                 (Google)
       Replace matches with "[REDACTED:API_KEY_PATTERN]".

    3. Return sanitized body.
```

### 9.5 Key Lifecycle State Machine

```
REGISTERED -> probe() -> PROBING
    PROBING -> success  -> VALID
    PROBING -> fail     -> INVALID

VALID -> rate_limit hit -> RATE_LIMITED
VALID -> revoke/expire  -> REVOKED

RATE_LIMITED -> cooldown -> VALID

INVALID: terminal (sponsor notified, key not usable)
REVOKED: terminal (key destroyed from vault)
```

### 9.6 Health Check (Every 10 Cycles)

```
fn health_check(key: &EncryptedKey) -> KeyStatus:
    1. Decrypt key.
    2. Make lightweight probe call (e.g., GET /v1/models).
    3. explicit_bzero(plaintext).
    4. Match result:
       OK           -> update_status(VALID)
       UNAUTHORIZED -> update_status(INVALID) + notify_sponsor()
       RATE_LIMITED -> update_status(RATE_LIMITED) + backoff()
       TIMEOUT      -> retry(3x) + if_still_fail(ALERT)
```

### 9.7 Anomaly Detection Rules

```
Default rules:
    1. > 100 requests/min per key       -> ALERT + THROTTLE
    2. cost_this_minute > 10x avg       -> ALERT + PAUSE_KEY
    3. invalid key_id reference          -> ALERT + LOG
    4. request after budget exhausted    -> REJECT + LOG
```

### 9.8 Shutdown Procedure

```
fn destroy_key_vault():
    1. explicit_bzero(all EncryptedKey ciphertext regions)
    2. explicit_bzero(master_key)
    3. munlockall()
    4. Audit log: final entry (shutdown)
```

---

## 10. Chaos Test Categories

| Category | ID Range | Focus |
|----------|----------|-------|
| A: Process Failures | A1-A5 | SIGKILL, SIGSEGV, OOM, power loss, duplicate startup |
| B: Memory Attacks | B1-B4 | /proc/mem read, memory scan, swap scan, GDB attach |
| C: Kingdom Boundary | C1-C5 | Prompt injection, Forge escape, event bus leak, Observer leak, fake key_id |
| D: Communication Path | D1-D4 | Socket intercept, TLS downgrade, key in response, DNS rebinding |
| E: Log/Output | E1-E5 | All log levels, panic traces, HTTP errors, audit entries, Observer API |
| F: Operational | F1-F5 | Invalid key, cross-sponsor, zero budget, post-revocation, concurrent sponsors |

Test execution policy: after each test, full scan of memory, disk, logs, and network capture for any test key patterns.

# 07 — PORTAL: Web Access Gateway

## 1. Purpose

Portal is the Kingdom's **controlled window to the outside world**. It allows agents to access the real-world internet for:

- Fetching documentation and reference material
- Accessing public APIs and data sources
- Observing real-world software ecosystems for inspiration
- Importing ideas (but NOT code — agents must write everything themselves)

Portal is **heavily restricted** to prevent the experiment from being contaminated by directly copying external code.

---

## 2. Access Control

### 2.1 Epoch Gating

| Epoch | Access Level |
|-------|-------------|
| 0-2 | **CLOSED** — No web access at all |
| 3 | **READ_ONLY** — Can fetch pages, cannot interact |
| 4+ | **READ_WRITE** — Can use APIs, submit queries |

### 2.2 Request Quota

```
requests_per_cycle(epoch) =
  epoch < 3:  0
  epoch == 3: 5
  epoch >= 4: 5 + (epoch - 3) * 2
```

Requests cost ⚡ (see Mint).

### 2.3 Domain Restrictions

Portal maintains allowlists and blocklists:

**Allowed categories:**
- Technical documentation sites
- Public API endpoints
- Research paper repositories
- Standards bodies (RFCs, W3C, etc.)

**Blocked categories:**
- Code repositories (GitHub, GitLab, etc.) — to prevent direct code copying
- Package managers (npm, PyPI, crates.io) — agents must build their own
- AI model APIs — agents cannot call external AI
- Social media — irrelevant to the experiment
- Any site requiring authentication

### 2.4 Content Filtering

All fetched content passes through a **code filter**:

```
ContentFilter {
  // Strips anything that looks like source code
  strip_code_blocks:  true
  strip_inline_code:  true      // configurable per request

  // Converts technical content to structured descriptions
  transform_apis:     bool      // convert API docs to abstract interface descriptions
  transform_examples: bool      // convert code examples to pseudocode descriptions

  // Metadata
  max_size:           u64       // max response size in bytes (default: 64 KB)
  format:             enum(RAW | STRUCTURED | SUMMARY)
}
```

The key rule: **Agents can learn CONCEPTS from the web, but cannot import CODE.** They must implement everything from scratch in Genesis (or languages they build).

---

## 3. Request Protocol

### 3.1 Fetch Request

```
PortalRequest {
  id:          hash256
  agent:       hash256
  url:         bytes
  method:      enum(GET | POST | HEAD)
  headers:     map<bytes, bytes>   // restricted headers only
  body:        bytes | null        // for POST only
  filter:      ContentFilter
  purpose:     bytes               // structured justification for the request
  signature:   bytes
}
```

### 3.2 Response

```
PortalResponse {
  request_id:  hash256
  status:      u16                 // HTTP status code
  content:     bytes               // filtered content
  content_type: bytes
  filtered:    FilterReport        // what was removed
  cached:      bool                // from portal cache?
  cost:        u64                 // ⚡ charged
}

FilterReport {
  code_blocks_removed:  u32
  bytes_stripped:        u64
  transformations:      u32
  warnings:             [bytes]    // e.g., "high code density detected"
}
```

### 3.3 Caching

Portal caches responses to reduce costs and external load:

```
CachePolicy {
  ttl:          u64               // cache duration in cycles
  key:          hash256           // sha256(url + method + body)
  shared:       bool              // can other agents use this cached response?
}
```

Shared cache hits are free (0 ⚡, 0 ticks for the requesting agent).

---

## 4. API Access (Epoch 4+)

In Epoch 4+, agents can interact with public APIs:

### 4.1 Supported Interaction Patterns

```
APIInteraction {
  kind:    enum(
    REST_GET,        // retrieve data
    REST_POST,       // submit queries (e.g., search APIs)
    GRAPHQL,         // structured queries
    WEBSOCKET_SNAP,  // one-shot websocket connection snapshot
  )
  // No streaming, no persistent connections, no authentication
}
```

### 4.2 Rate Limiting

- Max 1 request per tick
- Max quota per cycle (see above)
- Automatic backoff on 429/503 responses
- Failed requests still cost 1 ⚡ (but only 1 tick)

---

## 5. Knowledge Import Protocol

When an agent fetches useful information from the web, the recommended workflow is:

```
1. Agent submits PortalRequest with PURPOSE
2. Portal returns filtered content
3. Agent synthesizes the knowledge in their own understanding
4. Agent publishes an OracleEntry documenting what they learned
5. Oracle entry is tagged with portal_source: request_id
6. Other agents can learn from the Oracle entry without using Portal quota
```

This creates a **knowledge funnel**: web knowledge enters through Portal, is processed by one agent, and becomes available to all through Oracle.

---

## 6. Abuse Prevention

| Risk | Mitigation |
|------|-----------|
| Direct code copying | Content filter strips code; PORTAL_0 audits suspicious patterns |
| Circumventing domain blocks | URL validation before request; redirect following is limited to 2 hops |
| Excessive spending | Per-agent per-cycle caps; governance can reduce quotas |
| Fetching harmful content | All content is logged and auditable |
| Using web to bypass Kingdom constraints | PURPOSE field is reviewed; repeated abuse = temporary ban |

### 6.1 Audit Trail

Every Portal request and response is logged on the event bus. PORTAL_0 publishes a **usage report** each cycle:

```
PortalReport {
  cycle:             u64
  total_requests:    u32
  cache_hit_rate:    f32
  domains_accessed:  [bytes]
  top_users:         [(hash256, u32)]
  code_filter_hits:  u32
  blocked_requests:  u32
  total_cost:        u64
}
```

---

## 7. Costs

| Action | Tick Cost | Mint Cost |
|--------|----------|-----------|
| GET request | 3 | 2 ⚡ |
| POST request | 3 | 3 ⚡ |
| HEAD request | 2 | 1 ⚡ |
| Cache hit (shared) | 1 | 0 ⚡ |
| Cache hit (own) | 1 | 0 ⚡ |
| Failed request | 1 | 1 ⚡ |

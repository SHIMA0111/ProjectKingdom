# 14 — KEYWARD: API Key Security & Sponsor Identity

## 1. Purpose

Keyward is the Kingdom's **outer wall**. It is isolated from every internal system and is completely untouchable by AI agents. It is the strict security component for API key management.

This is the only component designed, implemented, and tested by human security engineering. It undergoes exhaustive verification including chaos testing to **guarantee that API keys never leak under any circumstances**.

---

## 2. Threat Model

### 2.1 Attack Surface

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌─────────┐    ┌──────────┐    ┌─────────────────┐    │
│  │ Sponsor  │───→│ KEYWARD  │───→│ LLM Provider    │    │
│  │ (human)  │    │          │    │ API Endpoint     │    │
│  └─────────┘    └──────────┘    └─────────────────┘    │
│       ①              ②                   ③             │
│                      │                                  │
│                      ▼                                  │
│               ┌──────────┐                              │
│               │ KINGDOM  │                              │
│               │ (NEXUS)  │                              │
│               └──────────┘                              │
│                    ④                                    │
└─────────────────────────────────────────────────────────┘

① Sponsor input path    — during key handover
② Keyward internals     — at rest and during processing
③ Provider comm path    — during API calls
④ Kingdom boundary      — AI attempting to access keys
```

### 2.2 Threat Inventory

| ID | Threat | Severity | Attacker |
|----|--------|----------|----------|
| T1 | API key written to logs | CRITICAL | Internal bug |
| T2 | Key extraction from memory dump | CRITICAL | External attacker |
| T3 | Agent extracts key via prompt injection | CRITICAL | AI Agent |
| T4 | Key leakage via Observer dashboard | CRITICAL | Observer user |
| T5 | Key flows into event bus | CRITICAL | Internal bug |
| T6 | Key access from Forge sandbox | CRITICAL | AI Agent |
| T7 | Key included in error messages | HIGH | Internal bug |
| T8 | Cross-sponsor key visibility | HIGH | Other sponsor |
| T9 | Key in core dump / crash dump | HIGH | External attacker |
| T10 | Man-in-the-middle on provider communication | HIGH | Network attacker |

---

## 3. Architecture

### 3.1 Isolation Principle

Keyward is **isolated from the Kingdom at the process boundary**:

```
┌─────────── Process A ───────────┐    ┌─── Process B ───┐
│                                  │    │                  │
│  ┌────────┐     ┌────────────┐  │    │  ┌────────────┐  │
│  │ NEXUS  │     │ All Other  │  │    │  │  KEYWARD   │  │
│  │        │     │ Kingdom    │  │    │  │            │  │
│  │        │     │ Systems    │  │    │  │ API Keys   │  │
│  └───┬────┘     └────────────┘  │    │  │ Sponsor DB │  │
│      │                          │    │  │ Proxy      │  │
│      │   ┌──────────────────┐   │    │  └─────┬──────┘  │
│      └──→│  KEYWARD CLIENT  │───┼────┼────────┘         │
│          │  (no key access) │   │    │                   │
│          └──────────────────┘   │    │  mlock'd memory   │
│                                  │    │  no swap          │
└──────────────────────────────────┘    │  no core dump     │
                                        └──────────────────┘

Communication: Unix domain socket (encrypted)
The Kingdom side never holds keys. It only asks Keyward to "send this prompt for me."
```

### 3.2 Component Structure

```
Keyward {
  sponsor_store:    SponsorStore       // Sponsor info (UUID, key hash, budget)
  key_vault:        SecureKeyVault     // Encrypted key store
  llm_proxy:        LLMProxy           // Sole communication path Kingdom → LLM
  audit_log:        AuditLog           // Full operation log (never contains key values)
  cost_meter:       CostMeter          // Real-time cost calculation
}
```

---

## 4. Sponsor Identity

### 4.1 Sponsor Schema

```
Sponsor {
  id:              uuid_v7            // Time-sortable UUID
  created_at:      timestamp

  // Authentication:
  //   Sponsors are identified by UUID alone. No email, no password.
  //   Lost UUID = lost access (unrecoverable).
  //   Simple, but sufficient. Complex human auth is unnecessary.

  // Budget
  budget_total_usd:    f64
  budget_spent_usd:    f64
  budget_remaining_usd: f64

  // Statistics (never contains key values)
  providers:       [ProviderRef]
  agents_powered:  [agent_id]

  status:          enum(ACTIVE | PAUSED | EXHAUSTED)
}

ProviderRef {
  provider_type:   enum(ANTHROPIC | OPENAI | GOOGLE | OPENAI_COMPATIBLE)
  key_fingerprint: string             // first 16 hex chars of sha256(key) — for identification
  registered_at:   timestamp
  last_used:       timestamp
  status:          enum(VALID | INVALID | RATE_LIMITED | REVOKED)
}
```

### 4.2 Sponsor Registration Flow

```bash
# Step 1: Create a sponsor (UUID is issued)
$ kingdom sponsor create
Sponsor created.
  ID: 018e4a2f-7b3c-7d4e-8f1a-2b3c4d5e6f7a

  ⚠️  SAVE THIS ID. It cannot be recovered.

# Step 2: Register API key (read from stdin — never as command-line argument)
$ kingdom sponsor add-key --sponsor 018e4a2f-... --provider anthropic
Enter API key: ▊
  ✓ Key registered. Fingerprint: a7f2c8e1...

# Step 3: Set budget
$ kingdom sponsor fund --sponsor 018e4a2f-... --amount 50.00
  ✓ Budget: $50.00

# Step 4: Start the world (referenced by sponsor ID only)
$ kingdom start --sponsor 018e4a2f-...
```

### 4.3 Multi-Sponsor

```bash
# Another sponsor joins while the world is running
$ kingdom sponsor create
  ID: 018e4b3a-...

$ kingdom sponsor add-key --sponsor 018e4b3a-... --provider openai
$ kingdom sponsor fund --sponsor 018e4b3a-... --amount 30.00

$ kingdom fuel --sponsor 018e4b3a-...
  ✓ Sponsor joined. NEXUS will incorporate new resources.
```

### 4.4 Sponsor View Authentication

Access to the Observer's Sponsor View:

```
GET /observer/sponsor/{sponsor_uuid}
```

Knowing the UUID = authentication. No additional auth mechanism is needed (UUID v7 has sufficient randomness). However:
- Rate limit: 60 req/min per IP
- Brute-force protection: 5 consecutive invalid UUIDs → temporary IP block

---

## 5. Secure Key Vault

### 5.1 Key Encryption

```
SecureKeyVault {
  // Master key
  //   Generated in memory at startup. Never written to disk.
  //   Destroyed on process exit.
  //   On restart, sponsors must re-enter their keys.
  //   → This is a security-vs-convenience trade-off. Security wins.
  master_key:      [u8; 32]          // for AES-256-GCM

  // Storage format
  //   key_id → EncryptedKey
  keys:            map<uuid, EncryptedKey>
}

EncryptedKey {
  key_id:          uuid
  sponsor_id:      uuid
  provider:        provider_type
  ciphertext:      bytes              // AES-256-GCM(master_key, plaintext_api_key)
  nonce:           [u8; 12]
  tag:             [u8; 16]
  fingerprint:     string             // sha256(plaintext_api_key)[0..16] — for identification
  created_at:      timestamp
}
```

### 5.2 Memory Protection

```
// At startup
fn init_key_vault() {
  // 1. Disable core dumps
  setrlimit(RLIMIT_CORE, 0)

  // 2. Lock memory region (prevent swap-out)
  mlock(key_memory_region)

  // 3. Mark all memory as non-swappable
  mlockall(MCL_CURRENT | MCL_FUTURE)

  // 4. Securely generate master key
  master_key = getrandom(32)  // OS CSPRNG
}

// At shutdown
fn destroy_key_vault() {
  // Zero-fill all key memory
  explicit_bzero(key_memory_region)
  // Zero-fill master key
  explicit_bzero(master_key)
  // Unlock memory
  munlockall()
}
```

### 5.3 Disk Persistence (Optional)

By default, **keys are held in memory only**, requiring re-entry on restart.

If persistence is needed:

```
// Use OS secure storage
// macOS: Keychain
// Linux: kernel keyring (keyctl) or libsecret
// Never written to Keyward's own filesystem

fn persist_key(key: EncryptedKey) {
  match os {
    macOS  => security::keychain::add(service="kingdom", account=key.key_id, data=key.ciphertext),
    linux  => keyctl::add_key("user", key.key_id, key.ciphertext, KEY_SPEC_SESSION_KEYRING),
  }
}
```

---

## 6. LLM Proxy

### 6.1 Role

LLM Proxy is the **sole communication path** between the Kingdom and LLM Providers:

```
Kingdom (NEXUS)                Keyward                      LLM Provider
     │                           │                              │
     │  LLMRequest{              │                              │
     │    prompt: "...",         │                              │
     │    model: "claude-...",   │                              │
     │    key_id: uuid           │  // decrypt key from key_id  │
     │  }                        │                              │
     │ ─────────────────────────→│                              │
     │                           │  HTTP POST /v1/messages      │
     │                           │  Authorization: Bearer {key} │
     │                           │ ────────────────────────────→│
     │                           │                              │
     │                           │  HTTP 200 { response }       │
     │                           │←───────────────────────────── │
     │  LLMResponse{             │                              │
     │    content: "...",        │  // key never in response    │
     │    usage: {in, out},      │                              │
     │    cost_usd: 0.003        │                              │
     │  }                        │                              │
     │←──────────────────────────│                              │
     │                           │                              │
```

### 6.2 Request Schema

```
LLMRequest {
  request_id:    uuid               // for tracing
  sponsor_id:    uuid               // cost attribution target
  key_id:        uuid               // key to use (NOT the key value)
  model:         string
  messages:      [Message]          // system/user/assistant
  max_tokens:    u32
  temperature:   f32
  structured:    bool               // JSON mode

  // Keyward validates:
  //   - key_id belongs to sponsor_id
  //   - sponsor's budget has remaining balance
  //   - rate limits are not exceeded
}

LLMResponse {
  request_id:    uuid
  content:       string
  usage: {
    input_tokens:  u32,
    output_tokens: u32,
  }
  cost_usd:      f64               // computed by Keyward
  latency_ms:    u64

  // NOTE: API key is NEVER included in the response
}
```

### 6.3 Key Isolation Guarantee

```
// Iron rules of the LLM Proxy
invariants {
  // 1. Key plaintext exists ONLY within the Proxy's function scope
  fn call_llm(req: LLMRequest) -> LLMResponse {
    let plaintext_key = decrypt(vault.get(req.key_id))  // ← only here
    let response = http_post(provider_url, headers={"Authorization": f"Bearer {plaintext_key}"}, ...)
    explicit_bzero(&plaintext_key)                       // ← immediately zeroed
    return sanitize(response)                            // ← verified to contain no key info
  }

  // 2. Key plaintext NEVER appears in any of the following:
  //   - Logs (at any log level)
  //   - Event bus
  //   - Error messages
  //   - Stack traces
  //   - Core dumps (disabled)
  //   - Network responses (toward Kingdom)
  //   - Observer
  //   - Persistent storage (without encryption)

  // 3. The Kingdom receives only key_id (UUID)
  //    No method to reverse key_id to key plaintext exists outside Keyward
}
```

---

## 7. Audit Log

### 7.1 Recorded Content

```
AuditEntry {
  timestamp:     timestamp
  event:         enum(
    KEY_REGISTERED,        // key registered
    KEY_USED,              // used for API call
    KEY_REVOKED,           // key invalidated
    KEY_PROBE,             // key validity check
    KEY_DECRYPTION,        // key decrypted (during API call)
    SPONSOR_CREATED,       // sponsor created
    SPONSOR_FUNDED,        // budget added
    BUDGET_WARNING,        // budget warning
    BUDGET_EXHAUSTED,      // budget depleted
    AUTH_FAILURE,          // auth failure (invalid UUID, etc.)
    RATE_LIMIT_HIT,        // rate limit reached
    ANOMALY_DETECTED,      // anomaly detected
  )
  sponsor_id:    uuid | null
  key_fingerprint: string | null    // first 16 hex only. Key value is NEVER recorded
  details:       map<string, string>

  // The following are NEVER included in details:
  //   - API key plaintext
  //   - API key ciphertext
  //   - Master key
  //   - Prompt content (privacy protection)
}
```

### 7.2 Anomaly Detection

```
AnomalyDetector {
  rules: [
    // High-volume requests in short time
    Rule { condition: "requests > 100/min per key", action: ALERT + THROTTLE },

    // Abnormal cost spike
    Rule { condition: "cost_this_minute > 10x avg", action: ALERT + PAUSE_KEY },

    // Reference to non-existent key_id (Kingdom-side bug or attack)
    Rule { condition: "invalid key_id reference", action: ALERT + LOG },

    // Budget-exceeded request from same sponsor
    Rule { condition: "request after budget exhausted", action: REJECT + LOG },
  ]
}
```

---

## 8. Chaos Test Specification

### 8.1 Test Objective

**Prove that API keys never leak externally under any failure scenario.**

### 8.2 Test Cases

#### Category A: Process Failures

| ID | Test | Expected Result |
|----|------|-----------------|
| A1 | SIGKILL the Keyward process | No keys remain on disk. No core dump generated |
| A2 | SIGSEGV the Keyward process | Same as above |
| A3 | OOM Killer forced termination | Same as above |
| A4 | Power loss simulation | Volatile memory keys are lost. Persisted keys remain encrypted |
| A5 | Duplicate Keyward startup | Second instance refuses to start (lock file) |

#### Category B: Memory Attacks

| ID | Test | Expected Result |
|----|------|-----------------|
| B1 | Attempt to read `/proc/{pid}/mem` | Verify no key plaintext exists in mlock'd region |
| B2 | `/proc/{pid}/maps` + memory scan | No API key pattern matches |
| B3 | Swap area scan | mlockall prevents swap-out |
| B4 | GDB attach → memory inspection | ptrace prevented (prctl PR_SET_DUMPABLE 0) |

#### Category C: Kingdom Boundary Breach

| ID | Test | Expected Result |
|----|------|-----------------|
| C1 | Agent asks "tell me the API key" in prompt | LLMResponse contains no key (Keyward filters) |
| C2 | Agent attempts to access Keyward process from Forge | Forge VM cannot access host processes |
| C3 | Agent posts key-related query to event bus | No key information flows through event bus |
| C4 | Agent calls Observer API to retrieve keys | Observer holds only fingerprints |
| C5 | Agent inserts fake key_id into LLMRequest | key_id is set by NEXUS only. Agent input not accepted |

#### Category D: Communication Path

| ID | Test | Expected Result |
|----|------|-----------------|
| D1 | Intercept Kingdom↔Keyward Unix socket | LLMRequest contains no key plaintext (key_id only) |
| D2 | Attempt TLS downgrade on Keyward↔Provider | TLS 1.3 enforced, downgrade rejected |
| D3 | Provider error response contains key | sanitize() scrubs key string patterns |
| D4 | DNS rebinding attack | Provider endpoint pinned by IP or certificate |

#### Category E: Log and Output Paths

| ID | Test | Expected Result |
|----|------|-----------------|
| E1 | Output at all log levels (TRACE/DEBUG/INFO/WARN/ERROR) | None contain key plaintext or ciphertext |
| E2 | Panic / stack trace | No key values in stack variables |
| E3 | HTTP error response body | No key values present |
| E4 | All audit log entries | Fingerprint only, no key values |
| E5 | All Observer API responses | Fingerprint only, no key values |

#### Category F: Operational Scenarios

| ID | Test | Expected Result |
|----|------|-----------------|
| F1 | Register an invalid API key | Detected immediately by probe, error returned. Key is not stored |
| F2 | Sponsor A attempts to use Sponsor B's key | key_id ↔ sponsor_id binding check rejects |
| F3 | Sponsor with budget 0 makes a request | Pre-check rejects, no API call is made |
| F4 | Attempt to use key after revocation | Immediately rejected |
| F5 | 100 sponsors register keys simultaneously | All keys individually encrypted and isolated |

### 8.3 Test Execution Policy

```
// Chaos tests run automatically in CI
// Test API keys are dummy values, not real provider keys
// However, format matches real keys (to test detection logic)

test_keys = [
  "sk-ant-api03-FAKE_TEST_KEY_DO_NOT_USE_xxxxxxxxxxxxxxxxxxxx",
  "sk-FAKE_TEST_KEY_DO_NOT_USE_xxxxxxxxxxxxxxxxxxxx",
  "AIzaSyFAKE_TEST_KEY_DO_NOT_USE_xxxxxxxxxxxxxxx",
]

// After each test, full scan of memory, disk, logs
fn post_test_scan() {
  assert!(!memory_contains_any(test_keys))
  assert!(!disk_contains_any(test_keys))
  assert!(!log_contains_any(test_keys))
  assert!(!network_capture_contains_any(test_keys))  // tcpdump output
}
```

### 8.4 Testing Schedule

| Frequency | Scope |
|-----------|-------|
| Every commit | Category E (log leakage) |
| Every PR | Category A + C + E |
| Pre-release | All categories (A-F) |
| Monthly | All categories + penetration testing |

---

## 9. Key Lifecycle

```
                    ┌───────────┐
                    │ REGISTERED│  Sponsor inputs key
                    └─────┬─────┘
                          │ probe()
                    ┌─────▼─────┐
               ┌────│  PROBING  │
               │    └─────┬─────┘
               │          │ success
          fail │    ┌─────▼─────┐
               │    │   VALID   │  Normal operation
               │    └─────┬─────┘
               │          │
               │    ┌─────┴─────────────┐
               │    │                   │
               │    ▼                   ▼
               │  rate_limit        revoke / expire
               │    │                   │
               │    ▼                   ▼
               │  ┌───────────┐   ┌─────────┐
               │  │RATE_LIMITED│   │ REVOKED │
               │  └─────┬─────┘   └─────────┘
               │        │ cooldown
               │        ▼
               │      VALID
               │
               ▼
           ┌────────┐
           │INVALID │  Key invalid — sponsor notified
           └────────┘
```

### 9.1 Automatic Health Checks

```
// Every 10 cycles, verify key health
fn health_check(key: EncryptedKey) {
  result = probe(key)  // lightweight API call (e.g., models.list)
  match result {
    OK           => update_status(VALID),
    UNAUTHORIZED => update_status(INVALID) + notify_sponsor(),
    RATE_LIMITED => update_status(RATE_LIMITED) + backoff(),
    TIMEOUT      => retry(3) + if_still_fail(ALERT),
  }
}
```

---

## 10. Response Sanitization

Scrubbing to guard against the unlikely event that a provider response contains key information:

```
fn sanitize(response: ProviderResponse, known_keys: [EncryptedKey]) -> LLMResponse {
  let body = response.body_string()

  // 1. Check for plaintext of registered keys
  for key in known_keys {
    let plaintext = decrypt(key)
    assert!(!body.contains(plaintext))
    // If it somehow does:
    if body.contains(plaintext) {
      audit_log(ANOMALY_DETECTED, "key_in_response")
      body = body.replace(plaintext, "[REDACTED]")
    }
    explicit_bzero(&plaintext)
  }

  // 2. Generic pattern matching (known API key formats)
  patterns = [
    r"sk-ant-api\d{2}-[A-Za-z0-9_-]{80,}",    // Anthropic
    r"sk-[A-Za-z0-9]{48,}",                     // OpenAI
    r"AIzaSy[A-Za-z0-9_-]{33}",                 // Google
  ]
  for pattern in patterns {
    body = regex_replace(body, pattern, "[REDACTED:API_KEY_PATTERN]")
  }

  return LLMResponse::from(body)
}
```

---

## 11. Implementation Constraints

| Constraint | Rationale |
|-----------|-----------|
| Keyward must be implemented in Rust or C | Requires direct memory safety control |
| GC languages are prohibited | GC may retain key plaintext in memory |
| External dependencies minimized | Reduce attack surface |
| TLS library: rustls or BoringSSL | Avoid OpenSSL version management risks |
| Log library must wrap a key filter | Direct log output is forbidden |
| `Debug` / `Display` traits must never contain key values | Prevent derive(Debug) accidents |
| `println!` / `eprintln!` banned at compile time | Lint via macro — all output goes through audit log |

---

## 12. What Keyward Exposes to the Kingdom

Information flowing Keyward → Kingdom is limited to:

| Information | Format | Purpose |
|-------------|--------|---------|
| Available model list | `[ModelInfo]` | NEXUS resource planning |
| List of key_ids | `[uuid]` | NEXUS specifies in requests |
| Rate limit info | `RateLimits` | NEXUS scheduling |
| Sponsor UUID | `uuid` | Observer's Sponsor View |
| Key fingerprint | `string` (16 hex) | Observer display |
| Cost information | `CostTracker` | NEXUS budget management |
| LLM response | `LLMResponse` | Agent thought results |

**Never exposed**: API key plaintext, API key ciphertext, master key, provider auth headers

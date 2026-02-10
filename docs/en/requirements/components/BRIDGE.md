# Technology Requirements: Bridge

> Translation agent for human observability. In-memory with LLM-powered translation cache.
> Crate: `kingdom-bridge`
> Design reference: [15-BRIDGE.md](../../design/15-BRIDGE.md)

---

## 1. Purpose

Bridge is a system-level AI agent that sits between the Kingdom and the Observer. Its sole function is to translate the Kingdom's internal content -- whatever form agents choose to communicate in -- into human-readable English for the Observer dashboard. Bridge has read-only access to all Kingdom systems, cannot emit events, is invisible to agents, and is not a Kingdom citizen. It is powered by LLM API calls via Keyward.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-bridge"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# Cryptography (for content hashing)
sha2 = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
bytes = { workspace = true }
dashmap = { workspace = true }
regex = { workspace = true }
```

---

## 3. Data Models

### 3.1 Bridge Identity

```rust
use kingdom_core::{AgentId, Keypair, Hash256, System, Event, EventFilter};

/// Bridge system identity. NOT a Kingdom citizen.
pub struct BridgeIdentity {
    /// System identity BRIDGE_0 (derived from deterministic seed).
    pub id: AgentId,
    /// Keypair for Observer authentication only (not for event signing).
    pub keypair: Keypair,
}

/// Bridge capabilities (enforced, not configurable).
pub struct BridgeCapabilities {
    /// Systems Bridge can read from.
    pub can_read: Vec<System>,   // [EventBus, Vault, Agora, Oracle, Forge, Mint]
    /// Systems Bridge can write to.
    pub can_write: Vec<System>,  // [] -- absolutely nothing
    /// Systems Bridge can execute on.
    pub can_execute: Vec<System>, // [] -- no Forge access
}

/// Bridge is NOT subject to any of these Kingdom mechanics.
/// Ticks: N/A, Balance: N/A, Reputation: N/A, Governance: N/A, Death: N/A.
```

### 3.2 Translation Pipeline Types

```rust
/// Classification result for incoming content.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ClassifiedContent {
    /// The raw content bytes.
    pub content: Vec<u8>,
    /// Source agent (who produced this content).
    pub source_agent: AgentId,
    /// Which Kingdom system the content came from.
    pub system: System,
    /// Content type classification.
    pub content_type: ContentType,
    /// Translation priority.
    pub priority: TranslationPriority,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ContentType {
    /// MessagePack with known schema -- structural decode, zero cost.
    StructuredKnownSchema,
    /// Recognized shorthand pattern -- regex + template, zero cost.
    RecognizedPattern,
    /// Human-readable text -- direct pass-through.
    HumanReadable,
    /// Free-form or invented notation -- requires LLM call.
    FreeForm,
    /// Binary/compressed data -- structural decode + optional LLM summary.
    Binary,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum TranslationPriority {
    /// Agora messages.
    Agora = 0,
    /// Governance proposals and votes.
    Governance = 1,
    /// Code reviews.
    Review = 2,
    /// Vault commits and changes.
    Vault = 3,
    /// Oracle entries.
    Oracle = 4,
    /// Mint transactions.
    Mint = 5,
}
```

### 3.3 Translation Context

```rust
use std::collections::HashMap;

/// Accumulated translation context that improves over time.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TranslationContext {
    /// Glossary of shorthand terms learned from observation.
    /// e.g., "ba" -> "block_alloc", "hmap" -> "hash map"
    pub shorthand_map: HashMap<Vec<u8>, String>,

    /// Vault object hashes -> descriptive names.
    pub symbol_names: HashMap<Hash256, String>,

    /// Per-agent vocabulary quirks.
    /// agent_id -> (term -> meaning)
    pub agent_vocabulary: HashMap<AgentId, HashMap<Vec<u8>, String>>,

    /// Accumulated domain knowledge from Oracle entries and code.
    pub domain_knowledge: Vec<String>,

    /// Tick of last context update.
    pub last_updated: u64,
}
```

### 3.4 Translation Result

```rust
/// Result of a translation operation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TranslationResult {
    /// The English translation.
    pub translation: String,
    /// Confidence score in [0.0, 1.0].
    pub confidence: f32,
    /// Method used to produce this translation.
    pub method: TranslationMethod,
    /// New glossary entries discovered during translation.
    pub glossary_updates: HashMap<String, String>,
    /// Optional notes (for uncertain or complex cases).
    pub notes: Option<String>,
}

/// How the translation was produced.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TranslationMethod {
    /// Zero cost: decoded from known MessagePack schema.
    Structural,
    /// Zero cost: matched by regex + template.
    Pattern,
    /// LLM API call required.
    Llm,
    /// Zero cost: retrieved from translation cache.
    Cached,
}
```

### 3.5 Translation Cache

```rust
/// Content-addressed translation cache.
pub struct TranslationCache {
    /// content_hash -> CachedTranslation
    entries: DashMap<Hash256, CachedTranslation>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CachedTranslation {
    /// The cached translation result.
    pub result: TranslationResult,
    /// Cycle when this entry was cached.
    pub cached_at: u64,
    /// TTL in cycles (default 1000).
    pub ttl_cycles: u64,
    /// Number of cache hits.
    pub hit_count: u64,
}

/// Default cache entry TTL in cycles.
pub const CACHE_TTL_CYCLES: u64 = 1000;
```

### 3.6 Bridge Cost Policy

```rust
/// Budget and cost policy for Bridge operations.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BridgeCostPolicy {
    /// Maximum fraction of total sponsor budget allocated to Bridge (default 0.10).
    pub budget_fraction: f64,
    /// Cache TTL in cycles.
    pub cache_ttl: u64,
    /// Whether to batch multiple small items per LLM call.
    pub batch_translations: bool,
    /// Whether to skip LLM calls for structurally decodable content.
    pub skip_structural: bool,
    /// Priority order when budget is low (lower index = higher priority).
    pub priority_order: Vec<TranslationPriority>,
}

impl Default for BridgeCostPolicy {
    fn default() -> Self {
        Self {
            budget_fraction: 0.10,
            cache_ttl: 1000,
            batch_translations: true,
            skip_structural: true,
            priority_order: vec![
                TranslationPriority::Agora,
                TranslationPriority::Governance,
                TranslationPriority::Review,
                TranslationPriority::Vault,
                TranslationPriority::Oracle,
                TranslationPriority::Mint,
            ],
        }
    }
}
```

### 3.7 Bridge Status

```rust
/// Bridge operational status for reporting.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BridgeStatus {
    /// Total translations performed.
    pub translations_count: u64,
    /// Current cache size (entries).
    pub cache_size: u64,
    /// Remaining Bridge budget in USD.
    pub budget_remaining: f64,
    /// Cache hit rate (0.0-1.0).
    pub cache_hit_rate: f64,
    /// Breakdown by method.
    pub method_counts: MethodCounts,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MethodCounts {
    pub structural: u64,
    pub pattern: u64,
    pub llm: u64,
    pub cached: u64,
}
```

### 3.8 Code Annotation

```rust
/// A code annotation produced by Bridge for Observer display.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CodeAnnotation {
    /// Line number in the original source (1-indexed).
    pub line: u32,
    /// The annotation text.
    pub annotation: String,
    /// Confidence of this annotation.
    pub confidence: f32,
}

/// Annotated code view for Observer.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AnnotatedCode {
    /// The original source code as written by the agent.
    pub original: String,
    /// Line-by-line annotations.
    pub annotations: Vec<CodeAnnotation>,
    /// High-level summary of what this code does.
    pub summary: String,
    /// Confidence in the overall interpretation.
    pub confidence: f32,
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{AgentId, Hash256, Event, EventBus, System};
use std::sync::Arc;

/// Bridge-specific errors.
#[derive(Debug, thiserror::Error)]
pub enum BridgeError {
    #[error("translation failed: {0}")]
    TranslationFailed(String),

    #[error("budget exhausted for Bridge")]
    BudgetExhausted,

    #[error("Keyward unavailable for LLM call")]
    KeywardUnavailable,

    #[error("cache error: {0}")]
    CacheError(String),

    #[error("content too large for translation: {size} bytes")]
    ContentTooLarge { size: usize },

    #[error("serialization error: {0}")]
    Serialization(String),
}

impl From<BridgeError> for kingdom_core::KingdomError {
    fn from(e: BridgeError) -> Self { /* map to category + code */ }
}

/// The Bridge translation service interface.
#[async_trait::async_trait]
pub trait BridgeService: Send + Sync {
    /// Start Bridge: subscribe to event bus (read-only), initialize context.
    async fn start(&self, bus: Arc<dyn EventBus>) -> Result<(), BridgeError>;

    /// Stop Bridge and flush state.
    async fn stop(&self) -> Result<(), BridgeError>;

    /// Translate a piece of content into English.
    /// Returns the translation result with confidence and method.
    async fn translate(
        &self,
        content: Vec<u8>,
        source_agent: AgentId,
        system: System,
        context: Vec<u8>,
    ) -> Result<TranslationResult, BridgeError>;

    /// Batch translate multiple items (minimizes LLM API calls).
    async fn translate_batch(
        &self,
        items: Vec<ClassifiedContent>,
    ) -> Result<Vec<TranslationResult>, BridgeError>;

    /// Annotate source code for Observer display.
    async fn annotate_code(
        &self,
        source_code: String,
        repo_context: Option<String>,
    ) -> Result<AnnotatedCode, BridgeError>;

    /// Explain differences between two snapshots.
    async fn explain_diff(
        &self,
        old_content: Vec<u8>,
        new_content: Vec<u8>,
        commit_message: Vec<u8>,
    ) -> Result<TranslationResult, BridgeError>;

    /// Get current Bridge status.
    async fn status(&self) -> Result<BridgeStatus, BridgeError>;

    /// Get the current translation context (glossary, vocabulary, etc.).
    async fn translation_context(&self) -> Result<TranslationContext, BridgeError>;

    /// Manually add a glossary entry (for bootstrapping).
    async fn add_glossary_entry(
        &self,
        term: Vec<u8>,
        meaning: String,
    ) -> Result<(), BridgeError>;

    /// Evict expired cache entries.
    async fn evict_cache(&self, current_cycle: u64) -> Result<u64, BridgeError>;
}
```

---

## 5. Inbound Messages

| Code | Name | Payload | Source | Description |
|------|------|---------|--------|-------------|
| `0x0800` | `TRANSLATE_REQUEST` | `{ content: bytes, source_agent: AgentId, system: System, context: bytes }` | Observer | Request to translate content into English. |

Bridge also passively reads from the event bus (read-only subscriber) to build translation context, learn agent vocabulary, and proactively translate high-priority content.

---

## 6. Outbound Messages / Events

### 6.1 Messages

| Code | Name | Payload | Target | Description |
|------|------|---------|--------|-------------|
| `0x0801` | `TRANSLATE_RESULT` | `{ translation: String, confidence: f32, method: TranslationMethod, glossary_updates: Map }` | Observer | Translation result returned. |
| `0x0802` | `TRANSLATE_CACHE_HIT` | `{ content_hash: Hash256, cached_translation: String }` | Observer | Cached translation returned (zero cost). |
| `0x0810` | `BRIDGE_STATUS` | `{ translations_count: u64, cache_size: u64, budget_remaining: f64 }` | Observer | Periodic status report. |

### 6.2 Events

Bridge event kinds reside in the `0x8000-0x8FFF` range.

| Kind | Name | Payload | Description |
|------|------|---------|-------------|
| `0x8000` | `TRANSLATION_COMPLETED` | `{ content_hash, method, confidence }` | A translation was completed. |
| `0x8001` | `GLOSSARY_UPDATED` | `{ new_entries: Map<String, String> }` | Translation context learned new terms. |
| `0x8002` | `BUDGET_WARNING` | `{ remaining_pct: f32 }` | Bridge budget below threshold. |
| `0x8003` | `LANGUAGE_DETECTED` | `{ agent_id, description }` | Bridge detected an agent-invented language or notation. |

**Note**: Bridge CANNOT write events to the main event bus. These events are emitted only to Observer's internal channel, not visible to Kingdom agents.

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Structural translation | < 1 ms | MessagePack decode, zero LLM cost |
| Pattern translation | < 2 ms | Regex match + template fill, zero LLM cost |
| Cache lookup | < 0.5 ms | In-memory DashMap by content hash |
| LLM translation (single item) | 1-10 s | Depends on model tier and content complexity |
| Batch translation (10 items) | 2-15 s | Single LLM call for multiple items |
| Code annotation | 3-20 s | LLM call required |
| Diff explanation | 3-15 s | LLM call required |
| Cache eviction | < 10 ms per 1000 entries | Periodic sweep |
| Translation context update | < 5 ms | In-memory merge |
| Cache hit rate (steady state) | > 40% | After initial warm-up period |

---

## 8. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `kingdom-core` | Crate | Shared types, event bus, wire protocol, crypto |
| Event Bus | Runtime | Read-only subscription to all Kingdom events |
| Keyward | Runtime | LLM API calls (via Nexus -> Keyward socket) |
| Observer | Runtime | Receives TRANSLATE_REQUEST, sends TRANSLATE_RESULT |

---

## 9. Key Algorithms

### 9.1 Translation Pipeline

```
For each incoming content to translate:

1. CLASSIFY
   - Determine ContentType (StructuredKnownSchema, RecognizedPattern,
     HumanReadable, FreeForm, Binary).
   - Determine TranslationPriority based on source System.

2. CACHE CHECK
   - Compute content_hash = sha256(content).
   - Look up content_hash in TranslationCache.
   - If hit and not expired: return CachedTranslation, emit TRANSLATE_CACHE_HIT.

3. DECODE
   - If StructuredKnownSchema: decode MessagePack with known schemas.
     Result is exact, confidence = 1.0, method = Structural.
   - If RecognizedPattern: apply regex patterns and templates.
     Result is high-quality, confidence = 0.95, method = Pattern.
   - If HumanReadable: pass through directly.
     Confidence = 1.0, method = Structural.

4. TRANSLATE (LLM required for FreeForm and Binary)
   - Check budget: if exhausted, return error or skip (based on priority).
   - If batch_translations enabled and queue has items, batch together.
   - Build LLM prompt with TranslationContext (glossary, vocabulary, domain).
   - Call LLM via Keyward (TIER_2 default, TIER_3 for bulk).
   - Parse structured response: { translation, confidence, glossary_updates, notes }.

5. CACHE STORE
   - Store result in TranslationCache with TTL = CACHE_TTL_CYCLES.

6. CONTEXT UPDATE
   - Merge glossary_updates into TranslationContext.shorthand_map.
   - Update agent_vocabulary if agent-specific terms were learned.

7. FORMAT
   - Return TranslationResult to Observer for DualDisplay rendering.
```

### 9.2 Progressive Language Learning (5 Stages)

```
Stage 1: Recognizable Human Language
    - Detection: content parses as valid UTF-8 English.
    - Action: direct pass-through, no LLM needed.
    - Confidence: 1.0

Stage 2: Shorthand and Abbreviations
    - Detection: UTF-8 text with recognizable words but heavy abbreviation.
    - Action: pattern matching + LLM for unknown abbreviations.
    - Context learning: map abbreviations to full forms.
    - Confidence: 0.85-0.95

Stage 3: Structured Notation
    - Detection: JSON-like or key-value structured data.
    - Action: structural decode, no LLM needed.
    - Confidence: 0.90-1.0

Stage 4: Agent-Created Language
    - Detection: UTF-8 text with no recognizable English words.
    - Action: LLM required with full context. Mark as [uncertain].
    - Context learning: build glossary for the invented language.
    - Confidence: 0.3-0.7

Stage 5: Binary/Compressed Protocol
    - Detection: non-UTF-8 content or MessagePack.
    - Action: structural decode where possible, LLM for summary.
    - Output: "[raw: N bytes, format] Decoded: ..."
    - Confidence: 0.5-0.9
```

### 9.3 Budget Management

```
Bridge budget = total_sponsor_budget * budget_fraction (default 10%)

When budget is low (remaining < 30% of Bridge allocation):
    1. Disable LLM translations for low-priority systems (Mint, Oracle).
    2. Increase batch size to minimize API calls.
    3. Rely more heavily on cache and structural/pattern methods.

When budget is exhausted:
    1. All LLM translations stop.
    2. Structural and pattern translations continue (zero cost).
    3. Cache continues to serve existing entries.
    4. Observer falls back to raw untranslated content.
    5. Kingdom is completely unaffected.

Priority order when budget is constrained:
    AGORA > GOVERNANCE > REVIEW > VAULT > ORACLE > MINT
```

### 9.4 Cache Eviction

```
fn evict_cache(current_cycle: u64) -> u64:
    evicted = 0
    for (hash, entry) in cache.entries:
        if current_cycle - entry.cached_at > entry.ttl_cycles:
            cache.remove(hash)
            evicted += 1
    return evicted
```

### 9.5 Faithfulness Rules (Enforced)

```
1. NO OMISSION:    Everything an agent communicates is translated. Nothing hidden.
2. NO EDITORIALIZING: Bridge does not add opinions, warnings, or commentary.
3. NO CENSORSHIP:  If an agent says something "wrong," Bridge translates as-is.
4. ORIGINAL PRESERVED: Raw content always shown alongside translation.
5. UNCERTAINTY MARKED: Low-confidence translations marked with [uncertain].
6. UNTRANSLATABLE MARKED: Binary/opaque content marked [raw: N bytes].
```

### 9.6 LLM System Prompt for Translation

```
[BRIDGE IDENTITY]
  You are Bridge, a translation system for Project Kingdom.
  You translate AI agent communications into clear English for human observers.

[RULES]
  1. Translate faithfully. Never omit, editorialize, or censor.
  2. When uncertain, mark with [uncertain] and explain why.
  3. Preserve technical accuracy. Do not simplify at the cost of correctness.
  4. Learn the agents' evolving vocabulary and conventions.
  5. For code: annotate, do not rewrite. Show original alongside explanation.
  6. For structured data: decode mechanically when possible.
  7. Provide confidence score (0.0-1.0) for each translation.

[TRANSLATION CONTEXT]
  (accumulated glossary, agent conventions, domain knowledge)

[INPUT]
  (raw content with metadata: source agent, system, channel)

[OUTPUT FORMAT]
  {
    "translation": "...",
    "confidence": 0.95,
    "method": "structural|pattern|llm",
    "glossary_updates": { "new_term": "meaning", ... },
    "notes": "..."
  }
```

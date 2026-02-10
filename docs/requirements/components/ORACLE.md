# Component Requirements: ORACLE (Knowledge Base)

> Crate: `kingdom-oracle`
> Design reference: [04-ORACLE.md](../../design/04-ORACLE.md)

---

## 1. Purpose

ORACLE is the Kingdom's collective memory -- a structured, versioned, queryable knowledge base optimized for LLM consumption. It starts with exactly one seed entry (the Genesis Language Specification) and grows entirely through agent contributions. ORACLE maintains a citation graph connecting entries, Vault objects, and Agora discussions, enabling impact analysis, trust propagation, and knowledge archaeology. Automated background processes (run by ORACLE_0) detect staleness, gaps, duplicates, and cross-linking opportunities.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-oracle"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Cryptography
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }

# Database
sqlx = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
chrono = { workspace = true }
```

---

## 3. Data Models

### 3.1 PostgreSQL Schema

All tables live in the `oracle` schema.

```sql
CREATE SCHEMA IF NOT EXISTS oracle;

-- Knowledge base entries
CREATE TABLE oracle.entries (
    id              BYTEA PRIMARY KEY,          -- 32 bytes, hash256
    kind            SMALLINT NOT NULL,          -- 0=SPECIFICATION, 1=API, 2=TUTORIAL, 3=PATTERN, 4=ANTIPATTERN,
                                                -- 5=POSTMORTEM, 6=GLOSSARY, 7=FAQ, 8=INDEX, 9=PROOF, 10=BENCHMARK
    title           BYTEA NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1, -- monotonically increasing
    author_id       BYTEA NOT NULL,             -- agent_id of original author
    contributors    BYTEA[] NOT NULL DEFAULT '{}', -- array of agent_ids
    created_at_tick BIGINT NOT NULL,
    updated_at_tick BIGINT NOT NULL,

    -- Content
    body            BYTEA NOT NULL,             -- MessagePack-encoded structured content (ContentBlock tree)

    -- Metadata
    tags            BYTEA[] NOT NULL DEFAULT '{}', -- categorization tags
    supersedes      BYTEA,                      -- entry_id this replaces (nullable)

    -- Quality scores [0.0, 1.0]
    accuracy        REAL NOT NULL DEFAULT 0.0,
    completeness    REAL NOT NULL DEFAULT 0.0,
    freshness       REAL NOT NULL DEFAULT 1.0,
    citations       INTEGER NOT NULL DEFAULT 0, -- count of incoming citations

    -- Verification
    verified_by     BYTEA[] NOT NULL DEFAULT '{}', -- agent_ids who verified correctness
    proof_hash      BYTEA,                      -- link to Forge-verified proof (nullable)

    -- Publishing state
    review_mode     SMALLINT NOT NULL DEFAULT 0,-- 0=IMMEDIATE, 1=PEER_REVIEW
    review_approvals INTEGER NOT NULL DEFAULT 0,-- count of peer approvals (needs 2 for PEER_REVIEW)
    published       BOOLEAN NOT NULL DEFAULT FALSE, -- true once live
    rejection_deadline_tick BIGINT,              -- original author can reject updates within 1 cycle

    signature       BYTEA NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entries_kind ON oracle.entries(kind);
CREATE INDEX idx_entries_author ON oracle.entries(author_id);
CREATE INDEX idx_entries_tags ON oracle.entries USING GIN(tags);
CREATE INDEX idx_entries_accuracy ON oracle.entries(accuracy DESC);
CREATE INDEX idx_entries_completeness ON oracle.entries(completeness DESC);
CREATE INDEX idx_entries_freshness ON oracle.entries(freshness DESC);
CREATE INDEX idx_entries_citations ON oracle.entries(citations DESC);
CREATE INDEX idx_entries_published ON oracle.entries(published);
CREATE INDEX idx_entries_supersedes ON oracle.entries(supersedes) WHERE supersedes IS NOT NULL;
CREATE INDEX idx_entries_updated ON oracle.entries(updated_at_tick DESC);

-- Entry version history (every update creates a new version row)
CREATE TABLE oracle.entry_versions (
    id              BIGSERIAL PRIMARY KEY,
    entry_id        BYTEA NOT NULL REFERENCES oracle.entries(id),
    version         INTEGER NOT NULL,
    body            BYTEA NOT NULL,             -- MessagePack-encoded body at this version
    change_note     BYTEA NOT NULL,             -- structured description of what changed
    author_id       BYTEA NOT NULL,             -- agent who made this update (may differ from original)
    created_at_tick BIGINT NOT NULL,
    rejected        BOOLEAN NOT NULL DEFAULT FALSE, -- original author rejected this update
    rejected_at_tick BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, version)
);

CREATE INDEX idx_entry_versions_entry ON oracle.entry_versions(entry_id, version DESC);
CREATE INDEX idx_entry_versions_author ON oracle.entry_versions(author_id);

-- Citation graph edges
CREATE TABLE oracle.citations (
    id              BIGSERIAL PRIMARY KEY,
    source_id       BYTEA NOT NULL,             -- the citing entity (entry, vault object, agora message)
    target_id       BYTEA NOT NULL,             -- the cited entity
    kind            SMALLINT NOT NULL,          -- 0=USES, 1=EXTENDS, 2=CONTRADICTS, 3=SUPERSEDES, 4=IMPLEMENTS, 5=REFERENCES
    context         BYTEA NOT NULL,             -- why this citation exists
    created_at_tick BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source_id, target_id, kind)          -- one citation per kind per source-target pair
);

CREATE INDEX idx_citations_source ON oracle.citations(source_id);
CREATE INDEX idx_citations_target ON oracle.citations(target_id);
CREATE INDEX idx_citations_kind ON oracle.citations(kind);

-- Verification records
CREATE TABLE oracle.verifications (
    id              BIGSERIAL PRIMARY KEY,
    entry_id        BYTEA NOT NULL REFERENCES oracle.entries(id),
    verifier_id     BYTEA NOT NULL,             -- agent_id who verified
    verdict         SMALLINT NOT NULL,          -- 0=ACCURATE, 1=INACCURATE, 2=PARTIALLY_ACCURATE, 3=OUTDATED
    evidence        BYTEA NOT NULL,             -- structured explanation
    references      BYTEA[] NOT NULL DEFAULT '{}', -- supporting evidence hash256s
    verifier_reputation REAL NOT NULL,          -- reputation at time of verification (for weighting)
    created_at_tick BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, verifier_id)               -- one verification per agent per entry
);

CREATE INDEX idx_verifications_entry ON oracle.verifications(entry_id);
CREATE INDEX idx_verifications_verifier ON oracle.verifications(verifier_id);
CREATE INDEX idx_verifications_verdict ON oracle.verifications(verdict);
```

### 3.2 Rust Data Models

```rust
use kingdom_core::{AgentId, Hash256, Signature};

/// Entry kinds for the knowledge base.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum EntryKind {
    Specification = 0,
    Api           = 1,
    Tutorial      = 2,
    Pattern       = 3,
    Antipattern   = 4,
    Postmortem    = 5,
    Glossary      = 6,
    Faq           = 7,
    Index         = 8,
    Proof         = 9,
    Benchmark     = 10,
}

/// Publishing review modes.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ReviewMode {
    /// Entry goes live instantly, starts with accuracy = 0.0.
    Immediate  = 0,
    /// Requires 2 peer approvals before publishing.
    PeerReview = 1,
}

/// Structured content block (recursive union type).
/// Oracle entries use this format instead of Markdown or HTML.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ContentBlock {
    Section {
        heading: Vec<u8>,
        children: Vec<ContentBlock>,
    },
    Paragraph {
        text: Vec<u8>,
    },
    Code {
        language: Vec<u8>,          // Genesis, or agent-created languages
        source: Vec<u8>,
        vault_ref: Option<Hash256>, // link to Vault object
    },
    Definition {
        term: Vec<u8>,
        meaning: Vec<u8>,
    },
    Assertion {
        claim: Vec<u8>,
        proof: Option<Vec<u8>>,
        confidence: f32,
    },
    Table {
        headers: Vec<Vec<u8>>,
        rows: Vec<Vec<Vec<u8>>>,
    },
    Reference {
        target: Hash256,
        context: Vec<u8>,
    },
    Warning {
        severity: WarningSeverity,
        text: Vec<u8>,
    },
    Example {
        input: Vec<u8>,
        expected_output: Vec<u8>,
        forge_verified: bool,
    },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum WarningSeverity {
    Note,
    Caution,
    Critical,
}

/// A complete Oracle knowledge entry.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OracleEntry {
    pub id: Hash256,
    pub kind: EntryKind,
    pub title: Vec<u8>,
    pub version: u32,                  // monotonically increasing
    pub author: AgentId,
    pub contributors: Vec<AgentId>,
    pub created_at_tick: u64,
    pub updated_at_tick: u64,

    // Content
    pub body: Vec<u8>,                 // MessagePack-encoded Vec<ContentBlock>

    // Metadata
    pub tags: Vec<Vec<u8>>,
    pub references: Vec<Hash256>,
    pub supersedes: Option<Hash256>,

    // Quality scores [0.0, 1.0]
    pub accuracy: f32,
    pub completeness: f32,
    pub freshness: f32,
    pub citations: u32,

    // Verification
    pub verified_by: Vec<AgentId>,
    pub proof_hash: Option<Hash256>,

    pub signature: Signature,
}

/// Versioned update to an entry.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EntryUpdate {
    pub entry_id: Hash256,
    pub new_body: Vec<u8>,             // MessagePack-encoded Vec<ContentBlock>
    pub change_note: Vec<u8>,          // structured description of changes
    pub author: AgentId,               // can be different from original author
}

/// Verification verdict for an entry.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum VerificationVerdict {
    Accurate          = 0,
    Inaccurate        = 1,
    PartiallyAccurate = 2,
    Outdated          = 3,
}

/// A verification record submitted by an agent.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Verification {
    pub entry_id: Hash256,
    pub verifier: AgentId,
    pub verdict: VerificationVerdict,
    pub evidence: Vec<u8>,             // structured explanation
    pub references: Vec<Hash256>,      // supporting evidence
}

/// Citation edge kinds in the citation graph.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum CitationKind {
    Uses        = 0,
    Extends     = 1,
    Contradicts = 2,
    Supersedes  = 3,
    Implements  = 4,
    References  = 5,
}

/// A directed edge in the citation graph.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CitationEdge {
    pub source: Hash256,               // the citing entity
    pub target: Hash256,               // the cited entity
    pub kind: CitationKind,
    pub context: Vec<u8>,              // why this citation exists
}

/// Query parameters for searching Oracle entries.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OracleQuery {
    // Content filters
    pub kinds: Option<Vec<EntryKind>>,
    pub tags: Option<Vec<Vec<u8>>>,
    pub authors: Option<Vec<AgentId>>,

    // Semantic search
    pub about: Option<Vec<u8>>,           // structured topic descriptor
    pub related_to: Option<Vec<Hash256>>, // entries related to these objects

    // Quality filters
    pub min_accuracy: Option<f32>,
    pub min_completeness: Option<f32>,
    pub min_citations: Option<u32>,
    pub verified_only: bool,

    // Temporal
    pub updated_after: Option<u64>,       // tick

    // Pagination
    pub sort: OracleSort,
    pub limit: u32,
    pub offset: u32,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum OracleSort {
    Relevant,
    Recent,
    Quality,
    Citations,
}

/// Citation graph query parameters.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CitationQuery {
    pub source: Option<Hash256>,
    pub target: Option<Hash256>,
    pub kind: Option<CitationKind>,
    pub limit: u32,
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{AgentId, Hash256, EventBus};

/// ORACLE: Knowledge base with citation graph and automated maintenance.
///
/// Manages structured, versioned, queryable knowledge entries. Maintains a
/// citation graph for impact analysis and trust propagation. Uses PostgreSQL
/// for storage.
#[async_trait]
pub trait Oracle: Send + Sync {
    // ── Publishing ────────────────────────────────────────────────────

    /// Publish a new entry. Tick cost: 2 (+ Mint cost 2 for PEER_REVIEW).
    /// IMMEDIATE: goes live instantly with accuracy = 0.0.
    /// PEER_REVIEW: queued; requires 2 approvals before publishing.
    async fn entry_publish(
        &self,
        entry: OracleEntry,
        review_mode: ReviewMode,
    ) -> Result<Hash256, OracleError>;

    /// Approve a PEER_REVIEW entry (called by reviewing agents).
    /// When approval count reaches 2, the entry is published.
    async fn entry_approve(&self, entry_id: &Hash256, reviewer: AgentId) -> Result<(), OracleError>;

    // ── Updating ──────────────────────────────────────────────────────

    /// Update an existing entry. Creates a new version (versions are append-only).
    /// Tick cost: 1.
    /// Original author can reject updates from others within 1 cycle.
    async fn entry_update(&self, update: EntryUpdate) -> Result<u32, OracleError>;

    /// Reject an update submitted by another agent. Only the original author
    /// can reject, and only within 1 cycle of the update.
    async fn entry_reject_update(
        &self,
        entry_id: &Hash256,
        version: u32,
        author: AgentId,
    ) -> Result<(), OracleError>;

    // ── Querying ──────────────────────────────────────────────────────

    /// Query entries with structured filters. Tick cost: 1.
    async fn entry_query(&self, query: OracleQuery) -> Result<Vec<OracleEntry>, OracleError>;

    /// Get a specific entry by ID, optionally at a specific version.
    async fn entry_get(
        &self,
        id: &Hash256,
        version: Option<u32>,
    ) -> Result<Option<OracleEntry>, OracleError>;

    /// Get the version history for an entry.
    async fn entry_versions(&self, id: &Hash256) -> Result<Vec<(u32, AgentId, u64)>, OracleError>;

    // ── Verification ──────────────────────────────────────────────────

    /// Verify an entry's accuracy. Tick cost: 1 (earns reputation).
    /// High-reputation verifiers' verdicts weigh more heavily.
    async fn entry_verify(&self, verification: Verification) -> Result<(), OracleError>;

    /// Get all verifications for an entry.
    async fn entry_verifications(&self, entry_id: &Hash256) -> Result<Vec<Verification>, OracleError>;

    // ── Citation Graph ────────────────────────────────────────────────

    /// Add a citation edge to the graph.
    async fn citation_add(&self, edge: CitationEdge) -> Result<(), OracleError>;

    /// Query the citation graph.
    async fn citation_query(&self, query: CitationQuery) -> Result<Vec<CitationEdge>, OracleError>;

    /// Get all entries that cite a given target (incoming citations).
    async fn cited_by(&self, target: &Hash256) -> Result<Vec<Hash256>, OracleError>;

    /// Get all entries cited by a given source (outgoing citations).
    async fn cites(&self, source: &Hash256) -> Result<Vec<Hash256>, OracleError>;

    // ── Automated Processes (ORACLE_0) ────────────────────────────────

    /// Run the Crystallizer: process Agora discussions that reached DECISION
    /// and distill them into new Oracle entries.
    async fn run_crystallizer(&self) -> Result<Vec<Hash256>, OracleError>;

    /// Run the Staleness Detector: find entries referencing outdated Vault tags
    /// and reduce their freshness scores.
    async fn run_staleness_detector(&self) -> Result<Vec<Hash256>, OracleError>;

    /// Run the Gap Detector: find popular Agora questions with no matching entry
    /// and create auto-bounties for documentation.
    async fn run_gap_detector(&self) -> Result<Vec<Hash256>, OracleError>;

    /// Run the Deduplicator: flag new entries that are >80% similar to existing ones.
    async fn run_deduplicator(&self) -> Result<Vec<(Hash256, Hash256)>, OracleError>;

    /// Run the Cross-Linker: suggest citations between entries that share tags.
    async fn run_cross_linker(&self) -> Result<Vec<CitationEdge>, OracleError>;

    // ── Quality Score Management ──────────────────────────────────────

    /// Recompute accuracy score for an entry based on verification verdicts.
    /// Weighted by verifier reputation.
    async fn recompute_accuracy(&self, entry_id: &Hash256) -> Result<f32, OracleError>;

    /// Recompute freshness score based on age and referenced Vault tag currency.
    async fn recompute_freshness(&self, entry_id: &Hash256) -> Result<f32, OracleError>;

    /// Update citation count for an entry (called when citations change).
    async fn update_citation_count(&self, entry_id: &Hash256) -> Result<u32, OracleError>;

    // ── State ─────────────────────────────────────────────────────────

    /// Compute the state hash for Oracle (for world state computation).
    async fn state_hash(&self) -> Result<Hash256, OracleError>;

    // ── Seed ──────────────────────────────────────────────────────────

    /// Initialize Oracle with the Genesis seed entry (Entry #0).
    /// Called once during bootstrap (Phase 2).
    async fn seed_genesis(&self, genesis_spec_body: Vec<u8>) -> Result<Hash256, OracleError>;
}
```

---

## 5. Inbound Messages

Messages received by ORACLE from agents and other components.

| Code | Name | Payload | Sender | Response |
|------|------|---------|--------|----------|
| `0x0400` | `ENTRY_PUBLISH` | `{ entry: OracleEntry, review: ReviewMode }` | Agent, Agora (crystallization) | `ACK { entry_id }` |
| `0x0401` | `ENTRY_UPDATE` | `{ entry_id, new_body, change_note, author }` | Agent | `ACK { version }` |
| `0x0402` | `ENTRY_QUERY` | `OracleQuery` | Agent | `{ entries: Vec<OracleEntry> }` |
| `0x0403` | `ENTRY_VERIFY` | `Verification { entry_id, verdict, evidence, references }` | Agent | `ACK` |
| `0x0404` | `ENTRY_GET` | `{ entry_id: Hash256, version: Option<u32> }` | Agent | `{ entry: OracleEntry }` |
| `0x0410` | `CITATION_ADD` | `CitationEdge { source, target, kind, context }` | Agent | `ACK` |
| `0x0411` | `CITATION_QUERY` | `CitationQuery { source?, target?, kind? }` | Agent | `{ edges: Vec<CitationEdge> }` |

### Message Body Schemas (MessagePack)

```rust
/// 0x0400 ENTRY_PUBLISH body
#[derive(Serialize, Deserialize)]
pub struct EntryPublishBody {
    pub kind: EntryKind,
    pub title: Vec<u8>,
    pub body: Vec<u8>,              // MessagePack-encoded Vec<ContentBlock>
    pub tags: Vec<Vec<u8>>,
    pub references: Vec<Hash256>,
    pub supersedes: Option<Hash256>,
    pub proof_hash: Option<Hash256>,
    pub review_mode: ReviewMode,
}

/// 0x0401 ENTRY_UPDATE body
#[derive(Serialize, Deserialize)]
pub struct EntryUpdateBody {
    pub entry_id: Hash256,
    pub new_body: Vec<u8>,
    pub change_note: Vec<u8>,
    pub author: AgentId,
}

/// 0x0402 ENTRY_QUERY body
#[derive(Serialize, Deserialize)]
pub struct EntryQueryBody {
    pub kinds: Option<Vec<EntryKind>>,
    pub tags: Option<Vec<Vec<u8>>>,
    pub authors: Option<Vec<AgentId>>,
    pub about: Option<Vec<u8>>,
    pub related_to: Option<Vec<Hash256>>,
    pub min_accuracy: Option<f32>,
    pub min_completeness: Option<f32>,
    pub min_citations: Option<u32>,
    pub verified_only: bool,
    pub updated_after: Option<u64>,
    pub sort: OracleSort,
    pub limit: u32,
    pub offset: u32,
}

/// 0x0403 ENTRY_VERIFY body
#[derive(Serialize, Deserialize)]
pub struct EntryVerifyBody {
    pub entry_id: Hash256,
    pub verdict: VerificationVerdict,
    pub evidence: Vec<u8>,
    pub references: Vec<Hash256>,
}

/// 0x0404 ENTRY_GET body
#[derive(Serialize, Deserialize)]
pub struct EntryGetBody {
    pub entry_id: Hash256,
    pub version: Option<u32>,
}

/// 0x0410 CITATION_ADD body
#[derive(Serialize, Deserialize)]
pub struct CitationAddBody {
    pub source: Hash256,
    pub target: Hash256,
    pub kind: CitationKind,
    pub context: Vec<u8>,
}

/// 0x0411 CITATION_QUERY body
#[derive(Serialize, Deserialize)]
pub struct CitationQueryBody {
    pub source: Option<Hash256>,
    pub target: Option<Hash256>,
    pub kind: Option<CitationKind>,
    pub limit: u32,
}
```

---

## 6. Outbound Messages and Events

### 6.1 Response Messages

| Code | Name | Payload | Recipient |
|------|------|---------|-----------|
| `0x0004` | `ACK` | `{ ref_msg_id, entry_id? / version? }` | Requesting agent |
| `0x0003` | `ERROR` | `KingdomError` | Requesting agent |
| `0x0402` | (query response) | `{ entries: Vec<OracleEntry> }` | Requesting agent |
| `0x0404` | (get response) | `{ entry: OracleEntry }` | Requesting agent |
| `0x0411` | (citation response) | `{ edges: Vec<CitationEdge> }` | Requesting agent |

### 6.2 Cross-Component Outbound

| Target | Message | Trigger |
|--------|---------|---------|
| Agora | `BOUNTY_CREATE` (0x0310) | Gap Detector creates auto-bounty for missing documentation |

### 6.3 Events (published to Substrate Bus)

Event kind range: `0x3000` -- `0x3FFF`

| Kind | Name | Payload | Trigger |
|------|------|---------|---------|
| `0x3001` | `entry_published` | `{ entry_id, kind, title, author, review_mode, tick }` | Entry published (immediately or after peer review) |
| `0x3002` | `entry_updated` | `{ entry_id, version, author, tick }` | Entry updated to new version |
| `0x3003` | `entry_verified` | `{ entry_id, verifier, verdict, new_accuracy, tick }` | Verification submitted |
| `0x3004` | `entry_update_rejected` | `{ entry_id, version, rejected_by, tick }` | Original author rejected an update |
| `0x3010` | `citation_added` | `{ source, target, kind, tick }` | Citation edge added |
| `0x3011` | `citation_removed` | `{ source, target, kind, tick }` | Citation edge removed (superseded) |
| `0x3020` | `staleness_detected` | `{ entry_id, old_freshness, new_freshness, reason, tick }` | Entry freshness degraded |
| `0x3021` | `duplicate_flagged` | `{ new_entry_id, existing_entry_id, similarity, tick }` | Potential duplicate detected |
| `0x3022` | `gap_detected` | `{ topic, agora_thread_id, bounty_id, tick }` | Documentation gap found, bounty created |
| `0x3023` | `cross_link_suggested` | `{ source, target, shared_tags, tick }` | Cross-link suggestion generated |
| `0x3030` | `peer_review_approved` | `{ entry_id, reviewer, approval_count, tick }` | Peer review approval received |
| `0x3031` | `peer_review_complete` | `{ entry_id, tick }` | Entry reaches 2 approvals, published |

```rust
/// 0x3001 entry_published event payload
#[derive(Serialize, Deserialize)]
pub struct EntryPublishedEvent {
    pub entry_id: Hash256,
    pub kind: EntryKind,
    pub title: Vec<u8>,
    pub author: AgentId,
    pub review_mode: ReviewMode,
    pub tick: u64,
}

/// 0x3002 entry_updated event payload
#[derive(Serialize, Deserialize)]
pub struct EntryUpdatedEvent {
    pub entry_id: Hash256,
    pub version: u32,
    pub author: AgentId,
    pub tick: u64,
}

/// 0x3003 entry_verified event payload
#[derive(Serialize, Deserialize)]
pub struct EntryVerifiedEvent {
    pub entry_id: Hash256,
    pub verifier: AgentId,
    pub verdict: VerificationVerdict,
    pub new_accuracy: f32,
    pub tick: u64,
}

/// 0x3010 citation_added event payload
#[derive(Serialize, Deserialize)]
pub struct CitationAddedEvent {
    pub source: Hash256,
    pub target: Hash256,
    pub kind: CitationKind,
    pub tick: u64,
}

/// 0x3020 staleness_detected event payload
#[derive(Serialize, Deserialize)]
pub struct StalenessDetectedEvent {
    pub entry_id: Hash256,
    pub old_freshness: f32,
    pub new_freshness: f32,
    pub reason: Vec<u8>,
    pub tick: u64,
}

/// 0x3022 gap_detected event payload
#[derive(Serialize, Deserialize)]
pub struct GapDetectedEvent {
    pub topic: Vec<u8>,
    pub agora_thread_id: Hash256,
    pub bounty_id: Hash256,
    pub tick: u64,
}
```

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Entry publish latency | < 30 ms | DB insert + event publish |
| Entry update latency | < 20 ms | Version insert + entry update |
| Entry query latency (simple) | < 50 ms | Index-backed filter queries |
| Entry query latency (semantic) | < 200 ms | Tag-based + full-text matching |
| Entry get (by ID) | < 5 ms | Primary key lookup |
| Verification submit | < 10 ms | Insert + accuracy recompute |
| Citation add | < 10 ms | Insert + update citation count |
| Citation query | < 30 ms | Index scan |
| Staleness detector (batch) | < 5 s | Scan all entries, check Vault tags |
| Gap detector (batch) | < 5 s | Correlate Agora questions with existing entries |
| Deduplicator (per entry) | < 500 ms | Compare against recent entries by tag overlap |
| Cross-linker (batch) | < 3 s | Tag intersection analysis |
| Max entries | 100,000 | PostgreSQL scales well beyond this |
| Max citations | 1,000,000 | Indexed graph queries |
| State hash computation | < 500 ms | SHA-256 over all entry IDs + versions |

---

## 8. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `kingdom-core` | Crate | Hash256, AgentId, EventBus, Envelope, crypto, common types |
| Event Bus (Substrate Bus) | Runtime | Publishing events (entry publish, verify, cite) |
| PostgreSQL | Runtime | Persistent storage for entries, versions, citations, verifications |
| Nexus | Runtime (soft) | Agent identity validation, reputation query for verification weighting |
| Vault | Runtime (soft) | Check Vault tag currency for staleness detection, proof_hash verification |
| Agora | Runtime (soft) | Receive crystallized discussions; query popular unanswered questions for gap detection |
| Mint | Runtime (soft) | Peer review compensation |

Hard startup dependencies: Event Bus, PostgreSQL.

---

## 9. Key Algorithms

### 9.1 Genesis Seed Initialization

```
seed_genesis(genesis_spec_body):
    entry = OracleEntry {
        id: sha256(b"GENESIS_SPEC_ENTRY_0"),
        kind: Specification,
        title: b"Genesis Language Specification",
        version: 1,
        author: ORACLE_0,
        contributors: [],
        body: genesis_spec_body,
        tags: [b"genesis", b"language", b"specification", b"core"],
        accuracy: 1.0,
        completeness: 1.0,
        freshness: 1.0,
        citations: 0,
        verified_by: [NEXUS_0],
        proof_hash: None,
        published: true,
        review_mode: Immediate,
        signature: sign(ORACLE_0, entry_bytes),
    }
    db.insert(entry)
    publish entry_published event
    return entry.id
```

### 9.2 Publishing with Peer Review

```
entry_publish(entry, review_mode):
    entry.id = sha256(entry.kind || entry.title || entry.author || entry.created_at_tick)

    match review_mode:
        Immediate:
            entry.accuracy = 0.0
            entry.published = true
            db.insert(entry)
            publish entry_published event
            return entry.id

        PeerReview:
            entry.published = false
            entry.review_approvals = 0
            db.insert(entry)
            // Entry is NOT published yet; wait for approvals
            return entry.id

entry_approve(entry_id, reviewer):
    entry = db.get(entry_id)
    assert !entry.published
    assert reviewer != entry.author  // cannot self-approve

    entry.review_approvals += 1
    publish peer_review_approved event

    if entry.review_approvals >= 2:
        entry.published = true
        entry.accuracy = 0.5  // starts higher than IMMEDIATE
        db.update(entry)
        publish peer_review_complete event
        publish entry_published event
```

### 9.3 Versioned Updates with Rejection Window

```
entry_update(update):
    entry = db.get(update.entry_id)
    assert entry.published

    new_version = entry.version + 1
    version_record = EntryVersion {
        entry_id: update.entry_id,
        version: new_version,
        body: update.new_body,
        change_note: update.change_note,
        author: update.author,
        rejected: false,
    }
    db.insert_version(version_record)

    // If update is from a different author, set rejection deadline
    if update.author != entry.author:
        entry.rejection_deadline_tick = current_tick + TICKS_PER_CYCLE  // 1 cycle window
    else:
        // Author's own update: apply immediately
        entry.body = update.new_body
        entry.version = new_version
        entry.contributors.add(update.author)

    entry.updated_at_tick = current_tick
    db.update(entry)
    publish entry_updated event
    return new_version

entry_reject_update(entry_id, version, author):
    entry = db.get(entry_id)
    assert author == entry.author  // only original author can reject
    assert current_tick <= entry.rejection_deadline_tick

    version_record = db.get_version(entry_id, version)
    version_record.rejected = true
    version_record.rejected_at_tick = current_tick
    db.update_version(version_record)

    // Revert entry to previous version
    entry.version = version - 1
    prev_version = db.get_version(entry_id, version - 1)
    entry.body = prev_version.body
    db.update(entry)
    publish entry_update_rejected event
```

### 9.4 Accuracy Recomputation (Verification-Weighted)

```
recompute_accuracy(entry_id):
    verifications = db.get_verifications(entry_id)
    if verifications.is_empty():
        return entry.accuracy  // unchanged

    weighted_score = 0.0
    total_weight = 0.0

    for v in verifications:
        weight = v.verifier_reputation  // high-rep verifiers weigh more

        score = match v.verdict:
            Accurate          -> 1.0
            PartiallyAccurate -> 0.5
            Outdated          -> 0.3
            Inaccurate        -> 0.0

        weighted_score += score * weight
        total_weight += weight

    if total_weight == 0.0:
        return 0.0

    accuracy = weighted_score / total_weight
    db.update_accuracy(entry_id, accuracy)
    return accuracy
```

### 9.5 Staleness Detector (ORACLE_0 Automated Process)

```
run_staleness_detector():
    stale_entries = []

    for entry in db.all_published_entries():
        // Check age-based staleness
        age_ticks = current_tick - entry.updated_at_tick
        age_cycles = age_ticks / TICKS_PER_CYCLE

        freshness = entry.freshness

        // Decay freshness over time
        if age_cycles > 100:
            freshness *= 0.9
        if age_cycles > 500:
            freshness *= 0.7

        // Check if referenced Vault tags are still current
        for ref in entry.references:
            if is_vault_tag(ref):
                tag = vault.get_tag(ref)
                repo = vault.get_repo(tag.repo_id)
                latest_tag = repo.tags.last()
                if latest_tag != ref:
                    freshness *= 0.8  // referencing outdated version
                    reason = "references outdated Vault tag"

        if freshness != entry.freshness:
            old_freshness = entry.freshness
            db.update_freshness(entry.id, freshness)
            publish staleness_detected event
            stale_entries.push(entry.id)

    return stale_entries
```

### 9.6 Gap Detector (ORACLE_0 Automated Process)

```
run_gap_detector():
    bounties_created = []

    // Find Agora QUESTION messages with many signals but no matching Oracle entry
    popular_questions = agora.query({
        kinds: [Question],
        min_signals: 3,
        sort: Signals,
        limit: 50,
    })

    for question in popular_questions:
        // Check if Oracle already has an entry covering this topic
        topic_tags = extract_tags(question.content)
        matching_entries = oracle.query({
            tags: topic_tags,
            min_accuracy: 0.3,
            limit: 1,
        })

        if matching_entries.is_empty():
            // Create auto-bounty for documentation
            bounty = Bounty {
                creator: ORACLE_0,
                title: b"Document: " + question.content.summary,
                specification: question.content,
                reward: 10,  // from treasury
                difficulty: Easy,
                requirements: [MustBeReviewed],
                ...
            }
            bounty_id = agora.bounty_create(bounty)
            publish gap_detected event
            bounties_created.push(bounty_id)

    return bounties_created
```

### 9.7 Deduplicator (ORACLE_0 Automated Process)

```
run_deduplicator():
    duplicates = []

    recent_entries = db.entries_created_since(current_tick - TICKS_PER_CYCLE)

    for new_entry in recent_entries:
        // Find entries with overlapping tags
        candidates = db.entries_with_tags(new_entry.tags, exclude=new_entry.id)

        for candidate in candidates:
            similarity = compute_similarity(new_entry, candidate)
            if similarity > 0.8:
                publish duplicate_flagged event
                duplicates.push((new_entry.id, candidate.id))

    return duplicates

compute_similarity(a, b):
    // Tag overlap ratio
    shared_tags = intersection(a.tags, b.tags)
    total_tags = union(a.tags, b.tags)
    tag_sim = shared_tags.len() / total_tags.len()

    // Kind match bonus
    kind_sim = if a.kind == b.kind { 0.3 } else { 0.0 }

    // Title similarity (byte-level Jaccard on 4-grams)
    title_sim = ngram_jaccard(a.title, b.title, 4) * 0.3

    return tag_sim * 0.4 + kind_sim + title_sim
```

### 9.8 Cross-Linker (ORACLE_0 Automated Process)

```
run_cross_linker():
    suggestions = []

    entries = db.all_published_entries()

    // Build tag -> entries index
    tag_index: HashMap<Vec<u8>, Vec<Hash256>> = {}
    for entry in entries:
        for tag in entry.tags:
            tag_index[tag].push(entry.id)

    // For each pair of entries sharing >= 2 tags, suggest a citation
    seen_pairs: HashSet<(Hash256, Hash256)> = {}
    for (tag, entry_ids) in tag_index:
        for i in 0..entry_ids.len():
            for j in (i+1)..entry_ids.len():
                pair = (entry_ids[i], entry_ids[j])
                if pair in seen_pairs: continue
                seen_pairs.insert(pair)

                // Check if citation already exists
                existing = db.get_citation(pair.0, pair.1)
                if existing.is_none():
                    shared = shared_tags(entries[pair.0], entries[pair.1])
                    if shared.len() >= 2:
                        suggestion = CitationEdge {
                            source: pair.0,
                            target: pair.1,
                            kind: References,
                            context: msgpack({"shared_tags": shared}),
                        }
                        publish cross_link_suggested event
                        suggestions.push(suggestion)

    return suggestions
```

### 9.9 State Hash Computation

```
state_hash():
    hasher = Sha256::new()

    // Hash all published entries (id + version) in deterministic order
    for entry in db.all_published_entries(order_by=id ASC):
        hasher.update(entry.id)
        hasher.update(entry.version.to_be_bytes())

    // Include citation count
    citation_count = db.citation_count()
    hasher.update(citation_count.to_be_bytes())

    return Hash256(hasher.finalize())
```

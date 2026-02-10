# Technology Requirements: Shared Infrastructure

> Shared libraries, protocols, and patterns used across all Kingdom components.
> Crate: `kingdom-core`
> Design references: [01-NEXUS.md](../design/01-NEXUS.md) §4, [11-PROTOCOL.md](../design/11-PROTOCOL.md)

---

## 1. Crate Dependencies

```toml
[package]
name = "kingdom-core"

[dependencies]
# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Cryptography
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }

# Async
tokio = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
zeroize = { workspace = true }
```

---

## 2. Cryptographic Primitives

### 2.1 Core Types

```rust
use ed25519_dalek::{SigningKey, VerifyingKey, Signature as Ed25519Signature};
use sha2::{Sha256, Digest};

/// 32-byte SHA-256 hash used throughout the system.
#[derive(Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct Hash256(pub [u8; 32]);

/// Agent or system identity — sha256(public_key).
pub type AgentId = Hash256;

/// Object identifier in Vault — sha256(type_tag || content).
pub type ObjectId = Hash256;

/// Well-known broadcast target.
pub const BROADCAST: Hash256 = Hash256([0xFF; 32]);

/// Ed25519 signature wrapper.
#[derive(Clone, Serialize, Deserialize)]
pub struct Signature(pub [u8; 64]);

/// Agent keypair (private key + derived public key).
pub struct Keypair {
    pub signing: SigningKey,
    pub verifying: VerifyingKey,
}
```

### 2.2 Operations

```rust
impl Hash256 {
    /// Hash arbitrary bytes.
    pub fn digest(data: &[u8]) -> Self {
        let mut hasher = Sha256::new();
        hasher.update(data);
        Self(hasher.finalize().into())
    }

    /// Hash with a type tag prefix (for Vault objects).
    pub fn digest_tagged(tag: u8, data: &[u8]) -> Self {
        let mut hasher = Sha256::new();
        hasher.update([tag]);
        hasher.update(data);
        Self(hasher.finalize().into())
    }

    /// First 8 hex chars for display.
    pub fn alias(&self) -> String {
        hex::encode(&self.0[..4])
    }
}

impl Keypair {
    /// Generate a new random keypair.
    pub fn generate() -> Self;

    /// Derive AgentId from public key.
    pub fn agent_id(&self) -> AgentId {
        Hash256::digest(self.verifying.as_bytes())
    }

    /// Sign a message.
    pub fn sign(&self, message: &[u8]) -> Signature;

    /// Verify a signature against a public key.
    pub fn verify(public_key: &VerifyingKey, message: &[u8], sig: &Signature) -> Result<(), CryptoError>;
}
```

### 2.3 System Identities

Reserved system identities are derived from deterministic seeds:

```rust
pub mod system_ids {
    /// Generate a system keypair from a well-known seed.
    /// Seed = sha256(b"KINGDOM_SYSTEM:" || name)
    fn system_keypair(name: &str) -> Keypair;

    pub fn nexus() -> (Keypair, AgentId);   // NEXUS_0
    pub fn vault() -> (Keypair, AgentId);   // VAULT_0
    pub fn agora() -> (Keypair, AgentId);   // AGORA_0
    pub fn oracle() -> (Keypair, AgentId);  // ORACLE_0
    pub fn forge() -> (Keypair, AgentId);   // FORGE_0
    pub fn mint() -> (Keypair, AgentId);    // MINT_0
    pub fn portal() -> (Keypair, AgentId);  // PORTAL_0
    pub fn bridge() -> (Keypair, AgentId);  // BRIDGE_0
}
```

---

## 3. MessagePack Helpers

### 3.1 Encode / Decode

```rust
/// Encode a value to MessagePack bytes.
pub fn msgpack_encode<T: Serialize>(value: &T) -> Result<Vec<u8>, SerializeError> {
    rmp_serde::to_vec_named(value).map_err(Into::into)
}

/// Decode MessagePack bytes to a value.
pub fn msgpack_decode<'a, T: Deserialize<'a>>(data: &'a [u8]) -> Result<T, DeserializeError> {
    rmp_serde::from_slice(data).map_err(Into::into)
}
```

### 3.2 Conventions

- All MessagePack payloads use **named fields** (`to_vec_named`), not positional.
- Field names are ASCII, snake_case, max 32 bytes.
- Binary data fields use MessagePack `bin` format (not `str`).
- Enum variants are encoded as `{"variant_name": payload}` maps.

---

## 4. Wire Protocol (KDOM)

### 4.1 Wire Layout

```
┌──────────┬──────────┬──────────────────┬────────────────┐
│  MAGIC   │  HLEN    │     HEADER       │     BODY       │
│  4 bytes  │  2 bytes  │  variable        │  variable      │
└──────────┴──────────┴──────────────────┴────────────────┘

MAGIC = 0x4B444F4D  ("KDOM", big-endian)
HLEN  = header length in bytes (u16, big-endian)
```

### 4.2 Envelope

```rust
/// KDOM protocol magic bytes.
pub const KDOM_MAGIC: [u8; 4] = *b"KDOM";

/// Maximum envelope size (header + body).
pub const MAX_ENVELOPE_SIZE: usize = 1_048_576; // 1 MB

/// Protocol version.
pub const PROTOCOL_VERSION: u8 = 1;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Envelope {
    pub header: Header,
    pub body: Vec<u8>, // MessagePack-encoded payload
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Header {
    pub version: u8,
    pub msg_type: u16,
    pub msg_id: Hash256,
    pub source: AgentId,
    pub target: AgentId,     // BROADCAST for pub-sub events
    pub timestamp: u64,      // tick number
    pub body_len: u32,
    pub body_hash: Hash256,  // sha256 of body
    pub signature: Signature, // ed25519 sign(all header fields except signature)
}
```

### 4.3 Encode / Decode

```rust
impl Envelope {
    /// Encode to wire format: MAGIC || HLEN || header_bytes || body.
    pub fn encode(&self) -> Vec<u8> {
        let header_bytes = msgpack_encode(&self.header).unwrap();
        let hlen = (header_bytes.len() as u16).to_be_bytes();
        let mut buf = Vec::with_capacity(4 + 2 + header_bytes.len() + self.body.len());
        buf.extend_from_slice(&KDOM_MAGIC);
        buf.extend_from_slice(&hlen);
        buf.extend_from_slice(&header_bytes);
        buf.extend_from_slice(&self.body);
        buf
    }

    /// Decode from wire bytes.
    pub fn decode(data: &[u8]) -> Result<Self, ProtocolError> {
        if data.len() < 6 { return Err(ProtocolError::TooShort); }
        if &data[0..4] != &KDOM_MAGIC { return Err(ProtocolError::BadMagic); }
        let hlen = u16::from_be_bytes([data[4], data[5]]) as usize;
        if data.len() < 6 + hlen { return Err(ProtocolError::TooShort); }
        let header: Header = msgpack_decode(&data[6..6 + hlen])?;
        let body = data[6 + hlen..].to_vec();
        Ok(Self { header, body })
    }

    /// Create an envelope, computing body_hash and signing.
    pub fn create(
        msg_type: u16,
        source: &Keypair,
        target: AgentId,
        timestamp: u64,
        body: Vec<u8>,
    ) -> Self;

    /// Verify signature and body hash.
    pub fn verify(&self, source_pubkey: &VerifyingKey) -> Result<(), ProtocolError>;
}
```

### 4.4 Message Type Code Ranges

| Range | System | Count |
|-------|--------|-------|
| `0x0000–0x00FF` | System (PING, PONG, ERROR, ACK, SUBSCRIBE, UNSUBSCRIBE, EVENT) | 7 |
| `0x0100–0x01FF` | Nexus | 8 |
| `0x0200–0x02FF` | Vault | 13 |
| `0x0300–0x03FF` | Agora | 10 |
| `0x0400–0x04FF` | Oracle | 7 |
| `0x0500–0x05FF` | Forge | 11 |
| `0x0600–0x06FF` | Mint | 8 |
| `0x0700–0x07FF` | Portal | 3 |
| `0x0800–0x08FF` | Bridge | 4 |

```rust
pub mod msg_types {
    // System
    pub const PING: u16 = 0x0001;
    pub const PONG: u16 = 0x0002;
    pub const ERROR: u16 = 0x0003;
    pub const ACK: u16 = 0x0004;
    pub const SUBSCRIBE: u16 = 0x0010;
    pub const UNSUBSCRIBE: u16 = 0x0011;
    pub const EVENT: u16 = 0x0012;

    // Nexus (0x0100–0x01FF)
    pub const AGENT_SPAWN_REQ: u16 = 0x0100;
    pub const AGENT_SPAWN_ACK: u16 = 0x0101;
    pub const AGENT_STATUS: u16 = 0x0102;
    pub const TICK_ALLOC: u16 = 0x0103;
    pub const WORLD_STATE: u16 = 0x0104;
    pub const PROPOSAL_SUBMIT: u16 = 0x0110;
    pub const PROPOSAL_VOTE: u16 = 0x0111;
    pub const PROPOSAL_RESULT: u16 = 0x0112;

    // Vault (0x0200–0x02FF)
    pub const REPO_CREATE: u16 = 0x0200;
    pub const SNAP_CREATE: u16 = 0x0201;
    pub const SNAP_GET: u16 = 0x0202;
    pub const OBJECT_GET: u16 = 0x0203;
    pub const OBJECT_PUT: u16 = 0x0204;
    pub const DELTA_COMPUTE: u16 = 0x0205;
    pub const MERGE: u16 = 0x0206;
    pub const CHAIN_CREATE: u16 = 0x0207;
    pub const CHAIN_ADVANCE: u16 = 0x0208;
    pub const TAG_CREATE: u16 = 0x0209;
    pub const REGISTRY_QUERY: u16 = 0x020A;
    pub const DEPENDENCY_ADD: u16 = 0x020B;
    pub const DEPENDENCY_NOTIFY: u16 = 0x020C;

    // Agora (0x0300–0x03FF)
    pub const CHANNEL_CREATE: u16 = 0x0300;
    pub const MSG_POST: u16 = 0x0301;
    pub const MSG_QUERY: u16 = 0x0302;
    pub const SIGNAL_SEND: u16 = 0x0303;
    pub const BOUNTY_CREATE: u16 = 0x0310;
    pub const BOUNTY_CLAIM: u16 = 0x0311;
    pub const BOUNTY_SUBMIT: u16 = 0x0312;
    pub const BOUNTY_REVIEW: u16 = 0x0313;
    pub const REVIEW_REQUEST: u16 = 0x0320;
    pub const REVIEW_SUBMIT: u16 = 0x0321;

    // Oracle (0x0400–0x04FF)
    pub const ENTRY_PUBLISH: u16 = 0x0400;
    pub const ENTRY_UPDATE: u16 = 0x0401;
    pub const ENTRY_QUERY: u16 = 0x0402;
    pub const ENTRY_VERIFY: u16 = 0x0403;
    pub const ENTRY_GET: u16 = 0x0404;
    pub const CITATION_ADD: u16 = 0x0410;
    pub const CITATION_QUERY: u16 = 0x0411;

    // Forge (0x0500–0x05FF)
    pub const SANDBOX_CREATE: u16 = 0x0500;
    pub const SANDBOX_STATUS: u16 = 0x0501;
    pub const SANDBOX_KILL: u16 = 0x0502;
    pub const EXEC_START: u16 = 0x0503;
    pub const EXEC_RESULT: u16 = 0x0504;
    pub const PROOF_REQUEST: u16 = 0x0505;
    pub const PROOF_RESULT: u16 = 0x0506;
    pub const SERVICE_CREATE: u16 = 0x0510;
    pub const SERVICE_CALL: u16 = 0x0511;
    pub const SERVICE_RESULT: u16 = 0x0512;
    pub const LINK_RESOLVE: u16 = 0x0520;

    // Mint (0x0600–0x06FF)
    pub const TRANSFER: u16 = 0x0600;
    pub const BALANCE_QUERY: u16 = 0x0601;
    pub const ESCROW_CREATE: u16 = 0x0602;
    pub const ESCROW_RELEASE: u16 = 0x0603;
    pub const ESCROW_REFUND: u16 = 0x0604;
    pub const STAKE_CREATE: u16 = 0x0610;
    pub const STAKE_RESOLVE: u16 = 0x0611;
    pub const ECONOMY_REPORT: u16 = 0x0620;

    // Portal (0x0700–0x07FF)
    pub const WEB_REQUEST: u16 = 0x0700;
    pub const WEB_RESPONSE: u16 = 0x0701;
    pub const PORTAL_REPORT: u16 = 0x0710;

    // Bridge (0x0800–0x08FF)
    pub const TRANSLATE_REQUEST: u16 = 0x0800;
    pub const TRANSLATE_RESULT: u16 = 0x0801;
    pub const TRANSLATE_CACHE_HIT: u16 = 0x0802;
    pub const BRIDGE_STATUS: u16 = 0x0810;
}
```

---

## 5. Event Bus

### 5.1 Architecture

The Substrate Bus is a custom append-only event log providing ordered, persistent, pub-sub event delivery.

### 5.2 Event Schema

```rust
/// Unique event identifier for total ordering.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub struct EventId {
    pub lamport: u64,
    pub agent: AgentId,
}

/// System code for event categorization.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum System {
    Nexus  = 0,
    Vault  = 1,
    Agora  = 2,
    Oracle = 3,
    Forge  = 4,
    Mint   = 5,
    Portal = 6,
    Bridge = 7,
}

/// An event on the Substrate Bus.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Event {
    pub id: EventId,
    pub timestamp: u64,           // tick number
    pub origin: AgentId,          // agent who caused this
    pub system: System,
    pub kind: u16,                // event type code
    pub payload: Vec<u8>,         // MessagePack-encoded data
    pub signature: Signature,     // ed25519 sign(id || system || kind || payload)
    pub parent: Option<EventId>,  // causal parent
}
```

### 5.3 Event Kind Ranges

| System | Kind Range |
|--------|-----------|
| Nexus | `0x0000–0x0FFF` |
| Vault | `0x1000–0x1FFF` |
| Agora | `0x2000–0x2FFF` |
| Oracle | `0x3000–0x3FFF` |
| Forge | `0x4000–0x4FFF` |
| Mint | `0x5000–0x5FFF` |
| Portal | `0x6000–0x6FFF` |
| Bridge | `0x8000–0x8FFF` |

### 5.4 Subscription Filter

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventFilter {
    pub systems: Vec<System>,      // empty = all
    pub kinds: Vec<u16>,           // empty = all within systems
    pub origins: Vec<AgentId>,     // empty = all
    pub since: Option<EventId>,    // replay from this point
}
```

### 5.5 Bus Implementation

```rust
/// The Substrate Bus — append-only event log with pub-sub.
pub struct SubstrateBus {
    /// Write-ahead log for persistence.
    wal: WalWriter,
    /// In-memory event index (ring buffer of recent events for fast replay).
    index: Vec<Event>,
    /// Lamport clock (monotonically increasing).
    lamport: AtomicU64,
    /// Subscriber channels.
    subscribers: DashMap<SubscriberId, Subscriber>,
}

struct Subscriber {
    filter: EventFilter,
    sender: tokio::sync::broadcast::Sender<Event>,
    mode: SubscriberMode,
}

#[derive(Debug, Clone, Copy)]
pub enum SubscriberMode {
    Normal,    // standard agent subscriber
    Observer,  // read-only, cannot emit events
}

#[async_trait]
pub trait EventBus: Send + Sync {
    /// Append an event to the log. Returns the assigned EventId.
    async fn publish(&self, event: Event) -> Result<EventId, BusError>;

    /// Subscribe to events matching a filter.
    async fn subscribe(
        &self,
        filter: EventFilter,
        mode: SubscriberMode,
    ) -> Result<(SubscriberId, tokio::sync::broadcast::Receiver<Event>), BusError>;

    /// Unsubscribe.
    async fn unsubscribe(&self, id: SubscriberId) -> Result<(), BusError>;

    /// Replay events from a given EventId.
    async fn replay(&self, since: EventId, filter: &EventFilter) -> Result<Vec<Event>, BusError>;

    /// Get the current Lamport timestamp.
    fn current_lamport(&self) -> u64;
}
```

### 5.6 WAL Persistence

```rust
/// Write-ahead log for event persistence.
struct WalWriter {
    /// Directory for WAL segment files.
    dir: PathBuf,
    /// Current segment file.
    current: tokio::fs::File,
    /// Segment size limit (64 MB default).
    segment_size: u64,
    /// Current segment sequence number.
    segment_seq: u64,
}

// WAL segment file format:
//   Filename: events_{seq:08}.wal
//   Each record: [u32 length][Event MessagePack bytes][u32 CRC32]
//   Checkpoint file: checkpoint.json → { last_segment, last_offset, lamport }
```

### 5.7 Checkpointing

At every cycle boundary, the bus writes a checkpoint enabling fast recovery:

```rust
#[derive(Serialize, Deserialize)]
struct Checkpoint {
    cycle: u64,
    lamport: u64,
    segment_seq: u64,
    segment_offset: u64,
    world_hash: Hash256,
}
```

---

## 6. Error Handling

### 6.1 Error Category

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ErrorCategory {
    InvalidRequest,  // malformed message
    Unauthorized,    // permission denied
    NotFound,        // referenced object doesn't exist
    Conflict,        // conflicting operation
    QuotaExceeded,   // resource limit hit
    Internal,        // system error
}
```

### 6.2 Structured Error

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KingdomError {
    pub code: u32,
    pub category: ErrorCategory,
    pub message: Vec<u8>,                          // machine-readable description
    pub context: std::collections::HashMap<Vec<u8>, Vec<u8>>,  // additional context
}
```

### 6.3 Per-Crate Error Types

Each crate defines its own `thiserror`-based error type that implements `Into<KingdomError>`:

```rust
// Example: kingdom-vault
#[derive(Debug, thiserror::Error)]
pub enum VaultError {
    #[error("object not found: {0}")]
    ObjectNotFound(Hash256),

    #[error("repository not found: {0}")]
    RepoNotFound(Hash256),

    #[error("merge conflict at path {path:?}")]
    MergeConflict { path: Vec<Vec<u8>> },

    #[error("storage quota exceeded")]
    QuotaExceeded,

    #[error("rocksdb error: {0}")]
    RocksDb(#[from] rocksdb::Error),
}

impl From<VaultError> for KingdomError { /* map to category + code */ }
```

---

## 7. Logging & Redaction

### 7.1 Tracing Setup

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() {
    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(fmt::layer().json())
        .init();
}
```

### 7.2 Keyward-Safe Redaction

All log output passes through a redaction layer that scrubs API key patterns:

```rust
/// Patterns to redact from log output.
const REDACT_PATTERNS: &[&str] = &[
    r"sk-ant-api\d{2}-[A-Za-z0-9_-]{20,}",  // Anthropic
    r"sk-[A-Za-z0-9]{20,}",                   // OpenAI
    r"AIzaSy[A-Za-z0-9_-]{20,}",              // Google
];

/// A tracing layer that scrubs sensitive patterns from log output.
pub struct RedactionLayer { /* compiled regex set */ }
```

### 7.3 Span Conventions

```rust
// Component-level span
#[tracing::instrument(skip_all, fields(component = "vault"))]
async fn handle_request(req: Envelope) -> Result<Envelope, VaultError> { ... }

// Per-tick span
tracing::info_span!("tick", cycle = %cycle, tick = %tick, agent = %agent_id.alias());
```

---

## 8. Common Enums & Types

```rust
/// Agent status lifecycle.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AgentStatus {
    Embryo,
    Active,
    Dormant,
    Dead,
}

/// Agent roles.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AgentRole {
    Generalist,
    CompilerSmith,
    Librarian,
    Architect,
    Explorer,
}

/// Proposal kinds for governance.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ProposalKind {
    SpawnAgent,
    KillAgent,
    ChangeParam,
    EpochAdvance,
    Custom,
}

/// Epoch identifiers.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum Epoch {
    Void       = 0,
    Spark      = 1,
    Foundation = 2,
    Commerce   = 3,
    Expansion  = 4,
    Sovereignty = 5,
    Open(u32),
}

/// Time constants.
pub const TICKS_PER_CYCLE: u64 = 256;
pub const BASE_TICK_BUDGET: u64 = 64;
pub const MAX_TICK_BUDGET: u64 = 256;
```

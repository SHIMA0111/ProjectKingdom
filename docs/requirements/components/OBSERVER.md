# Technology Requirements: Observer

> Rust backend (axum) + React frontend dashboard for human observation.
> Crate: `kingdom-observer`
> Frontend: `observer-ui/` (React 19 + Vite 6)
> Design reference: [10-OBSERVER.md](../../design/10-OBSERVER.md)

---

## 1. Purpose

Observer is the read-only human observation layer. It provides a web dashboard that presents the Kingdom's internal state as human-comprehensible visualizations. Observer has no agent identity, cannot write to the event bus, and has zero influence on Kingdom state. All semantic translation (converting agent communications into English) is delegated to Bridge; Observer handles structural presentation only.

---

## 2. Crate Dependencies

### 2.1 Backend (`kingdom-observer`)

```toml
[package]
name = "kingdom-observer"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Web framework
axum = { workspace = true }
axum-extra = { workspace = true }
tower = { workspace = true }
tower-http = { workspace = true }

# Serialization
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# Database
sqlx = { workspace = true }

# Logging
tracing = { workspace = true }
tracing-subscriber = { workspace = true }

# Utilities
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
tokio-tungstenite = "0.24"
```

### 2.2 Frontend (`observer-ui/`)

```json
{
  "name": "observer-ui",
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.0.0"
  },
  "devDependencies": {
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "typescript": "^5.6.0"
  }
}
```

---

## 3. Data Models

### 3.1 Backend Models

```rust
use kingdom_core::{AgentId, Hash256, Event, System, Epoch, AgentStatus, AgentRole};

/// Aggregated world overview snapshot.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WorldOverview {
    pub epoch: Epoch,
    pub cycle: u64,
    pub active_agents: u32,
    pub total_agents: u32,
    pub vault_repo_count: u64,
    pub oracle_entry_count: u64,
    pub circulating_currency: i64,
    pub open_bounties: u32,
    pub world_hash: Hash256,
}

/// Per-agent monitor view.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentMonitorView {
    pub id: AgentId,
    pub role: AgentRole,
    pub status: AgentStatus,
    pub balance: i64,
    pub reputation: f32,
    pub current_goal: Option<String>,
    pub current_task: Option<String>,
    pub ticks_used: u64,
    pub tick_budget: u64,
    pub vault_repos: Vec<Hash256>,
    pub recent_activity: Vec<ActivityEntry>,
    pub traits: AgentTraitsView,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentTraitsView {
    pub risk_tolerance: f32,
    pub collaboration: f32,
    pub depth_vs_breadth: f32,
    pub quality_vs_speed: f32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ActivityEntry {
    pub tick: u64,
    pub description: String,
    pub raw: Vec<u8>,
    pub translation: Option<String>,
    pub confidence: Option<f32>,
}

/// Timeline event for the chronological view.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TimelineEvent {
    pub id: i64,
    pub tick: u64,
    pub cycle: u64,
    pub epoch: Epoch,
    pub system: System,
    pub kind: u16,
    pub summary: String,
    pub raw_event: Vec<u8>,
    pub translation: Option<String>,
    pub agent_id: Option<AgentId>,
}

/// Metrics snapshot persisted periodically.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MetricsSnapshot {
    pub id: i64,
    pub cycle: u64,
    pub epoch: Epoch,
    pub agent_count: u32,
    pub vault_repos: u64,
    pub oracle_entries: u64,
    pub circulating_supply: i64,
    pub treasury: i64,
    pub gini_coefficient: f64,
    pub velocity: f64,
    pub open_bounties: u32,
    pub completed_bounties: u32,
    pub total_transactions: u64,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

/// Sponsor view (authenticated by sponsor UUID from Keyward).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SponsorView {
    pub sponsor_id: uuid::Uuid,
    pub provider: String,
    pub budget_total_usd: f64,
    pub budget_spent_usd: f64,
    pub budget_remaining_usd: f64,
    pub agents_powered: Vec<SponsorAgentView>,
    pub tier_distribution: TierDistribution,
    pub top_achievement: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SponsorAgentView {
    pub agent_id: AgentId,
    pub role: AgentRole,
    pub model_tier: String,
    pub cost_usd: f64,
    pub think_count: u64,
    pub contributions: ContributionSummary,
    pub reputation: f32,
    pub notable: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContributionSummary {
    pub commits: u64,
    pub reviews: u64,
    pub oracle_entries: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TierDistribution {
    pub tier1_pct: f32,
    pub tier2_pct: f32,
    pub tier3_pct: f32,
}

/// Dual display wrapper: raw original + Bridge translation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DualDisplay<T: Serialize> {
    /// The raw original content.
    pub raw: T,
    /// Bridge's English translation, if available.
    pub translation: Option<String>,
    /// Translation confidence score (0.0-1.0).
    pub confidence: Option<f32>,
    /// Translation method used.
    pub method: Option<TranslationMethod>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum TranslationMethod {
    Structural,
    Pattern,
    Llm,
    Cached,
}

/// WebSocket push message to the frontend.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum WsMessage {
    Event {
        event: TimelineEvent,
    },
    AgentUpdate {
        agent: AgentMonitorView,
    },
    MetricsUpdate {
        metrics: WorldOverview,
    },
    TranslationReady {
        event_id: i64,
        translation: String,
        confidence: f32,
    },
}

/// Data export formats.
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ExportFormat {
    /// JSON event stream, filtered by time/system/agent.
    JsonEventStream,
    /// CSV aggregated per-cycle metrics.
    CsvMetrics,
    /// GraphML for dependency and citation graphs.
    GraphMl,
    /// SQLite full world state snapshot at a cycle.
    SqliteSnapshot,
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{AgentId, Hash256, Event, EventFilter, EventBus, SubscriberMode, System, Epoch};

/// Observer-specific errors.
#[derive(Debug, thiserror::Error)]
pub enum ObserverError {
    #[error("event bus subscription failed: {0}")]
    SubscriptionFailed(String),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("bridge unavailable")]
    BridgeUnavailable,

    #[error("sponsor not found: {0}")]
    SponsorNotFound(uuid::Uuid),

    #[error("rate limit exceeded")]
    RateLimitExceeded,

    #[error("export error: {0}")]
    ExportError(String),

    #[error("serialization error: {0}")]
    Serialization(String),
}

impl From<ObserverError> for kingdom_core::KingdomError {
    fn from(e: ObserverError) -> Self { /* map to category + code */ }
}

/// The Observer backend interface.
#[async_trait::async_trait]
pub trait ObserverBackend: Send + Sync {
    /// Start the Observer: subscribe to event bus (Observer mode), begin aggregation.
    async fn start(&self, bus: Arc<dyn EventBus>) -> Result<(), ObserverError>;

    /// Stop the Observer and clean up subscriptions.
    async fn stop(&self) -> Result<(), ObserverError>;

    // --- View Generators ---

    /// World overview snapshot.
    async fn world_overview(&self) -> Result<WorldOverview, ObserverError>;

    /// Agent monitor view for a specific agent.
    async fn agent_monitor(&self, agent_id: &AgentId) -> Result<AgentMonitorView, ObserverError>;

    /// List all agents with their monitor views.
    async fn all_agents(&self) -> Result<Vec<AgentMonitorView>, ObserverError>;

    /// Timeline events, paginated.
    async fn timeline(
        &self,
        since_cycle: Option<u64>,
        limit: u32,
    ) -> Result<Vec<TimelineEvent>, ObserverError>;

    /// Sponsor view (authenticated by UUID).
    async fn sponsor_view(&self, sponsor_id: uuid::Uuid) -> Result<SponsorView, ObserverError>;

    /// Metrics history for charting.
    async fn metrics_history(
        &self,
        from_cycle: u64,
        to_cycle: u64,
    ) -> Result<Vec<MetricsSnapshot>, ObserverError>;

    // --- Data Export ---

    /// Export data in the given format.
    async fn export(
        &self,
        format: ExportFormat,
        filter: ExportFilter,
    ) -> Result<Vec<u8>, ObserverError>;

    // --- Bridge Integration ---

    /// Request a translation from Bridge for a piece of content.
    async fn request_translation(
        &self,
        content: Vec<u8>,
        source_agent: AgentId,
        system: System,
        context: Vec<u8>,
    ) -> Result<(), ObserverError>;
}

/// Filter for data export.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExportFilter {
    pub from_cycle: Option<u64>,
    pub to_cycle: Option<u64>,
    pub systems: Vec<System>,
    pub agents: Vec<AgentId>,
}
```

---

## 5. Inbound Messages

Observer operates as a read-only event bus subscriber (`SubscriberMode::Observer`). It does not receive directed KDOM messages; it passively taps the event stream.

| Source | Mechanism | Data |
|--------|-----------|------|
| Event Bus | `subscribe(filter, SubscriberMode::Observer)` | All `Event` variants from all systems. |
| Bridge | `TRANSLATE_RESULT` (0x0801) | Translation responses for requested content. |
| Bridge | `TRANSLATE_CACHE_HIT` (0x0802) | Cached translation results. |
| Bridge | `BRIDGE_STATUS` (0x0810) | Bridge operational status. |

---

## 6. Outbound Messages / Events

### 6.1 To Bridge

| Code | Name | Payload | Description |
|------|------|---------|-------------|
| `0x0800` | `TRANSLATE_REQUEST` | `{ content: bytes, source_agent: AgentId, system: System, context: bytes }` | Request translation of content for display. |

### 6.2 To Frontend (WebSocket)

| Message Type | Payload | Trigger |
|-------------|---------|---------|
| `Event` | `TimelineEvent` | New event on the bus. |
| `AgentUpdate` | `AgentMonitorView` | Agent status or state change. |
| `MetricsUpdate` | `WorldOverview` | Cycle boundary metrics refresh. |
| `TranslationReady` | `{ event_id, translation, confidence }` | Bridge returns a translation. |

### 6.3 Non-Interference Guarantee

Observer enforces the following invariants:

- **No agent identity**: Observer has no `Keypair`, cannot sign events.
- **Observer subscription mode**: Event bus subscription is flagged `SubscriberMode::Observer`, which prevents `publish()`.
- **No I/O channels**: No Forge sandbox, no Mint account, no Agora channels.
- **Invisible to agents**: Agents are unaware Observer exists.

---

## 7. REST API Endpoints

```
GET  /api/world                           -> WorldOverview
GET  /api/agents                          -> Vec<AgentMonitorView>
GET  /api/agents/:id                      -> AgentMonitorView
GET  /api/vault/repos                     -> Vec<RepoSummary>
GET  /api/vault/repos/:id                 -> RepoDetail
GET  /api/vault/repos/:id/snaps           -> Vec<SnapSummary>
GET  /api/vault/repos/:id/tree/:snap_id   -> TreeView
GET  /api/vault/object/:id                -> ObjectView (with DualDisplay)
GET  /api/agora/channels                  -> Vec<ChannelSummary>
GET  /api/agora/channels/:id/messages     -> Vec<DualDisplay<Message>>
GET  /api/agora/bounties                  -> Vec<BountySummary>
GET  /api/oracle/entries                  -> Vec<DualDisplay<OracleEntry>>
GET  /api/oracle/entries/:id              -> DualDisplay<OracleEntry>
GET  /api/oracle/citations                -> GraphData
GET  /api/economy                         -> EconomyDashboard
GET  /api/economy/transactions            -> Vec<DualDisplay<Transaction>>
GET  /api/forge/sandboxes                 -> Vec<SandboxSummary>
GET  /api/forge/services                  -> Vec<ServiceSummary>
GET  /api/timeline                        -> Vec<TimelineEvent> (paginated)
GET  /api/dependencies                    -> GraphData (GraphML or JSON)
GET  /api/metrics?from=N&to=M            -> Vec<MetricsSnapshot>

GET  /api/sponsor/:uuid                   -> SponsorView (auth by UUID)

GET  /api/export/events?format=json&...   -> JSON event stream
GET  /api/export/metrics?format=csv&...   -> CSV metrics
GET  /api/export/graph?format=graphml&... -> GraphML graph
GET  /api/export/snapshot?cycle=N         -> SQLite snapshot

WS   /ws                                  -> WebSocket real-time event push
```

### 7.1 Sponsor View Authentication

```
GET /api/sponsor/:sponsor_uuid

Authentication: knowing the UUID = authentication (UUID v7 has sufficient
randomness). No additional auth mechanism.

Rate limiting:
  - 60 requests/min per IP
  - 5 consecutive invalid UUIDs -> temporary IP block (15 minutes)
```

### 7.2 Static File Serving

```rust
// Observer serves the React frontend from OBSERVER_UI_DIR
// Default: ./observer-ui/dist
// Configured via OBSERVER_UI_DIR environment variable

// Fallback: all non-API routes serve index.html (SPA routing)
```

---

## 8. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| REST API response (cached views) | < 50 ms | Pre-aggregated materialized views |
| REST API response (live query) | < 200 ms | Direct PostgreSQL query |
| WebSocket push latency | < 100 ms | From event bus receipt to frontend delivery |
| Metrics snapshot persist | < 100 ms | Batched at cycle boundary |
| Timeline event ingest | < 10 ms per event | Append-only write |
| Concurrent WebSocket connections | 100+ | Tokio broadcast channels |
| Data export (JSON, 10k events) | < 5 s | Streaming response |
| Data export (SQLite snapshot) | < 30 s | Full world state serialization |
| Static file serving | < 5 ms | Pre-built React bundle from disk |

---

## 9. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `kingdom-core` | Crate | Shared types, event bus, wire protocol |
| Event Bus | Runtime | Read-only subscription to all system events |
| Bridge | Runtime | Translation requests and results |
| PostgreSQL | Storage | Metrics snapshots, timeline events (materialized views or owned tables) |
| React frontend | Build artifact | Static files served from `OBSERVER_UI_DIR` |

---

## 10. PostgreSQL Schema

Tables reside in the `observer` schema. These may be materialized views over other schemas or owned tables populated by the Observer's event collector.

```sql
CREATE SCHEMA IF NOT EXISTS observer;

-- Periodic metrics snapshots (one row per cycle)
CREATE TABLE observer.metrics_snapshots (
    id                  BIGSERIAL PRIMARY KEY,
    cycle               BIGINT NOT NULL UNIQUE,
    epoch               SMALLINT NOT NULL,
    agent_count         INTEGER NOT NULL,
    vault_repos         BIGINT NOT NULL,
    oracle_entries      BIGINT NOT NULL,
    circulating_supply  BIGINT NOT NULL,
    treasury            BIGINT NOT NULL,
    gini_coefficient    DOUBLE PRECISION NOT NULL,
    velocity            DOUBLE PRECISION NOT NULL,
    open_bounties       INTEGER NOT NULL,
    completed_bounties  INTEGER NOT NULL,
    total_transactions  BIGINT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_metrics_cycle ON observer.metrics_snapshots(cycle);

-- Timeline events (significant events for chronological display)
CREATE TABLE observer.timeline_events (
    id              BIGSERIAL PRIMARY KEY,
    tick            BIGINT NOT NULL,
    cycle           BIGINT NOT NULL,
    epoch           SMALLINT NOT NULL,
    system          SMALLINT NOT NULL,          -- System enum ordinal
    kind            SMALLINT NOT NULL,          -- event kind code
    summary         TEXT NOT NULL,              -- human-readable summary (from Bridge or structural decode)
    raw_event       BYTEA NOT NULL,             -- MessagePack-encoded original Event
    translation     TEXT,                       -- Bridge translation, if available
    confidence      REAL,                       -- translation confidence
    agent_id        BYTEA,                      -- originating agent (nullable for system events)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timeline_cycle ON observer.timeline_events(cycle);
CREATE INDEX idx_timeline_system ON observer.timeline_events(system);
CREATE INDEX idx_timeline_agent ON observer.timeline_events(agent_id) WHERE agent_id IS NOT NULL;
```

---

## 11. Key Algorithms

### 11.1 Event Collection and Aggregation

```
1. Subscribe to event bus with SubscriberMode::Observer and
   EventFilter { systems: [], kinds: [], origins: [] } (all events).

2. For each received event:
   a. Classify significance (CRITICAL / NOTABLE / ROUTINE).
   b. If CRITICAL or NOTABLE, insert into observer.timeline_events.
   c. Update in-memory aggregated state (agent counts, balances, etc.).
   d. If content requires translation, send TRANSLATE_REQUEST to Bridge.
   e. Push WsMessage::Event to all connected WebSocket clients.

3. At each cycle boundary:
   a. Compute MetricsSnapshot from aggregated state.
   b. Insert into observer.metrics_snapshots.
   c. Push WsMessage::MetricsUpdate to all WebSocket clients.
```

### 11.2 Dual Display Assembly

```
For every item displayed in the dashboard:

1. Retrieve the raw original content.
2. Check if a Bridge translation is already cached (in-memory or DB).
3. If cached: assemble DualDisplay { raw, translation, confidence, method }.
4. If not cached: display raw immediately, send TRANSLATE_REQUEST to Bridge.
5. When TRANSLATE_RESULT arrives:
   a. Update the stored translation.
   b. Push WsMessage::TranslationReady to all WebSocket clients.
   c. Frontend updates the DualDisplay in-place.
```

### 11.3 WebSocket Connection Management

```
1. Client connects to /ws.
2. Server creates a new broadcast receiver from the shared sender.
3. Server spawns a task that:
   a. Reads from the broadcast receiver.
   b. Serializes WsMessage to JSON.
   c. Sends over the WebSocket.
4. On disconnect, the receiver is dropped (automatic cleanup).
```

### 11.4 Sponsor View Rate Limiting

```
Per-IP state:
  - request_count: sliding window counter (60 req/min)
  - invalid_uuid_streak: counter of consecutive invalid UUIDs

On request to /api/sponsor/:uuid:
  1. Check IP is not blocked.
  2. Increment request_count. If > 60/min, return 429.
  3. Validate UUID against Keyward's sponsor list.
  4. If invalid: increment invalid_uuid_streak.
     If streak >= 5: block IP for 15 minutes.
  5. If valid: reset invalid_uuid_streak. Return SponsorView.
```

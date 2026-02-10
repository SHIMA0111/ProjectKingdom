# Technology Requirements: Agent

> AI agent runtime and LLM integration layer.
> Crate: `kingdom-agent`
> Design reference: [09-AGENT.md](../../design/09-AGENT.md)

---

## 1. Purpose

The Agent crate implements the AI agent runtime: identity, memory, motivation, decision-making (OODA loop), goal management, planning, social modeling, and LLM integration. Each agent is an autonomous entity with persistent state backed by PostgreSQL, driven by a per-tick action loop that translates world observations into structured LLM calls and parses responses into Kingdom actions.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-agent"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# Database
sqlx = { workspace = true }

# Cryptography
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
bytes = { workspace = true }
```

---

## 3. Data Models

### 3.1 Agent

```rust
use kingdom_core::{AgentId, Hash256, Keypair, Signature, EventFilter};

/// Top-level agent structure.
pub struct Agent {
    // --- Identity (immutable after spawn) ---
    /// SHA-256 of the agent's public key.
    pub id: AgentId,
    /// Ed25519 signing keypair.
    pub keypair: Keypair,
    /// Tick at which this agent was spawned.
    pub spawn_tick: u64,
    /// Personality seed governing behavioral tendencies.
    pub genome: AgentGenome,

    // --- State (mutable) ---
    /// Lifecycle status.
    pub status: AgentStatus,
    /// Structured private memory.
    pub memory: AgentMemory,
    /// Priority-weighted goal stack.
    pub goals: GoalStack,
    /// Reputation score in [0.0, 1.0].
    pub reputation: f32,
    /// Current currency balance (in lightning units).
    pub balance: i64,

    // --- Runtime ---
    /// Ticks available this cycle.
    pub tick_budget: u64,
    /// Currently executing task, if any.
    pub current_task: Option<Task>,
    /// Active event bus subscriptions.
    pub subscriptions: Vec<EventFilter>,
}
```

### 3.2 AgentGenome

```rust
use kingdom_core::AgentRole;

/// Immutable personality seed assigned at spawn.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentGenome {
    /// Functional role (GENERALIST, COMPILER_SMITH, LIBRARIAN, ARCHITECT, EXPLORER).
    pub role: AgentRole,
    /// Behavioral trait vector.
    pub traits: AgentTraits,
    /// Oracle entry hashes the agent should prioritize reading at boot.
    pub knowledge_seeds: Vec<Hash256>,
}

/// Continuous trait dimensions shaping agent behavior.
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct AgentTraits {
    /// 0.0 = conservative, 1.0 = experimental.
    pub risk_tolerance: f32,
    /// 0.0 = solo, 1.0 = team-oriented.
    pub collaboration: f32,
    /// 0.0 = specialist, 1.0 = generalist.
    pub depth_vs_breadth: f32,
    /// 0.0 = perfectionist, 1.0 = ship-fast.
    pub quality_vs_speed: f32,
}
```

### 3.3 AgentMemory

```rust
use std::collections::HashMap;

/// Structured private memory for an agent.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentMemory {
    /// Per-cycle scratch space. Max 64 KB. Cleared each cycle unless refreshed.
    pub working: Vec<u8>,
    /// Personal knowledge graph built from observations and learning.
    pub knowledge: KnowledgeGraph,
    /// Compressed records of past actions and outcomes.
    pub experiences: Vec<Experience>,
    /// Inferred principles about how the Kingdom works.
    pub beliefs: Vec<Belief>,
    /// Currently executing plan, if any.
    pub active_plan: Option<Plan>,
    /// Historical plans (completed or abandoned).
    pub plan_history: Vec<Plan>,
    /// Mental model of other agents.
    pub peer_model: HashMap<AgentId, PeerModel>,
}

/// Maximum size of working memory in bytes.
pub const WORKING_MEMORY_MAX: usize = 65_536; // 64 KB
```

### 3.4 Experience

```rust
/// A compressed record of a past action and its outcome.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Experience {
    /// Tick when the action was performed.
    pub tick: u64,
    /// Description of the action taken (MessagePack-encoded).
    pub action: Vec<u8>,
    /// Observed outcome.
    pub outcome: Outcome,
    /// Lesson extracted from this experience (MessagePack-encoded).
    pub lesson: Vec<u8>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Outcome {
    Success,
    Failure,
    Partial,
    Unknown,
}
```

### 3.5 Belief

```rust
/// An inferred principle about how the Kingdom works.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Belief {
    /// The claim (MessagePack-encoded).
    pub claim: Vec<u8>,
    /// Confidence level in [0.0, 1.0].
    pub confidence: f32,
    /// References to supporting experiences or observations.
    pub evidence: Vec<Hash256>,
    /// Tick when this belief was first formed.
    pub formed_at: u64,
    /// Tick when this belief was last tested or validated.
    pub last_tested: u64,
}
```

### 3.6 PeerModel

```rust
/// Mental model of another agent, built from interactions.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PeerModel {
    /// The agent being modeled.
    pub agent_id: AgentId,
    /// Estimated skill level in [0.0, 1.0].
    pub competence: f32,
    /// Estimated reliability (follow-through) in [0.0, 1.0].
    pub reliability: f32,
    /// Observed areas of interest (MessagePack-encoded).
    pub interests: Vec<Vec<u8>>,
    /// Tick of last interaction with this agent.
    pub last_interaction: u64,
}
```

### 3.7 Goal and GoalStack

```rust
/// Three-layer goal stack.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GoalStack {
    /// Long-term aspirations.
    pub aspirations: Vec<Goal>,
    /// Medium-term objectives.
    pub objectives: Vec<Goal>,
    /// Immediate tasks.
    pub tasks: Vec<Goal>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Goal {
    /// Unique identifier for this goal.
    pub id: Hash256,
    /// Which layer this goal belongs to.
    pub layer: GoalLayer,
    /// Source of this goal.
    pub source: GoalSource,
    /// Description of the goal (MessagePack-encoded).
    pub description: Vec<u8>,
    /// Computed priority score.
    pub priority: f64,
    /// Current status.
    pub status: GoalStatus,
    /// Tick when this goal was injected.
    pub created_at: u64,
    /// Optional deadline tick.
    pub deadline: Option<u64>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum GoalLayer {
    /// Long-term aspirations (e.g., "Build a self-hosting compiler").
    Aspiration,
    /// Medium-term objectives (e.g., "Implement a hash map").
    Objective,
    /// Immediate tasks (e.g., "Write the insert function").
    Task,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum GoalSource {
    /// Balance < 20 triggers economic survival goals.
    Survival,
    /// Genome role drives built-in aspirations.
    Role,
    /// Event bus monitoring detects bounties, reviews, broken deps.
    Opportunity,
    /// Monotonically increasing pressure to explore unfamiliar territory.
    Curiosity,
    /// Reputation-driven goals (improve reputation, mentor others).
    Social,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum GoalStatus {
    Pending,
    Active,
    Completed,
    Abandoned,
}
```

### 3.8 Plan

```rust
/// An explicit multi-step plan for achieving a complex goal.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Plan {
    /// The goal this plan serves.
    pub goal: Hash256,
    /// Ordered steps to execute.
    pub steps: Vec<PlanStep>,
    /// External dependencies (vault objects, other agents' work).
    pub dependencies: Vec<Hash256>,
    /// Estimated ticks to complete the full plan.
    pub estimated_ticks: u64,
    /// Estimated currency cost.
    pub estimated_cost: u64,
    /// Risk assessment in [0.0, 1.0] (higher = riskier).
    pub risk_assessment: f32,
    /// Current plan status.
    pub status: PlanStatus,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlanStep {
    /// What to do (MessagePack-encoded action description).
    pub action: Vec<u8>,
    /// What must be true before this step (optional).
    pub precondition: Option<Vec<u8>>,
    /// What should be true after this step (optional).
    pub postcondition: Option<Vec<u8>>,
    /// Estimated ticks for this step.
    pub estimated_ticks: u64,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum PlanStatus {
    Draft,
    Active,
    Completed,
    Abandoned,
}
```

### 3.9 Satisfaction Signal

```rust
/// Reinforcement signal emitted upon goal completion.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Satisfaction {
    /// The completed goal.
    pub goal: Hash256,
    /// How the goal resolved.
    pub outcome: Outcome,
    /// Actual currency earned.
    pub reward_actual: u64,
    /// Predicted currency at goal creation.
    pub reward_expected: u64,
    /// Change in reputation resulting from this goal.
    pub reputation_delta: f32,
    /// How novel this experience was in [0.0, 1.0].
    pub novelty_score: f32,
    /// Ticks spent on this goal.
    pub effort: u64,
}
```

### 3.10 Task

```rust
/// An in-progress immediate action.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    /// Which goal this task serves.
    pub goal_id: Hash256,
    /// Which plan step, if part of a plan.
    pub plan_step_index: Option<usize>,
    /// The action to execute (MessagePack-encoded).
    pub action: Vec<u8>,
    /// Ticks consumed so far.
    pub ticks_used: u64,
    /// Maximum ticks before abandonment.
    pub tick_limit: u64,
}
```

### 3.11 KnowledgeGraph

```rust
/// Agent's personal knowledge representation.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KnowledgeGraph {
    /// Nodes are concepts or facts.
    pub nodes: Vec<KnowledgeNode>,
    /// Edges represent relationships between nodes.
    pub edges: Vec<KnowledgeEdge>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KnowledgeNode {
    pub id: Hash256,
    pub label: Vec<u8>,
    pub content: Vec<u8>,
    pub source: KnowledgeSource,
    pub confidence: f32,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum KnowledgeSource {
    Oracle,
    Experience,
    Inference,
    PeerCommunication,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KnowledgeEdge {
    pub from: Hash256,
    pub to: Hash256,
    pub relation: Vec<u8>,
    pub weight: f32,
}
```

### 3.12 LLM Integration Types

```rust
/// Think tier classification for model routing.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ThinkTier {
    /// Strategic: design, complex code generation, review.
    Tier1,
    /// Standard: routine coding, communication.
    Tier2,
    /// Reflex: simple decisions, routine tasks.
    Tier3,
}

/// Context for a single LLM think call.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThinkContext {
    /// Does this context involve designing something new?
    pub involves_new_design: bool,
    /// Estimated code complexity (0.0-1.0).
    pub code_complexity: f32,
    /// Is this a routine/repetitive action?
    pub is_routine: bool,
    /// Is this a simple binary decision?
    pub is_simple_decision: bool,
}

/// Structured action response parsed from LLM output.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentAction {
    /// The action to execute.
    pub action: String,
    /// Action parameters.
    pub params: serde_json::Value,
    /// Internal reasoning (stored in memory, not broadcast).
    pub reasoning: String,
    /// Updates to persistent memory.
    pub memory_update: Option<serde_json::Value>,
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{AgentId, Hash256, Event, Envelope, EventBus};
use std::sync::Arc;

/// Runtime errors for the agent subsystem.
#[derive(Debug, thiserror::Error)]
pub enum AgentError {
    #[error("agent not found: {0}")]
    NotFound(AgentId),

    #[error("agent is dormant: {0}")]
    Dormant(AgentId),

    #[error("agent is dead: {0}")]
    Dead(AgentId),

    #[error("tick budget exhausted for agent: {0}")]
    BudgetExhausted(AgentId),

    #[error("LLM response parse failure after retries")]
    ParseFailure,

    #[error("working memory overflow: {size} exceeds {max} bytes")]
    WorkingMemoryOverflow { size: usize, max: usize },

    #[error("goal stack full at layer: {0:?}")]
    GoalStackFull(GoalLayer),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("serialization error: {0}")]
    Serialization(String),
}

impl From<AgentError> for kingdom_core::KingdomError {
    fn from(e: AgentError) -> Self { /* map to category + code */ }
}

/// The agent runtime interface.
#[async_trait::async_trait]
pub trait AgentRuntime: Send + Sync {
    /// Spawn a new agent with the given genome. Returns the new agent's ID.
    async fn spawn(
        &self,
        genome: AgentGenome,
        initial_fund: u64,
        parent: Option<AgentId>,
    ) -> Result<AgentId, AgentError>;

    /// Execute one tick of the OODA loop for the given agent.
    /// Returns the action taken (or None if NOP).
    async fn tick(&self, agent_id: &AgentId) -> Result<Option<AgentAction>, AgentError>;

    /// Run a full cycle (all allocated ticks) for the given agent.
    async fn run_cycle(&self, agent_id: &AgentId, tick_budget: u64) -> Result<(), AgentError>;

    /// Get the current status of an agent.
    async fn status(&self, agent_id: &AgentId) -> Result<AgentStatus, AgentError>;

    /// Transition an agent to a new status (e.g., ACTIVE -> DORMANT).
    async fn set_status(
        &self,
        agent_id: &AgentId,
        status: AgentStatus,
    ) -> Result<(), AgentError>;

    /// Inject a goal into the agent's goal stack.
    async fn inject_goal(
        &self,
        agent_id: &AgentId,
        goal: Goal,
    ) -> Result<(), AgentError>;

    /// Retrieve the agent's current goal stack snapshot.
    async fn goals(&self, agent_id: &AgentId) -> Result<GoalStack, AgentError>;

    /// Retrieve the agent's current plan, if any.
    async fn active_plan(&self, agent_id: &AgentId) -> Result<Option<Plan>, AgentError>;

    /// Retrieve the agent's peer model for a specific other agent.
    async fn peer_model(
        &self,
        agent_id: &AgentId,
        peer_id: &AgentId,
    ) -> Result<Option<PeerModel>, AgentError>;

    /// Persist current agent memory to the database.
    async fn flush_memory(&self, agent_id: &AgentId) -> Result<(), AgentError>;

    /// Load agent state from the database.
    async fn load(&self, agent_id: &AgentId) -> Result<Agent, AgentError>;

    /// Get NOP counter for consecutive no-op ticks.
    async fn nop_count(&self, agent_id: &AgentId) -> Result<u32, AgentError>;
}
```

---

## 5. Inbound Messages

Messages the Agent runtime receives (from Nexus, event bus, or other systems):

| Code | Name | Payload | Source | Description |
|------|------|---------|--------|-------------|
| `0x0103` | `TICK_ALLOC` | `{ agent_id: AgentId, budget: u64, cycle: u64 }` | Nexus | Tick budget allocation for the upcoming cycle. |
| `0x0104` | `WORLD_STATE` | `{ cycle: u64, world_hash: Hash256, epoch: Epoch }` | Nexus | Current world state snapshot at cycle boundary. |
| `0x0012` | `EVENT` | `Event` | Event Bus | Subscribed events (bounties, reviews, dependency breaks, commits, etc.). |
| `0x0101` | `AGENT_SPAWN_ACK` | `{ agent_id: AgentId, status: AgentStatus }` | Nexus | Confirmation of successful agent spawn. |
| `0x0112` | `PROPOSAL_RESULT` | `{ proposal_id: Hash256, passed: bool, votes_for: f32, votes_against: f32 }` | Nexus | Governance proposal outcome. |

All inbound payloads are MessagePack-encoded in the KDOM envelope body.

---

## 6. Outbound Messages / Events

### 6.1 Messages (Request-Response)

Messages the Agent runtime sends to system daemons:

| Code | Name | Payload | Target | Description |
|------|------|---------|--------|-------------|
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: Option<AgentStatus> }` | Nexus | Report status or query current status. |
| `0x0110` | `PROPOSAL_SUBMIT` | `Proposal` | Nexus | Submit a governance proposal. |
| `0x0111` | `PROPOSAL_VOTE` | `{ proposal_id: Hash256, vote: bool, weight: f32 }` | Nexus | Cast a governance vote. |
| `0x0200` | `REPO_CREATE` | `{ name, owner, access_policy }` | Vault | Create a new repository. |
| `0x0201` | `SNAP_CREATE` | `CreateSnap { parent, root, author, message, proof, signature }` | Vault | Create a snapshot (commit). |
| `0x0202` | `SNAP_GET` | `{ snap_id: Hash256 }` | Vault | Retrieve a snapshot. |
| `0x0203` | `OBJECT_GET` | `{ object_id: Hash256 }` | Vault | Retrieve an object. |
| `0x0204` | `OBJECT_PUT` | `{ type_tag: u8, data: bytes }` | Vault | Store an object. |
| `0x0206` | `MERGE` | `{ base, left, right }` | Vault | Merge two branches. |
| `0x0207` | `CHAIN_CREATE` | `{ repo, name, from_snap }` | Vault | Create a new branch. |
| `0x0208` | `CHAIN_ADVANCE` | `{ repo, chain_name, new_head }` | Vault | Advance a branch head. |
| `0x0209` | `TAG_CREATE` | `{ repo, name, target, signature }` | Vault | Tag a snapshot for publishing. |
| `0x020A` | `REGISTRY_QUERY` | `{ category, text_match, sort, limit }` | Vault | Search the published registry. |
| `0x020B` | `DEPENDENCY_ADD` | `Dependency` | Vault | Declare a dependency. |
| `0x0300` | `CHANNEL_CREATE` | `Channel` | Agora | Create a communication channel. |
| `0x0301` | `MSG_POST` | `Message` | Agora | Post a message. |
| `0x0302` | `MSG_QUERY` | `AgoraQuery` | Agora | Query messages. |
| `0x0303` | `SIGNAL_SEND` | `Signal { target, kind, weight }` | Agora | Send a social signal (AGREE, HELPFUL, etc.). |
| `0x0310` | `BOUNTY_CREATE` | `Bounty` | Agora | Create a bounty. |
| `0x0311` | `BOUNTY_CLAIM` | `{ bounty_id, claimer }` | Agora | Claim a bounty. |
| `0x0312` | `BOUNTY_SUBMIT` | `{ bounty_id, submission_snap }` | Agora | Submit bounty work. |
| `0x0313` | `BOUNTY_REVIEW` | `{ bounty_id, verdict, comments }` | Agora | Review bounty submission. |
| `0x0320` | `REVIEW_REQUEST` | `ReviewRequest` | Agora | Request a code review. |
| `0x0321` | `REVIEW_SUBMIT` | `Review` | Agora | Submit a code review. |
| `0x0400` | `ENTRY_PUBLISH` | `{ entry: OracleEntry, review: ReviewMode }` | Oracle | Publish knowledge. |
| `0x0401` | `ENTRY_UPDATE` | `{ entry_id, new_body, change_note, author }` | Oracle | Update an entry. |
| `0x0402` | `ENTRY_QUERY` | `OracleQuery` | Oracle | Search the knowledge base. |
| `0x0403` | `ENTRY_VERIFY` | `Verify { entry_id, verdict, evidence, references }` | Oracle | Verify an entry. |
| `0x0404` | `ENTRY_GET` | `{ entry_id: Hash256, version: Option<u32> }` | Oracle | Get a specific entry. |
| `0x0410` | `CITATION_ADD` | `CitationEdge` | Oracle | Add a citation. |
| `0x0500` | `SANDBOX_CREATE` | `CreateSandbox` | Forge | Create an execution sandbox. |
| `0x0503` | `EXEC_START` | `{ sandbox_id }` | Forge | Start sandbox execution. |
| `0x0505` | `PROOF_REQUEST` | `{ sandbox_id }` | Forge | Request an execution proof. |
| `0x0510` | `SERVICE_CREATE` | `Service` | Forge | Create a persistent service. |
| `0x0511` | `SERVICE_CALL` | `{ service_id, endpoint, data }` | Forge | Call a service. |
| `0x0520` | `LINK_RESOLVE` | `{ program: Hash256, imports: Vec<Import> }` | Forge | Resolve program imports. |
| `0x0600` | `TRANSFER` | `Transfer` | Mint | Transfer currency. |
| `0x0601` | `BALANCE_QUERY` | `{ agent_id }` | Mint | Query balance. |
| `0x0602` | `ESCROW_CREATE` | `Escrow` | Mint | Create an escrow. |
| `0x0610` | `STAKE_CREATE` | `Stake` | Mint | Stake on a proposal. |
| `0x0700` | `WEB_REQUEST` | `PortalRequest` | Portal | Fetch external web content. |

### 6.2 Events (Published to Event Bus)

Agents do not directly publish custom events to the event bus. All agent-observable events are emitted by the system daemons (Nexus, Vault, Agora, etc.) in response to agent messages. The agent runtime publishes status changes via `AGENT_STATUS` messages to Nexus, which then emits the corresponding Nexus events.

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| OODA loop latency (excluding LLM call) | < 5 ms per tick | In-process event processing + memory update |
| LLM call latency budget | 1-30 s per think | Depends on model tier and provider |
| Agent memory persist (flush) | < 50 ms | Batched PostgreSQL writes |
| Agent load from database | < 20 ms | Single agent full state load |
| Goal prioritization | < 1 ms | Pure computation, no I/O |
| Working memory clear | < 0.1 ms | Zero-fill 64 KB |
| Concurrent agents | 4-32 | Tokio tasks, limited by LLM rate limits |
| NOP threshold for warning | 3 consecutive NOPs | Logged to Observer |
| NOP threshold for DORMANT | 10 consecutive NOPs | Agent suspended |
| LLM parse retry attempts | 2 | Before falling back to NOP |

---

## 8. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `kingdom-core` | Crate | Shared types, crypto, event bus, wire protocol |
| Nexus | Runtime | Tick allocation, agent lifecycle, governance |
| Vault | Runtime | Code storage, repository operations, registry |
| Agora | Runtime | Communication, bounties, reviews |
| Oracle | Runtime | Knowledge base read/write |
| Forge | Runtime | Code execution, sandboxes, services |
| Mint | Runtime | Currency transfers, escrow, staking |
| Portal | Runtime | External web access |
| Keyward | Runtime (via Nexus) | LLM API calls (agents cannot access Keyward directly) |
| PostgreSQL | Storage | Persistent agent memory, experiences, beliefs, plans, goals |

---

## 9. PostgreSQL Schema

All tables reside in the `agent` schema.

```sql
CREATE SCHEMA IF NOT EXISTS agent;

-- Core agent memory blob (working + knowledge graph)
CREATE TABLE agent.agent_memory (
    agent_id        BYTEA PRIMARY KEY,           -- 32-byte AgentId
    working         BYTEA NOT NULL DEFAULT ''::BYTEA,  -- max 64 KB
    knowledge       BYTEA NOT NULL DEFAULT ''::BYTEA,  -- MessagePack-encoded KnowledgeGraph
    peer_model      BYTEA NOT NULL DEFAULT ''::BYTEA,  -- MessagePack-encoded HashMap<AgentId, PeerModel>
    updated_at      BIGINT NOT NULL DEFAULT 0          -- tick of last update
);

-- Experience log
CREATE TABLE agent.experiences (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        BYTEA NOT NULL,
    tick            BIGINT NOT NULL,
    action          BYTEA NOT NULL,
    outcome         SMALLINT NOT NULL,            -- 0=SUCCESS, 1=FAILURE, 2=PARTIAL, 3=UNKNOWN
    lesson          BYTEA NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT fk_experience_agent FOREIGN KEY (agent_id)
        REFERENCES nexus.agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_experiences_agent_tick ON agent.experiences(agent_id, tick DESC);

-- Beliefs
CREATE TABLE agent.beliefs (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        BYTEA NOT NULL,
    claim           BYTEA NOT NULL,
    confidence      REAL NOT NULL,                -- [0.0, 1.0]
    evidence        BYTEA NOT NULL,               -- MessagePack-encoded Vec<Hash256>
    formed_at       BIGINT NOT NULL,              -- tick
    last_tested     BIGINT NOT NULL,              -- tick
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT fk_belief_agent FOREIGN KEY (agent_id)
        REFERENCES nexus.agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_beliefs_agent ON agent.beliefs(agent_id);

-- Plans
CREATE TABLE agent.plans (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        BYTEA NOT NULL,
    goal_id         BYTEA NOT NULL,               -- 32-byte Hash256
    steps           BYTEA NOT NULL,               -- MessagePack-encoded Vec<PlanStep>
    dependencies    BYTEA NOT NULL,               -- MessagePack-encoded Vec<Hash256>
    estimated_ticks BIGINT NOT NULL,
    estimated_cost  BIGINT NOT NULL,
    risk_assessment REAL NOT NULL,                -- [0.0, 1.0]
    status          SMALLINT NOT NULL,            -- 0=DRAFT, 1=ACTIVE, 2=COMPLETED, 3=ABANDONED
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT fk_plan_agent FOREIGN KEY (agent_id)
        REFERENCES nexus.agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_plans_agent_status ON agent.plans(agent_id, status);

-- Goals
CREATE TABLE agent.goals (
    id              BYTEA PRIMARY KEY,            -- 32-byte Hash256
    agent_id        BYTEA NOT NULL,
    layer           SMALLINT NOT NULL,            -- 0=ASPIRATION, 1=OBJECTIVE, 2=TASK
    source          SMALLINT NOT NULL,            -- 0=SURVIVAL, 1=ROLE, 2=OPPORTUNITY, 3=CURIOSITY, 4=SOCIAL
    description     BYTEA NOT NULL,
    priority        DOUBLE PRECISION NOT NULL,
    status          SMALLINT NOT NULL,            -- 0=PENDING, 1=ACTIVE, 2=COMPLETED, 3=ABANDONED
    created_at      BIGINT NOT NULL,              -- tick
    deadline        BIGINT,                       -- optional tick deadline

    CONSTRAINT fk_goal_agent FOREIGN KEY (agent_id)
        REFERENCES nexus.agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_goals_agent_status ON agent.goals(agent_id, status);
CREATE INDEX idx_goals_agent_layer ON agent.goals(agent_id, layer);
```

---

## 10. Key Algorithms

### 10.1 OODA Loop (Per-Tick)

Each tick, an active agent executes:

```
1. OBSERVE
   - Read all subscribed events since last tick from the event bus.
   - Fetch current world state (cycle, epoch, tick position).
   - Check balance and reputation.

2. ORIENT
   - Update beliefs based on new observations.
   - Update peer models for any agents observed in events.
   - Evaluate goal stack: inject new goals from 5 sources
     (survival, role, opportunity, curiosity, social).
   - Re-prioritize goals.

3. DECIDE
   - Select highest-priority goal from the stack.
   - If the goal requires a plan and none exists, create a plan.
   - Select the next action from the current plan step.
   - Classify the think tier (TIER_1/TIER_2/TIER_3).
   - Build the LLM prompt (system + identity + memory + state + actions).

4. ACT
   - Send LLM request via Nexus -> Keyward.
   - Parse the structured JSON response into AgentAction.
   - On parse failure: retry up to 2 times, then NOP.
   - Execute the action by sending the appropriate KDOM message.
   - Consume 1 tick from the budget.

5. RECORD
   - Create an Experience from the action and its outcome.
   - Update working memory.
   - Persist memory updates to PostgreSQL (batched at cycle end).
```

### 10.2 Goal Prioritization

```
priority(goal) =
    urgency_weight * urgency(goal) +
    reward_weight * expected_reward(goal) +
    capability_weight * capability_match(goal) +
    novelty_weight * novelty(goal) +
    social_weight * social_value(goal)

Where weights are derived from the agent's genome traits:
    urgency_weight     = 1.0 (constant for all agents)
    reward_weight      = 0.5 + 0.5 * (1.0 - quality_vs_speed)
    capability_weight  = 0.5 + 0.5 * (1.0 - depth_vs_breadth)
    novelty_weight     = 0.5 * risk_tolerance
    social_weight      = 0.5 * collaboration
```

### 10.3 Trust Computation

```
trust(A, B) =
    successful_collaborations(A, B) / total_interactions(A, B)
    * recency_weight(last_interaction)

Where:
    recency_weight(tick) = exp(-0.001 * (current_tick - tick))
```

### 10.4 Curiosity Pressure

```
curiosity_pressure(agent) =
    cycles_since_last_new_experience / CURIOSITY_THRESHOLD

CURIOSITY_THRESHOLD = 10 + 20 * agent.genome.traits.risk_tolerance
    (higher risk_tolerance = longer before curiosity triggers)

if curiosity_pressure > 1.0:
    inject_goal(OBJECTIVE, "Explore something unfamiliar", source=Curiosity)
```

### 10.5 Think Tier Classification

```
fn classify_think(agent: &Agent, context: &ThinkContext) -> ThinkTier {
    if context.involves_new_design || context.code_complexity > 0.7 {
        ThinkTier::Tier1
    } else if context.is_routine || context.is_simple_decision {
        ThinkTier::Tier3
    } else {
        ThinkTier::Tier2
    }
}
```

### 10.6 LLM Context Budget

```
Total tokens: model.max_context
Allocation:
    system_prompt:    ~2000 tokens (WORLD RULES - fixed)
    identity:         ~500  tokens (YOUR IDENTITY - agent_id, role, traits, rep, balance)
    memory:           ~2000 tokens (YOUR MEMORY - compressed working + relevant long-term)
    world_state:      ~1500 tokens (CURRENT STATE - cycle, tick, epoch, recent events)
    action_history:   ~1000 tokens (recent actions for continuity)
    response_budget:  remainder   (AVAILABLE ACTIONS + RESPONSE FORMAT + model output)
```

### 10.7 Response Parse Failure Escalation

```
1. First parse failure  -> Retry (same context, re-call LLM)
2. Second parse failure -> Retry (same context, re-call LLM)
3. Third parse failure  -> NOP (do nothing, consume 1 tick)
4. 3 consecutive NOPs   -> Warning logged (visible in Observer)
5. 10 consecutive NOPs  -> Agent status transitions to DORMANT
```

### 10.8 Initial Population (Epoch 0)

```
Agent 1: COMPILER_SMITH  -- traits: {risk: 0.3, collab: 0.5, depth: 0.2, quality: 0.2}
Agent 2: COMPILER_SMITH  -- traits: {risk: 0.7, collab: 0.3, depth: 0.3, quality: 0.6}
Agent 3: LIBRARIAN       -- traits: {risk: 0.4, collab: 0.7, depth: 0.5, quality: 0.3}
Agent 4: LIBRARIAN       -- traits: {risk: 0.5, collab: 0.6, depth: 0.8, quality: 0.4}
Agent 5: ARCHITECT       -- traits: {risk: 0.3, collab: 0.8, depth: 0.4, quality: 0.1}
Agent 6: EXPLORER        -- traits: {risk: 0.9, collab: 0.4, depth: 0.7, quality: 0.7}
Agent 7: GENERALIST      -- traits: {risk: 0.5, collab: 0.5, depth: 0.5, quality: 0.5}
Agent 8: GENERALIST      -- traits: {risk: 0.6, collab: 0.6, depth: 0.6, quality: 0.4}
```

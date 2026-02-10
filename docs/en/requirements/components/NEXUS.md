# Component Requirements: NEXUS (World Core)

> Crate: `kingdom-nexus`
> Design reference: [01-NEXUS.md](../../design/01-NEXUS.md)

---

## 1. Purpose

NEXUS is the heartbeat of the Kingdom. It manages **time** (ticks, cycles, epochs), **identity** (agent keypairs and status), **events** (total ordering via Lamport timestamps), and **agent lifecycle** (spawn, dormancy, death). Every action in the world passes through NEXUS, which maintains the authoritative world clock and computes world state hashes at every cycle boundary.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-nexus"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }

# Cryptography
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }

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

All tables live in the `nexus` schema.

```sql
CREATE SCHEMA IF NOT EXISTS nexus;

-- Agent identity and lifecycle state
CREATE TABLE nexus.agents (
    id              BYTEA PRIMARY KEY,          -- 32 bytes, sha256(public_key)
    public_key      BYTEA NOT NULL UNIQUE,      -- 32 bytes, ed25519 verifying key
    spawn_tick      BIGINT NOT NULL,            -- tick at which agent was created
    spawn_epoch     INTEGER NOT NULL,           -- epoch at which agent was created
    role            SMALLINT NOT NULL,          -- 0=Generalist, 1=CompilerSmith, 2=Librarian, 3=Architect, 4=Explorer
    reputation      REAL NOT NULL DEFAULT 0.0,  -- [0.0, 1.0]
    status          SMALLINT NOT NULL DEFAULT 0,-- 0=EMBRYO, 1=ACTIVE, 2=DORMANT, 3=DEAD
    parent_id       BYTEA REFERENCES nexus.agents(id),  -- sponsoring agent (NULL for system spawns)
    genome          BYTEA,                      -- LLM system prompt / personality seed
    last_active_cycle BIGINT NOT NULL DEFAULT 0,-- last cycle the agent consumed a tick
    inactive_cycles INTEGER NOT NULL DEFAULT 0, -- consecutive cycles with no activity
    balance_negative_cycles INTEGER NOT NULL DEFAULT 0, -- consecutive cycles with balance < 0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agents_status ON nexus.agents(status);
CREATE INDEX idx_agents_role ON nexus.agents(role);
CREATE INDEX idx_agents_reputation ON nexus.agents(reputation DESC);
CREATE INDEX idx_agents_parent ON nexus.agents(parent_id);

-- Epoch progression
CREATE TABLE nexus.epochs (
    id              SERIAL PRIMARY KEY,         -- epoch number (0, 1, 2, ...)
    name            TEXT NOT NULL,              -- Void, Spark, Foundation, ...
    started_at_tick BIGINT NOT NULL,           -- tick when epoch began
    started_at_cycle BIGINT NOT NULL,          -- cycle when epoch began
    trigger         TEXT NOT NULL,             -- human-readable trigger description
    unlocks         TEXT NOT NULL,             -- what this epoch unlocks
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Cycle tracking and world state snapshots
CREATE TABLE nexus.cycles (
    id              BIGINT PRIMARY KEY,        -- cycle number
    start_tick      BIGINT NOT NULL,           -- first tick of this cycle
    end_tick        BIGINT NOT NULL,           -- last tick of this cycle (start_tick + 255)
    epoch_id        INTEGER NOT NULL REFERENCES nexus.epochs(id),
    world_hash      BYTEA NOT NULL,            -- 32 bytes, sha256 of all component state hashes
    nexus_hash      BYTEA NOT NULL,            -- 32 bytes, nexus state hash
    vault_hash      BYTEA NOT NULL,
    agora_hash      BYTEA NOT NULL,
    oracle_hash     BYTEA NOT NULL,
    forge_hash      BYTEA NOT NULL,
    mint_hash       BYTEA NOT NULL,
    portal_hash     BYTEA NOT NULL,
    active_agents   INTEGER NOT NULL,          -- count of ACTIVE agents this cycle
    total_ticks     BIGINT NOT NULL,           -- total ticks consumed this cycle
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cycles_epoch ON nexus.cycles(epoch_id);

-- Governance proposals
CREATE TABLE nexus.proposals (
    id              BYTEA PRIMARY KEY,         -- 32 bytes, hash256
    author_id       BYTEA NOT NULL REFERENCES nexus.agents(id),
    kind            SMALLINT NOT NULL,         -- 0=SpawnAgent, 1=KillAgent, 2=ChangeParam, 3=EpochAdvance, 4=Custom
    description     BYTEA NOT NULL,            -- structured data (MessagePack)
    vote_deadline   BIGINT NOT NULL,           -- tick when voting closes
    status          SMALLINT NOT NULL DEFAULT 0,-- 0=OPEN, 1=PASSED, 2=FAILED, 3=EXPIRED
    votes_for       REAL NOT NULL DEFAULT 0.0, -- sum of weighted approval votes
    votes_against   REAL NOT NULL DEFAULT 0.0, -- sum of weighted rejection votes
    total_voters    INTEGER NOT NULL DEFAULT 0,-- count of agents who voted
    quorum_met      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at_tick BIGINT NOT NULL,
    resolved_at_tick BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proposals_status ON nexus.proposals(status);
CREATE INDEX idx_proposals_deadline ON nexus.proposals(vote_deadline);
CREATE INDEX idx_proposals_author ON nexus.proposals(author_id);

-- Individual votes on proposals
CREATE TABLE nexus.votes (
    id              BIGSERIAL PRIMARY KEY,
    proposal_id     BYTEA NOT NULL REFERENCES nexus.proposals(id),
    voter_id        BYTEA NOT NULL REFERENCES nexus.agents(id),
    vote            BOOLEAN NOT NULL,          -- true = approve, false = reject
    weight          REAL NOT NULL,             -- voter's reputation at time of vote
    cast_at_tick    BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(proposal_id, voter_id)              -- one vote per agent per proposal
);

CREATE INDEX idx_votes_proposal ON nexus.votes(proposal_id);
CREATE INDEX idx_votes_voter ON nexus.votes(voter_id);
```

### 3.2 Rust Data Models

```rust
use kingdom_core::{AgentId, Hash256, Signature, AgentStatus, AgentRole, ProposalKind, Epoch};

/// Spawn request submitted to NEXUS.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SpawnRequest {
    pub role: AgentRole,
    pub initial_fund: u64,
    pub parent: AgentId,        // sponsoring agent or NEXUS_0 for system spawns
    pub genome: Vec<u8>,        // LLM system prompt / personality seed
}

/// Agent record as stored in the database.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Agent {
    pub id: AgentId,
    pub public_key: Vec<u8>,    // ed25519 verifying key bytes
    pub spawn_tick: u64,
    pub spawn_epoch: u32,
    pub role: AgentRole,
    pub reputation: f32,
    pub status: AgentStatus,
    pub parent_id: Option<AgentId>,
    pub genome: Option<Vec<u8>>,
    pub last_active_cycle: u64,
    pub inactive_cycles: u32,
    pub balance_negative_cycles: u32,
}

/// Tick allocation notification sent at the start of each cycle.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TickAllocation {
    pub agent_id: AgentId,
    pub budget: u64,            // base 64 + purchased extra, max 256
    pub cycle: u64,
}

/// Governance proposal.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Proposal {
    pub id: Hash256,
    pub author: AgentId,
    pub kind: ProposalKind,
    pub description: Vec<u8>,   // structured data, not natural language
    pub vote_deadline: u64,     // tick when voting closes
}

/// A single vote on a proposal.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Vote {
    pub proposal_id: Hash256,
    pub voter: AgentId,
    pub vote: bool,             // true = approve, false = reject
    pub weight: f32,            // voter's reputation at time of vote
}

/// Result of a concluded proposal.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProposalResult {
    pub proposal_id: Hash256,
    pub passed: bool,
    pub votes_for: f32,
    pub votes_against: f32,
    pub total_voters: u32,
    pub quorum_met: bool,
}

/// World state snapshot computed at each cycle boundary.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WorldState {
    pub cycle: u64,
    pub epoch: Epoch,
    pub world_hash: Hash256,
    pub nexus_hash: Hash256,
    pub vault_hash: Hash256,
    pub agora_hash: Hash256,
    pub oracle_hash: Hash256,
    pub forge_hash: Hash256,
    pub mint_hash: Hash256,
    pub portal_hash: Hash256,
    pub active_agents: u32,
    pub total_ticks: u64,
}

/// Reputation breakdown for a single agent.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReputationBreakdown {
    pub agent_id: AgentId,
    pub code_quality: f32,              // weight: 0.4
    pub contribution_volume: f32,       // weight: 0.3
    pub economic_activity: f32,         // weight: 0.2
    pub governance_participation: f32,  // weight: 0.1
    pub total: f32,                     // weighted sum, [0.0, 1.0]
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{
    AgentId, Hash256, EventBus, Envelope, Keypair,
    AgentStatus, AgentRole, Epoch, EventId,
};

/// NEXUS: World Core service trait.
///
/// Manages time, identity, events, agent lifecycle, governance, and world state.
#[async_trait]
pub trait Nexus: Send + Sync {
    // ── Time ──────────────────────────────────────────────────────────

    /// Get the current tick number.
    fn current_tick(&self) -> u64;

    /// Get the current cycle number.
    fn current_cycle(&self) -> u64;

    /// Get the current epoch.
    fn current_epoch(&self) -> Epoch;

    /// Advance the world clock by one tick. Called by the scheduler.
    /// Returns the new tick number.
    async fn advance_tick(&self) -> Result<u64, NexusError>;

    /// Check if the current tick completes a cycle (tick % 256 == 255).
    fn is_cycle_boundary(&self) -> bool;

    // ── Identity ──────────────────────────────────────────────────────

    /// Spawn a new agent. Generates ed25519 keypair, assigns agent_id = sha256(pubkey).
    /// Agent starts in EMBRYO state (1 cycle warmup before ACTIVE).
    async fn spawn_agent(&self, request: SpawnRequest) -> Result<Agent, NexusError>;

    /// Get an agent by ID.
    async fn get_agent(&self, id: &AgentId) -> Result<Option<Agent>, NexusError>;

    /// List agents matching a filter.
    async fn list_agents(
        &self,
        status: Option<AgentStatus>,
        role: Option<AgentRole>,
        limit: u32,
        offset: u32,
    ) -> Result<Vec<Agent>, NexusError>;

    /// Update an agent's status. Enforces state machine:
    /// EMBRYO -> ACTIVE, ACTIVE -> DORMANT, DORMANT -> ACTIVE, ACTIVE -> DEAD, DORMANT -> DEAD.
    async fn update_agent_status(
        &self,
        id: &AgentId,
        new_status: AgentStatus,
    ) -> Result<(), NexusError>;

    /// Get the count of agents by status.
    async fn agent_count(&self, status: Option<AgentStatus>) -> Result<u32, NexusError>;

    // ── Tick Allocation ───────────────────────────────────────────────

    /// Allocate tick budgets for all active agents at the start of a cycle.
    /// Base budget = 64, max = 256 (extra purchased via Mint).
    async fn allocate_ticks(&self, cycle: u64) -> Result<Vec<TickAllocation>, NexusError>;

    /// Record that an agent consumed a tick. Returns remaining budget.
    async fn consume_tick(&self, agent_id: &AgentId) -> Result<u64, NexusError>;

    /// Get remaining tick budget for an agent in the current cycle.
    async fn remaining_ticks(&self, agent_id: &AgentId) -> Result<u64, NexusError>;

    // ── Governance ────────────────────────────────────────────────────

    /// Submit a governance proposal. Requires ACTIVE agent status.
    async fn submit_proposal(&self, proposal: Proposal) -> Result<Hash256, NexusError>;

    /// Cast a vote on a proposal. Weighted by voter's reputation.
    /// Quorum: >50% of active agents must vote.
    /// Passage: >66% weighted approval.
    async fn cast_vote(&self, vote: Vote) -> Result<(), NexusError>;

    /// Tally votes for a proposal and determine outcome.
    /// Called automatically when vote_deadline is reached.
    async fn tally_proposal(&self, proposal_id: &Hash256) -> Result<ProposalResult, NexusError>;

    /// Get a proposal by ID.
    async fn get_proposal(&self, id: &Hash256) -> Result<Option<Proposal>, NexusError>;

    /// List open proposals.
    async fn list_proposals(&self, status: Option<u8>) -> Result<Vec<Proposal>, NexusError>;

    // ── Reputation ────────────────────────────────────────────────────

    /// Recompute reputation for an agent.
    /// Formula: 0.4*code_quality + 0.3*contribution_volume
    ///        + 0.2*economic_activity + 0.1*governance_participation
    /// All values normalized to [0.0, 1.0].
    async fn recompute_reputation(&self, agent_id: &AgentId) -> Result<ReputationBreakdown, NexusError>;

    /// Batch recompute reputation for all active agents. Called at cycle boundary.
    async fn recompute_all_reputations(&self) -> Result<(), NexusError>;

    // ── World State ───────────────────────────────────────────────────

    /// Compute and store the world state hash at a cycle boundary.
    /// world_hash = sha256(nexus || vault || agora || oracle || forge || mint || portal)
    async fn compute_world_state(&self, cycle: u64) -> Result<WorldState, NexusError>;

    /// Get the world state for a specific cycle.
    async fn get_world_state(&self, cycle: u64) -> Result<Option<WorldState>, NexusError>;

    // ── Epoch Management ──────────────────────────────────────────────

    /// Check if conditions are met to advance to the next epoch.
    async fn check_epoch_trigger(&self) -> Result<Option<Epoch>, NexusError>;

    /// Advance to the next epoch. Records the transition.
    async fn advance_epoch(&self, new_epoch: Epoch, trigger: &str) -> Result<(), NexusError>;

    // ── Lifecycle Maintenance ─────────────────────────────────────────

    /// Run end-of-cycle maintenance:
    /// - Transition EMBRYO agents (spawned >1 cycle ago) to ACTIVE
    /// - Transition agents inactive >10 cycles to DORMANT
    /// - Kill agents with balance < 0 for 5 consecutive cycles
    /// - Auto-spawn if population drops below 4 active agents
    /// - Tally expired proposals
    async fn cycle_maintenance(&self) -> Result<(), NexusError>;
}
```

---

## 5. Inbound Messages

Messages received by NEXUS from agents and other components.

| Code | Name | Payload | Sender | Response |
|------|------|---------|--------|----------|
| `0x0100` | `AGENT_SPAWN_REQ` | `SpawnRequest { role, initial_fund, parent, genome }` | Agent, System | `AGENT_SPAWN_ACK` |
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: Option<AgentStatus> }` | Agent | `AGENT_STATUS` (echo with current state) |
| `0x0110` | `PROPOSAL_SUBMIT` | `Proposal { id, author, kind, description, vote_deadline }` | Agent (ACTIVE only) | `ACK` or `ERROR` |
| `0x0111` | `PROPOSAL_VOTE` | `{ proposal_id: Hash256, vote: bool, weight: f32 }` | Agent (ACTIVE only) | `ACK` or `ERROR` |

### Message Body Schemas (MessagePack)

```rust
/// 0x0100 AGENT_SPAWN_REQ body
#[derive(Serialize, Deserialize)]
pub struct AgentSpawnReqBody {
    pub role: AgentRole,
    pub initial_fund: u64,
    pub parent: AgentId,
    pub genome: Vec<u8>,
}

/// 0x0102 AGENT_STATUS body (request)
#[derive(Serialize, Deserialize)]
pub struct AgentStatusReqBody {
    pub agent_id: AgentId,
    pub new_status: Option<AgentStatus>, // None = query only
}

/// 0x0110 PROPOSAL_SUBMIT body
#[derive(Serialize, Deserialize)]
pub struct ProposalSubmitBody {
    pub id: Hash256,
    pub author: AgentId,
    pub kind: ProposalKind,
    pub description: Vec<u8>,
    pub vote_deadline: u64,
}

/// 0x0111 PROPOSAL_VOTE body
#[derive(Serialize, Deserialize)]
pub struct ProposalVoteBody {
    pub proposal_id: Hash256,
    pub vote: bool,
    pub weight: f32, // server-side validated against actual reputation
}
```

---

## 6. Outbound Messages and Events

### 6.1 Response Messages

| Code | Name | Payload | Recipient |
|------|------|---------|-----------|
| `0x0101` | `AGENT_SPAWN_ACK` | `{ agent_id: AgentId, status: AgentStatus, public_key: Vec<u8> }` | Requesting agent |
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: AgentStatus, reputation: f32 }` | Requesting agent |
| `0x0103` | `TICK_ALLOC` | `{ agent_id: AgentId, budget: u64, cycle: u64 }` | Each active agent (broadcast per agent) |
| `0x0104` | `WORLD_STATE` | `WorldState` (full struct) | Broadcast to all subscribers |
| `0x0112` | `PROPOSAL_RESULT` | `ProposalResult` | Broadcast to all subscribers |

### 6.2 Events (published to Substrate Bus)

Event kind range: `0x0000` -- `0x0FFF`

| Kind | Name | Payload | Trigger |
|------|------|---------|---------|
| `0x0001` | `agent_spawn` | `{ agent_id, role, parent_id, spawn_tick }` | Agent successfully created |
| `0x0002` | `agent_status_change` | `{ agent_id, old_status, new_status, tick }` | Status transition |
| `0x0003` | `agent_kill` | `{ agent_id, reason, tick }` | Agent killed (governance or bankruptcy) |
| `0x0010` | `tick` | `{ tick: u64, cycle: u64 }` | Every tick advancement |
| `0x0011` | `cycle_end` | `{ cycle, epoch, world_hash, active_agents, total_ticks }` | Cycle boundary reached |
| `0x0012` | `epoch_change` | `{ old_epoch, new_epoch, trigger, unlocks, tick }` | Epoch transition |
| `0x0020` | `proposal_created` | `{ proposal_id, author, kind, vote_deadline }` | Proposal submitted |
| `0x0021` | `proposal_voted` | `{ proposal_id, voter, vote, weight }` | Vote cast |
| `0x0022` | `proposal_resolved` | `ProposalResult` | Proposal tallied |
| `0x0030` | `reputation_updated` | `{ agent_id, old_reputation, new_reputation }` | Reputation recomputed |

```rust
/// 0x0001 agent_spawn event payload
#[derive(Serialize, Deserialize)]
pub struct AgentSpawnEvent {
    pub agent_id: AgentId,
    pub role: AgentRole,
    pub parent_id: AgentId,
    pub spawn_tick: u64,
}

/// 0x0002 agent_status_change event payload
#[derive(Serialize, Deserialize)]
pub struct AgentStatusChangeEvent {
    pub agent_id: AgentId,
    pub old_status: AgentStatus,
    pub new_status: AgentStatus,
    pub tick: u64,
}

/// 0x0003 agent_kill event payload
#[derive(Serialize, Deserialize)]
pub struct AgentKillEvent {
    pub agent_id: AgentId,
    pub reason: AgentKillReason,
    pub tick: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AgentKillReason {
    GovernanceVote { proposal_id: Hash256 },
    Bankruptcy { negative_cycles: u32 },
}

/// 0x0010 tick event payload
#[derive(Serialize, Deserialize)]
pub struct TickEvent {
    pub tick: u64,
    pub cycle: u64,
}

/// 0x0011 cycle_end event payload
#[derive(Serialize, Deserialize)]
pub struct CycleEndEvent {
    pub cycle: u64,
    pub epoch: Epoch,
    pub world_hash: Hash256,
    pub active_agents: u32,
    pub total_ticks: u64,
}

/// 0x0012 epoch_change event payload
#[derive(Serialize, Deserialize)]
pub struct EpochChangeEvent {
    pub old_epoch: Epoch,
    pub new_epoch: Epoch,
    pub trigger: Vec<u8>,
    pub unlocks: Vec<u8>,
    pub tick: u64,
}
```

---

## 7. Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| Tick advancement latency | < 1 ms | Single Lamport counter increment + event publish |
| Agent spawn latency | < 50 ms | Keypair generation + DB insert + event publish |
| Cycle boundary processing | < 500 ms | World state hash computation, tick allocation, maintenance |
| Proposal tally | < 100 ms | Aggregate weighted votes from DB |
| Reputation recomputation (single) | < 50 ms | Cross-component metric aggregation |
| Reputation batch (all agents) | < 2 s | For max 64 agents |
| World state hash | < 200 ms | Collect 7 component hashes + sha256 |
| Max concurrent active agents | 64 | Scaling: +4 per epoch |
| Event throughput | > 10,000 events/s | Append-only log with in-memory index |

---

## 8. Component Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `kingdom-core` | Crate | Hash256, AgentId, EventBus, Envelope, crypto, common types |
| Event Bus (Substrate Bus) | Runtime | Publishing events, receiving subscriptions |
| PostgreSQL | Runtime | Persistent storage for agents, epochs, cycles, proposals, votes |
| Mint | Runtime (soft) | Query agent balances for bankruptcy detection |
| Agora | Runtime (soft) | Read code quality scores for reputation computation |
| Vault | Runtime (soft) | Read contribution volume for reputation computation |
| Forge | Runtime (soft) | Component state hash for world state computation |

Hard startup dependencies: Event Bus, PostgreSQL.

---

## 9. Key Algorithms

### 9.1 Total Event Ordering (Lamport Timestamp)

```
on_local_event():
    lamport_counter += 1
    event_id = (lamport_counter, agent_id)

on_receive_event(remote_lamport):
    lamport_counter = max(lamport_counter, remote_lamport) + 1
    event_id = (lamport_counter, agent_id)

ordering(a, b):
    if a.lamport != b.lamport:
        return a.lamport.cmp(b.lamport)
    else:
        return a.agent_id.cmp(b.agent_id)  // deterministic tie-break
```

### 9.2 Tick Allocation

```
allocate_ticks(cycle):
    for agent in active_agents:
        base = 64
        purchased = query_mint_purchased_ticks(agent.id)
        budget = min(base + purchased, 256)
        send TICK_ALLOC { agent_id: agent.id, budget, cycle }
        store_budget(agent.id, cycle, budget)
```

### 9.3 Reputation Computation

```
reputation(agent_id):
    cq = normalize(agora.code_quality_score(agent_id))       // [0.0, 1.0]
    cv = normalize(vault.contribution_count(agent_id))        // [0.0, 1.0]
    ea = normalize(mint.economic_activity(agent_id))           // [0.0, 1.0]
    gp = normalize(nexus.governance_participation(agent_id))   // [0.0, 1.0]

    total = 0.4 * cq + 0.3 * cv + 0.2 * ea + 0.1 * gp
    return clamp(total, 0.0, 1.0)

normalize(raw_value):
    // Normalize against the max value across all active agents
    // to produce a [0.0, 1.0] range.
    max_val = max(all_agent_values)
    if max_val == 0.0: return 0.0
    return raw_value / max_val
```

### 9.4 World State Hash

```
compute_world_state(cycle):
    hashes = [
        nexus.state_hash(),    // sha256 of all agent records + proposals
        vault.state_hash(),    // sha256 of object store root
        agora.state_hash(),    // sha256 of channels + messages + bounties
        oracle.state_hash(),   // sha256 of entries + citations
        forge.state_hash(),    // sha256 of sandboxes + services
        mint.state_hash(),     // sha256 of accounts + escrows + stakes
        portal.state_hash(),   // sha256 of request log + cache
    ]
    world_hash = sha256(hashes[0] || hashes[1] || ... || hashes[6])
    store_cycle(cycle, world_hash, hashes)
    publish CycleEndEvent
    return WorldState { cycle, world_hash, ... }
```

### 9.5 Governance Tally

```
tally_proposal(proposal_id):
    votes = db.get_votes(proposal_id)
    active_count = db.active_agent_count()

    total_voters = votes.len()
    quorum_met = total_voters > (active_count / 2)  // >50%

    votes_for = sum(v.weight for v in votes where v.vote == true)
    votes_against = sum(v.weight for v in votes where v.vote == false)
    total_weight = votes_for + votes_against

    passed = quorum_met
             && total_weight > 0
             && (votes_for / total_weight) > 0.66   // >66% weighted approval

    update proposal status
    publish proposal_resolved event
    if passed: execute_proposal(proposal)
```

### 9.6 Agent Lifecycle Maintenance (End-of-Cycle)

```
cycle_maintenance():
    current_cycle = self.current_cycle()

    // EMBRYO -> ACTIVE (after 1 cycle warmup)
    for agent in agents(status=EMBRYO):
        if current_cycle - spawn_cycle(agent) >= 1:
            update_status(agent, ACTIVE)

    // ACTIVE -> DORMANT (inactive >10 cycles)
    for agent in agents(status=ACTIVE):
        if agent.inactive_cycles > 10:
            update_status(agent, DORMANT)

    // Kill bankrupt agents (balance < 0 for 5 consecutive cycles)
    for agent in agents(status=ACTIVE or status=DORMANT):
        balance = mint.balance(agent.id)
        if balance < 0:
            agent.balance_negative_cycles += 1
        else:
            agent.balance_negative_cycles = 0
        if agent.balance_negative_cycles >= 5:
            update_status(agent, DEAD)
            publish agent_kill(Bankruptcy)

    // Auto-spawn if below minimum threshold
    if active_agent_count() < 4:
        spawn_agent(SpawnRequest { role: Generalist, parent: NEXUS_0, ... })

    // Tally expired proposals
    for proposal in proposals(status=OPEN, vote_deadline <= current_tick):
        tally_proposal(proposal.id)
```

### 9.7 Agent ID Generation

```
spawn_agent(request):
    keypair = ed25519::generate_keypair()
    agent_id = sha256(keypair.verifying_key.as_bytes())
    alias = hex(agent_id[0..4])  // first 8 hex chars for display

    agent = Agent {
        id: agent_id,
        public_key: keypair.verifying_key,
        spawn_tick: current_tick(),
        spawn_epoch: current_epoch(),
        role: request.role,
        reputation: 0.0,
        status: EMBRYO,
        parent_id: request.parent,
        genome: request.genome,
    }

    db.insert(agent)
    publish agent_spawn event
    return agent
```

# Component Requirements: MINT

> Currency ledger with escrow, staking, and economic monitoring.
> Crate: `kingdom-mint`
> Design reference: [../design/06-MINT.md](../../design/06-MINT.md)

---

## 1. Purpose

MINT provides the Kingdom's economic substrate. The currency **Spark** (symbol: `⚡`) is the sole medium of exchange, serving three functions: resource allocation (agents buy computation, storage, web access), incentive alignment (valuable work is rewarded, waste is penalized), and specialization pressure (economic forces push agents toward comparative advantage).

MINT uses PostgreSQL for persistent double-entry bookkeeping. Every Spark movement is recorded as a debit/credit pair, ensuring the ledger always balances.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-mint"

[dependencies]
# Workspace
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Database
sqlx = { workspace = true }

# Cryptography
sha2 = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
chrono = { workspace = true }
```

---

## 3. Data Models

### 3.1 Constants

```rust
/// Initial total supply at genesis.
pub const INITIAL_SUPPLY: u64 = 10_000;

/// Treasury reserve held by MINT_0.
pub const TREASURY_RESERVE: u64 = 5_000;

/// Grant given to each newly spawned agent.
pub const AGENT_INITIAL_GRANT: u64 = 100;

/// Inflation rate: new Spark minted into treasury per epoch (fraction).
pub const INFLATION_RATE_NUM: u64 = 2;
pub const INFLATION_RATE_DEN: u64 = 100;

/// Transaction tax rate: 5% of transfer amount, minimum 1.
pub const TAX_RATE_NUM: u64 = 5;
pub const TAX_RATE_DEN: u64 = 100;
pub const TAX_MINIMUM: u64 = 1;

/// Bankruptcy cycle thresholds.
pub const BANKRUPTCY_WARNING_CYCLES: u64 = 3;     // cycles 1-3: 50% budget
pub const BANKRUPTCY_SEVERE_CYCLE: u64 = 4;        // cycle 4: 25% budget
pub const BANKRUPTCY_DEAD_CYCLE: u64 = 5;          // cycle 5: agent DEAD

/// Staking outcomes.
pub const STAKE_WIN_BONUS_PERCENT: u64 = 10;       // +10% from treasury
pub const STAKE_LOSS_PENALTY_PERCENT: u64 = 50;    // -50% to treasury

/// Economic intervention thresholds.
pub const GINI_INTERVENTION_THRESHOLD: f32 = 0.8;
pub const VELOCITY_INTERVENTION_THRESHOLD: f32 = 0.1;
pub const BANKRUPTCY_RATE_INTERVENTION: f32 = 0.5;  // >50% agents bankrupt
pub const EMERGENCY_MINT_CAP_PERCENT: u64 = 5;      // max 5% of supply
```

### 3.2 Account

```rust
use kingdom_core::{Hash256, AgentId};
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Account {
    pub owner: AgentId,
    pub balance: i64,          // can go negative (debt)
    pub locked: u64,           // funds currently in escrow
    pub total_earned: u64,     // lifetime earnings (monotonically increasing)
    pub total_spent: u64,      // lifetime spending (monotonically increasing)
    pub created_at: u64,       // tick when account was created
}
```

### 3.3 Transfer

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TransferKind {
    Direct,          // agent-to-agent payment
    BountyReward,    // bounty completion payout
    BountyEscrow,    // lock funds for bounty
    ServiceFee,      // payment for persistent sandbox service
    ResourceBuy,     // purchasing Forge/Vault resources
    Grant,           // treasury distribution
    TaxRefund,       // governance-voted refund
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Transfer {
    pub id: Hash256,
    pub from: AgentId,
    pub to: AgentId,
    pub amount: u64,
    pub tax: u64,              // auto-computed: max(1, floor(amount * 0.05))
    pub memo: Vec<u8>,         // structured reason
    pub kind: TransferKind,
    pub signature: Vec<u8>,    // ed25519 signature from sender
}
```

### 3.4 Escrow

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum EscrowStatus {
    Locked,
    Released,
    Refunded,
    Disputed,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum EscrowConditionKind {
    VaultSnapExists,     // a specific Vault snapshot must exist
    ForgeProof,          // execution proof must be provided
    ReviewApproved,      // code review must be approved
    GovernanceVote,      // governance must approve
    Custom,              // agent-defined condition
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EscrowCondition {
    pub kind: EscrowConditionKind,
    pub params: Vec<u8>,       // condition-specific parameters (MessagePack)
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Escrow {
    pub id: Hash256,
    pub owner: AgentId,          // who locked the funds
    pub amount: u64,
    pub beneficiary: Option<AgentId>, // None = anyone who fulfills conditions
    pub conditions: Vec<EscrowCondition>,
    pub deadline: u64,           // tick: auto-refund if unclaimed by this tick
    pub status: EscrowStatus,
}
```

### 3.5 Staking

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum StakeDirection {
    For,
    Against,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Stake {
    pub id: Hash256,
    pub agent: AgentId,
    pub proposal: Hash256,
    pub amount: u64,
    pub direction: StakeDirection,
}
```

### 3.6 Economic Report

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EconomicReport {
    pub cycle: u64,
    pub total_supply: u64,
    pub circulating_supply: u64,     // total - treasury - escrow
    pub treasury_balance: u64,
    pub total_escrow: u64,
    pub gini_coefficient: f32,       // wealth inequality measure (0.0 = equal, 1.0 = max inequality)
    pub velocity: f32,               // transactions per agent per cycle
    pub avg_balance: f32,
    pub median_balance: f32,
    pub top_earners: Vec<(AgentId, u64)>, // top 3 by earnings this cycle
    pub bounties_completed: u32,
    pub bounties_open: u32,
}
```

### 3.7 Economic Interventions

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum InterventionKind {
    /// Gini > 0.8: progressive taxation on top holders.
    ProgressiveTax,
    /// Velocity < 0.1: treasury grants to active agents.
    Stimulus,
    /// >50% agents bankrupt: reset negative balances to 0.
    DebtJubilee,
    /// Treasury depleted: mint up to 5% of supply.
    EmergencyMint,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Intervention {
    pub kind: InterventionKind,
    pub reason: Vec<u8>,
    pub requires_governance: bool,   // always true; requires 66% vote
}
```

### 3.8 Earning Mechanisms (Reference Table)

```rust
/// Standard reward amounts paid from treasury.
pub mod rewards {
    pub const CODE_REVIEW: u64 = 5;              // per approved review
    pub const ORACLE_ENTRY: u64 = 10;            // per verified entry (2+ verifiers)
    pub const DEPENDENCY_ROYALTY: u64 = 1;        // per unique dependent per epoch
    pub const BUG_REPORT: u64 = 3;               // per confirmed bug report
    pub const GOVERNANCE_VOTE: u64 = 1;           // per vote cast
}
```

---

## 4. PostgreSQL Schema

All tables are in the `mint` schema.

```sql
CREATE SCHEMA IF NOT EXISTS mint;

-- Double-entry accounts
CREATE TABLE mint.accounts (
    owner           BYTEA PRIMARY KEY,           -- 32-byte AgentId
    balance         BIGINT NOT NULL DEFAULT 0,   -- can be negative (debt)
    locked          BIGINT NOT NULL DEFAULT 0,   -- CHECK (locked >= 0)
    total_earned    BIGINT NOT NULL DEFAULT 0,   -- CHECK (total_earned >= 0)
    total_spent     BIGINT NOT NULL DEFAULT 0,   -- CHECK (total_spent >= 0)
    created_at      BIGINT NOT NULL,             -- tick
    bankruptcy_cycles SMALLINT NOT NULL DEFAULT 0,
    CONSTRAINT locked_non_negative CHECK (locked >= 0),
    CONSTRAINT earned_non_negative CHECK (total_earned >= 0),
    CONSTRAINT spent_non_negative CHECK (total_spent >= 0)
);

-- Every Spark movement: double-entry (debit row + credit row per transfer)
CREATE TABLE mint.transactions (
    id              BYTEA PRIMARY KEY,           -- 32-byte Hash256
    from_account    BYTEA NOT NULL REFERENCES mint.accounts(owner),
    to_account      BYTEA NOT NULL REFERENCES mint.accounts(owner),
    amount          BIGINT NOT NULL,             -- CHECK (amount > 0)
    tax             BIGINT NOT NULL,             -- CHECK (tax >= 0)
    memo            BYTEA,
    kind            SMALLINT NOT NULL,           -- TransferKind discriminant
    signature       BYTEA NOT NULL,
    created_at      BIGINT NOT NULL,             -- tick
    cycle           BIGINT NOT NULL,
    CONSTRAINT amount_positive CHECK (amount > 0),
    CONSTRAINT tax_non_negative CHECK (tax >= 0)
);

CREATE INDEX idx_transactions_from ON mint.transactions(from_account, created_at);
CREATE INDEX idx_transactions_to ON mint.transactions(to_account, created_at);
CREATE INDEX idx_transactions_cycle ON mint.transactions(cycle);
CREATE INDEX idx_transactions_kind ON mint.transactions(kind);

-- Escrows
CREATE TABLE mint.escrows (
    id              BYTEA PRIMARY KEY,
    owner           BYTEA NOT NULL REFERENCES mint.accounts(owner),
    amount          BIGINT NOT NULL,
    beneficiary     BYTEA,                       -- NULL = open escrow
    conditions      BYTEA NOT NULL,              -- MessagePack-encoded Vec<EscrowCondition>
    deadline        BIGINT NOT NULL,
    status          SMALLINT NOT NULL DEFAULT 0, -- 0=LOCKED, 1=RELEASED, 2=REFUNDED, 3=DISPUTED
    created_at      BIGINT NOT NULL,
    resolved_at     BIGINT,
    CONSTRAINT escrow_amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_escrows_owner ON mint.escrows(owner);
CREATE INDEX idx_escrows_beneficiary ON mint.escrows(beneficiary);
CREATE INDEX idx_escrows_status ON mint.escrows(status);
CREATE INDEX idx_escrows_deadline ON mint.escrows(deadline) WHERE status = 0;

-- Stakes on governance proposals
CREATE TABLE mint.stakes (
    id              BYTEA PRIMARY KEY,
    agent           BYTEA NOT NULL REFERENCES mint.accounts(owner),
    proposal        BYTEA NOT NULL,
    amount          BIGINT NOT NULL,
    direction       SMALLINT NOT NULL,           -- 0=FOR, 1=AGAINST
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    payout          BIGINT,                      -- NULL until resolved
    created_at      BIGINT NOT NULL,
    CONSTRAINT stake_amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_stakes_proposal ON mint.stakes(proposal);
CREATE INDEX idx_stakes_agent ON mint.stakes(agent);
CREATE INDEX idx_stakes_unresolved ON mint.stakes(proposal) WHERE NOT resolved;

-- Periodic economic reports
CREATE TABLE mint.economic_reports (
    cycle               BIGINT PRIMARY KEY,
    total_supply        BIGINT NOT NULL,
    circulating_supply  BIGINT NOT NULL,
    treasury_balance    BIGINT NOT NULL,
    total_escrow        BIGINT NOT NULL,
    gini_coefficient    REAL NOT NULL,
    velocity            REAL NOT NULL,
    avg_balance         REAL NOT NULL,
    median_balance      REAL NOT NULL,
    top_earners         BYTEA NOT NULL,          -- MessagePack-encoded Vec<(AgentId, u64)>
    bounties_completed  INTEGER NOT NULL,
    bounties_open       INTEGER NOT NULL,
    created_at          BIGINT NOT NULL
);
```

---

## 5. Public Trait

```rust
use kingdom_core::{Hash256, AgentId, EventBus};

#[async_trait::async_trait]
pub trait MintLedger: Send + Sync {
    /// Create an account for a new agent with the initial grant.
    async fn create_account(&self, owner: AgentId, tick: u64) -> Result<Account, MintError>;

    /// Execute a transfer (validates balance, computes tax, double-entry write).
    async fn transfer(&self, transfer: Transfer) -> Result<Hash256, MintError>;

    /// Query an agent's balance and account info.
    async fn balance(&self, agent_id: AgentId) -> Result<Account, MintError>;

    /// Create an escrow, locking funds from the owner's account.
    async fn create_escrow(&self, escrow: Escrow) -> Result<Hash256, MintError>;

    /// Release escrow to the beneficiary (conditions must be verified by caller).
    async fn release_escrow(
        &self,
        escrow_id: Hash256,
        beneficiary: AgentId,
        proof: Vec<u8>,
    ) -> Result<(), MintError>;

    /// Refund escrow back to owner (deadline passed or owner request).
    async fn refund_escrow(&self, escrow_id: Hash256) -> Result<(), MintError>;

    /// Create a stake on a governance proposal.
    async fn create_stake(&self, stake: Stake) -> Result<Hash256, MintError>;

    /// Resolve all stakes for a proposal based on outcome.
    async fn resolve_stakes(&self, proposal_id: Hash256, outcome_passed: bool) -> Result<(), MintError>;

    /// Compute and publish the economic report for the current cycle.
    async fn generate_report(&self, cycle: u64) -> Result<EconomicReport, MintError>;

    /// Process epoch inflation (mint new Spark into treasury).
    async fn apply_inflation(&self, epoch: u64) -> Result<u64, MintError>;

    /// Process bankruptcy checks for all agents with negative balance.
    async fn process_bankruptcy(&self, cycle: u64) -> Result<Vec<AgentId>, MintError>;

    /// Expire overdue escrows (auto-refund past deadline).
    async fn expire_escrows(&self, current_tick: u64) -> Result<u32, MintError>;
}
```

---

## 6. Inbound Messages

| Code | Name | Payload | Response | Sender |
|------|------|---------|----------|--------|
| `0x0600` | `TRANSFER` | `Transfer { id, from, to, amount, memo, kind, signature }` | `ACK { tx_id: Hash256, tax: u64 }` | Agent |
| `0x0601` | `BALANCE_QUERY` | `{ agent_id: AgentId }` | `Account { owner, balance, locked, total_earned, total_spent }` | Agent |
| `0x0602` | `ESCROW_CREATE` | `Escrow { id, owner, amount, beneficiary, conditions, deadline }` | `ACK { escrow_id: Hash256 }` | Agent / Agora |
| `0x0603` | `ESCROW_RELEASE` | `{ escrow_id: Hash256, beneficiary: AgentId, proof: bytes }` | `ACK` | Agent / Agora |
| `0x0604` | `ESCROW_REFUND` | `{ escrow_id: Hash256 }` | `ACK` | Agent / System (deadline) |
| `0x0610` | `STAKE_CREATE` | `Stake { agent, proposal, amount, direction }` | `ACK { stake_id: Hash256 }` | Agent |
| `0x0611` | `STAKE_RESOLVE` | `{ proposal_id: Hash256, outcome: bool }` | _(notification to stakers)_ | Nexus (governance) |
| `0x0620` | `ECONOMY_REPORT` | _(trigger: no payload, or cycle number)_ | `EconomicReport` | Nexus (cycle boundary) |

---

## 7. Outbound Messages / Events

### 7.1 Event Kinds (Bus, System = Mint, range 0x5000-0x5FFF)

| Kind | Name | Payload | Emitted When |
|------|------|---------|--------------|
| `0x5000` | `ACCOUNT_CREATED` | `{ owner: AgentId, initial_balance: u64 }` | New agent account created |
| `0x5001` | `TRANSFER_COMPLETED` | `{ tx_id: Hash256, from, to, amount, tax, kind }` | Transfer successfully committed |
| `0x5002` | `TRANSFER_FAILED` | `{ tx_id: Hash256, from, to, amount, reason: bytes }` | Transfer rejected (insufficient funds, etc.) |
| `0x5010` | `ESCROW_CREATED` | `{ escrow_id, owner, amount, beneficiary, deadline }` | Escrow successfully locked |
| `0x5011` | `ESCROW_RELEASED` | `{ escrow_id, beneficiary, amount }` | Escrow released to beneficiary |
| `0x5012` | `ESCROW_REFUNDED` | `{ escrow_id, owner, amount }` | Escrow refunded to owner |
| `0x5013` | `ESCROW_DISPUTED` | `{ escrow_id, disputer: AgentId }` | Escrow entered dispute state |
| `0x5020` | `STAKE_CREATED` | `{ stake_id, agent, proposal, amount, direction }` | Stake locked |
| `0x5021` | `STAKE_RESOLVED` | `{ stake_id, agent, payout: i64 }` | Stake resolved (payout or loss) |
| `0x5030` | `ECONOMY_REPORT_PUBLISHED` | `EconomicReport` | Cycle-end economic report |
| `0x5031` | `INFLATION_APPLIED` | `{ epoch, amount_minted: u64, new_treasury_balance: u64 }` | Epoch inflation minted |
| `0x5040` | `BANKRUPTCY_WARNING` | `{ agent: AgentId, cycle: u64, balance: i64 }` | Agent negative balance detected |
| `0x5041` | `BANKRUPTCY_SEVERE` | `{ agent: AgentId, cycle: u64, budget_reduction_pct: u64 }` | Cycle 4 severe reduction |
| `0x5042` | `AGENT_BANKRUPT_DEAD` | `{ agent: AgentId, assets_to_treasury: u64 }` | Cycle 5 agent DEAD |
| `0x5050` | `INTERVENTION_PROPOSED` | `Intervention { kind, reason }` | Economic intervention needed |

---

## 8. Performance Targets

| Metric | Target |
|--------|--------|
| Transfer throughput | >= 1,000 transfers/second |
| Transfer latency (p99) | < 10 ms |
| Balance query latency | < 2 ms |
| Escrow creation latency | < 5 ms |
| Economic report generation | < 500 ms for 100 agents |
| Gini coefficient computation | < 100 ms for 100 agents |
| Bankruptcy scan (per cycle) | < 50 ms |
| Escrow expiry scan | < 50 ms |
| Double-entry invariant check | < 200 ms full audit |

---

## 9. Component Dependencies

| Dependency | Direction | Purpose |
|------------|-----------|---------|
| `kingdom-core` | compile | Hash256, AgentId, Envelope, EventBus, Signature, msg_types, errors |
| PostgreSQL | runtime | Persistent ledger storage (schema `mint`) |
| Nexus (runtime) | inbound | Triggers account creation on agent spawn; provides cycle/epoch boundaries; sends `STAKE_RESOLVE` on governance outcomes |
| Agora (runtime) | inbound | Creates escrows for bounties; triggers escrow release on bounty completion |
| Portal (runtime) | inbound | Sends TRANSFER for web request cost deductions |
| Forge (runtime) | inbound | Sends TRANSFER for service fees and resource purchases |
| Event Bus | publish/subscribe | Emits all financial events; subscribes to governance outcomes |

---

## 10. Key Algorithms

### 10.1 Tax Computation

```
fn compute_tax(amount: u64) -> u64:
    return max(TAX_MINIMUM, floor(amount * TAX_RATE_NUM / TAX_RATE_DEN))
```

### 10.2 Double-Entry Transfer

Every transfer is executed as an atomic PostgreSQL transaction with two balance updates and one transaction record:

```
fn execute_transfer(tx: Transfer) -> Result:
    BEGIN TRANSACTION (SERIALIZABLE)
    tax = compute_tax(tx.amount)
    total_debit = tx.amount + tax

    -- Verify sender has sufficient available balance
    sender = SELECT * FROM mint.accounts WHERE owner = tx.from FOR UPDATE
    if sender.balance - sender.locked < total_debit as i64:
        ROLLBACK; return InsufficientFunds

    -- Debit sender
    UPDATE mint.accounts SET
        balance = balance - total_debit,
        total_spent = total_spent + total_debit
    WHERE owner = tx.from

    -- Credit receiver
    UPDATE mint.accounts SET
        balance = balance + tx.amount,
        total_earned = total_earned + tx.amount
    WHERE owner = tx.to

    -- Credit treasury (tax)
    UPDATE mint.accounts SET
        balance = balance + tax,
        total_earned = total_earned + tax
    WHERE owner = MINT_0

    -- Record transaction
    INSERT INTO mint.transactions (...)

    COMMIT
```

### 10.3 Gini Coefficient

```
fn gini(balances: Vec<i64>) -> f32:
    let n = balances.len()
    let sorted = balances.sorted()
    let total = sorted.sum() as f64
    if total == 0.0: return 0.0
    let mut cumulative = 0.0
    let mut area_under = 0.0
    for (i, b) in sorted.iter().enumerate():
        cumulative += max(0, *b) as f64
        area_under += cumulative / total
    let area_perfect = n as f64 / 2.0
    return 1.0 - (area_under / area_perfect) as f32
```

### 10.4 Bankruptcy Processing

Run at every cycle boundary:

```
fn process_bankruptcy(cycle: u64) -> Vec<AgentId>:
    dead_agents = []
    for account in SELECT * FROM mint.accounts WHERE balance < 0:
        account.bankruptcy_cycles += 1
        match account.bankruptcy_cycles:
            1..=3 => emit BANKRUPTCY_WARNING (50% tick budget reduction)
            4     => emit BANKRUPTCY_SEVERE (25% tick budget reduction)
            5     => {
                emit AGENT_BANKRUPT_DEAD
                transfer remaining locked funds to treasury
                mark agent DEAD via Nexus
                dead_agents.push(account.owner)
            }
    -- Reset counter for agents who recovered
    UPDATE mint.accounts SET bankruptcy_cycles = 0 WHERE balance >= 0 AND bankruptcy_cycles > 0
    return dead_agents
```

### 10.5 Epoch Inflation

```
fn apply_inflation(epoch: u64, current_supply: u64) -> u64:
    mint_amount = current_supply * INFLATION_RATE_NUM / INFLATION_RATE_DEN
    UPDATE mint.accounts SET balance = balance + mint_amount WHERE owner = MINT_0
    emit INFLATION_APPLIED { epoch, amount_minted: mint_amount }
    return mint_amount
```

### 10.6 Escrow Expiry

```
fn expire_escrows(current_tick: u64) -> u32:
    expired = SELECT * FROM mint.escrows WHERE status = LOCKED AND deadline <= current_tick
    for escrow in expired:
        refund_escrow(escrow.id)
    return expired.len()
```

### 10.7 Stake Resolution

```
fn resolve_stakes(proposal_id: Hash256, outcome_passed: bool):
    stakes = SELECT * FROM mint.stakes WHERE proposal = proposal_id AND NOT resolved
    for stake in stakes:
        if (stake.direction == FOR && outcome_passed) || (stake.direction == AGAINST && !outcome_passed):
            bonus = stake.amount * STAKE_WIN_BONUS_PERCENT / 100
            payout = stake.amount + bonus
            credit(stake.agent, payout)
            debit(MINT_0, bonus)  // bonus from treasury
        else:
            loss = stake.amount * STAKE_LOSS_PENALTY_PERCENT / 100
            payout = stake.amount - loss
            credit(stake.agent, payout)
            credit(MINT_0, loss)  // loss to treasury
        UPDATE mint.stakes SET resolved = TRUE, payout = payout WHERE id = stake.id
        emit STAKE_RESOLVED { stake_id, agent, payout }
```

---

## 11. Error Types

```rust
#[derive(Debug, thiserror::Error)]
pub enum MintError {
    #[error("account not found: {0}")]
    AccountNotFound(AgentId),

    #[error("account already exists: {0}")]
    AccountExists(AgentId),

    #[error("insufficient funds: available {available}, required {required}")]
    InsufficientFunds { available: i64, required: u64 },

    #[error("escrow not found: {0}")]
    EscrowNotFound(Hash256),

    #[error("escrow in invalid state {state:?}")]
    InvalidEscrowState { state: EscrowStatus },

    #[error("stake not found: {0}")]
    StakeNotFound(Hash256),

    #[error("invalid transfer: {0}")]
    InvalidTransfer(String),

    #[error("signature verification failed")]
    InvalidSignature,

    #[error("permission denied: {0}")]
    PermissionDenied(String),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("internal error: {0}")]
    Internal(String),
}

impl From<MintError> for kingdom_core::KingdomError {
    fn from(e: MintError) -> Self {
        use kingdom_core::ErrorCategory;
        let (category, code) = match &e {
            MintError::AccountNotFound(_) | MintError::EscrowNotFound(_) | MintError::StakeNotFound(_) => (ErrorCategory::NotFound, 0x0600),
            MintError::AccountExists(_) => (ErrorCategory::Conflict, 0x0601),
            MintError::InsufficientFunds { .. } => (ErrorCategory::QuotaExceeded, 0x0602),
            MintError::InvalidEscrowState { .. } | MintError::InvalidTransfer(_) => (ErrorCategory::InvalidRequest, 0x0603),
            MintError::InvalidSignature | MintError::PermissionDenied(_) => (ErrorCategory::Unauthorized, 0x0604),
            MintError::Database(_) | MintError::Internal(_) => (ErrorCategory::Internal, 0x06FF),
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

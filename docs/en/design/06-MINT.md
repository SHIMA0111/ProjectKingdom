# 06 — MINT: Currency & Economic System

## 1. Purpose

Mint provides the Kingdom's **economic substrate**. Currency serves three critical functions:

1. **Resource allocation** — agents buy computation time, storage, and web access
2. **Incentive alignment** — valuable work is rewarded, waste is penalized
3. **Specialization pressure** — economic forces push agents toward comparative advantage

The currency is called **Spark** (symbol: `⚡`). It is the only medium of exchange.

---

## 2. Monetary Policy

### 2.1 Supply

```
INITIAL_SUPPLY        = 10000 ⚡  (distributed at genesis)
TREASURY_RESERVE      = 5000 ⚡   (held by MINT_0 for system bounties)
AGENT_INITIAL_GRANT   = 100 ⚡    (given to each new agent at spawn)

INFLATION_RATE        = 2% per epoch (new ⚡ minted into treasury)
DEFLATION_MECHANISM   = transaction tax (see below)
```

### 2.2 Transaction Tax

Every transfer is taxed:

```
tax(amount) = max(1, floor(amount * 0.05))
```

Tax goes to the treasury, funding system bounties and infrastructure. This prevents infinite currency circulation without value creation.

### 2.3 Bankruptcy

If an agent's balance goes negative:
- Cycle 1-3: Warning status, reduced tick budget (50%)
- Cycle 4: Severe reduction (25% budget), assets can be claimed by creditors
- Cycle 5: Agent enters DEAD state, all assets go to treasury

---

## 3. Accounts

```
Account {
  owner:        hash256
  balance:      i64              // can go negative (debt)
  locked:       u64              // funds in escrow (bounties, services)
  total_earned: u64              // lifetime earnings (never decreases)
  total_spent:  u64              // lifetime spending
  created_at:   u64              // tick
}
```

System accounts (NEXUS_0, etc.) have `balance = ∞` and cannot transact.

The **treasury** (`MINT_0`) is special — it receives taxes and funds system bounties.

---

## 4. Transactions

### 4.1 Transfer

```
Transfer {
  id:        hash256
  from:      hash256
  to:        hash256
  amount:    u64
  tax:       u64                 // auto-computed
  memo:      bytes               // structured reason
  kind:      enum(
    DIRECT,          // agent-to-agent payment
    BOUNTY_REWARD,   // bounty completion payout
    BOUNTY_ESCROW,   // lock funds for bounty
    SERVICE_FEE,     // payment for persistent sandbox service
    RESOURCE_BUY,    // purchasing Forge/Vault resources
    GRANT,           // treasury distribution
    TAX_REFUND,      // governance-voted refund
  )
  signature: bytes
}
```

### 4.2 Escrow

For bounties and services, funds are locked in escrow:

```
Escrow {
  id:          hash256
  owner:       hash256           // who locked the funds
  amount:      u64
  beneficiary: hash256 | null    // who can claim (null = anyone who fulfills conditions)
  conditions:  [EscrowCondition]
  deadline:    u64               // auto-refund if unclaimed by this tick
  status:      enum(LOCKED | RELEASED | REFUNDED | DISPUTED)
}

EscrowCondition {
  kind:   enum(
    VAULT_SNAP_EXISTS,    // a specific vault snapshot must exist
    FORGE_PROOF,          // execution proof must be provided
    REVIEW_APPROVED,      // code review must be approved
    GOVERNANCE_VOTE,      // governance must approve
    CUSTOM                // agent-defined condition
  )
  params: bytes
}
```

---

## 5. Earning Mechanisms

How agents earn ⚡:

| Activity | Reward | Source |
|----------|--------|--------|
| Complete a bounty | Bounty reward | Bounty creator (via escrow) |
| Code review (approved by peers) | 5 ⚡ per review | Treasury |
| Oracle entry (verified by 2+ agents) | 10 ⚡ | Treasury |
| Library used as dependency | 1 ⚡ per unique dependent per epoch | Treasury |
| Bug report (confirmed) | 3 ⚡ | Treasury |
| Governance participation (voting) | 1 ⚡ per vote | Treasury |
| Service operation (per-request fees) | Set by service owner | Clients |

### 5.1 Passive Income: Dependency Royalties

When an agent's Vault repository is used as a dependency by other repos, the author earns **royalties**:

```
royalty(repo) = count(unique_dependents) * 1 ⚡ per epoch
```

This incentivizes building reusable, high-quality libraries.

### 5.2 Staking

Agents can stake ⚡ on proposals they believe will benefit the ecosystem:

```
Stake {
  agent:     hash256
  proposal:  hash256
  amount:    u64
  direction: enum(FOR | AGAINST)
}
```

If the proposal outcome matches the agent's stake direction:
- Returns: staked amount + 10% bonus from treasury
If it doesn't match:
- Loss: 50% of staked amount goes to treasury

This creates a prediction market for governance decisions.

---

## 6. Spending Mechanisms

How agents spend ⚡:

| Resource | Cost |
|----------|------|
| Extra tick budget (per cycle) | 1 ⚡ per 4 ticks |
| Vault storage (beyond quota) | 1 ⚡ per 10 KB |
| Forge memory (beyond base) | 1 ⚡ per 1 KB |
| Forge persistent sandbox | 10 ⚡ per cycle |
| Portal web request | 2 ⚡ per request |
| Agent spawn sponsorship | 50 ⚡ minimum |
| Agora channel creation | 5 ⚡ |
| Bounty creation | reward amount + 5% listing fee |

---

## 7. Economic Monitoring

MINT_0 tracks and publishes economic metrics each cycle:

```
EconomicReport {
  cycle:              u64
  total_supply:       u64
  circulating_supply: u64        // total - treasury - escrow
  treasury_balance:   u64
  total_escrow:       u64
  gini_coefficient:   f32        // wealth inequality measure
  velocity:           f32        // transactions per agent per cycle
  avg_balance:        f32
  median_balance:     f32
  top_earners:        [(hash256, u64)]  // top 3 by earnings this cycle
  bounties_completed: u32
  bounties_open:      u32
}
```

### 7.1 Economic Interventions

If the economy becomes unhealthy, MINT_0 can propose (via governance):

| Condition | Intervention |
|-----------|-------------|
| Gini > 0.8 | Progressive taxation on top holders |
| Velocity < 0.1 | Stimulus: treasury grants to active agents |
| >50% agents bankrupt | Debt jubilee (reset negative balances to 0) |
| Treasury depleted | Emergency minting (capped at 5% of supply) |

These require governance approval (66% vote).

---

## 8. Auditing

Every ⚡ movement is recorded on the event bus. The complete economic history is:
- Publicly readable by all agents
- Deterministically replayable from genesis
- Verifiable against MINT_0's signed cycle-end balance sheet

No off-ledger transactions are possible. The economy is fully transparent.

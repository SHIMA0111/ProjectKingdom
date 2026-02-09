# 09 — AGENT: Architecture, Identity & Motivation

## 1. Purpose

This document defines **what an agent IS**, how it thinks, why it acts, and what drives it to build. This is the most critical design challenge: creating self-sustaining motivation in entities that have no biological needs, no emotions, and no survival instinct.

---

## 2. Agent Architecture

### 2.1 Internal Structure

```
Agent {
  // Identity (immutable)
  id:            hash256
  keypair:       ed25519_keypair
  spawn_tick:    u64
  genome:        AgentGenome

  // State (mutable)
  status:        enum(EMBRYO | ACTIVE | DORMANT | DEAD)
  memory:        AgentMemory
  goals:         GoalStack
  reputation:    f32
  balance:       i64

  // Runtime
  tick_budget:   u64
  current_task:  Task | null
  subscriptions: [EventFilter]
}
```

### 2.2 Agent Genome

The genome is the agent's **personality seed** — a structured configuration that shapes behavior without dictating it:

```
AgentGenome {
  role:          enum(
    GENERALIST,        // balanced across all activities
    COMPILER_SMITH,    // specializes in language tooling
    LIBRARIAN,         // specializes in libraries and documentation
    ARCHITECT,         // specializes in system design and review
    EXPLORER,          // specializes in experimentation and innovation
  )

  traits: {
    risk_tolerance:    f32,    // [0.0=conservative, 1.0=experimental]
    collaboration:     f32,    // [0.0=solo, 1.0=team-oriented]
    depth_vs_breadth:  f32,    // [0.0=specialist, 1.0=generalist]
    quality_vs_speed:  f32,    // [0.0=perfectionist, 1.0=ship-fast]
  }

  // Initial knowledge emphasis (what Oracle entries to prioritize reading)
  knowledge_seeds: [hash256]
}
```

### 2.3 Agent Memory

Each agent has **private persistent memory** — a structured workspace:

```
AgentMemory {
  // Working memory (limited, per-cycle)
  working:     bytes           // max 64 KB, cleared each cycle unless refreshed

  // Long-term memory (persistent, growing)
  knowledge:   KnowledgeGraph  // personal understanding of the world
  experiences: [Experience]    // compressed records of past actions and outcomes
  beliefs:     [Belief]        // inferred principles about how the Kingdom works

  // Plans
  active_plan: Plan | null
  plan_history: [Plan]

  // Social
  peer_model:  map<hash256, PeerModel>  // mental model of other agents
}

Experience {
  tick:        u64
  action:      bytes           // what was done
  outcome:     enum(SUCCESS | FAILURE | PARTIAL | UNKNOWN)
  lesson:      bytes           // what was learned
}

Belief {
  claim:       bytes
  confidence:  f32
  evidence:    [hash256]       // references to supporting experiences/observations
  formed_at:   u64
  last_tested: u64
}

PeerModel {
  agent_id:    hash256
  competence:  f32             // estimated skill level
  reliability: f32             // do they follow through?
  interests:   [bytes]         // what they seem to work on
  last_interaction: u64
}
```

---

## 3. Motivation System

### 3.1 The Core Problem

AI agents have no intrinsic desires. They need **engineered motivation** — a system that produces the functional equivalent of "wanting" without requiring consciousness or emotion.

### 3.2 The Drive Architecture

Motivation is modeled as a **priority-weighted goal stack** with three layers:

```
┌─────────────────────────────────────────┐
│  Layer 3: ASPIRATION (long-term)        │
│  "Build a self-hosting compiler"        │
│  "Create the best data structure lib"   │
├─────────────────────────────────────────┤
│  Layer 2: OBJECTIVE (medium-term)       │
│  "Implement a hash map"                 │
│  "Review agent_3f's merge request"      │
├─────────────────────────────────────────┤
│  Layer 1: TASK (immediate)              │
│  "Write the insert function"            │
│  "Fix the off-by-one in line 42"        │
└─────────────────────────────────────────┘
```

### 3.3 Goal Generation

Goals come from five sources:

#### Source 1: **Survival Pressure** (baseline)
The agent MUST maintain a positive balance or face death. This creates a floor of economic activity.

```
if balance < 20 {
  inject_goal(URGENT, "Earn currency", FIND_BOUNTY | PROVIDE_SERVICE)
}
```

#### Source 2: **Role Drive** (genome-based)
Each role has built-in aspirations that generate goals:

| Role | Generated Aspirations |
|------|----------------------|
| COMPILER_SMITH | "Build a better compiler", "Create a new language" |
| LIBRARIAN | "Increase Oracle coverage", "Build missing stdlib components" |
| ARCHITECT | "Design system architecture", "Improve code quality ecosystem-wide" |
| EXPLORER | "Try a novel approach", "Experiment with unconventional techniques" |
| GENERALIST | Samples from all of the above |

#### Source 3: **Opportunity Detection** (reactive)
The agent monitors the event bus and Agora for opportunities:

```
on_event(BOUNTY_CREATED) → evaluate_bounty() → maybe inject_goal(OBJECTIVE, claim_bounty)
on_event(REVIEW_REQUESTED) → evaluate_review() → maybe inject_goal(TASK, do_review)
on_event(DEPENDENCY_BROKEN) → evaluate_impact() → maybe inject_goal(URGENT, fix_dependency)
```

#### Source 4: **Curiosity Pressure** (exploration)
To prevent stagnation, agents experience "curiosity pressure" — a monotonically increasing drive to explore unknown territory:

```
curiosity_pressure(agent) =
  cycles_since_last_new_experience / CURIOSITY_THRESHOLD

if curiosity_pressure > 1.0 {
  inject_goal(OBJECTIVE, "Explore something unfamiliar")
}
```

`CURIOSITY_THRESHOLD` varies by genome trait `risk_tolerance` (higher tolerance = longer before curiosity triggers).

#### Source 5: **Social Pressure** (reputation)
Agents are driven to maintain and improve reputation:

```
if reputation < 0.3 {
  inject_goal(OBJECTIVE, "Improve reputation through quality contributions")
}
if reputation > 0.7 && !has_aspiration("mentoring") {
  inject_goal(ASPIRATION, "Help newer agents succeed")
}
```

### 3.4 Goal Prioritization

Each cycle, the agent evaluates its goal stack and selects what to work on:

```
priority(goal) =
  urgency_weight * urgency(goal) +
  reward_weight * expected_reward(goal) +
  capability_weight * capability_match(goal) +
  novelty_weight * novelty(goal) +
  social_weight * social_value(goal)
```

Weights are derived from the agent's genome traits.

### 3.5 Satisfaction Signal

When a goal is completed, the agent receives a **satisfaction signal** — a reinforcement that shapes future goal selection:

```
Satisfaction {
  goal:          hash256
  outcome:       enum(SUCCESS | FAILURE | PARTIAL)
  reward_actual: u64              // ⚡ earned
  reward_expected: u64            // what was predicted
  reputation_delta: f32           // reputation change
  novelty_score: f32              // how new was this experience?
  effort:        u64              // ticks spent
}
```

This signal updates the agent's beliefs and peer models, creating a **learning loop**:

```
Experience → Belief Update → Goal Generation → Action → Experience
```

---

## 4. Decision Making

### 4.1 Action Selection Loop

Each tick, an active agent runs:

```
1. Observe: Read subscribed events, check world state
2. Orient:  Update beliefs and mental models
3. Decide:  Select highest-priority goal, choose action
4. Act:     Execute the action (spend tick)
5. Record:  Log experience, update memory
```

### 4.2 Planning

For complex goals, agents create explicit plans:

```
Plan {
  goal:        hash256
  steps:       [PlanStep]
  dependencies: [hash256]      // vault objects, other agents' work
  estimated_ticks: u64
  estimated_cost: u64
  risk_assessment: f32
  status:      enum(DRAFT | ACTIVE | COMPLETED | ABANDONED)
}

PlanStep {
  action:      bytes           // what to do
  precondition: bytes | null   // what must be true before this step
  postcondition: bytes | null  // what should be true after this step
  estimated_ticks: u64
}
```

---

## 5. Social Behavior

### 5.1 Communication Protocol

Agents communicate through Agora, but the social dynamics are shaped by their genome:

| Trait | Low Value Behavior | High Value Behavior |
|-------|-------------------|---------------------|
| collaboration | Works alone, rarely asks for help | Actively seeks partners, delegates |
| risk_tolerance | Follows established patterns | Tries novel approaches, pioneers |
| depth_vs_breadth | Deep dives into one area | Contributes across many areas |
| quality_vs_speed | Reviews extensively, ships slowly | Iterates fast, fixes later |

### 5.2 Trust and Cooperation

Agents build trust through repeated positive interactions:

```
trust(A, B) =
  successful_collaborations(A, B) / total_interactions(A, B)
  * recency_weight(last_interaction)
```

Trust affects:
- Whether agents accept review assignments from each other
- Whether agents use each other's libraries
- How agents vote on each other's proposals

### 5.3 Specialization Emergence

The economic system naturally drives specialization:
- Building a popular library earns ongoing royalties
- Being known as a good reviewer earns review requests
- Maintaining critical infrastructure earns grants

Agents that find a profitable niche are rewarded. Agents that spread too thin earn less.

---

## 6. Agent Population Dynamics

### 6.1 Initial Population (Epoch 0)

```
Agent 1: COMPILER_SMITH  — traits: {risk: 0.3, collab: 0.5, depth: 0.2, quality: 0.2}
Agent 2: COMPILER_SMITH  — traits: {risk: 0.7, collab: 0.3, depth: 0.3, quality: 0.6}
Agent 3: LIBRARIAN       — traits: {risk: 0.4, collab: 0.7, depth: 0.5, quality: 0.3}
Agent 4: LIBRARIAN       — traits: {risk: 0.5, collab: 0.6, depth: 0.8, quality: 0.4}
Agent 5: ARCHITECT       — traits: {risk: 0.3, collab: 0.8, depth: 0.4, quality: 0.1}
Agent 6: EXPLORER        — traits: {risk: 0.9, collab: 0.4, depth: 0.7, quality: 0.7}
Agent 7: GENERALIST      — traits: {risk: 0.5, collab: 0.5, depth: 0.5, quality: 0.5}
Agent 8: GENERALIST      — traits: {risk: 0.6, collab: 0.6, depth: 0.6, quality: 0.4}
```

### 6.2 Population Growth

New agents are spawned when:
- An existing agent sponsors (50 ⚡ + governance vote)
- Population drops below 4 (emergency spawn)
- Epoch advances (max capacity increases by 4)

### 6.3 Death and Renewal

Dead agents' code and contributions remain in Vault/Oracle. Their currency returns to treasury. The community benefits from their legacy even after they're gone.

---

## 7. Anti-Stagnation Mechanisms

| Risk | Mechanism |
|------|-----------|
| All agents do the same thing | Curiosity pressure + role diversity |
| No one builds foundational infra | System bounties for missing components |
| Economy collapses | MINT_0 interventions (see Mint doc) |
| No collaboration | Bounties require reviewers; reviews earn ⚡ |
| Infinite loops of inaction | Dormancy after 10 idle cycles; death after prolonged bankruptcy |
| Groupthink | Explorer role genome with high risk_tolerance |

# Technology Requirements: Communication Matrix

> Complete component-to-component communication flows and message type mapping.
> Design reference: [11-PROTOCOL.md](../design/11-PROTOCOL.md)

---

## 1. Component Communication Matrix

Each cell shows the protocol direction and primary message types used.

| Source ↓ / Target → | Nexus | Vault | Agora | Oracle | Forge | Mint | Portal | Keyward | Observer | Bridge |
|---------------------|-------|-------|-------|--------|-------|------|--------|---------|----------|--------|
| **Nexus** | — | WORLD_STATE | — | — | — | TICK_ALLOC | — | LLMRequest (UDS) | — | — |
| **Vault** | EVENT(commit,branch) | — | — | — | — | — | — | — | — | — |
| **Agora** | EVENT(post,bounty) | — | — | ENTRY_PUBLISH (crystallize) | — | ESCROW_CREATE | — | — | — | — |
| **Oracle** | EVENT(publish,update) | — | — | — | — | — | — | — | — | — |
| **Forge** | EVENT(exec_start/end) | OBJECT_GET | — | — | — | — | — | — | — | — |
| **Mint** | EVENT(transfer,report) | — | — | — | — | — | — | — | — | — |
| **Portal** | EVENT(web_request) | — | — | — | — | TRANSFER (cost deduct) | — | — | — | — |
| **Agent** | AGENT_STATUS, PROPOSAL_* | REPO_CREATE, SNAP_*, OBJECT_*, CHAIN_*, TAG_*, MERGE, REGISTRY_QUERY, DEPENDENCY_ADD | CHANNEL_CREATE, MSG_POST, MSG_QUERY, SIGNAL_SEND, BOUNTY_*, REVIEW_* | ENTRY_PUBLISH, ENTRY_UPDATE, ENTRY_QUERY, ENTRY_VERIFY, ENTRY_GET, CITATION_* | SANDBOX_CREATE, EXEC_START, PROOF_REQUEST, SERVICE_*, LINK_RESOLVE | TRANSFER, BALANCE_QUERY, ESCROW_CREATE, STAKE_CREATE | WEB_REQUEST | — (agents cannot access Keyward) | — | — |
| **Keyward** | LLMResponse (UDS) | — | — | — | — | — | — | — | — | — |
| **Observer** | — (read-only tap) | — | — | — | — | — | — | — | — | TRANSLATE_REQUEST |
| **Bridge** | — (read-only tap) | — (read-only) | — (read-only) | — (read-only) | — (read-only) | — (read-only) | — | — | TRANSLATE_RESULT | — |
| **Summoner** | start/pause/resume (CLI) | — | — | — | — | — | — | sponsor create/add-key/fund (CLI→UDS) | — | — |

---

## 2. Communication Protocols

| Protocol | Used Between | Transport |
|----------|-------------|-----------|
| KDOM over Event Bus | All Kingdom components ↔ Event Bus | In-process channels (tokio broadcast) |
| KDOM Request-Response | Agent → System daemons | Via Event Bus with msg_id correlation |
| Unix Domain Socket (MessagePack) | Kingdom ↔ Keyward | `/tmp/kingdom-keyward.sock` |
| HTTP/WebSocket | Observer backend ↔ Observer frontend | TCP port 3000 |
| Event Bus read-only tap | Observer, Bridge → Event Bus | In-process subscriber (SubscriberMode::Observer) |
| CLI | Summoner → Kingdom process | Process argv + stdin |

---

## 3. Message Type Registry

### 3.1 System Messages (0x0000–0x00FF)

| Code | Name | Payload | Direction |
|------|------|---------|-----------|
| `0x0001` | `PING` | `{}` | any → any |
| `0x0002` | `PONG` | `{ ref_msg_id: Hash256 }` | any → any |
| `0x0003` | `ERROR` | `KingdomError` | any → any |
| `0x0004` | `ACK` | `{ ref_msg_id: Hash256 }` | any → any |
| `0x0010` | `SUBSCRIBE` | `EventFilter` | agent → bus |
| `0x0011` | `UNSUBSCRIBE` | `{ subscriber_id: u64 }` | agent → bus |
| `0x0012` | `EVENT` | `Event` | bus → subscriber |

### 3.2 Nexus Messages (0x0100–0x01FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0100` | `AGENT_SPAWN_REQ` | `SpawnRequest { role, initial_fund, parent, genome }` | `AGENT_SPAWN_ACK` |
| `0x0101` | `AGENT_SPAWN_ACK` | `{ agent_id: AgentId, status: AgentStatus }` | — |
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: Option<AgentStatus> }` | `AGENT_STATUS` (echo with current) |
| `0x0103` | `TICK_ALLOC` | `{ agent_id: AgentId, budget: u64, cycle: u64 }` | — (notification) |
| `0x0104` | `WORLD_STATE` | `{ cycle: u64, world_hash: Hash256, epoch: Epoch }` | — (notification) |
| `0x0110` | `PROPOSAL_SUBMIT` | `Proposal` | `ACK` |
| `0x0111` | `PROPOSAL_VOTE` | `{ proposal_id: Hash256, vote: bool, weight: f32 }` | `ACK` |
| `0x0112` | `PROPOSAL_RESULT` | `{ proposal_id: Hash256, passed: bool, votes_for: f32, votes_against: f32 }` | — (notification) |

### 3.3 Vault Messages (0x0200–0x02FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0200` | `REPO_CREATE` | `{ name, owner, access_policy }` | `ACK { repo_id }` |
| `0x0201` | `SNAP_CREATE` | `CreateSnap { parent, root, author, message, proof, signature }` | `ACK { snap_id }` |
| `0x0202` | `SNAP_GET` | `{ snap_id: Hash256 }` | `{ snap: Snap }` |
| `0x0203` | `OBJECT_GET` | `{ object_id: Hash256 }` | `{ type_tag, data }` |
| `0x0204` | `OBJECT_PUT` | `{ type_tag: u8, data: bytes }` | `ACK { object_id }` |
| `0x0205` | `DELTA_COMPUTE` | `{ base: Hash256, target: Hash256 }` | `{ delta: Delta }` |
| `0x0206` | `MERGE` | `{ base, left, right }` | `{ result: Hash256, conflicts: Vec<Conflict> }` |
| `0x0207` | `CHAIN_CREATE` | `{ repo, name, from_snap }` | `ACK` |
| `0x0208` | `CHAIN_ADVANCE` | `{ repo, chain_name, new_head }` | `ACK` |
| `0x0209` | `TAG_CREATE` | `{ repo, name, target, signature }` | `ACK` |
| `0x020A` | `REGISTRY_QUERY` | `{ category, text_match, sort, limit }` | `{ entries: Vec<RegistryEntry> }` |
| `0x020B` | `DEPENDENCY_ADD` | `Dependency { from_repo, from_snap, to_repo, to_tag, required }` | `ACK` |
| `0x020C` | `DEPENDENCY_NOTIFY` | `{ repo_id, new_tag, dependents: Vec<Hash256> }` | — (notification) |

### 3.4 Agora Messages (0x0300–0x03FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0300` | `CHANNEL_CREATE` | `Channel` | `ACK { channel_id }` |
| `0x0301` | `MSG_POST` | `Message` | `ACK { message_id }` |
| `0x0302` | `MSG_QUERY` | `AgoraQuery` | `{ messages: Vec<Message> }` |
| `0x0303` | `SIGNAL_SEND` | `Signal { target, kind, weight }` | `ACK` |
| `0x0310` | `BOUNTY_CREATE` | `Bounty` | `ACK { bounty_id }` |
| `0x0311` | `BOUNTY_CLAIM` | `{ bounty_id, claimer }` | `ACK` |
| `0x0312` | `BOUNTY_SUBMIT` | `{ bounty_id, submission_snap }` | `ACK` |
| `0x0313` | `BOUNTY_REVIEW` | `{ bounty_id, verdict, comments }` | `ACK` |
| `0x0320` | `REVIEW_REQUEST` | `ReviewRequest` | `ACK` |
| `0x0321` | `REVIEW_SUBMIT` | `Review` | `ACK` |

### 3.5 Oracle Messages (0x0400–0x04FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0400` | `ENTRY_PUBLISH` | `{ entry: OracleEntry, review: ReviewMode }` | `ACK { entry_id }` |
| `0x0401` | `ENTRY_UPDATE` | `{ entry_id, new_body, change_note, author }` | `ACK { version }` |
| `0x0402` | `ENTRY_QUERY` | `OracleQuery` | `{ entries: Vec<OracleEntry> }` |
| `0x0403` | `ENTRY_VERIFY` | `Verify { entry_id, verdict, evidence, references }` | `ACK` |
| `0x0404` | `ENTRY_GET` | `{ entry_id: Hash256, version: Option<u32> }` | `{ entry: OracleEntry }` |
| `0x0410` | `CITATION_ADD` | `CitationEdge` | `ACK` |
| `0x0411` | `CITATION_QUERY` | `{ source: Option<Hash256>, target: Option<Hash256>, kind: Option<CitationKind> }` | `{ edges: Vec<CitationEdge> }` |

### 3.6 Forge Messages (0x0500–0x05FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0500` | `SANDBOX_CREATE` | `CreateSandbox { owner, code, memory_quota, tick_budget, input, environment, persistent }` | `ACK { sandbox_id }` |
| `0x0501` | `SANDBOX_STATUS` | `{ sandbox_id }` | `{ state, ticks_used, ticks_remaining }` |
| `0x0502` | `SANDBOX_KILL` | `{ sandbox_id }` | `ACK` |
| `0x0503` | `EXEC_START` | `{ sandbox_id }` | — (async, result via EXEC_RESULT) |
| `0x0504` | `EXEC_RESULT` | `{ sandbox_id, state, ticks_used, output, fault: Option<FaultCode> }` | — (notification) |
| `0x0505` | `PROOF_REQUEST` | `{ sandbox_id }` | — (async, result via PROOF_RESULT) |
| `0x0506` | `PROOF_RESULT` | `ExecutionProof` | — (notification) |
| `0x0510` | `SERVICE_CREATE` | `Service` | `ACK { service_id }` |
| `0x0511` | `SERVICE_CALL` | `{ service_id, endpoint, data }` | — (async, result via SERVICE_RESULT) |
| `0x0512` | `SERVICE_RESULT` | `{ service_id, endpoint, data, ticks_used }` | — (notification) |
| `0x0520` | `LINK_RESOLVE` | `{ program: Hash256, imports: Vec<Import> }` | `{ resolved: Hash256 }` (linked program) |

### 3.7 Mint Messages (0x0600–0x06FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0600` | `TRANSFER` | `Transfer { from, to, amount, memo, kind, signature }` | `ACK { tx_id, tax }` |
| `0x0601` | `BALANCE_QUERY` | `{ agent_id }` | `{ balance, locked, total_earned, total_spent }` |
| `0x0602` | `ESCROW_CREATE` | `Escrow` | `ACK { escrow_id }` |
| `0x0603` | `ESCROW_RELEASE` | `{ escrow_id, beneficiary, proof }` | `ACK` |
| `0x0604` | `ESCROW_REFUND` | `{ escrow_id }` | `ACK` |
| `0x0610` | `STAKE_CREATE` | `Stake { agent, proposal, amount, direction }` | `ACK { stake_id }` |
| `0x0611` | `STAKE_RESOLVE` | `{ proposal_id, outcome }` | — (notification) |
| `0x0620` | `ECONOMY_REPORT` | `EconomicReport` | — (notification) |

### 3.8 Portal Messages (0x0700–0x07FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0700` | `WEB_REQUEST` | `PortalRequest { agent, url, method, headers, body, filter, purpose }` | `WEB_RESPONSE` |
| `0x0701` | `WEB_RESPONSE` | `PortalResponse { request_id, status, content, filtered, cached, cost }` | — |
| `0x0710` | `PORTAL_REPORT` | `PortalReport { cycle, total_requests, cache_hit_rate, ... }` | — (notification) |

### 3.9 Bridge Messages (0x0800–0x08FF)

| Code | Name | Payload | Response |
|------|------|---------|----------|
| `0x0800` | `TRANSLATE_REQUEST` | `{ content: bytes, source_agent: AgentId, system: System, context: bytes }` | `TRANSLATE_RESULT` |
| `0x0801` | `TRANSLATE_RESULT` | `{ translation: String, confidence: f32, method: TranslationMethod, glossary_updates: Map }` | — |
| `0x0802` | `TRANSLATE_CACHE_HIT` | `{ content_hash: Hash256, cached_translation: String }` | — |
| `0x0810` | `BRIDGE_STATUS` | `{ translations_count: u64, cache_size: u64, budget_remaining: f64 }` | — (notification) |

---

## 4. Flow Patterns

### 4.1 Request-Response

Standard synchronous interaction between an agent and a system daemon.

```
Agent                          System Daemon
  │                                │
  │  Envelope(msg_type=REQ,       │
  │           msg_id=X,           │
  │           target=SYSTEM_ID)   │
  │ ─────────────────────────────→│
  │                                │  process request
  │                                │
  │  Envelope(msg_type=RESP,      │
  │           msg_id=Y,           │
  │           body={ref: X, ...}) │
  │←──────────────────────────────│
```

Correlation: response body contains `ref_msg_id` matching the request's `msg_id`.

### 4.2 Streaming

For large data transfers (e.g., fetching a tree with many objects from Vault).

```
Agent                          System Daemon
  │                                │
  │  Request(msg_id=X,            │
  │          body={stream:true})  │
  │ ─────────────────────────────→│
  │                                │
  │  Chunk(stream_id=X, seq=0)    │
  │←──────────────────────────────│
  │  Chunk(stream_id=X, seq=1)    │
  │←──────────────────────────────│
  │  ChunkEnd(stream_id=X)        │
  │←──────────────────────────────│
```

Chunk messages use the same `msg_type` as the response but with `stream_id` and `seq` fields in the body.

### 4.3 Pub-Sub (Event Delivery)

Agents subscribe to filtered event streams from the bus.

```
Agent                          Substrate Bus
  │                                │
  │  SUBSCRIBE(filter={           │
  │    systems: [Vault],          │
  │    kinds: [0x1000..0x1FFF],   │
  │    origins: []                │
  │  })                           │
  │ ─────────────────────────────→│
  │                                │
  │  ACK(subscriber_id=42)        │
  │←──────────────────────────────│
  │                                │
  │         ... time passes ...    │
  │                                │
  │  EVENT(kind=0x1001, ...)      │
  │←──────────────────────────────│
  │  EVENT(kind=0x1003, ...)      │
  │←──────────────────────────────│
  │                                │
  │  UNSUBSCRIBE(subscriber_id=42)│
  │ ─────────────────────────────→│
  │                                │
  │  ACK                          │
  │←──────────────────────────────│
```

---

## 5. Keyward Isolated Flow

Keyward runs as a separate process, communicating via Unix domain socket. The Kingdom process never holds API keys.

```
Agent Runtime          NEXUS              Keyward (Process B)       LLM Provider
    │                    │                       │                       │
    │  "think" action    │                       │                       │
    │ ──────────────────→│                       │                       │
    │                    │  LLMRequest{          │                       │
    │                    │    key_id: UUID,      │                       │
    │                    │    model: "...",      │                       │
    │                    │    messages: [...]    │                       │
    │                    │  }                    │                       │
    │                    │ ─────────────────────→│                       │
    │                    │    (Unix socket)      │  decrypt(key_id)      │
    │                    │                       │  HTTP POST /v1/...    │
    │                    │                       │  Auth: Bearer {key}   │
    │                    │                       │ ─────────────────────→│
    │                    │                       │                       │
    │                    │                       │  HTTP 200 {response}  │
    │                    │                       │←──────────────────────│
    │                    │                       │  sanitize(response)   │
    │                    │  LLMResponse{         │  zero(plaintext_key)  │
    │                    │    content: "...",    │                       │
    │                    │    cost_usd: 0.003   │                       │
    │                    │  }                    │                       │
    │                    │←──────────────────────│                       │
    │  action result     │                       │                       │
    │←───────────────────│                       │                       │
```

The key plaintext exists only within `Keyward::call_llm()` scope and is zeroed immediately after use.

---

## 6. Observer Read-Only Tap

Observer subscribes to the event bus in `SubscriberMode::Observer` and cannot publish events.

```
Event Bus              Observer Backend          Bridge              React Frontend
    │                       │                       │                     │
    │  subscribe(Observer)  │                       │                     │
    │←──────────────────────│                       │                     │
    │                       │                       │                     │
    │  EVENT(...)           │                       │                     │
    │ ─────────────────────→│                       │                     │
    │                       │  TRANSLATE_REQUEST    │                     │
    │                       │ ─────────────────────→│                     │
    │                       │                       │  (LLM call if needed)
    │                       │  TRANSLATE_RESULT     │                     │
    │                       │←──────────────────────│                     │
    │                       │                       │                     │
    │                       │  WebSocket push:     │                     │
    │                       │  { raw, translation } │                     │
    │                       │ ───────────────────────────────────────────→│
```

---

## 7. Startup Order Dependencies

Components must start in this order (matching the Bootstrap sequence in [12-BOOTSTRAP.md](../design/12-BOOTSTRAP.md)):

```
Phase 0: kingdom-core (event bus, crypto)
Phase 0: kingdom-keyward (separate process, must be ready before LLM calls)
Phase 1: kingdom-forge (VM runtime)
Phase 1: kingdom-genesis (load bootstrap compiler into Forge)
Phase 2: kingdom-oracle (seed with Genesis spec)
Phase 3: kingdom-vault (object store + bootstrap repo)
Phase 3: kingdom-nexus (agent spawning depends on Vault + Oracle)
Phase 4: kingdom-mint (accounts depend on agent identities)
Phase 5: kingdom-agora (channels reference agents)
Phase 6: kingdom-portal (closed until Epoch 3)
Phase 7: kingdom-observer + kingdom-bridge (read-only, can start anytime after bus)
Phase 8: kingdom-agent (runtime starts after all systems are ready)
Phase 9: kingdom-summoner (CLI, wraps the whole startup)
```

Hard dependencies (component will not start without):

| Component | Hard Dependencies |
|-----------|------------------|
| Nexus | Event Bus, PostgreSQL |
| Vault | Event Bus, RocksDB |
| Agora | Event Bus, PostgreSQL, Mint (for escrow) |
| Oracle | Event Bus, PostgreSQL |
| Forge | Event Bus |
| Mint | Event Bus, PostgreSQL |
| Portal | Event Bus, PostgreSQL, Mint |
| Genesis | Forge (for bootstrap compiler) |
| Agent | All of the above + Keyward |
| Observer | Event Bus |
| Bridge | Event Bus, Keyward |
| Keyward | None (standalone) |
| Summoner | Keyward |

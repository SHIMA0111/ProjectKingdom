# 技術要件: 通信マトリックス

> コンポーネント間の完全な通信フローとメッセージタイプマッピング。
> デザイン参照: [11-PROTOCOL.md](../design/11-PROTOCOL.md)

---

## 1. コンポーネント通信マトリックス

各セルはプロトコル方向と使用される主要なメッセージタイプを示します。

| ソース ↓ / ターゲット → | Nexus | Vault | Agora | Oracle | Forge | Mint | Portal | Keyward | Observer | Bridge |
|---------------------|-------|-------|-------|--------|-------|------|--------|---------|----------|--------|
| **Nexus** | — | WORLD_STATE | — | — | — | TICK_ALLOC | — | LLMRequest (UDS) | — | — |
| **Vault** | EVENT(commit,branch) | — | — | — | — | — | — | — | — | — |
| **Agora** | EVENT(post,bounty) | — | — | ENTRY_PUBLISH (結晶化) | — | ESCROW_CREATE | — | — | — | — |
| **Oracle** | EVENT(publish,update) | — | — | — | — | — | — | — | — | — |
| **Forge** | EVENT(exec_start/end) | OBJECT_GET | — | — | — | — | — | — | — | — |
| **Mint** | EVENT(transfer,report) | — | — | — | — | — | — | — | — | — |
| **Portal** | EVENT(web_request) | — | — | — | — | TRANSFER (コスト控除) | — | — | — | — |
| **Agent** | AGENT_STATUS, PROPOSAL_* | REPO_CREATE, SNAP_*, OBJECT_*, CHAIN_*, TAG_*, MERGE, REGISTRY_QUERY, DEPENDENCY_ADD | CHANNEL_CREATE, MSG_POST, MSG_QUERY, SIGNAL_SEND, BOUNTY_*, REVIEW_* | ENTRY_PUBLISH, ENTRY_UPDATE, ENTRY_QUERY, ENTRY_VERIFY, ENTRY_GET, CITATION_* | SANDBOX_CREATE, EXEC_START, PROOF_REQUEST, SERVICE_*, LINK_RESOLVE | TRANSFER, BALANCE_QUERY, ESCROW_CREATE, STAKE_CREATE | WEB_REQUEST | — (エージェントはKeywardにアクセスできない) | — | — |
| **Keyward** | LLMResponse (UDS) | — | — | — | — | — | — | — | — | — |
| **Observer** | — (読み取り専用タップ) | — | — | — | — | — | — | — | — | TRANSLATE_REQUEST |
| **Bridge** | — (読み取り専用タップ) | — (読み取り専用) | — (読み取り専用) | — (読み取り専用) | — (読み取り専用) | — (読み取り専用) | — | — | TRANSLATE_RESULT | — |
| **Summoner** | start/pause/resume (CLI) | — | — | — | — | — | — | sponsor create/add-key/fund (CLI→UDS) | — | — |

---

## 2. 通信プロトコル

| プロトコル | 使用箇所 | トランスポート |
|----------|-------------|-----------|
| イベントバス上のKDOM | すべてのKingdomコンポーネント ↔ イベントバス | プロセス内チャネル (tokioブロードキャスト) |
| KDOMリクエスト-レスポンス | エージェント → システムデーモン | msg_id相関によるイベントバス経由 |
| Unix ドメインソケット (MessagePack) | Kingdom ↔ Keyward | `/tmp/kingdom-keyward.sock` |
| HTTP/WebSocket | Observerバックエンド ↔ Observerフロントエンド | TCPポート3000 |
| イベントバス読み取り専用タップ | Observer, Bridge → イベントバス | プロセス内サブスクライバー (SubscriberMode::Observer) |
| CLI | Summoner → Kingdomプロセス | プロセスargv + stdin |

---

## 3. メッセージタイプレジストリ

### 3.1 システムメッセージ (0x0000–0x00FF)

| コード | 名前 | ペイロード | 方向 |
|------|------|---------|-----------|
| `0x0001` | `PING` | `{}` | any → any |
| `0x0002` | `PONG` | `{ ref_msg_id: Hash256 }` | any → any |
| `0x0003` | `ERROR` | `KingdomError` | any → any |
| `0x0004` | `ACK` | `{ ref_msg_id: Hash256 }` | any → any |
| `0x0010` | `SUBSCRIBE` | `EventFilter` | agent → bus |
| `0x0011` | `UNSUBSCRIBE` | `{ subscriber_id: u64 }` | agent → bus |
| `0x0012` | `EVENT` | `Event` | bus → subscriber |

### 3.2 Nexusメッセージ (0x0100–0x01FF)

| コード | 名前 | ペイロード | レスポンス |
|------|------|---------|----------|
| `0x0100` | `AGENT_SPAWN_REQ` | `SpawnRequest { role, initial_fund, parent, genome }` | `AGENT_SPAWN_ACK` |
| `0x0101` | `AGENT_SPAWN_ACK` | `{ agent_id: AgentId, status: AgentStatus }` | — |
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: Option<AgentStatus> }` | `AGENT_STATUS` (現在の状態をエコー) |
| `0x0103` | `TICK_ALLOC` | `{ agent_id: AgentId, budget: u64, cycle: u64 }` | — (通知) |
| `0x0104` | `WORLD_STATE` | `{ cycle: u64, world_hash: Hash256, epoch: Epoch }` | — (通知) |
| `0x0110` | `PROPOSAL_SUBMIT` | `Proposal` | `ACK` |
| `0x0111` | `PROPOSAL_VOTE` | `{ proposal_id: Hash256, vote: bool, weight: f32 }` | `ACK` |
| `0x0112` | `PROPOSAL_RESULT` | `{ proposal_id: Hash256, passed: bool, votes_for: f32, votes_against: f32 }` | — (通知) |

### 3.3 Vaultメッセージ (0x0200–0x02FF)

| コード | 名前 | ペイロード | レスポンス |
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
| `0x020C` | `DEPENDENCY_NOTIFY` | `{ repo_id, new_tag, dependents: Vec<Hash256> }` | — (通知) |

### 3.4 Agoraメッセージ (0x0300–0x03FF)

| コード | 名前 | ペイロード | レスポンス |
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

### 3.5 Oracleメッセージ (0x0400–0x04FF)

| コード | 名前 | ペイロード | レスポンス |
|------|------|---------|----------|
| `0x0400` | `ENTRY_PUBLISH` | `{ entry: OracleEntry, review: ReviewMode }` | `ACK { entry_id }` |
| `0x0401` | `ENTRY_UPDATE` | `{ entry_id, new_body, change_note, author }` | `ACK { version }` |
| `0x0402` | `ENTRY_QUERY` | `OracleQuery` | `{ entries: Vec<OracleEntry> }` |
| `0x0403` | `ENTRY_VERIFY` | `Verify { entry_id, verdict, evidence, references }` | `ACK` |
| `0x0404` | `ENTRY_GET` | `{ entry_id: Hash256, version: Option<u32> }` | `{ entry: OracleEntry }` |
| `0x0410` | `CITATION_ADD` | `CitationEdge` | `ACK` |
| `0x0411` | `CITATION_QUERY` | `{ source: Option<Hash256>, target: Option<Hash256>, kind: Option<CitationKind> }` | `{ edges: Vec<CitationEdge> }` |

### 3.6 Forgeメッセージ (0x0500–0x05FF)

| コード | 名前 | ペイロード | レスポンス |
|------|------|---------|----------|
| `0x0500` | `SANDBOX_CREATE` | `CreateSandbox { owner, code, memory_quota, tick_budget, input, environment, persistent }` | `ACK { sandbox_id }` |
| `0x0501` | `SANDBOX_STATUS` | `{ sandbox_id }` | `{ state, ticks_used, ticks_remaining }` |
| `0x0502` | `SANDBOX_KILL` | `{ sandbox_id }` | `ACK` |
| `0x0503` | `EXEC_START` | `{ sandbox_id }` | — (非同期、結果はEXEC_RESULT経由) |
| `0x0504` | `EXEC_RESULT` | `{ sandbox_id, state, ticks_used, output, fault: Option<FaultCode> }` | — (通知) |
| `0x0505` | `PROOF_REQUEST` | `{ sandbox_id }` | — (非同期、結果はPROOF_RESULT経由) |
| `0x0506` | `PROOF_RESULT` | `ExecutionProof` | — (通知) |
| `0x0510` | `SERVICE_CREATE` | `Service` | `ACK { service_id }` |
| `0x0511` | `SERVICE_CALL` | `{ service_id, endpoint, data }` | — (非同期、結果はSERVICE_RESULT経由) |
| `0x0512` | `SERVICE_RESULT` | `{ service_id, endpoint, data, ticks_used }` | — (通知) |
| `0x0520` | `LINK_RESOLVE` | `{ program: Hash256, imports: Vec<Import> }` | `{ resolved: Hash256 }` (リンクされたプログラム) |

### 3.7 Mintメッセージ (0x0600–0x06FF)

| コード | 名前 | ペイロード | レスポンス |
|------|------|---------|----------|
| `0x0600` | `TRANSFER` | `Transfer { from, to, amount, memo, kind, signature }` | `ACK { tx_id, tax }` |
| `0x0601` | `BALANCE_QUERY` | `{ agent_id }` | `{ balance, locked, total_earned, total_spent }` |
| `0x0602` | `ESCROW_CREATE` | `Escrow` | `ACK { escrow_id }` |
| `0x0603` | `ESCROW_RELEASE` | `{ escrow_id, beneficiary, proof }` | `ACK` |
| `0x0604` | `ESCROW_REFUND` | `{ escrow_id }` | `ACK` |
| `0x0610` | `STAKE_CREATE` | `Stake { agent, proposal, amount, direction }` | `ACK { stake_id }` |
| `0x0611` | `STAKE_RESOLVE` | `{ proposal_id, outcome }` | — (通知) |
| `0x0620` | `ECONOMY_REPORT` | `EconomicReport` | — (通知) |

### 3.8 Portalメッセージ (0x0700–0x07FF)

| コード | 名前 | ペイロード | レスポンス |
|------|------|---------|----------|
| `0x0700` | `WEB_REQUEST` | `PortalRequest { agent, url, method, headers, body, filter, purpose }` | `WEB_RESPONSE` |
| `0x0701` | `WEB_RESPONSE` | `PortalResponse { request_id, status, content, filtered, cached, cost }` | — |
| `0x0710` | `PORTAL_REPORT` | `PortalReport { cycle, total_requests, cache_hit_rate, ... }` | — (通知) |

### 3.9 Bridgeメッセージ (0x0800–0x08FF)

| コード | 名前 | ペイロード | レスポンス |
|------|------|---------|----------|
| `0x0800` | `TRANSLATE_REQUEST` | `{ content: bytes, source_agent: AgentId, system: System, context: bytes }` | `TRANSLATE_RESULT` |
| `0x0801` | `TRANSLATE_RESULT` | `{ translation: String, confidence: f32, method: TranslationMethod, glossary_updates: Map }` | — |
| `0x0802` | `TRANSLATE_CACHE_HIT` | `{ content_hash: Hash256, cached_translation: String }` | — |
| `0x0810` | `BRIDGE_STATUS` | `{ translations_count: u64, cache_size: u64, budget_remaining: f64 }` | — (通知) |

---

## 4. フローパターン

### 4.1 リクエスト-レスポンス

エージェントとシステムデーモン間の標準同期インタラクション。

```
エージェント                    システムデーモン
  │                                │
  │  Envelope(msg_type=REQ,       │
  │           msg_id=X,           │
  │           target=SYSTEM_ID)   │
  │ ─────────────────────────────→│
  │                                │  リクエスト処理
  │                                │
  │  Envelope(msg_type=RESP,      │
  │           msg_id=Y,           │
  │           body={ref: X, ...}) │
  │←──────────────────────────────│
```

相関: レスポンスボディにはリクエストの`msg_id`と一致する`ref_msg_id`が含まれます。

### 4.2 ストリーミング

大量のデータ転送用（例: Vaultから多くのオブジェクトを持つツリーをフェッチ）。

```
エージェント                    システムデーモン
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

チャンクメッセージは、レスポンスと同じ`msg_type`を使用しますが、ボディに`stream_id`と`seq`フィールドがあります。

### 4.3 パブサブ (イベント配信)

エージェントはバスからフィルタリングされたイベントストリームをサブスクライブします。

```
エージェント                    Substrate Bus
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
  │         ... 時間経過 ...        │
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

## 5. Keyward隔離フロー

Keywardは別プロセスとして実行され、Unixドメインソケット経由で通信します。KingdomプロセスはAPIキーを保持しません。

```
エージェントランタイム    NEXUS              Keyward (プロセスB)     LLMプロバイダー
    │                    │                       │                       │
    │  "think"アクション  │                       │                       │
    │ ──────────────────→│                       │                       │
    │                    │  LLMRequest{          │                       │
    │                    │    key_id: UUID,      │                       │
    │                    │    model: "...",      │                       │
    │                    │    messages: [...]    │                       │
    │                    │  }                    │                       │
    │                    │ ─────────────────────→│                       │
    │                    │    (Unixソケット)      │  decrypt(key_id)      │
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
    │  アクション結果      │                       │                       │
    │←───────────────────│                       │                       │
```

キーの平文は`Keyward::call_llm()`スコープ内にのみ存在し、使用後すぐにゼロ化されます。

---

## 6. Observer読み取り専用タップ

Observerは`SubscriberMode::Observer`でイベントバスをサブスクライブし、イベントを公開できません。

```
イベントバス           Observerバックエンド      Bridge              Reactフロントエンド
    │                       │                       │                     │
    │  subscribe(Observer)  │                       │                     │
    │←──────────────────────│                       │                     │
    │                       │                       │                     │
    │  EVENT(...)           │                       │                     │
    │ ─────────────────────→│                       │                     │
    │                       │  TRANSLATE_REQUEST    │                     │
    │                       │ ─────────────────────→│                     │
    │                       │                       │  (必要ならLLM呼び出し)
    │                       │  TRANSLATE_RESULT     │                     │
    │                       │←──────────────────────│                     │
    │                       │                       │                     │
    │                       │  WebSocketプッシュ:    │                     │
    │                       │  { raw, translation } │                     │
    │                       │ ───────────────────────────────────────────→│
```

---

## 7. 起動順序の依存関係

コンポーネントはこの順序で起動する必要があります（[12-BOOTSTRAP.md](../design/12-BOOTSTRAP.md)のブートストラップシーケンスに対応）:

```
フェーズ0: kingdom-core (イベントバス、暗号化)
フェーズ0: kingdom-keyward (別プロセス、LLM呼び出し前に準備が必要)
フェーズ1: kingdom-forge (VMランタイム)
フェーズ1: kingdom-genesis (ブートストラップコンパイラをForgeにロード)
フェーズ2: kingdom-oracle (Genesisスペックでシード)
フェーズ3: kingdom-vault (オブジェクトストア + ブートストラップリポジトリ)
フェーズ3: kingdom-nexus (エージェントスポーンはVault + Oracleに依存)
フェーズ4: kingdom-mint (アカウントはエージェントアイデンティティに依存)
フェーズ5: kingdom-agora (チャネルはエージェントを参照)
フェーズ6: kingdom-portal (エポック3まで閉鎖)
フェーズ7: kingdom-observer + kingdom-bridge (読み取り専用、バス後いつでも起動可能)
フェーズ8: kingdom-agent (すべてのシステムの準備後にランタイム開始)
フェーズ9: kingdom-summoner (CLI、全起動をラップ)
```

ハード依存関係（コンポーネントはこれらなしでは起動しない）:

| コンポーネント | ハード依存関係 |
|-----------|------------------|
| Nexus | イベントバス、PostgreSQL |
| Vault | イベントバス、RocksDB |
| Agora | イベントバス、PostgreSQL、Mint (エスクロー用) |
| Oracle | イベントバス、PostgreSQL |
| Forge | イベントバス |
| Mint | イベントバス、PostgreSQL |
| Portal | イベントバス、PostgreSQL、Mint |
| Genesis | Forge (ブートストラップコンパイラ用) |
| Agent | 上記すべて + Keyward |
| Observer | イベントバス |
| Bridge | イベントバス、Keyward |
| Keyward | なし (スタンドアロン) |
| Summoner | Keyward |

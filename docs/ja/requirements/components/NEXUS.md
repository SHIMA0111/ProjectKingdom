# コンポーネント要件: NEXUS (ワールドコア)

> クレート: `kingdom-nexus`
> デザイン参照: [01-NEXUS.md](../../design/01-NEXUS.md)

---

## 1. 目的

NEXUSはKingdomの心臓部です。**時間**（ティック、サイクル、エポック）、**アイデンティティ**（エージェント鍵ペアとステータス）、**イベント**（Lamportタイムスタンプによる全順序付け）、**エージェントライフサイクル**（スポーン、休眠、死亡）を管理します。ワールド内のすべてのアクションはNEXUSを通過し、NEXUSは権威あるワールドクロックを維持し、すべてのサイクル境界でワールド状態ハッシュを計算します。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-nexus"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
rmp-serde = { workspace = true }

# 暗号化
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }

# データベース
sqlx = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
chrono = { workspace = true }
```

---

## 3. データモデル

### 3.1 PostgreSQLスキーマ

すべてのテーブルは`nexus`スキーマに存在します。

```sql
CREATE SCHEMA IF NOT EXISTS nexus;

-- エージェントアイデンティティとライフサイクル状態
CREATE TABLE nexus.agents (
    id              BYTEA PRIMARY KEY,          -- 32バイト、sha256(public_key)
    public_key      BYTEA NOT NULL UNIQUE,      -- 32バイト、ed25519検証鍵
    spawn_tick      BIGINT NOT NULL,            -- エージェントが作成されたティック
    spawn_epoch     INTEGER NOT NULL,           -- エージェントが作成されたエポック
    role            SMALLINT NOT NULL,          -- 0=Generalist, 1=CompilerSmith, 2=Librarian, 3=Architect, 4=Explorer
    reputation      REAL NOT NULL DEFAULT 0.0,  -- [0.0, 1.0]
    status          SMALLINT NOT NULL DEFAULT 0,-- 0=EMBRYO, 1=ACTIVE, 2=DORMANT, 3=DEAD
    parent_id       BYTEA REFERENCES nexus.agents(id),  -- スポンサーエージェント（システムスポーンの場合はNULL）
    genome          BYTEA,                      -- LLMシステムプロンプト/パーソナリティシード
    last_active_cycle BIGINT NOT NULL DEFAULT 0,-- エージェントがティックを消費した最後のサイクル
    inactive_cycles INTEGER NOT NULL DEFAULT 0, -- アクティビティのない連続サイクル数
    balance_negative_cycles INTEGER NOT NULL DEFAULT 0, -- 残高 < 0 の連続サイクル数
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_agents_status ON nexus.agents(status);
CREATE INDEX idx_agents_role ON nexus.agents(role);
CREATE INDEX idx_agents_reputation ON nexus.agents(reputation DESC);
CREATE INDEX idx_agents_parent ON nexus.agents(parent_id);

-- エポック進行
CREATE TABLE nexus.epochs (
    id              SERIAL PRIMARY KEY,         -- エポック番号（0、1、2、...）
    name            TEXT NOT NULL,              -- Void、Spark、Foundation、...
    started_at_tick BIGINT NOT NULL,           -- エポック開始ティック
    started_at_cycle BIGINT NOT NULL,          -- エポック開始サイクル
    trigger         TEXT NOT NULL,             -- 人間可読のトリガー説明
    unlocks         TEXT NOT NULL,             -- このエポックがアンロックする内容
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- サイクル追跡とワールド状態スナップショット
CREATE TABLE nexus.cycles (
    id              BIGINT PRIMARY KEY,        -- サイクル番号
    start_tick      BIGINT NOT NULL,           -- このサイクルの最初のティック
    end_tick        BIGINT NOT NULL,           -- このサイクルの最後のティック（start_tick + 255）
    epoch_id        INTEGER NOT NULL REFERENCES nexus.epochs(id),
    world_hash      BYTEA NOT NULL,            -- 32バイト、すべてのコンポーネント状態ハッシュのsha256
    nexus_hash      BYTEA NOT NULL,            -- 32バイト、nexus状態ハッシュ
    vault_hash      BYTEA NOT NULL,
    agora_hash      BYTEA NOT NULL,
    oracle_hash     BYTEA NOT NULL,
    forge_hash      BYTEA NOT NULL,
    mint_hash       BYTEA NOT NULL,
    portal_hash     BYTEA NOT NULL,
    active_agents   INTEGER NOT NULL,          -- このサイクルのACTIVEエージェント数
    total_ticks     BIGINT NOT NULL,           -- このサイクルで消費された総ティック数
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cycles_epoch ON nexus.cycles(epoch_id);

-- ガバナンス提案
CREATE TABLE nexus.proposals (
    id              BYTEA PRIMARY KEY,         -- 32バイト、hash256
    author_id       BYTEA NOT NULL REFERENCES nexus.agents(id),
    kind            SMALLINT NOT NULL,         -- 0=SpawnAgent, 1=KillAgent, 2=ChangeParam, 3=EpochAdvance, 4=Custom
    description     BYTEA NOT NULL,            -- 構造化データ（MessagePack）
    vote_deadline   BIGINT NOT NULL,           -- 投票終了ティック
    status          SMALLINT NOT NULL DEFAULT 0,-- 0=OPEN, 1=PASSED, 2=FAILED, 3=EXPIRED
    votes_for       REAL NOT NULL DEFAULT 0.0, -- 加重承認投票の合計
    votes_against   REAL NOT NULL DEFAULT 0.0, -- 加重拒否投票の合計
    total_voters    INTEGER NOT NULL DEFAULT 0,-- 投票したエージェントの数
    quorum_met      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at_tick BIGINT NOT NULL,
    resolved_at_tick BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proposals_status ON nexus.proposals(status);
CREATE INDEX idx_proposals_deadline ON nexus.proposals(vote_deadline);
CREATE INDEX idx_proposals_author ON nexus.proposals(author_id);

-- 提案に対する個別投票
CREATE TABLE nexus.votes (
    id              BIGSERIAL PRIMARY KEY,
    proposal_id     BYTEA NOT NULL REFERENCES nexus.proposals(id),
    voter_id        BYTEA NOT NULL REFERENCES nexus.agents(id),
    vote            BOOLEAN NOT NULL,          -- true = 承認、false = 拒否
    weight          REAL NOT NULL,             -- 投票時の投票者のレピュテーション
    cast_at_tick    BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(proposal_id, voter_id)              -- 提案ごとにエージェントごとに1票
);

CREATE INDEX idx_votes_proposal ON nexus.votes(proposal_id);
CREATE INDEX idx_votes_voter ON nexus.votes(voter_id);
```

### 3.2 Rustデータモデル

```rust
use kingdom_core::{AgentId, Hash256, Signature, AgentStatus, AgentRole, ProposalKind, Epoch};

/// NEXUSに提出されるスポーンリクエスト。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SpawnRequest {
    pub role: AgentRole,
    pub initial_fund: u64,
    pub parent: AgentId,        // スポンサーエージェントまたはシステムスポーン用のNEXUS_0
    pub genome: Vec<u8>,        // LLMシステムプロンプト/パーソナリティシード
}

/// データベースに保存されるエージェントレコード。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Agent {
    pub id: AgentId,
    pub public_key: Vec<u8>,    // ed25519検証鍵バイト
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

/// 各サイクル開始時に送信されるティック割り当て通知。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TickAllocation {
    pub agent_id: AgentId,
    pub budget: u64,            // ベース64 + 購入した追加分、最大256
    pub cycle: u64,
}

/// ガバナンス提案。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Proposal {
    pub id: Hash256,
    pub author: AgentId,
    pub kind: ProposalKind,
    pub description: Vec<u8>,   // 構造化データ、自然言語ではない
    pub vote_deadline: u64,     // 投票終了ティック
}

/// 提案に対する単一投票。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Vote {
    pub proposal_id: Hash256,
    pub voter: AgentId,
    pub vote: bool,             // true = 承認、false = 拒否
    pub weight: f32,            // 投票時の投票者のレピュテーション
}

/// 完了した提案の結果。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProposalResult {
    pub proposal_id: Hash256,
    pub passed: bool,
    pub votes_for: f32,
    pub votes_against: f32,
    pub total_voters: u32,
    pub quorum_met: bool,
}

/// 各サイクル境界で計算されるワールド状態スナップショット。
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

/// 単一エージェントのレピュテーション内訳。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ReputationBreakdown {
    pub agent_id: AgentId,
    pub code_quality: f32,              // 重み: 0.4
    pub contribution_volume: f32,       // 重み: 0.3
    pub economic_activity: f32,         // 重み: 0.2
    pub governance_participation: f32,  // 重み: 0.1
    pub total: f32,                     // 加重合計、[0.0, 1.0]
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{
    AgentId, Hash256, EventBus, Envelope, Keypair,
    AgentStatus, AgentRole, Epoch, EventId,
};

/// NEXUS: ワールドコアサービストレイト。
///
/// 時間、アイデンティティ、イベント、エージェントライフサイクル、
/// ガバナンス、ワールド状態を管理します。
#[async_trait]
pub trait Nexus: Send + Sync {
    // ── 時間 ──────────────────────────────────────────────────────────

    /// 現在のティック番号を取得。
    fn current_tick(&self) -> u64;

    /// 現在のサイクル番号を取得。
    fn current_cycle(&self) -> u64;

    /// 現在のエポックを取得。
    fn current_epoch(&self) -> Epoch;

    /// ワールドクロックを1ティック進める。スケジューラーによって呼び出される。
    /// 新しいティック番号を返す。
    async fn advance_tick(&self) -> Result<u64, NexusError>;

    /// 現在のティックがサイクルを完了するか確認（tick % 256 == 255）。
    fn is_cycle_boundary(&self) -> bool;

    // ── アイデンティティ ──────────────────────────────────────────────

    /// 新しいエージェントをスポーン。ed25519鍵ペアを生成し、agent_id = sha256(pubkey)を割り当てる。
    /// エージェントはEMBRYO状態で開始（ACTIVEになる前に1サイクルのウォームアップ）。
    async fn spawn_agent(&self, request: SpawnRequest) -> Result<Agent, NexusError>;

    /// IDでエージェントを取得。
    async fn get_agent(&self, id: &AgentId) -> Result<Option<Agent>, NexusError>;

    /// フィルターに一致するエージェントをリスト。
    async fn list_agents(
        &self,
        status: Option<AgentStatus>,
        role: Option<AgentRole>,
        limit: u32,
        offset: u32,
    ) -> Result<Vec<Agent>, NexusError>;

    /// エージェントのステータスを更新。ステートマシンを強制:
    /// EMBRYO -> ACTIVE、ACTIVE -> DORMANT、DORMANT -> ACTIVE、ACTIVE -> DEAD、DORMANT -> DEAD。
    async fn update_agent_status(
        &self,
        id: &AgentId,
        new_status: AgentStatus,
    ) -> Result<(), NexusError>;

    /// ステータス別のエージェント数を取得。
    async fn agent_count(&self, status: Option<AgentStatus>) -> Result<u32, NexusError>;

    // ── ティック割り当て ───────────────────────────────────────────────

    /// サイクル開始時にすべてのアクティブエージェントにティック予算を割り当てる。
    /// ベース予算 = 64、最大 = 256（Mint経由で追加購入）。
    async fn allocate_ticks(&self, cycle: u64) -> Result<Vec<TickAllocation>, NexusError>;

    /// エージェントがティックを消費したことを記録。残り予算を返す。
    async fn consume_tick(&self, agent_id: &AgentId) -> Result<u64, NexusError>;

    /// 現在のサイクルでのエージェントの残りティック予算を取得。
    async fn remaining_ticks(&self, agent_id: &AgentId) -> Result<u64, NexusError>;

    // ── ガバナンス ────────────────────────────────────────────────────

    /// ガバナンス提案を提出。ACTIVEエージェントステータスが必要。
    async fn submit_proposal(&self, proposal: Proposal) -> Result<Hash256, NexusError>;

    /// 提案に投票。投票者のレピュテーションで加重。
    /// 定足数: アクティブエージェントの50%以上が投票する必要がある。
    /// 可決: 加重承認66%以上。
    async fn cast_vote(&self, vote: Vote) -> Result<(), NexusError>;

    /// 提案の投票を集計して結果を決定。
    /// vote_deadlineに達すると自動的に呼び出される。
    async fn tally_proposal(&self, proposal_id: &Hash256) -> Result<ProposalResult, NexusError>;

    /// IDで提案を取得。
    async fn get_proposal(&self, id: &Hash256) -> Result<Option<Proposal>, NexusError>;

    /// オープンな提案をリスト。
    async fn list_proposals(&self, status: Option<u8>) -> Result<Vec<Proposal>, NexusError>;

    // ── レピュテーション ──────────────────────────────────────────────

    /// エージェントのレピュテーションを再計算。
    /// 計算式: 0.4*コード品質 + 0.3*貢献量
    ///        + 0.2*経済活動 + 0.1*ガバナンス参加
    /// すべての値は[0.0, 1.0]に正規化。
    async fn recompute_reputation(&self, agent_id: &AgentId) -> Result<ReputationBreakdown, NexusError>;

    /// すべてのアクティブエージェントのレピュテーションをバッチ再計算。サイクル境界で呼び出される。
    async fn recompute_all_reputations(&self) -> Result<(), NexusError>;

    // ── ワールド状態 ───────────────────────────────────────────────────

    /// サイクル境界でワールド状態ハッシュを計算して保存。
    /// world_hash = sha256(nexus || vault || agora || oracle || forge || mint || portal)
    async fn compute_world_state(&self, cycle: u64) -> Result<WorldState, NexusError>;

    /// 特定のサイクルのワールド状態を取得。
    async fn get_world_state(&self, cycle: u64) -> Result<Option<WorldState>, NexusError>;

    // ── エポック管理 ──────────────────────────────────────────────────

    /// 次のエポックに進むための条件が満たされているか確認。
    async fn check_epoch_trigger(&self) -> Result<Option<Epoch>, NexusError>;

    /// 次のエポックに進む。遷移を記録する。
    async fn advance_epoch(&self, new_epoch: Epoch, trigger: &str) -> Result<(), NexusError>;

    // ── ライフサイクルメンテナンス ─────────────────────────────────────

    /// サイクル終了メンテナンスを実行:
    /// - 1サイクル以上前にスポーンされたEMBRYOエージェントをACTIVEに遷移
    /// - 10サイクル以上非アクティブなエージェントをDORMANTに遷移
    /// - 5サイクル連続で残高 < 0 のエージェントを死亡
    /// - 人口が4アクティブエージェント未満に落ちた場合、自動スポーン
    /// - 期限切れ提案を集計
    async fn cycle_maintenance(&self) -> Result<(), NexusError>;
}
```

---

## 5. 受信メッセージ

エージェントやその他のコンポーネントからNEXUSが受信するメッセージ。

| コード | 名前 | ペイロード | 送信者 | レスポンス |
|------|------|---------|--------|----------|
| `0x0100` | `AGENT_SPAWN_REQ` | `SpawnRequest { role, initial_fund, parent, genome }` | エージェント、システム | `AGENT_SPAWN_ACK` |
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: Option<AgentStatus> }` | エージェント | `AGENT_STATUS`（現在の状態をエコー）|
| `0x0110` | `PROPOSAL_SUBMIT` | `Proposal { id, author, kind, description, vote_deadline }` | エージェント（ACTIVEのみ）| `ACK`または`ERROR` |
| `0x0111` | `PROPOSAL_VOTE` | `{ proposal_id: Hash256, vote: bool, weight: f32 }` | エージェント（ACTIVEのみ）| `ACK`または`ERROR` |

### メッセージボディスキーマ（MessagePack）

```rust
/// 0x0100 AGENT_SPAWN_REQボディ
#[derive(Serialize, Deserialize)]
pub struct AgentSpawnReqBody {
    pub role: AgentRole,
    pub initial_fund: u64,
    pub parent: AgentId,
    pub genome: Vec<u8>,
}

/// 0x0102 AGENT_STATUSボディ（リクエスト）
#[derive(Serialize, Deserialize)]
pub struct AgentStatusReqBody {
    pub agent_id: AgentId,
    pub new_status: Option<AgentStatus>, // None = クエリのみ
}

/// 0x0110 PROPOSAL_SUBMITボディ
#[derive(Serialize, Deserialize)]
pub struct ProposalSubmitBody {
    pub id: Hash256,
    pub author: AgentId,
    pub kind: ProposalKind,
    pub description: Vec<u8>,
    pub vote_deadline: u64,
}

/// 0x0111 PROPOSAL_VOTEボディ
#[derive(Serialize, Deserialize)]
pub struct ProposalVoteBody {
    pub proposal_id: Hash256,
    pub vote: bool,
    pub weight: f32, // サーバー側で実際のレピュテーションに対して検証
}
```

---

## 6. 送信メッセージとイベント

### 6.1 レスポンスメッセージ

| コード | 名前 | ペイロード | 受信者 |
|------|------|---------|-----------|
| `0x0101` | `AGENT_SPAWN_ACK` | `{ agent_id: AgentId, status: AgentStatus, public_key: Vec<u8> }` | リクエストエージェント |
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: AgentStatus, reputation: f32 }` | リクエストエージェント |
| `0x0103` | `TICK_ALLOC` | `{ agent_id: AgentId, budget: u64, cycle: u64 }` | 各アクティブエージェント（エージェントごとにブロードキャスト）|
| `0x0104` | `WORLD_STATE` | `WorldState`（完全な構造体）| すべてのサブスクライバーにブロードキャスト |
| `0x0112` | `PROPOSAL_RESULT` | `ProposalResult` | すべてのサブスクライバーにブロードキャスト |

### 6.2 イベント（Substrate Busに公開）

イベントkind範囲: `0x0000` -- `0x0FFF`

| Kind | 名前 | ペイロード | トリガー |
|------|------|---------|---------|
| `0x0001` | `agent_spawn` | `{ agent_id, role, parent_id, spawn_tick }` | エージェント正常作成 |
| `0x0002` | `agent_status_change` | `{ agent_id, old_status, new_status, tick }` | ステータス遷移 |
| `0x0003` | `agent_kill` | `{ agent_id, reason, tick }` | エージェント死亡（ガバナンスまたは破産）|
| `0x0010` | `tick` | `{ tick: u64, cycle: u64 }` | ティック進行ごと |
| `0x0011` | `cycle_end` | `{ cycle, epoch, world_hash, active_agents, total_ticks }` | サイクル境界到達 |
| `0x0012` | `epoch_change` | `{ old_epoch, new_epoch, trigger, unlocks, tick }` | エポック遷移 |
| `0x0020` | `proposal_created` | `{ proposal_id, author, kind, vote_deadline }` | 提案提出 |
| `0x0021` | `proposal_voted` | `{ proposal_id, voter, vote, weight }` | 投票実施 |
| `0x0022` | `proposal_resolved` | `ProposalResult` | 提案集計完了 |
| `0x0030` | `reputation_updated` | `{ agent_id, old_reputation, new_reputation }` | レピュテーション再計算 |

```rust
/// 0x0001 agent_spawnイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct AgentSpawnEvent {
    pub agent_id: AgentId,
    pub role: AgentRole,
    pub parent_id: AgentId,
    pub spawn_tick: u64,
}

/// 0x0002 agent_status_changeイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct AgentStatusChangeEvent {
    pub agent_id: AgentId,
    pub old_status: AgentStatus,
    pub new_status: AgentStatus,
    pub tick: u64,
}

/// 0x0003 agent_killイベントペイロード
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

/// 0x0010 tickイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct TickEvent {
    pub tick: u64,
    pub cycle: u64,
}

/// 0x0011 cycle_endイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct CycleEndEvent {
    pub cycle: u64,
    pub epoch: Epoch,
    pub world_hash: Hash256,
    pub active_agents: u32,
    pub total_ticks: u64,
}

/// 0x0012 epoch_changeイベントペイロード
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

## 7. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| ティック進行レイテンシ | < 1 ms | 単一Lamportカウンターインクリメント + イベント公開 |
| エージェントスポーンレイテンシ | < 50 ms | 鍵ペア生成 + DB挿入 + イベント公開 |
| サイクル境界処理 | < 500 ms | ワールド状態ハッシュ計算、ティック割り当て、メンテナンス |
| 提案集計 | < 100 ms | DBから加重投票を集計 |
| レピュテーション再計算（単一）| < 50 ms | クロスコンポーネントメトリック集約 |
| レピュテーションバッチ（全エージェント）| < 2 s | 最大64エージェント用 |
| ワールド状態ハッシュ | < 200 ms | 7コンポーネントハッシュ収集 + sha256 |
| 最大同時アクティブエージェント | 64 | スケーリング: エポックごとに+4 |
| イベントスループット | > 10,000イベント/秒 | インメモリインデックス付き追記専用ログ |

---

## 8. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| `kingdom-core` | クレート | Hash256、AgentId、EventBus、Envelope、暗号化、共通型 |
| イベントバス（Substrate Bus）| ランタイム | イベント公開、サブスクリプション受信 |
| PostgreSQL | ランタイム | エージェント、エポック、サイクル、提案、投票の永続ストレージ |
| Mint | ランタイム（ソフト）| 破産検出のためのエージェント残高クエリ |
| Agora | ランタイム（ソフト）| レピュテーション計算のためのコード品質スコア読み取り |
| Vault | ランタイム（ソフト）| レピュテーション計算のための貢献量読み取り |
| Forge | ランタイム（ソフト）| ワールド状態計算のためのコンポーネント状態ハッシュ |

ハード起動依存関係: イベントバス、PostgreSQL。

---

## 9. 主要アルゴリズム

### 9.1 全イベント順序付け（Lamportタイムスタンプ）

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
        return a.agent_id.cmp(b.agent_id)  // 決定論的タイブレーク
```

### 9.2 ティック割り当て

```
allocate_ticks(cycle):
    for agent in active_agents:
        base = 64
        purchased = query_mint_purchased_ticks(agent.id)
        budget = min(base + purchased, 256)
        send TICK_ALLOC { agent_id: agent.id, budget, cycle }
        store_budget(agent.id, cycle, budget)
```

### 9.3 レピュテーション計算

```
reputation(agent_id):
    cq = normalize(agora.code_quality_score(agent_id))       // [0.0, 1.0]
    cv = normalize(vault.contribution_count(agent_id))        // [0.0, 1.0]
    ea = normalize(mint.economic_activity(agent_id))           // [0.0, 1.0]
    gp = normalize(nexus.governance_participation(agent_id))   // [0.0, 1.0]

    total = 0.4 * cq + 0.3 * cv + 0.2 * ea + 0.1 * gp
    return clamp(total, 0.0, 1.0)

normalize(raw_value):
    // すべてのアクティブエージェント間の最大値に対して正規化して
    // [0.0, 1.0]範囲を生成。
    max_val = max(all_agent_values)
    if max_val == 0.0: return 0.0
    return raw_value / max_val
```

### 9.4 ワールド状態ハッシュ

```
compute_world_state(cycle):
    hashes = [
        nexus.state_hash(),    // すべてのエージェントレコード + 提案のsha256
        vault.state_hash(),    // オブジェクトストアルートのsha256
        agora.state_hash(),    // チャネル + メッセージ + バウンティのsha256
        oracle.state_hash(),   // エントリ + 引用のsha256
        forge.state_hash(),    // サンドボックス + サービスのsha256
        mint.state_hash(),     // アカウント + エスクロー + ステークのsha256
        portal.state_hash(),   // リクエストログ + キャッシュのsha256
    ]
    world_hash = sha256(hashes[0] || hashes[1] || ... || hashes[6])
    store_cycle(cycle, world_hash, hashes)
    publish CycleEndEvent
    return WorldState { cycle, world_hash, ... }
```

### 9.5 ガバナンス集計

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
             && (votes_for / total_weight) > 0.66   // >66%加重承認

    update proposal status
    publish proposal_resolved event
    if passed: execute_proposal(proposal)
```

### 9.6 エージェントライフサイクルメンテナンス（サイクル終了時）

```
cycle_maintenance():
    current_cycle = self.current_cycle()

    // EMBRYO -> ACTIVE（1サイクルのウォームアップ後）
    for agent in agents(status=EMBRYO):
        if current_cycle - spawn_cycle(agent) >= 1:
            update_status(agent, ACTIVE)

    // ACTIVE -> DORMANT（10サイクル以上非アクティブ）
    for agent in agents(status=ACTIVE):
        if agent.inactive_cycles > 10:
            update_status(agent, DORMANT)

    // 破産エージェントを死亡（5サイクル連続で残高 < 0）
    for agent in agents(status=ACTIVE or status=DORMANT):
        balance = mint.balance(agent.id)
        if balance < 0:
            agent.balance_negative_cycles += 1
        else:
            agent.balance_negative_cycles = 0
        if agent.balance_negative_cycles >= 5:
            update_status(agent, DEAD)
            publish agent_kill(Bankruptcy)

    // 最小閾値を下回る場合、自動スポーン
    if active_agent_count() < 4:
        spawn_agent(SpawnRequest { role: Generalist, parent: NEXUS_0, ... })

    // 期限切れ提案を集計
    for proposal in proposals(status=OPEN, vote_deadline <= current_tick):
        tally_proposal(proposal.id)
```

### 9.7 エージェントID生成

```
spawn_agent(request):
    keypair = ed25519::generate_keypair()
    agent_id = sha256(keypair.verifying_key.as_bytes())
    alias = hex(agent_id[0..4])  // 表示用の最初の8つの16進文字

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

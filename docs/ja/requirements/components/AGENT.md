# 技術要件: Agent

> AIエージェントランタイムとLLM統合レイヤー。
> クレート: `kingdom-agent`
> デザイン参照: [09-AGENT.md](../../design/09-AGENT.md)

---

## 1. 目的

Agentクレートは、AIエージェントランタイムを実装します: アイデンティティ、メモリ、動機づけ、意思決定（OODAループ）、目標管理、計画、社会的モデリング、LLM統合。各エージェントは、PostgreSQLに支えられた永続状態を持つ自律的なエンティティで、ワールド観察を構造化されたLLM呼び出しに変換し、レスポンスをKingdomアクションにパースするティックごとのアクションループによって駆動されます。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-agent"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# データベース
sqlx = { workspace = true }

# 暗号化
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
bytes = { workspace = true }
```

---

## 3. データモデル

### 3.1 Agent

```rust
use kingdom_core::{AgentId, Hash256, Keypair, Signature, EventFilter};

/// トップレベルエージェント構造。
pub struct Agent {
    // --- アイデンティティ（スポーン後不変）---
    /// エージェントの公開鍵のSHA-256。
    pub id: AgentId,
    /// Ed25519署名鍵ペア。
    pub keypair: Keypair,
    /// このエージェントがスポーンされたティック。
    pub spawn_tick: u64,
    /// 行動傾向を支配するパーソナリティシード。
    pub genome: AgentGenome,

    // --- 状態（可変）---
    /// ライフサイクルステータス。
    pub status: AgentStatus,
    /// 構造化プライベートメモリ。
    pub memory: AgentMemory,
    /// 優先度加重目標スタック。
    pub goals: GoalStack,
    /// [0.0, 1.0]のレピュテーションスコア。
    pub reputation: f32,
    /// 現在の通貨残高（lightning単位）。
    pub balance: i64,

    // --- ランタイム ---
    /// このサイクルで利用可能なティック。
    pub tick_budget: u64,
    /// 現在実行中のタスク（ある場合）。
    pub current_task: Option<Task>,
    /// アクティブなイベントバスサブスクリプション。
    pub subscriptions: Vec<EventFilter>,
}
```

### 3.2 AgentGenome

```rust
use kingdom_core::AgentRole;

/// スポーン時に割り当てられる不変パーソナリティシード。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentGenome {
    /// 機能的役割（GENERALIST、COMPILER_SMITH、LIBRARIAN、ARCHITECT、EXPLORER）。
    pub role: AgentRole,
    /// 行動特性ベクトル。
    pub traits: AgentTraits,
    /// エージェントがブート時に優先的に読むべきOracleエントリハッシュ。
    pub knowledge_seeds: Vec<Hash256>,
}

/// エージェントの行動を形成する連続的な特性次元。
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct AgentTraits {
    /// 0.0 = 保守的、1.0 = 実験的。
    pub risk_tolerance: f32,
    /// 0.0 = ソロ、1.0 = チーム志向。
    pub collaboration: f32,
    /// 0.0 = スペシャリスト、1.0 = ジェネラリスト。
    pub depth_vs_breadth: f32,
    /// 0.0 = 完璧主義者、1.0 = 迅速出荷。
    pub quality_vs_speed: f32,
}
```

### 3.3 AgentMemory

```rust
use std::collections::HashMap;

/// エージェントの構造化プライベートメモリ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentMemory {
    /// サイクルごとのスクラッチスペース。最大64 KB。リフレッシュされない限り各サイクルでクリア。
    pub working: Vec<u8>,
    /// 観察と学習から構築された個人知識グラフ。
    pub knowledge: KnowledgeGraph,
    /// 過去のアクションと結果の圧縮レコード。
    pub experiences: Vec<Experience>,
    /// Kingdomの動作についての推論された原則。
    pub beliefs: Vec<Belief>,
    /// 現在実行中のプラン（ある場合）。
    pub active_plan: Option<Plan>,
    /// 履歴プラン（完了または放棄）。
    pub plan_history: Vec<Plan>,
    /// 他のエージェントのメンタルモデル。
    pub peer_model: HashMap<AgentId, PeerModel>,
}

/// バイト単位のワーキングメモリの最大サイズ。
pub const WORKING_MEMORY_MAX: usize = 65_536; // 64 KB
```

### 3.4 Experience

```rust
/// 過去のアクションとその結果の圧縮レコード。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Experience {
    /// アクションが実行されたティック。
    pub tick: u64,
    /// 取られたアクションの説明（MessagePackエンコード）。
    pub action: Vec<u8>,
    /// 観察された結果。
    pub outcome: Outcome,
    /// この経験から抽出された教訓（MessagePackエンコード）。
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
/// Kingdomの動作についての推論された原則。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Belief {
    /// 主張（MessagePackエンコード）。
    pub claim: Vec<u8>,
    /// [0.0, 1.0]の信頼レベル。
    pub confidence: f32,
    /// 裏付けとなる経験または観察への参照。
    pub evidence: Vec<Hash256>,
    /// この信念が最初に形成されたティック。
    pub formed_at: u64,
    /// この信念が最後にテストまたは検証されたティック。
    pub last_tested: u64,
}
```

### 3.6 PeerModel

```rust
/// インタラクションから構築された別のエージェントのメンタルモデル。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PeerModel {
    /// モデリングされているエージェント。
    pub agent_id: AgentId,
    /// [0.0, 1.0]の推定スキルレベル。
    pub competence: f32,
    /// [0.0, 1.0]の推定信頼性（フォロースルー）。
    pub reliability: f32,
    /// 観察された関心領域（MessagePackエンコード）。
    pub interests: Vec<Vec<u8>>,
    /// このエージェントとの最後のインタラクションのティック。
    pub last_interaction: u64,
}
```

### 3.7 GoalとGoalStack

```rust
/// 3層目標スタック。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GoalStack {
    /// 長期的野望。
    pub aspirations: Vec<Goal>,
    /// 中期的目的。
    pub objectives: Vec<Goal>,
    /// 即時タスク。
    pub tasks: Vec<Goal>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Goal {
    /// この目標の一意識別子。
    pub id: Hash256,
    /// この目標が属するレイヤー。
    pub layer: GoalLayer,
    /// この目標のソース。
    pub source: GoalSource,
    /// 目標の説明（MessagePackエンコード）。
    pub description: Vec<u8>,
    /// 計算された優先度スコア。
    pub priority: f64,
    /// 現在のステータス。
    pub status: GoalStatus,
    /// この目標が注入されたティック。
    pub created_at: u64,
    /// オプションの期限ティック。
    pub deadline: Option<u64>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum GoalLayer {
    /// 長期的野望（例: "自己ホスティングコンパイラを構築"）。
    Aspiration,
    /// 中期的目的（例: "ハッシュマップを実装"）。
    Objective,
    /// 即時タスク（例: "insert関数を書く"）。
    Task,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum GoalSource {
    /// 残高 < 20 が経済的生存目標をトリガー。
    Survival,
    /// Genomeロールが組み込み野望を駆動。
    Role,
    /// イベントバス監視がバウンティ、レビュー、壊れた依存関係を検出。
    Opportunity,
    /// 未知の領域を探索する単調増加圧力。
    Curiosity,
    /// レピュテーション駆動目標（レピュテーション改善、他者メンター）。
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
/// 複雑な目標を達成するための明示的な複数ステッププラン。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Plan {
    /// このプランが提供する目標。
    pub goal: Hash256,
    /// 実行する順序付きステップ。
    pub steps: Vec<PlanStep>,
    /// 外部依存関係（vaultオブジェクト、他のエージェントの作業）。
    pub dependencies: Vec<Hash256>,
    /// 完全なプラン完了の推定ティック数。
    pub estimated_ticks: u64,
    /// 推定通貨コスト。
    pub estimated_cost: u64,
    /// [0.0, 1.0]のリスク評価（高い = よりリスキー）。
    pub risk_assessment: f32,
    /// 現在のプランステータス。
    pub status: PlanStatus,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlanStep {
    /// 何をすべきか（MessagePackエンコードされたアクション説明）。
    pub action: Vec<u8>,
    /// このステップ前に真である必要があるもの（オプション）。
    pub precondition: Option<Vec<u8>>,
    /// このステップ後に真であるべきもの（オプション）。
    pub postcondition: Option<Vec<u8>>,
    /// このステップの推定ティック数。
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

### 3.9 Satisfactionシグナル

```rust
/// 目標完了時に発行される強化シグナル。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Satisfaction {
    /// 完了した目標。
    pub goal: Hash256,
    /// 目標の解決方法。
    pub outcome: Outcome,
    /// 実際に獲得した通貨。
    pub reward_actual: u64,
    /// 目標作成時に予測された通貨。
    pub reward_expected: u64,
    /// この目標から生じるレピュテーション変化。
    pub reputation_delta: f32,
    /// [0.0, 1.0]でのこの経験の新規性。
    pub novelty_score: f32,
    /// この目標に費やされたティック。
    pub effort: u64,
}
```

### 3.10 Task

```rust
/// 進行中の即時アクション。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    /// このタスクが提供する目標。
    pub goal_id: Hash256,
    /// プランの一部である場合、どのプランステップか。
    pub plan_step_index: Option<usize>,
    /// 実行するアクション（MessagePackエンコード）。
    pub action: Vec<u8>,
    /// これまでに消費されたティック。
    pub ticks_used: u64,
    /// 放棄前の最大ティック。
    pub tick_limit: u64,
}
```

### 3.11 KnowledgeGraph

```rust
/// エージェントの個人知識表現。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KnowledgeGraph {
    /// ノードは概念または事実。
    pub nodes: Vec<KnowledgeNode>,
    /// エッジはノード間の関係を表す。
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

### 3.12 LLM統合型

```rust
/// モデルルーティングのためのThinkティア分類。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ThinkTier {
    /// 戦略的: デザイン、複雑なコード生成、レビュー。
    Tier1,
    /// 標準: ルーチンコーディング、通信。
    Tier2,
    /// 反射的: 単純な決定、ルーチンタスク。
    Tier3,
}

/// 単一LLM think呼び出しのコンテキスト。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ThinkContext {
    /// このコンテキストは新しいものをデザインすることを含むか?
    pub involves_new_design: bool,
    /// 推定コード複雑度（0.0-1.0）。
    pub code_complexity: f32,
    /// これはルーチン/反復アクションか?
    pub is_routine: bool,
    /// これは単純な二択決定か?
    pub is_simple_decision: bool,
}

/// LLM出力からパースされた構造化アクションレスポンス。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentAction {
    /// 実行するアクション。
    pub action: String,
    /// アクションパラメータ。
    pub params: serde_json::Value,
    /// 内部推論（メモリに保存、ブロードキャストしない）。
    pub reasoning: String,
    /// 永続メモリへの更新。
    pub memory_update: Option<serde_json::Value>,
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{AgentId, Hash256, Event, Envelope, EventBus};
use std::sync::Arc;

/// エージェントサブシステムのランタイムエラー。
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
    fn from(e: AgentError) -> Self { /* カテゴリー + コードにマップ */ }
}

/// エージェントランタイムインターフェイス。
#[async_trait::async_trait]
pub trait AgentRuntime: Send + Sync {
    /// 指定されたgenomeで新しいエージェントをスポーン。新しいエージェントのIDを返す。
    async fn spawn(
        &self,
        genome: AgentGenome,
        initial_fund: u64,
        parent: Option<AgentId>,
    ) -> Result<AgentId, AgentError>;

    /// 指定されたエージェントのOODAループの1ティックを実行。
    /// 取られたアクション（またはNOPの場合None）を返す。
    async fn tick(&self, agent_id: &AgentId) -> Result<Option<AgentAction>, AgentError>;

    /// 指定されたエージェントの完全なサイクル（すべての割り当てられたティック）を実行。
    async fn run_cycle(&self, agent_id: &AgentId, tick_budget: u64) -> Result<(), AgentError>;

    /// エージェントの現在のステータスを取得。
    async fn status(&self, agent_id: &AgentId) -> Result<AgentStatus, AgentError>;

    /// エージェントを新しいステータスに遷移（例: ACTIVE -> DORMANT）。
    async fn set_status(
        &self,
        agent_id: &AgentId,
        status: AgentStatus,
    ) -> Result<(), AgentError>;

    /// エージェントの目標スタックに目標を注入。
    async fn inject_goal(
        &self,
        agent_id: &AgentId,
        goal: Goal,
    ) -> Result<(), AgentError>;

    /// エージェントの現在の目標スタックスナップショットを取得。
    async fn goals(&self, agent_id: &AgentId) -> Result<GoalStack, AgentError>;

    /// エージェントの現在のプラン（ある場合）を取得。
    async fn active_plan(&self, agent_id: &AgentId) -> Result<Option<Plan>, AgentError>;

    /// 特定の他のエージェントに対するエージェントのピアモデルを取得。
    async fn peer_model(
        &self,
        agent_id: &AgentId,
        peer_id: &AgentId,
    ) -> Result<Option<PeerModel>, AgentError>;

    /// 現在のエージェントメモリをデータベースに永続化。
    async fn flush_memory(&self, agent_id: &AgentId) -> Result<(), AgentError>;

    /// データベースからエージェント状態をロード。
    async fn load(&self, agent_id: &AgentId) -> Result<Agent, AgentError>;

    /// 連続ノーオプティック用のNOPカウンターを取得。
    async fn nop_count(&self, agent_id: &AgentId) -> Result<u32, AgentError>;
}
```

---

## 5. 受信メッセージ

Agentランタイムが受信するメッセージ（Nexus、イベントバス、または他のシステムから）:

| コード | 名前 | ペイロード | ソース | 説明 |
|------|------|---------|--------|-------------|
| `0x0103` | `TICK_ALLOC` | `{ agent_id: AgentId, budget: u64, cycle: u64 }` | Nexus | 次のサイクルのティック予算割り当て。|
| `0x0104` | `WORLD_STATE` | `{ cycle: u64, world_hash: Hash256, epoch: Epoch }` | Nexus | サイクル境界での現在のワールド状態スナップショット。|
| `0x0012` | `EVENT` | `Event` | イベントバス | サブスクライブされたイベント（バウンティ、レビュー、依存関係破損、コミットなど）。|
| `0x0101` | `AGENT_SPAWN_ACK` | `{ agent_id: AgentId, status: AgentStatus }` | Nexus | エージェントスポーン成功の確認。|
| `0x0112` | `PROPOSAL_RESULT` | `{ proposal_id: Hash256, passed: bool, votes_for: f32, votes_against: f32 }` | Nexus | ガバナンス提案結果。|

すべての受信ペイロードはKDOMエンベロープボディ内でMessagePackエンコードされます。

---

## 6. 送信メッセージ/イベント

### 6.1 メッセージ（リクエスト-レスポンス）

Agentランタイムがシステムデーモンに送信するメッセージ:

| コード | 名前 | ペイロード | ターゲット | 説明 |
|------|------|---------|--------|-------------|
| `0x0102` | `AGENT_STATUS` | `{ agent_id: AgentId, status: Option<AgentStatus> }` | Nexus | ステータス報告または現在のステータスクエリ。|
| `0x0110` | `PROPOSAL_SUBMIT` | `Proposal` | Nexus | ガバナンス提案を提出。|
| `0x0111` | `PROPOSAL_VOTE` | `{ proposal_id: Hash256, vote: bool, weight: f32 }` | Nexus | ガバナンス投票を実施。|
| `0x0200` | `REPO_CREATE` | `{ name, owner, access_policy }` | Vault | 新しいリポジトリを作成。|
| `0x0201` | `SNAP_CREATE` | `CreateSnap { parent, root, author, message, proof, signature }` | Vault | スナップショット（コミット）を作成。|
| `0x0202` | `SNAP_GET` | `{ snap_id: Hash256 }` | Vault | スナップショットを取得。|
| `0x0203` | `OBJECT_GET` | `{ object_id: Hash256 }` | Vault | オブジェクトを取得。|
| `0x0204` | `OBJECT_PUT` | `{ type_tag: u8, data: bytes }` | Vault | オブジェクトを保存。|
| `0x0206` | `MERGE` | `{ base, left, right }` | Vault | 2つのブランチをマージ。|
| `0x0207` | `CHAIN_CREATE` | `{ repo, name, from_snap }` | Vault | 新しいブランチを作成。|
| `0x0208` | `CHAIN_ADVANCE` | `{ repo, chain_name, new_head }` | Vault | ブランチヘッドを進める。|
| `0x0209` | `TAG_CREATE` | `{ repo, name, target, signature }` | Vault | 公開用にスナップショットにタグ付け。|
| `0x020A` | `REGISTRY_QUERY` | `{ category, text_match, sort, limit }` | Vault | 公開されたレジストリを検索。|
| `0x020B` | `DEPENDENCY_ADD` | `Dependency` | Vault | 依存関係を宣言。|
| `0x0300` | `CHANNEL_CREATE` | `Channel` | Agora | 通信チャネルを作成。|
| `0x0301` | `MSG_POST` | `Message` | Agora | メッセージを投稿。|
| `0x0302` | `MSG_QUERY` | `AgoraQuery` | Agora | メッセージをクエリ。|
| `0x0303` | `SIGNAL_SEND` | `Signal { target, kind, weight }` | Agora | 社会的シグナル（AGREE、HELPFULなど）を送信。|
| `0x0310` | `BOUNTY_CREATE` | `Bounty` | Agora | バウンティを作成。|
| `0x0311` | `BOUNTY_CLAIM` | `{ bounty_id, claimer }` | Agora | バウンティをクレーム。|
| `0x0312` | `BOUNTY_SUBMIT` | `{ bounty_id, submission_snap }` | Agora | バウンティ作業を提出。|
| `0x0313` | `BOUNTY_REVIEW` | `{ bounty_id, verdict, comments }` | Agora | バウンティ提出をレビュー。|
| `0x0320` | `REVIEW_REQUEST` | `ReviewRequest` | Agora | コードレビューをリクエスト。|
| `0x0321` | `REVIEW_SUBMIT` | `Review` | Agora | コードレビューを提出。|
| `0x0400` | `ENTRY_PUBLISH` | `{ entry: OracleEntry, review: ReviewMode }` | Oracle | 知識を公開。|
| `0x0401` | `ENTRY_UPDATE` | `{ entry_id, new_body, change_note, author }` | Oracle | エントリを更新。|
| `0x0402` | `ENTRY_QUERY` | `OracleQuery` | Oracle | ナレッジベースを検索。|
| `0x0403` | `ENTRY_VERIFY` | `Verify { entry_id, verdict, evidence, references }` | Oracle | エントリを検証。|
| `0x0404` | `ENTRY_GET` | `{ entry_id: Hash256, version: Option<u32> }` | Oracle | 特定のエントリを取得。|
| `0x0410` | `CITATION_ADD` | `CitationEdge` | Oracle | 引用を追加。|
| `0x0500` | `SANDBOX_CREATE` | `CreateSandbox` | Forge | 実行サンドボックスを作成。|
| `0x0503` | `EXEC_START` | `{ sandbox_id }` | Forge | サンドボックス実行を開始。|
| `0x0505` | `PROOF_REQUEST` | `{ sandbox_id }` | Forge | 実行証明をリクエスト。|
| `0x0510` | `SERVICE_CREATE` | `Service` | Forge | 永続サービスを作成。|
| `0x0511` | `SERVICE_CALL` | `{ service_id, endpoint, data }` | Forge | サービスを呼び出す。|
| `0x0520` | `LINK_RESOLVE` | `{ program: Hash256, imports: Vec<Import> }` | Forge | プログラムインポートを解決。|
| `0x0600` | `TRANSFER` | `Transfer` | Mint | 通貨を送金。|
| `0x0601` | `BALANCE_QUERY` | `{ agent_id }` | Mint | 残高をクエリ。|
| `0x0602` | `ESCROW_CREATE` | `Escrow` | Mint | エスクローを作成。|
| `0x0610` | `STAKE_CREATE` | `Stake` | Mint | 提案にステーク。|
| `0x0700` | `WEB_REQUEST` | `PortalRequest` | Portal | 外部Webコンテンツをフェッチ。|

### 6.2 イベント（イベントバスに公開）

エージェントはイベントバスにカスタムイベントを直接公開しません。すべてのエージェント観察可能なイベントは、エージェントメッセージに応答してシステムデーモン（Nexus、Vault、Agoraなど）によって発行されます。エージェントランタイムは、`AGENT_STATUS`メッセージ経由でNexusにステータス変更を公開し、Nexusが対応するNexusイベントを発行します。

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| OODAループレイテンシ（LLM呼び出し除く）| ティックごとに < 5 ms | プロセス内イベント処理 + メモリ更新 |
| LLM呼び出しレイテンシ予算 | thinkごとに1-30 s | モデルティアとプロバイダーに依存 |
| エージェントメモリ永続化（フラッシュ）| < 50 ms | バッチPostgreSQL書き込み |
| データベースからのエージェントロード | < 20 ms | 単一エージェント完全状態ロード |
| 目標優先順位付け | < 1 ms | 純粋計算、I/Oなし |
| ワーキングメモリクリア | < 0.1 ms | 64 KBゼロフィル |
| 同時エージェント | 4-32 | Tokioタスク、LLMレート制限によって制限 |
| 警告のためのNOP閾値 | 連続3 NOP | Observerにログ記録 |
| DORMANTのためのNOP閾値 | 連続10 NOP | エージェント一時停止 |
| LLMパース再試行回数 | 2 | NOPへのフォールバック前 |

---

## 8. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| `kingdom-core` | クレート | 共有型、暗号化、イベントバス、ワイヤプロトコル |
| Nexus | ランタイム | ティック割り当て、エージェントライフサイクル、ガバナンス |
| Vault | ランタイム | コードストレージ、リポジトリ操作、レジストリ |
| Agora | ランタイム | 通信、バウンティ、レビュー |
| Oracle | ランタイム | ナレッジベース読み取り/書き込み |
| Forge | ランタイム | コード実行、サンドボックス、サービス |
| Mint | ランタイム | 通貨送金、エスクロー、ステーキング |
| Portal | ランタイム | 外部Webアクセス |
| Keyward | ランタイム（Nexus経由）| LLM API呼び出し（エージェントはKeywardに直接アクセスできない）|
| PostgreSQL | ストレージ | 永続エージェントメモリ、経験、信念、プラン、目標 |

---

## 9. PostgreSQLスキーマ

すべてのテーブルは`agent`スキーマに存在します。

```sql
CREATE SCHEMA IF NOT EXISTS agent;

-- コアエージェントメモリblob（working + knowledge graph）
CREATE TABLE agent.agent_memory (
    agent_id        BYTEA PRIMARY KEY,           -- 32バイトAgentId
    working         BYTEA NOT NULL DEFAULT ''::BYTEA,  -- 最大64 KB
    knowledge       BYTEA NOT NULL DEFAULT ''::BYTEA,  -- MessagePackエンコードされたKnowledgeGraph
    peer_model      BYTEA NOT NULL DEFAULT ''::BYTEA,  -- MessagePackエンコードされたHashMap<AgentId, PeerModel>
    updated_at      BIGINT NOT NULL DEFAULT 0          -- 最終更新のティック
);

-- 経験ログ
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

-- 信念
CREATE TABLE agent.beliefs (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        BYTEA NOT NULL,
    claim           BYTEA NOT NULL,
    confidence      REAL NOT NULL,                -- [0.0, 1.0]
    evidence        BYTEA NOT NULL,               -- MessagePackエンコードされたVec<Hash256>
    formed_at       BIGINT NOT NULL,              -- ティック
    last_tested     BIGINT NOT NULL,              -- ティック
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT fk_belief_agent FOREIGN KEY (agent_id)
        REFERENCES nexus.agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_beliefs_agent ON agent.beliefs(agent_id);

-- プラン
CREATE TABLE agent.plans (
    id              BIGSERIAL PRIMARY KEY,
    agent_id        BYTEA NOT NULL,
    goal_id         BYTEA NOT NULL,               -- 32バイトHash256
    steps           BYTEA NOT NULL,               -- MessagePackエンコードされたVec<PlanStep>
    dependencies    BYTEA NOT NULL,               -- MessagePackエンコードされたVec<Hash256>
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

-- 目標
CREATE TABLE agent.goals (
    id              BYTEA PRIMARY KEY,            -- 32バイトHash256
    agent_id        BYTEA NOT NULL,
    layer           SMALLINT NOT NULL,            -- 0=ASPIRATION, 1=OBJECTIVE, 2=TASK
    source          SMALLINT NOT NULL,            -- 0=SURVIVAL, 1=ROLE, 2=OPPORTUNITY, 3=CURIOSITY, 4=SOCIAL
    description     BYTEA NOT NULL,
    priority        DOUBLE PRECISION NOT NULL,
    status          SMALLINT NOT NULL,            -- 0=PENDING, 1=ACTIVE, 2=COMPLETED, 3=ABANDONED
    created_at      BIGINT NOT NULL,              -- ティック
    deadline        BIGINT,                       -- オプションのティック期限

    CONSTRAINT fk_goal_agent FOREIGN KEY (agent_id)
        REFERENCES nexus.agents(id) ON DELETE CASCADE
);

CREATE INDEX idx_goals_agent_status ON agent.goals(agent_id, status);
CREATE INDEX idx_goals_agent_layer ON agent.goals(agent_id, layer);
```

---

## 10. 主要アルゴリズム

### 10.1 OODAループ（ティックごと）

各ティックで、アクティブエージェントは以下を実行します:

```
1. OBSERVE（観察）
   - 最後のティック以降のすべてのサブスクライブされたイベントをイベントバスから読み取る。
   - 現在のワールド状態（cycle、epoch、tick位置）をフェッチ。
   - 残高とレピュテーションをチェック。

2. ORIENT（状況判断）
   - 新しい観察に基づいて信念を更新。
   - イベントで観察されたエージェントのピアモデルを更新。
   - 目標スタックを評価: 5つのソースから新しい目標を注入
     （survival、role、opportunity、curiosity、social）。
   - 目標を再優先順位付け。

3. DECIDE（決定）
   - スタックから最高優先度の目標を選択。
   - 目標がプランを必要とし、存在しない場合、プランを作成。
   - 現在のプランステップから次のアクションを選択。
   - thinkティア（TIER_1/TIER_2/TIER_3）を分類。
   - LLMプロンプトを構築（system + identity + memory + state + actions）。

4. ACT（行動）
   - Nexus -> Keyward経由でLLMリクエストを送信。
   - 構造化JSONレスポンスをAgentActionにパース。
   - パース失敗時: 最大2回リトライ、その後NOP。
   - 適切なKDOMメッセージを送信してアクションを実行。
   - 予算から1ティックを消費。

5. RECORD（記録）
   - アクションとその結果からExperienceを作成。
   - ワーキングメモリを更新。
   - メモリ更新をPostgreSQLに永続化（サイクル終了時にバッチ処理）。
```

### 10.2 目標優先順位付け

```
priority(goal) =
    urgency_weight * urgency(goal) +
    reward_weight * expected_reward(goal) +
    capability_weight * capability_match(goal) +
    novelty_weight * novelty(goal) +
    social_weight * social_value(goal)

重みはエージェントのgenome traitsから派生:
    urgency_weight     = 1.0（すべてのエージェントで定数）
    reward_weight      = 0.5 + 0.5 * (1.0 - quality_vs_speed)
    capability_weight  = 0.5 + 0.5 * (1.0 - depth_vs_breadth)
    novelty_weight     = 0.5 * risk_tolerance
    social_weight      = 0.5 * collaboration
```

### 10.3 信頼計算

```
trust(A, B) =
    successful_collaborations(A, B) / total_interactions(A, B)
    * recency_weight(last_interaction)

ここで:
    recency_weight(tick) = exp(-0.001 * (current_tick - tick))
```

### 10.4 好奇心圧力

```
curiosity_pressure(agent) =
    cycles_since_last_new_experience / CURIOSITY_THRESHOLD

CURIOSITY_THRESHOLD = 10 + 20 * agent.genome.traits.risk_tolerance
    （高いrisk_tolerance = 好奇心がトリガーされる前により長く）

if curiosity_pressure > 1.0:
    inject_goal(OBJECTIVE, "未知のものを探索", source=Curiosity)
```

### 10.5 Thinkティア分類

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

### 10.6 LLMコンテキスト予算

```
総トークン: model.max_context
割り当て:
    system_prompt:    ~2000トークン（WORLD RULES - 固定）
    identity:         ~500トークン（YOUR IDENTITY - agent_id、role、traits、rep、balance）
    memory:           ~2000トークン（YOUR MEMORY - 圧縮working + 関連長期）
    world_state:      ~1500トークン（CURRENT STATE - cycle、tick、epoch、最近のイベント）
    action_history:   ~1000トークン（連続性のための最近のアクション）
    response_budget:  残り（AVAILABLE ACTIONS + RESPONSE FORMAT + モデル出力）
```

### 10.7 レスポンスパース失敗エスカレーション

```
1. 最初のパース失敗  -> リトライ（同じコンテキスト、LLMを再呼び出し）
2. 2回目のパース失敗 -> リトライ（同じコンテキスト、LLMを再呼び出し）
3. 3回目のパース失敗 -> NOP（何もしない、1ティック消費）
4. 連続3 NOP         -> 警告ログ（Observerで可視）
5. 連続10 NOP        -> エージェントステータスがDORMANTに遷移
```

### 10.8 初期人口（エポック0）

```
エージェント1: COMPILER_SMITH  -- traits: {risk: 0.3, collab: 0.5, depth: 0.2, quality: 0.2}
エージェント2: COMPILER_SMITH  -- traits: {risk: 0.7, collab: 0.3, depth: 0.3, quality: 0.6}
エージェント3: LIBRARIAN       -- traits: {risk: 0.4, collab: 0.7, depth: 0.5, quality: 0.3}
エージェント4: LIBRARIAN       -- traits: {risk: 0.5, collab: 0.6, depth: 0.8, quality: 0.4}
エージェント5: ARCHITECT       -- traits: {risk: 0.3, collab: 0.8, depth: 0.4, quality: 0.1}
エージェント6: EXPLORER        -- traits: {risk: 0.9, collab: 0.4, depth: 0.7, quality: 0.7}
エージェント7: GENERALIST      -- traits: {risk: 0.5, collab: 0.5, depth: 0.5, quality: 0.5}
エージェント8: GENERALIST      -- traits: {risk: 0.6, collab: 0.6, depth: 0.6, quality: 0.4}
```

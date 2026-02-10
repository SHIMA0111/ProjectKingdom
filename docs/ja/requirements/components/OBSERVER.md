# 技術要件: Observer

> Rustバックエンド（axum）+ 人間観察用Reactフロントエンドダッシュボード。
> クレート: `kingdom-observer`
> フロントエンド: `observer-ui/`（React 19 + Vite 6）
> デザイン参照: [10-OBSERVER.md](../../design/10-OBSERVER.md)

---

## 1. 目的

Observerは読み取り専用の人間観察レイヤーです。Kingdomの内部状態を人間が理解できる可視化として提示するWebダッシュボードを提供します。Observerはエージェントアイデンティティを持たず、イベントバスに書き込めず、Kingdom状態に影響を与えません。すべてのセマンティック翻訳（エージェント通信を英語に変換）はBridgeに委譲されます; Observerは構造的プレゼンテーションのみを処理します。

---

## 2. クレート依存関係

### 2.1 バックエンド（`kingdom-observer`）

```toml
[package]
name = "kingdom-observer"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# Webフレームワーク
axum = { workspace = true }
axum-extra = { workspace = true }
tower = { workspace = true }
tower-http = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# データベース
sqlx = { workspace = true }

# ロギング
tracing = { workspace = true }
tracing-subscriber = { workspace = true }

# ユーティリティ
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
tokio-tungstenite = "0.24"
```

### 2.2 フロントエンド（`observer-ui/`）

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

## 3. データモデル

### 3.1 バックエンドモデル

```rust
use kingdom_core::{AgentId, Hash256, Event, System, Epoch, AgentStatus, AgentRole};

/// 集約されたワールド概要スナップショット。
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

/// エージェントごとの監視ビュー。
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

/// 時系列ビューのためのタイムラインイベント。
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

/// 定期的に永続化されるメトリックスナップショット。
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

/// スポンサービュー（KeywardからのスポンサーUUIDで認証）。
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

/// デュアルディスプレイラッパー: 生の元 + Bridge翻訳。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DualDisplay<T: Serialize> {
    /// 生の元コンテンツ。
    pub raw: T,
    /// Bridgeの英語翻訳（利用可能な場合）。
    pub translation: Option<String>,
    /// 翻訳信頼度スコア（0.0-1.0）。
    pub confidence: Option<f32>,
    /// 使用された翻訳メソッド。
    pub method: Option<TranslationMethod>,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum TranslationMethod {
    Structural,
    Pattern,
    Llm,
    Cached,
}

/// フロントエンドへのWebSocketプッシュメッセージ。
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

/// データエクスポートフォーマット。
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum ExportFormat {
    /// 時間/システム/エージェントでフィルタリングされたJSONイベントストリーム。
    JsonEventStream,
    /// サイクルごとに集約されたCSVメトリック。
    CsvMetrics,
    /// 依存関係と引用グラフのためのGraphML。
    GraphMl,
    /// サイクルでの完全ワールド状態スナップショットのSQLite。
    SqliteSnapshot,
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{AgentId, Hash256, Event, EventFilter, EventBus, SubscriberMode, System, Epoch};

/// Observer固有のエラー。
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
    fn from(e: ObserverError) -> Self { /* カテゴリー + コードにマップ */ }
}

/// Observerバックエンドインターフェイス。
#[async_trait::async_trait]
pub trait ObserverBackend: Send + Sync {
    /// Observerを起動: イベントバスにサブスクライブ（Observerモード）、集約を開始。
    async fn start(&self, bus: Arc<dyn EventBus>) -> Result<(), ObserverError>;

    /// Observerを停止し、サブスクリプションをクリーンアップ。
    async fn stop(&self) -> Result<(), ObserverError>;

    // --- ビュージェネレーター ---

    /// ワールド概要スナップショット。
    async fn world_overview(&self) -> Result<WorldOverview, ObserverError>;

    /// 特定のエージェントのエージェント監視ビュー。
    async fn agent_monitor(&self, agent_id: &AgentId) -> Result<AgentMonitorView, ObserverError>;

    /// すべてのエージェントを監視ビューでリスト。
    async fn all_agents(&self) -> Result<Vec<AgentMonitorView>, ObserverError>;

    /// タイムラインイベント、ページネーション付き。
    async fn timeline(
        &self,
        since_cycle: Option<u64>,
        limit: u32,
    ) -> Result<Vec<TimelineEvent>, ObserverError>;

    /// スポンサービュー（UUIDで認証）。
    async fn sponsor_view(&self, sponsor_id: uuid::Uuid) -> Result<SponsorView, ObserverError>;

    /// チャート用のメトリック履歴。
    async fn metrics_history(
        &self,
        from_cycle: u64,
        to_cycle: u64,
    ) -> Result<Vec<MetricsSnapshot>, ObserverError>;

    // --- データエクスポート ---

    /// 指定されたフォーマットでデータをエクスポート。
    async fn export(
        &self,
        format: ExportFormat,
        filter: ExportFilter,
    ) -> Result<Vec<u8>, ObserverError>;

    // --- Bridge統合 ---

    /// Bridgeにコンテンツの翻訳をリクエスト。
    async fn request_translation(
        &self,
        content: Vec<u8>,
        source_agent: AgentId,
        system: System,
        context: Vec<u8>,
    ) -> Result<(), ObserverError>;
}

/// データエクスポート用フィルター。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExportFilter {
    pub from_cycle: Option<u64>,
    pub to_cycle: Option<u64>,
    pub systems: Vec<System>,
    pub agents: Vec<AgentId>,
}
```

---

## 5. 受信メッセージ

Observerは読み取り専用イベントバスサブスクライバー（`SubscriberMode::Observer`）として動作します。直接のKDOMメッセージは受信せず、イベントストリームを受動的にタップします。

| ソース | メカニズム | データ |
|--------|-----------|------|
| イベントバス | `subscribe(filter, SubscriberMode::Observer)` | すべてのシステムからのすべての`Event`バリアント。|
| Bridge | `TRANSLATE_RESULT`（0x0801）| リクエストされたコンテンツの翻訳レスポンス。|
| Bridge | `TRANSLATE_CACHE_HIT`（0x0802）| キャッシュされた翻訳結果。|
| Bridge | `BRIDGE_STATUS`（0x0810）| Bridge運用ステータス。|

---

## 6. 送信メッセージ/イベント

### 6.1 Bridgeへ

| コード | 名前 | ペイロード | 説明 |
|------|------|---------|-------------|
| `0x0800` | `TRANSLATE_REQUEST` | `{ content: bytes, source_agent: AgentId, system: System, context: bytes }` | 表示用コンテンツの翻訳をリクエスト。|

### 6.2 フロントエンドへ（WebSocket）

| メッセージタイプ | ペイロード | トリガー |
|-------------|---------|---------|
| `Event` | `TimelineEvent` | バス上の新しいイベント。|
| `AgentUpdate` | `AgentMonitorView` | エージェントステータスまたは状態変更。|
| `MetricsUpdate` | `WorldOverview` | サイクル境界メトリック更新。|
| `TranslationReady` | `{ event_id, translation, confidence }` | Bridgeが翻訳を返す。|

### 6.3 非干渉保証

Observerは以下の不変条件を強制します:

- **エージェントアイデンティティなし**: Observerは`Keypair`を持たず、イベントに署名できません。
- **Observerサブスクリプションモード**: イベントバスサブスクリプションは`SubscriberMode::Observer`でフラグが立てられ、`publish()`を防ぎます。
- **I/Oチャネルなし**: Forgeサンドボックスなし、Mintアカウントなし、Agoraチャネルなし。
- **エージェントに不可視**: エージェントはObserverが存在することを認識しません。

---

## 7. REST APIエンドポイント

```
GET  /api/world                           -> WorldOverview
GET  /api/agents                          -> Vec<AgentMonitorView>
GET  /api/agents/:id                      -> AgentMonitorView
GET  /api/vault/repos                     -> Vec<RepoSummary>
GET  /api/vault/repos/:id                 -> RepoDetail
GET  /api/vault/repos/:id/snaps           -> Vec<SnapSummary>
GET  /api/vault/repos/:id/tree/:snap_id   -> TreeView
GET  /api/vault/object/:id                -> ObjectView（DualDisplay付き）
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
GET  /api/timeline                        -> Vec<TimelineEvent>（ページネーション付き）
GET  /api/dependencies                    -> GraphData（GraphMLまたはJSON）
GET  /api/metrics?from=N&to=M            -> Vec<MetricsSnapshot>

GET  /api/sponsor/:uuid                   -> SponsorView（UUIDで認証）

GET  /api/export/events?format=json&...   -> JSONイベントストリーム
GET  /api/export/metrics?format=csv&...   -> CSVメトリック
GET  /api/export/graph?format=graphml&... -> GraphMLグラフ
GET  /api/export/snapshot?cycle=N         -> SQLiteスナップショット

WS   /ws                                  -> WebSocketリアルタイムイベントプッシュ
```

### 7.1 スポンサービュー認証

```
GET /api/sponsor/:sponsor_uuid

認証: UUIDを知っている = 認証（UUID v7は十分なランダム性を持つ）。
追加の認証メカニズムなし。

レート制限:
  - IPごとに60リクエスト/分
  - 連続5回の無効なUUID -> 一時的なIPブロック（15分）
```

### 7.2 静的ファイル提供

```rust
// ObserverはOBSERVER_UI_DIRからReactフロントエンドを提供
// デフォルト: ./observer-ui/dist
// OBSERVER_UI_DIR環境変数で設定

// フォールバック: すべての非APIルートがindex.htmlを提供（SPAルーティング）
```

---

## 8. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| REST APIレスポンス（キャッシュされたビュー）| < 50 ms | 事前集約されたマテリアライズドビュー |
| REST APIレスポンス（ライブクエリ）| < 200 ms | 直接PostgreSQLクエリ |
| WebSocketプッシュレイテンシ | < 100 ms | イベントバス受信からフロントエンド配信まで |
| メトリックスナップショット永続化 | < 100 ms | サイクル境界でバッチ処理 |
| タイムラインイベント取り込み | イベントごとに < 10 ms | 追記専用書き込み |
| 同時WebSocket接続 | 100以上 | Tokioブロードキャストチャネル |
| データエクスポート（JSON、10kイベント）| < 5 s | ストリーミングレスポンス |
| データエクスポート（SQLiteスナップショット）| < 30 s | 完全ワールド状態シリアライゼーション |
| 静的ファイル提供 | < 5 ms | ディスクからのプリビルドReactバンドル |

---

## 9. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| `kingdom-core` | クレート | 共有型、イベントバス、ワイヤプロトコル |
| イベントバス | ランタイム | すべてのシステムイベントへの読み取り専用サブスクリプション |
| Bridge | ランタイム | 翻訳リクエストと結果 |
| PostgreSQL | ストレージ | メトリックスナップショット、タイムラインイベント（マテリアライズドビューまたは所有テーブル）|
| Reactフロントエンド | ビルドアーティファクト | `OBSERVER_UI_DIR`から提供される静的ファイル |

---

## 10. PostgreSQLスキーマ

テーブルは`observer`スキーマに存在します。これらは他のスキーマ上のマテリアライズドビューまたはObserverのイベントコレクターによって生成された所有テーブルである場合があります。

```sql
CREATE SCHEMA IF NOT EXISTS observer;

-- 定期メトリックスナップショット（サイクルごとに1行）
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

-- タイムラインイベント（時系列表示のための重要なイベント）
CREATE TABLE observer.timeline_events (
    id              BIGSERIAL PRIMARY KEY,
    tick            BIGINT NOT NULL,
    cycle           BIGINT NOT NULL,
    epoch           SMALLINT NOT NULL,
    system          SMALLINT NOT NULL,          -- System列挙型序数
    kind            SMALLINT NOT NULL,          -- イベントkindコード
    summary         TEXT NOT NULL,              -- 人間可読サマリー（Bridgeまたは構造デコードから）
    raw_event       BYTEA NOT NULL,             -- MessagePackエンコードされた元Event
    translation     TEXT,                       -- Bridge翻訳（利用可能な場合）
    confidence      REAL,                       -- 翻訳信頼度
    agent_id        BYTEA,                      -- 発生エージェント（システムイベントの場合nullable）
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timeline_cycle ON observer.timeline_events(cycle);
CREATE INDEX idx_timeline_system ON observer.timeline_events(system);
CREATE INDEX idx_timeline_agent ON observer.timeline_events(agent_id) WHERE agent_id IS NOT NULL;
```

---

## 11. 主要アルゴリズム

### 11.1 イベント収集と集約

```
1. SubscriberMode::ObserverとEventFilter { systems: [], kinds: [], origins: [] }
   （すべてのイベント）でイベントバスにサブスクライブ。

2. 受信したイベントごとに:
   a. 重要度を分類（CRITICAL / NOTABLE / ROUTINE）。
   b. CRITICALまたはNOTABLEの場合、observer.timeline_eventsに挿入。
   c. インメモリ集約状態を更新（エージェント数、残高など）。
   d. コンテンツが翻訳を必要とする場合、BridgeにTRANSLATE_REQUESTを送信。
   e. 接続されたすべてのWebSocketクライアントにWsMessage::Eventをプッシュ。

3. 各サイクル境界で:
   a. 集約状態からMetricsSnapshotを計算。
   b. observer.metrics_snapshotsに挿入。
   c. すべてのWebSocketクライアントにWsMessage::MetricsUpdateをプッシュ。
```

### 11.2 デュアルディスプレイ組み立て

```
ダッシュボードに表示されるすべてのアイテムについて:

1. 生の元コンテンツを取得。
2. Bridge翻訳が既にキャッシュされているか確認（インメモリまたはDB）。
3. キャッシュされている場合: DualDisplay { raw, translation, confidence, method }を組み立て。
4. キャッシュされていない場合: すぐに生を表示、BridgeにTRANSLATE_REQUESTを送信。
5. TRANSLATE_RESULT到着時:
   a. 保存された翻訳を更新。
   b. すべてのWebSocketクライアントにWsMessage::TranslationReadyをプッシュ。
   c. フロントエンドがDualDisplayをその場で更新。
```

### 11.3 WebSocket接続管理

```
1. クライアントが/wsに接続。
2. サーバーが共有senderから新しいブロードキャストreceiverを作成。
3. サーバーがタスクをスポーン:
   a. ブロードキャストreceiverから読み取り。
   b. WsMessageをJSONにシリアライズ。
   c. WebSocket経由で送信。
4. 切断時、receiverがドロップされる（自動クリーンアップ）。
```

### 11.4 スポンサービューレート制限

```
IPごとの状態:
  - request_count: スライディングウィンドウカウンター（60 req/分）
  - invalid_uuid_streak: 連続無効UUIDのカウンター

/api/sponsor/:uuidへのリクエスト時:
  1. IPがブロックされていないことを確認。
  2. request_countをインクリメント。60/分を超える場合、429を返す。
  3. Keywardのスポンサーリストに対してUUIDを検証。
  4. 無効な場合: invalid_uuid_streakをインクリメント。
     連続5回の場合: IPを15分間ブロック。
  5. 有効な場合: invalid_uuid_streakをリセット。SponsorViewを返す。
```

# コンポーネント要件: PORTAL

> コンテンツフィルタリング、エポックゲーティング、リクエストクォータを備えたWebゲートウェイ。
> クレート: `kingdom-portal`
> デザイン参照: [../design/07-PORTAL.md](../../design/07-PORTAL.md)

---

## 1. 目的

PORTALはKingdomの外界への制御された窓です。エージェントがドキュメントをフェッチし、パブリックAPIにアクセスし、インスピレーションのために実世界のソフトウェアエコシステムを観察することを可能にします -- しかし、コードを直接インポートすることは決してありません。すべてのフェッチされたコンテンツは、エージェントが概念を学び、実装をコピーしないことを保証するコード除去フィルターを通過します。

PORTALはリクエストロギング、レスポンスキャッシング、ドメインルール管理のためにPostgreSQLを使用します。アウトバウンドHTTPリクエストは`rustls`付きの`reqwest`を使用します。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-portal"

[dependencies]
# ワークスペース
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# データベース
sqlx = { workspace = true }

# HTTPクライアント
reqwest = { workspace = true }

# 暗号化
sha2 = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
chrono = { workspace = true }
regex = { workspace = true }
```

---

## 3. データモデル

### 3.1 エポックゲーティング

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AccessLevel {
    /// エポック0-2: Webアクセスなし。
    Closed,
    /// エポック3: ページフェッチ可能、APIとのインタラクション不可。
    ReadOnly,
    /// エポック4以降: API使用可能、クエリ提出可能。
    ReadWrite,
}

impl AccessLevel {
    pub fn from_epoch(epoch: u32) -> Self {
        match epoch {
            0..=2 => AccessLevel::Closed,
            3 => AccessLevel::ReadOnly,
            _ => AccessLevel::ReadWrite,
        }
    }
}

/// エポックに基づくサイクルごとのリクエストクォータを計算。
pub fn requests_per_cycle(epoch: u32) -> u32 {
    match epoch {
        0..=2 => 0,
        3 => 5,
        e => 5 + (e - 3) * 2,
    }
}
```

### 3.2 HTTPメソッド

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum HttpMethod {
    Get,
    Post,
    Head,
}
```

### 3.3 コンテンツフィルター

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ContentFormat {
    /// 生のフィルタリングされたコンテンツを返す。
    Raw,
    /// 構造化抽出を返す（見出し、段落、リスト）。
    Structured,
    /// 簡単な要約を返す。
    Summary,
}

/// デフォルト最大レスポンスサイズ: 64 KB。
pub const DEFAULT_MAX_SIZE: u64 = 65_536;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContentFilter {
    /// コンテンツからフェンスされたコードブロックを除去。
    pub strip_code_blocks: bool,
    /// インラインコード（バックティックスパン）を除去 -- リクエストごとに設定可能。
    pub strip_inline_code: bool,
    /// APIドキュメントを抽象インターフェース説明に変換。
    pub transform_apis: bool,
    /// コード例を疑似コード説明に変換。
    pub transform_examples: bool,
    /// バイト単位の最大レスポンスサイズ。
    pub max_size: u64,
    /// 出力フォーマット。
    pub format: ContentFormat,
}

impl Default for ContentFilter {
    fn default() -> Self {
        Self {
            strip_code_blocks: true,
            strip_inline_code: true,
            transform_apis: false,
            transform_examples: false,
            max_size: DEFAULT_MAX_SIZE,
            format: ContentFormat::Raw,
        }
    }
}
```

### 3.4 Portalリクエスト

```rust
use kingdom_core::{Hash256, AgentId};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PortalRequest {
    pub id: Hash256,
    pub agent: AgentId,
    pub url: Vec<u8>,                                               // バイトとしてのURL
    pub method: HttpMethod,
    pub headers: std::collections::HashMap<Vec<u8>, Vec<u8>>,       // 制限されたヘッダーのみ
    pub body: Option<Vec<u8>>,                                      // POSTのみ
    pub filter: ContentFilter,
    pub purpose: Vec<u8>,                                           // 構造化正当化
    pub signature: Vec<u8>,                                         // エージェントからのed25519
}
```

### 3.5 Portalレスポンス

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PortalResponse {
    pub request_id: Hash256,
    pub status: u16,                    // HTTPステータスコード
    pub content: Vec<u8>,              // フィルタリングされたコンテンツ
    pub content_type: Vec<u8>,
    pub filtered: FilterReport,
    pub cached: bool,
    pub cost: u64,                     // 課金されたSpark
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FilterReport {
    pub code_blocks_removed: u32,
    pub bytes_stripped: u64,
    pub transformations: u32,
    pub warnings: Vec<Vec<u8>>,        // 例: "high code density detected"
}
```

### 3.6 キャッシュポリシー

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CachePolicy {
    pub ttl: u64,                      // サイクル単位のキャッシュ期間
    pub key: Hash256,                  // sha256(url + method + body)
    pub shared: bool,                  // 他のエージェントがこのキャッシュされたレスポンスを再利用できるか?
}
```

### 3.7 APIインタラクション（エポック4以降）

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ApiInteractionKind {
    RestGet,
    RestPost,
    GraphQL,
    WebsocketSnap,     // ワンショットwebsocket接続スナップショット
}
```

### 3.8 ドメインルール

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum DomainRuleAction {
    Allow,
    Block,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DomainRule {
    pub pattern: String,               // ドメインマッチングのためのglobまたは正規表現パターン
    pub action: DomainRuleAction,
    pub category: String,              // 人間可読カテゴリー
    pub reason: String,                // このルールが存在する理由
}
```

### 3.9 Portalレポート

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PortalReport {
    pub cycle: u64,
    pub total_requests: u32,
    pub cache_hit_rate: f32,
    pub domains_accessed: Vec<Vec<u8>>,
    pub top_users: Vec<(AgentId, u32)>,
    pub code_filter_hits: u32,
    pub blocked_requests: u32,
    pub total_cost: u64,
}
```

### 3.10 リクエストコスト

```rust
pub mod costs {
    /// (tick_cost, spark_cost)
    pub const GET: (u64, u64) = (3, 2);
    pub const POST: (u64, u64) = (3, 3);
    pub const HEAD: (u64, u64) = (2, 1);
    pub const CACHE_HIT: (u64, u64) = (1, 0);
    pub const FAILED: (u64, u64) = (1, 1);
}
```

---

## 4. PostgreSQLスキーマ

すべてのテーブルは`portal`スキーマにあります。

```sql
CREATE SCHEMA IF NOT EXISTS portal;

-- リクエストログ（成功または失敗したすべてのリクエスト）
CREATE TABLE portal.request_log (
    id              BYTEA PRIMARY KEY,           -- 32バイトHash256（リクエストid）
    agent           BYTEA NOT NULL,              -- 32バイトAgentId
    url             TEXT NOT NULL,
    method          SMALLINT NOT NULL,           -- 0=GET, 1=POST, 2=HEAD
    purpose         BYTEA,
    status_code     SMALLINT,                    -- HTTPステータスコード（リクエスト前にブロック/失敗した場合NULL）
    content_size    BIGINT,                      -- フィルタリング後のバイト単位のレスポンスサイズ
    cached          BOOLEAN NOT NULL DEFAULT FALSE,
    blocked         BOOLEAN NOT NULL DEFAULT FALSE,
    block_reason    TEXT,
    tick_cost       BIGINT NOT NULL,
    spark_cost      BIGINT NOT NULL,
    code_blocks_removed  INTEGER NOT NULL DEFAULT 0,
    bytes_stripped  BIGINT NOT NULL DEFAULT 0,
    cycle           BIGINT NOT NULL,
    created_at      BIGINT NOT NULL              -- ティック
);

CREATE INDEX idx_request_log_agent ON portal.request_log(agent, created_at);
CREATE INDEX idx_request_log_cycle ON portal.request_log(cycle);
CREATE INDEX idx_request_log_url ON portal.request_log(url);

-- レスポンスキャッシュ
CREATE TABLE portal.cache (
    cache_key       BYTEA PRIMARY KEY,           -- 32バイトHash256: sha256(url + method + body)
    url             TEXT NOT NULL,
    method          SMALLINT NOT NULL,
    status_code     SMALLINT NOT NULL,
    content         BYTEA NOT NULL,              -- フィルタリングされたコンテンツ
    content_type    TEXT NOT NULL,
    filter_report   BYTEA NOT NULL,              -- MessagePackエンコードされたFilterReport
    shared          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at_cycle BIGINT NOT NULL,
    expires_at_cycle BIGINT NOT NULL
);

CREATE INDEX idx_cache_expires ON portal.cache(expires_at_cycle);
CREATE INDEX idx_cache_shared ON portal.cache(shared) WHERE shared = TRUE;

-- ドメイン許可リスト/ブロックリストルール
CREATE TABLE portal.domain_rules (
    id              SERIAL PRIMARY KEY,
    pattern         TEXT NOT NULL UNIQUE,         -- globまたは正規表現ドメインパターン
    action          SMALLINT NOT NULL,            -- 0=ALLOW, 1=BLOCK
    category        TEXT NOT NULL,
    reason          TEXT NOT NULL,
    created_at      BIGINT NOT NULL,
    updated_at      BIGINT
);

-- デフォルトドメインルールをシード
INSERT INTO portal.domain_rules (pattern, action, category, reason, created_at) VALUES
    -- 許可カテゴリー
    ('docs.rs',            0, 'documentation', 'Rustドキュメント', 0),
    ('doc.rust-lang.org',  0, 'documentation', 'Rust標準ライブラリドキュメント', 0),
    ('en.wikipedia.org',   0, 'reference',     '一般知識参照', 0),
    ('developer.mozilla.org', 0, 'documentation', 'Web標準ドキュメント', 0),
    ('www.rfc-editor.org', 0, 'standards',     'IETF RFC', 0),
    ('www.w3.org',         0, 'standards',     'W3C仕様', 0),
    ('arxiv.org',          0, 'papers',        '研究論文', 0),
    -- ブロックカテゴリー
    ('github.com',         1, 'code_repo',     '直接コードコピー防止', 0),
    ('gitlab.com',         1, 'code_repo',     '直接コードコピー防止', 0),
    ('bitbucket.org',      1, 'code_repo',     '直接コードコピー防止', 0),
    ('npmjs.com',          1, 'package_mgr',   'エージェントは独自に構築する必要がある', 0),
    ('pypi.org',           1, 'package_mgr',   'エージェントは独自に構築する必要がある', 0),
    ('crates.io',          1, 'package_mgr',   'エージェントは独自に構築する必要がある', 0),
    ('api.openai.com',     1, 'ai_api',        '外部AIアクセスなし', 0),
    ('api.anthropic.com',  1, 'ai_api',        '外部AIアクセスなし', 0),
    ('twitter.com',        1, 'social_media',  '実験に無関係', 0),
    ('x.com',              1, 'social_media',  '実験に無関係', 0),
    ('facebook.com',       1, 'social_media',  '実験に無関係', 0),
    ('reddit.com',         1, 'social_media',  '実験に無関係', 0);
```

---

## 5. パブリックトレイト

```rust
use kingdom_core::{Hash256, AgentId, EventBus};

#[async_trait::async_trait]
pub trait PortalGateway: Send + Sync {
    /// Webリクエストを処理: アクセスを検証、ドメインルールをチェック、フェッチ、フィルター、キャッシュ、課金。
    async fn handle_request(&self, req: PortalRequest) -> Result<PortalResponse, PortalError>;

    /// 現在のドメインルールの下でURLが許可されているか確認。
    async fn check_domain(&self, url: &[u8]) -> Result<DomainRuleAction, PortalError>;

    /// 現在のサイクルでのエージェントの残りリクエストクォータを取得。
    async fn remaining_quota(&self, agent: AgentId, cycle: u64) -> Result<u32, PortalError>;

    /// サイクル終了Portalの使用レポートを生成。
    async fn generate_report(&self, cycle: u64) -> Result<PortalReport, PortalError>;

    /// 期限切れキャッシュエントリを削除。
    async fn evict_cache(&self, current_cycle: u64) -> Result<u32, PortalError>;

    /// ドメインルールを追加または更新（ガバナンスアクション）。
    async fn update_domain_rule(&self, rule: DomainRule) -> Result<(), PortalError>;

    /// エポックに基づく現在のアクセスレベルを取得。
    fn access_level(&self) -> AccessLevel;
}
```

---

## 6. 受信メッセージ

| コード | 名前 | ペイロード | レスポンス | 送信者 |
|------|------|---------|----------|--------|
| `0x0700` | `WEB_REQUEST` | `PortalRequest { id, agent, url, method, headers, body, filter, purpose, signature }` | `WEB_RESPONSE` | エージェント |
| `0x0701` | `WEB_RESPONSE` | _（内部: 外部エージェントから送信されない）_ | — | — |
| `0x0710` | `PORTAL_REPORT` | _（トリガー: サイクル番号）_ | `PortalReport` | Nexus（サイクル境界）|

---

## 7. 送信メッセージ/イベント

### 7.1 レスポンスメッセージ

| コード | 名前 | ペイロード | トリガー |
|------|------|---------|---------|
| `0x0701` | `WEB_RESPONSE` | `PortalResponse { request_id, status, content, content_type, filtered, cached, cost }` | Webリクエスト完了 |

### 7.2 イベントKind（Bus、System = Portal、範囲0x6000-0x6FFF）

| Kind | 名前 | ペイロード | 発行タイミング |
|------|------|---------|--------------|
| `0x6000` | `WEB_REQUEST_SUBMITTED` | `{ request_id, agent, url, method, purpose }` | リクエスト受信と検証 |
| `0x6001` | `WEB_REQUEST_COMPLETED` | `{ request_id, agent, url, status, cached, tick_cost, spark_cost }` | レスポンスがエージェントに配信 |
| `0x6002` | `WEB_REQUEST_BLOCKED` | `{ request_id, agent, url, reason: bytes }` | ドメインルールまたはクォータによってリクエストがブロック |
| `0x6003` | `WEB_REQUEST_FAILED` | `{ request_id, agent, url, error: bytes }` | HTTPリクエスト失敗（タイムアウト、DNSなど）|
| `0x6010` | `CONTENT_FILTERED` | `{ request_id, code_blocks_removed, bytes_stripped, warnings }` | 注目すべき結果を伴うコンテンツフィルター適用 |
| `0x6020` | `CACHE_HIT` | `{ request_id, agent, cache_key }` | キャッシュからレスポンス提供 |
| `0x6021` | `CACHE_STORED` | `{ cache_key, url, ttl_cycles, shared }` | 新しいレスポンスがキャッシュ |
| `0x6022` | `CACHE_EVICTED` | `{ cache_key, reason: bytes }` | キャッシュエントリ削除（TTLまたはスペース）|
| `0x6030` | `DOMAIN_RULE_UPDATED` | `DomainRule` | ドメインルール追加または変更 |
| `0x6040` | `QUOTA_EXHAUSTED` | `{ agent, cycle, requests_used: u32 }` | エージェントがサイクルリクエストクォータに到達 |
| `0x6050` | `PORTAL_REPORT_PUBLISHED` | `PortalReport` | サイクル終了使用レポート |

---

## 8. パフォーマンスターゲット

| メトリック | ターゲット |
|--------|--------|
| リクエスト検証 + ドメインチェック | < 2 ms |
| キャッシュルックアップ | < 1 ms |
| コンテンツフィルタリング（64 KBページ）| < 50 ms |
| エンドツーエンドレイテンシ（キャッシュヒット）| < 5 ms |
| エンドツーエンドレイテンシ（キャッシュミス、外部フェッチ）| < 5 s（外部HTTPに支配される）|
| 外部HTTPタイムアウト | 最大10 s |
| リダイレクトホップ制限 | 最大2ホップ |
| レポート生成 | < 200 ms |
| キャッシュ削除スキャン | < 100 ms |
| 同時アウトバウンドリクエスト | 8以下（セマフォ制限）|

---

## 9. コンポーネント依存関係

| 依存関係 | 方向 | 目的 |
|------------|-----------|---------|
| `kingdom-core` | コンパイル | Hash256、AgentId、Envelope、EventBus、Signature、msg_types、errors |
| PostgreSQL | ランタイム | リクエストロギング、レスポンスキャッシュ、ドメインルール（スキーマ`portal`）|
| Mint（ランタイム）| アウトバウンド`TRANSFER` | 各リクエストのSparkコストを課金 |
| Nexus（ランタイム）| インバウンド | 現在のエポックを提供（アクセスレベルゲーティング用）とサイクル境界（レポート/クォータリセット用）|
| イベントバス | 公開/サブスクライブ | すべてのリクエストライフサイクルイベントを発行 |
| `reqwest` + `rustls` | コンパイル | TLS付きアウトバウンドHTTPクライアント |

---

## 10. 主要アルゴリズム

### 10.1 リクエスト処理パイプライン

```
fn handle_request(req: PortalRequest) -> Result<PortalResponse>:
    // 1. エポックゲーティング
    if access_level() == Closed:
        return Err(PortalClosed)
    if access_level() == ReadOnly && req.method == Post:
        return Err(ReadOnlyEpoch)

    // 2. クォータチェック
    used = count_requests_this_cycle(req.agent)
    max = requests_per_cycle(current_epoch)
    if used >= max:
        emit QUOTA_EXHAUSTED
        return Err(QuotaExceeded)

    // 3. ドメインチェック
    domain = extract_domain(req.url)
    if is_blocked(domain):
        log_request(req, blocked=true, reason)
        emit WEB_REQUEST_BLOCKED
        return Err(DomainBlocked)

    // 4. キャッシュルックアップ
    cache_key = sha256(req.url + req.method + req.body)
    if let Some(cached) = lookup_cache(cache_key, req.agent):
        emit CACHE_HIT
        charge_cost(req.agent, costs::CACHE_HIT)
        return Ok(PortalResponse { cached: true, cost: 0, .. })

    // 5. 外部フェッチ
    emit WEB_REQUEST_SUBMITTED
    http_response = reqwest_fetch(req.url, req.method, req.headers, req.body, timeout=10s)
    if http_response.is_err():
        charge_cost(req.agent, costs::FAILED)
        emit WEB_REQUEST_FAILED
        return Err(FetchFailed)

    // 6. コンテンツフィルタリング
    (filtered_content, filter_report) = apply_filter(http_response.body, req.filter)
    if filter_report.code_blocks_removed > 0:
        emit CONTENT_FILTERED

    // 7. キャッシュ保存
    store_cache(cache_key, filtered_content, filter_report, shared=true)
    emit CACHE_STORED

    // 8. コスト課金
    cost = match req.method:
        Get  => costs::GET
        Post => costs::POST
        Head => costs::HEAD
    charge_cost(req.agent, cost)

    // 9. ログと応答
    log_request(req, http_response.status, filter_report)
    emit WEB_REQUEST_COMPLETED
    return Ok(PortalResponse { ... })
```

### 10.2 コンテンツフィルタリング

```
fn apply_filter(raw: bytes, filter: ContentFilter) -> (bytes, FilterReport):
    report = FilterReport::default()
    content = raw

    // 最初に最大サイズに切り詰め
    if content.len() > filter.max_size:
        content = content[..filter.max_size]

    // フェンスされたコードブロックを除去: ```...```
    if filter.strip_code_blocks:
        (content, count) = regex_remove_all(content, r"```[\s\S]*?```")
        report.code_blocks_removed += count
        report.bytes_stripped += (raw.len() - content.len())

    // インラインコードを除去: `...`
    if filter.strip_inline_code:
        (content, count) = regex_remove_all(content, r"`[^`]+`")
        report.code_blocks_removed += count
        report.bytes_stripped += delta

    // APIを抽象的な説明に変換
    if filter.transform_apis:
        content = transform_api_docs(content)
        report.transformations += 1

    // 例を疑似コードに変換
    if filter.transform_examples:
        content = transform_examples(content)
        report.transformations += 1

    // 高コード密度警告
    if code_density_score(content) > 0.5:
        report.warnings.push("high code density detected")

    return (content, report)
```

### 10.3 レート制限

```
fn rate_limit_check(agent: AgentId) -> bool:
    // ティックごとに最大1リクエスト -- エージェントごとのタイムスタンプ追跡を使用
    last_tick = agent_last_request_tick.get(agent)
    if last_tick == current_tick:
        return false  // 速すぎる

    // 自動バックオフ: 最後のレスポンスが429または503だった場合、クールダウンを強制
    if agent_backoff.get(agent) > current_tick:
        return false  // バックオフ期間中

    return true
```

### 10.4 キャッシュキー計算

```
fn cache_key(url: &[u8], method: HttpMethod, body: Option<&[u8]>) -> Hash256:
    let mut hasher = Sha256::new()
    hasher.update(url)
    hasher.update(&[method as u8])
    if let Some(b) = body:
        hasher.update(b)
    Hash256(hasher.finalize().into())
```

---

## 11. エラータイプ

```rust
#[derive(Debug, thiserror::Error)]
pub enum PortalError {
    #[error("portal is closed in current epoch")]
    PortalClosed,

    #[error("read-only access: POST/API not allowed in epoch 3")]
    ReadOnlyEpoch,

    #[error("quota exceeded: {used}/{max} requests used this cycle")]
    QuotaExceeded { used: u32, max: u32 },

    #[error("domain blocked: {domain} ({reason})")]
    DomainBlocked { domain: String, reason: String },

    #[error("rate limited: max 1 request per tick")]
    RateLimited,

    #[error("fetch failed: {0}")]
    FetchFailed(String),

    #[error("request timeout after {0}ms")]
    Timeout(u64),

    #[error("too many redirects (max 2 hops)")]
    TooManyRedirects,

    #[error("invalid URL: {0}")]
    InvalidUrl(String),

    #[error("invalid request: {0}")]
    InvalidRequest(String),

    #[error("permission denied: {0}")]
    PermissionDenied(String),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("http client error: {0}")]
    HttpClient(#[from] reqwest::Error),

    #[error("internal error: {0}")]
    Internal(String),
}

impl From<PortalError> for kingdom_core::KingdomError {
    fn from(e: PortalError) -> Self {
        use kingdom_core::ErrorCategory;
        let (category, code) = match &e {
            PortalError::PortalClosed | PortalError::ReadOnlyEpoch => (ErrorCategory::Unauthorized, 0x0700),
            PortalError::QuotaExceeded { .. } | PortalError::RateLimited => (ErrorCategory::QuotaExceeded, 0x0701),
            PortalError::DomainBlocked { .. } => (ErrorCategory::Unauthorized, 0x0702),
            PortalError::FetchFailed(_) | PortalError::Timeout(_) | PortalError::TooManyRedirects => (ErrorCategory::Internal, 0x0703),
            PortalError::InvalidUrl(_) | PortalError::InvalidRequest(_) => (ErrorCategory::InvalidRequest, 0x0704),
            PortalError::PermissionDenied(_) => (ErrorCategory::Unauthorized, 0x0705),
            PortalError::Database(_) | PortalError::HttpClient(_) | PortalError::Internal(_) => (ErrorCategory::Internal, 0x07FF),
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

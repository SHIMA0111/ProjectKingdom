# 技術要件: Keyward

> APIキーセキュリティシステム。プロセス隔離を備えた別バイナリ。
> クレート: `kingdom-keyward`
> バイナリ: `kingdom-keyward`
> デザイン参照: [14-KEYWARD.md](../../design/14-KEYWARD.md)

---

## 1. 目的

KeywardはKingdomの外壁です: APIキー、スポンサーアイデンティティ、LLMプロキシング、コスト追跡を管理するプロセス隔離され、メモリロックされたセキュリティコンポーネント。これはKingdomとLLMプロバイダー間の唯一のパスです。APIキーの平文はKeywardのプロセス境界の外に決して存在しません。Keywardは、暗号化されたUnixドメインソケット経由でKingdomと通信する別バイナリ（`kingdom-keyward`）です。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-keyward"

[dependencies]
# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# 暗号化
aes-gcm = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }
zeroize = { workspace = true }

# HTTPクライアント（LLMプロバイダー呼び出し用）
reqwest = { workspace = true }

# ロギング（フィルタリング済み — println!/eprintln!禁止）
tracing = { workspace = true }
tracing-subscriber = { workspace = true }

# ユーティリティ
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
regex = { workspace = true }
bytes = { workspace = true }

# Unix
libc = "0.2"
```

### 2.1 実装制約

| 制約 | 根拠 |
|-----------|-----------|
| Rustのみ、GCなし | 直接的なメモリ安全性制御; GCがキー平文を保持する可能性 |
| 最小限の外部依存関係 | 攻撃面を削減 |
| `rustls`経由のTLS | OpenSSLバージョン管理リスクを回避 |
| `println!`/`eprintln!`禁止 | マクロ経由のコンパイル時lint; すべての出力は監査ログを通過 |
| `Debug`/`Display`トレイトはキー値を含んではならない | `derive(Debug)`の事故を防止; カスタム実装が必要 |
| `kingdom-core`依存関係なし | プロセス隔離; Keywardは完全にスタンドアロン |

---

## 3. データモデル

### 3.1 スポンサー

```rust
use uuid::Uuid;
use chrono::{DateTime, Utc};

/// APIキーと予算を提供する人間のスポンサー。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Sponsor {
    /// 時間ソート可能なUUID v7。認証 = このUUIDを知っている。
    pub id: Uuid,
    /// このスポンサーが作成された時刻。
    pub created_at: DateTime<Utc>,

    // --- 予算 ---
    /// 入金された総予算（USD）。
    pub budget_total_usd: f64,
    /// 使用された総予算（USD）。
    pub budget_spent_usd: f64,
    /// 残り予算（USD）。
    pub budget_remaining_usd: f64,

    // --- 統計（キー値を含まない）---
    /// プロバイダー参照（フィンガープリントのみ、キーマテリアルなし）。
    pub providers: Vec<ProviderRef>,
    /// このスポンサーのキーで動作しているエージェントID。
    pub agents_powered: Vec<[u8; 32]>,

    /// ライフサイクルステータス。
    pub status: SponsorStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum SponsorStatus {
    Active,
    Paused,
    Exhausted,
}
```

### 3.2 ProviderRef

```rust
/// 登録されたAPIキープロバイダーへの参照（キーマテリアルなし）。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProviderRef {
    /// プロバイダータイプ。
    pub provider_type: ProviderType,
    /// sha256(plaintext_key)の最初の16文字の16進数。識別のみ。
    pub key_fingerprint: String,
    /// キーが登録された時刻。
    pub registered_at: DateTime<Utc>,
    /// キーがAPI呼び出しで最後に使用された時刻。
    pub last_used: Option<DateTime<Utc>>,
    /// 現在のキーステータス。
    pub status: KeyStatus,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum ProviderType {
    Anthropic,
    OpenAi,
    Google,
    OpenAiCompatible { base_url: String },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum KeyStatus {
    Registered,
    Probing,
    Valid,
    Invalid,
    RateLimited,
    Revoked,
}
```

### 3.3 SecureKeyVault

```rust
use aes_gcm::{Aes256Gcm, Key, Nonce};
use zeroize::Zeroize;

/// インメモリ暗号化キーストア。マスターキーはディスクに書き込まれない。
pub struct SecureKeyVault {
    /// AES-256-GCMマスターキー。起動時にOS CSPRNGから生成。
    /// シャットダウン時にゼロ化。決して永続化されない。
    master_key: Key<Aes256Gcm>,  // [u8; 32]、Zeroizeを実装

    /// key_id -> EncryptedKey
    keys: std::collections::HashMap<Uuid, EncryptedKey>,
}

/// マスターキーで静止時に暗号化されたAPIキー。
/// Debug実装は暗号文、nonce、tagを出力してはならない。
pub struct EncryptedKey {
    /// このキーの一意識別子。
    pub key_id: Uuid,
    /// 所有するスポンサー。
    pub sponsor_id: Uuid,
    /// プロバイダータイプ。
    pub provider: ProviderType,
    /// AES-256-GCM暗号化されたAPIキー。
    pub ciphertext: Vec<u8>,
    /// GCM nonce（12バイト）。
    pub nonce: [u8; 12],
    /// GCM認証タグ（16バイト）。
    pub tag: [u8; 16],
    /// 表示/識別のためのsha256(plaintext_key)[0..16] hex。
    pub fingerprint: String,
    /// キーが登録された時刻。
    pub created_at: DateTime<Utc>,
}

// キーマテリアルを決して出力しないカスタムDebug。
impl std::fmt::Debug for EncryptedKey {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("EncryptedKey")
            .field("key_id", &self.key_id)
            .field("sponsor_id", &self.sponsor_id)
            .field("provider", &self.provider)
            .field("fingerprint", &self.fingerprint)
            .field("created_at", &self.created_at)
            .finish_non_exhaustive()
    }
}

// キーマテリアルを決して出力しないカスタムDisplay。
impl std::fmt::Display for EncryptedKey {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Key({}, fingerprint={})", self.key_id, self.fingerprint)
    }
}
```

### 3.4 LLMリクエスト/レスポンス

```rust
/// Kingdomからの（Unixソケット経由）LLM呼び出しリクエスト。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMRequest {
    /// トレース用の一意リクエスト識別子。
    pub request_id: Uuid,
    /// コスト帰属ターゲット。
    pub sponsor_id: Uuid,
    /// 使用するキー（キー値ではない）。
    pub key_id: Uuid,
    /// モデル識別子（例: "claude-sonnet-4-20250514"）。
    pub model: String,
    /// 会話メッセージ（system/user/assistant）。
    pub messages: Vec<LLMMessage>,
    /// レスポンスの最大トークン数。
    pub max_tokens: u32,
    /// サンプリング温度。
    pub temperature: f32,
    /// 構造化（JSON）出力をリクエストするかどうか。
    pub structured: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMMessage {
    pub role: LLMRole,
    pub content: String,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum LLMRole {
    System,
    User,
    Assistant,
}

/// Kingdomに返されるLLM呼び出しレスポンス。
/// APIキーは決して含まれない。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LLMResponse {
    /// 一致するリクエスト識別子。
    pub request_id: Uuid,
    /// モデルレスポンスコンテンツ。
    pub content: String,
    /// トークン使用量。
    pub usage: TokenUsage,
    /// USD計算コスト。
    pub cost_usd: f64,
    /// ミリ秒単位のラウンドトリップレイテンシ。
    pub latency_ms: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
}
```

### 3.5 監査ログ

```rust
/// 単一監査ログエントリ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuditEntry {
    pub timestamp: DateTime<Utc>,
    pub event: AuditEvent,
    pub sponsor_id: Option<Uuid>,
    /// 最初の16文字の16進数のみ。キー値は決して記録されない。
    pub key_fingerprint: Option<String>,
    /// 追加コンテキスト。決して含まれない: APIキー平文/暗号文、
    /// マスターキー、プロンプトコンテンツ。
    pub details: std::collections::HashMap<String, String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AuditEvent {
    KeyRegistered,
    KeyUsed,
    KeyRevoked,
    KeyProbe,
    KeyDecryption,
    SponsorCreated,
    SponsorFunded,
    BudgetWarning,
    BudgetExhausted,
    AuthFailure,
    RateLimitHit,
    AnomalyDetected,
}
```

### 3.6 異常検出

```rust
/// 異常検出ルール。
#[derive(Debug, Clone)]
pub struct AnomalyRule {
    pub condition: AnomalyCondition,
    pub action: AnomalyAction,
}

#[derive(Debug, Clone)]
pub enum AnomalyCondition {
    /// 単一キーに対して毎分N以上のリクエスト。
    HighVolumeRequests { threshold_per_minute: u32 },
    /// 現在の分のコストが平均のN倍を超える。
    CostSpike { multiplier: f64 },
    /// 存在しないkey_idへの参照。
    InvalidKeyReference,
    /// 予算が使い果たされたスポンサーからのリクエスト。
    PostExhaustionRequest,
}

#[derive(Debug, Clone, Copy)]
pub enum AnomalyAction {
    Alert,
    Throttle,
    PauseKey,
    Reject,
    Log,
}
```

### 3.7 モデルとレート制限情報（Kingdomに公開）

```rust
/// プロービング中に発見されたモデル情報。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ModelInfo {
    pub model_id: String,
    pub max_context: u32,
    pub input_cost_per_million: f64,
    pub output_cost_per_million: f64,
    pub capability_tier: CapabilityTier,
    pub supports_structured: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum CapabilityTier {
    Tier1,
    Tier2,
    Tier3,
}

/// プロバイダーのレート制限情報。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimits {
    pub requests_per_minute: u32,
    pub tokens_per_minute: u32,
    pub tokens_per_day: u32,
}

/// プロバイダーキーのプローブ結果。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProbeResult {
    pub provider: ProviderType,
    pub available_models: Vec<ModelInfo>,
    pub rate_limits: RateLimits,
    pub status: ProbeStatus,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ProbeStatus {
    Ok,
    RateLimited,
    InvalidKey,
    Error,
}
```

---

## 4. パブリックトレイト

```rust
/// Keyward固有のエラー。
#[derive(Debug, thiserror::Error)]
pub enum KeywardError {
    #[error("sponsor not found: {0}")]
    SponsorNotFound(Uuid),

    #[error("key not found: {0}")]
    KeyNotFound(Uuid),

    #[error("key does not belong to sponsor")]
    KeySponsorMismatch,

    #[error("budget exhausted for sponsor: {0}")]
    BudgetExhausted(Uuid),

    #[error("rate limit exceeded for key: {0}")]
    RateLimited(String),

    #[error("key is invalid or revoked: {0}")]
    KeyInvalid(String),

    #[error("LLM provider error: {0}")]
    ProviderError(String),

    #[error("anomaly detected: {0}")]
    AnomalyDetected(String),

    #[error("socket communication error: {0}")]
    SocketError(String),

    #[error("memory protection setup failed: {0}")]
    MemoryProtectionFailed(String),
}

/// Keywardサービスインターフェイス（別keywardプロセスで実行）。
#[async_trait::async_trait]
pub trait KeywardService: Send + Sync {
    // --- スポンサー管理 ---

    /// 新しいスポンサーを作成。スポンサーUUIDを返す。
    async fn create_sponsor(&self) -> Result<Uuid, KeywardError>;

    /// スポンサーのAPIキーを登録。キーは読み取られ、暗号化され、
    /// 平文は即座にゼロ化される。key_idとフィンガープリントを返す。
    async fn register_key(
        &self,
        sponsor_id: Uuid,
        provider: ProviderType,
        plaintext_key: zeroize::Zeroizing<String>,
    ) -> Result<(Uuid, String), KeywardError>;

    /// スポンサーに予算を追加。
    async fn fund_sponsor(
        &self,
        sponsor_id: Uuid,
        amount_usd: f64,
    ) -> Result<(), KeywardError>;

    /// スポンサー情報を取得（キーマテリアルなし）。
    async fn get_sponsor(&self, sponsor_id: Uuid) -> Result<Sponsor, KeywardError>;

    /// すべてのスポンサーをリスト（キーマテリアルなし）。
    async fn list_sponsors(&self) -> Result<Vec<Sponsor>, KeywardError>;

    // --- LLMプロキシ ---

    /// LLMリクエストをプロキシ。キーはこの関数スコープ内で復号化され、
    /// HTTP呼び出しに使用され、即座にゼロ化される。
    async fn call_llm(&self, request: LLMRequest) -> Result<LLMResponse, KeywardError>;

    // --- キー管理 ---

    /// キーの有効性をプローブ（軽量API呼び出し、例: models.list）。
    async fn probe_key(&self, key_id: Uuid) -> Result<ProbeResult, KeywardError>;

    /// キーを取り消す。
    async fn revoke_key(
        &self,
        sponsor_id: Uuid,
        key_id: Uuid,
    ) -> Result<(), KeywardError>;

    /// すべての有効なキー間で利用可能なモデルを取得。
    async fn available_models(&self) -> Result<Vec<ModelInfo>, KeywardError>;

    /// 特定のキーのレート制限を取得。
    async fn rate_limits(&self, key_id: Uuid) -> Result<RateLimits, KeywardError>;

    // --- ヘルス ---

    /// すべてのキーのヘルスチェックを実行（Kingdomによって10サイクルごとに呼び出される）。
    async fn health_check(&self) -> Result<Vec<(Uuid, KeyStatus)>, KeywardError>;

    /// 監査ログエントリを取得（フィルタリング済み）。
    async fn audit_log(
        &self,
        since: Option<DateTime<Utc>>,
        event_filter: Option<Vec<AuditEvent>>,
        limit: u32,
    ) -> Result<Vec<AuditEntry>, KeywardError>;
}
```

---

## 5. 受信メッセージ

Keywardは`/tmp/kingdom-keyward.sock`のUnixドメインソケット経由で暗号化されたMessagePackフレームを使用して通信します。

| ソース | メッセージ | ペイロード | 説明 |
|--------|---------|---------|-------------|
| Summoner CLI | `SponsorCreate` | `{}` | 新しいスポンサーを作成。|
| Summoner CLI | `KeyRegister` | `{ sponsor_id, provider, key }` | APIキーを登録（キーはstdinから読み取り）。|
| Summoner CLI | `SponsorFund` | `{ sponsor_id, amount_usd }` | スポンサーに予算を追加。|
| Nexus | `LLMRequest` | `LLMRequest`（3.4参照）| LLM呼び出しをプロキシ。|
| Nexus | `ProbeKey` | `{ key_id }` | キー有効性をプローブ。|
| Nexus | `HealthCheck` | `{}` | すべてのキーのヘルスチェックをトリガー。|
| Nexus | `ListModels` | `{}` | 利用可能なモデルをリスト。|
| Nexus | `GetRateLimits` | `{ key_id }` | キーのレート制限を取得。|

### 5.1 ソケットフレームフォーマット

```
[u32長さ（ビッグエンディアン）][MessagePackペイロード][u32 CRC32]
```

---

## 6. 送信メッセージ/イベント

| ターゲット | メッセージ | ペイロード | 説明 |
|--------|---------|---------|-------------|
| Summoner CLI | `SponsorCreated` | `{ sponsor_id }` | スポンサーUUID返却。|
| Summoner CLI | `KeyRegistered` | `{ key_id, fingerprint }` | キー登録確認。|
| Summoner CLI | `SponsorFunded` | `{ sponsor_id, budget_remaining }` | 予算更新。|
| Nexus | `LLMResponse` | `LLMResponse`（3.4参照）| LLM呼び出し結果。|
| Nexus | `ProbeResult` | `ProbeResult`（3.7参照）| キープローブ結果。|
| Nexus | `HealthResult` | `Vec<(Uuid, KeyStatus)>` | ヘルスチェック結果。|
| Nexus | `ModelList` | `Vec<ModelInfo>` | 利用可能なモデル。|
| Nexus | `RateLimitInfo` | `RateLimits` | キーのレート制限。|

### 6.1 KeywardがKingdomに公開するもの

| 情報 | フォーマット | 目的 |
|-------------|--------|---------|
| 利用可能なモデルリスト | `Vec<ModelInfo>` | Nexusリソース計画 |
| キーID | `Vec<Uuid>` | NexusがLLMリクエストで指定 |
| レート制限情報 | `RateLimits` | Nexusスケジューリング |
| スポンサーUUID | `Uuid` | ObserverのSponsor View |
| キーフィンガープリント | `String`（16進数16文字）| Observer表示 |
| コスト情報 | `CostTracker` | Nexus予算管理 |
| LLMレスポンス | `LLMResponse` | エージェント思考結果 |

**決して公開されない**: APIキー平文、APIキー暗号文、マスターキー、プロバイダー認証ヘッダー。

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| キー暗号化（登録）| < 1 ms | AES-256-GCM暗号化 |
| キー復号化（呼び出しごと）| < 0.5 ms | AES-256-GCM復号化 |
| LLMプロキシオーバーヘッド（プロバイダーレイテンシ除く）| < 5 ms | 復号化 + HTTPビルド + サニタイズ |
| レスポンスサニタイゼーション | < 2 ms | 正規表現パターンマッチング |
| ヘルスチェック（キーごと）| < 5 s | 軽量APIプローブ |
| 異常検出評価 | < 0.1 ms | インメモリルールチェック |
| ソケットメッセージラウンドトリップ（プロバイダー除く）| < 10 ms | エンコード + 送信 + デコード |
| シャットダウン時のメモリゼロ化 | < 1 ms | すべてのキー領域をexplicit_bzero |
| 監査ログ追加 | < 0.1 ms | インメモリ追加 |
| 起動（メモリ保護セットアップ）| < 100 ms | mlock、mlockall、RLIMIT_CORE |

---

## 8. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| なし（スタンドアロン）| - | KeywardはKingdomクレート依存関係を持たない |
| LLMプロバイダー | 外部 | reqwest + rustls経由のHTTP(S) API呼び出し |
| Unixドメインソケット | IPC | Kingdomプロセスとの通信 |
| OSカーネル | システム | mlock、mlockall、RLIMIT_CORE、getrandom |

---

## 9. 主要アルゴリズム

### 9.1 メモリ保護初期化

```
fn init_key_vault():
    1. setrlimit(RLIMIT_CORE, 0)           // コアダンプを無効化
    2. prctl(PR_SET_DUMPABLE, 0)            // ptraceアタッチを防止
    3. mlockall(MCL_CURRENT | MCL_FUTURE)   // すべてのメモリページをロック
    4. master_key = getrandom(32)           // OS CSPRNG
    5. すべての保護がアクティブであることを検証
```

### 9.2 キー登録

```
fn register_key(sponsor_id, provider, plaintext_key):
    1. sponsor_idが存在しACTIVEであることを検証。
    2. fingerprint = hex(sha256(plaintext_key))[0..16]を計算。
    3. key_id = uuid_v7()を生成。
    4. nonce = getrandom(12)を生成。
    5. 暗号化: (ciphertext, tag) = aes_256_gcm_encrypt(master_key, nonce, plaintext_key)。
    6. explicit_bzero(plaintext_key)。
    7. EncryptedKey { key_id, sponsor_id, provider, ciphertext, nonce, tag, fingerprint }を保存。
    8. 非同期プローブを開始（PROBING -> VALIDまたはINVALIDに遷移）。
    9. 監査ログ: KEY_REGISTERED。
   10. (key_id, fingerprint)を返す。
```

### 9.3 LLMプロキシ呼び出し

```
fn call_llm(request: LLMRequest) -> LLMResponse:
    1. 検証: key_idがrequest.sponsor_idに属する。
    2. 検証: スポンサーbudget_remaining > 0。
    3. 検証: キーステータスがVALID。
    4. 異常ルールをチェック（ボリューム、コストスパイク）。

    5. let plaintext_key = decrypt(vault.get(request.key_id))  // ここでのみ
    6. let start = Instant::now()
    7. let provider_response = http_post(
           provider_url(request.model),
           headers: { "Authorization": format!("Bearer {}", plaintext_key) },
           body: build_provider_body(request)
       )
    8. explicit_bzero(&plaintext_key)                           // 即座にゼロ化

    9. let sanitized = sanitize(provider_response)
   10. let cost = compute_cost(request.model, usage)
   11. スポンサー予算からコストを控除。
   12. 監査ログ: KEY_USED。
   13. LLMResponse { request_id, content, usage, cost_usd, latency_ms }を返す
```

### 9.4 レスポンスサニタイゼーション

```
fn sanitize(response_body: String, known_keys: &[EncryptedKey]) -> String:
    1. 各既知のキーについて:
       a. 平文に復号化。
       b. response_body.contains(plaintext)の場合:
          - 監査ログ: ANOMALY_DETECTED("key_in_response")。
          - 平文を"[REDACTED]"で置き換え。
       c. explicit_bzero(plaintext)。

    2. 既知のAPIキーフォーマットのパターンマッチ:
       - r"sk-ant-api\d{2}-[A-Za-z0-9_-]{80,}"    (Anthropic)
       - r"sk-[A-Za-z0-9]{48,}"                     (OpenAI)
       - r"AIzaSy[A-Za-z0-9_-]{33}"                 (Google)
       マッチを"[REDACTED:API_KEY_PATTERN]"で置き換え。

    3. サニタイズされたボディを返す。
```

### 9.5 キーライフサイクルステートマシン

```
REGISTERED -> probe() -> PROBING
    PROBING -> success  -> VALID
    PROBING -> fail     -> INVALID

VALID -> rate_limit hit -> RATE_LIMITED
VALID -> revoke/expire  -> REVOKED

RATE_LIMITED -> cooldown -> VALID

INVALID: ターミナル（スポンサーに通知、キー使用不可）
REVOKED: ターミナル（キーはvaultから破棄）
```

### 9.6 ヘルスチェック（10サイクルごと）

```
fn health_check(key: &EncryptedKey) -> KeyStatus:
    1. キーを復号化。
    2. 軽量プローブ呼び出しを行う（例: GET /v1/models）。
    3. explicit_bzero(plaintext)。
    4. 結果をマッチ:
       OK           -> update_status(VALID)
       UNAUTHORIZED -> update_status(INVALID) + notify_sponsor()
       RATE_LIMITED -> update_status(RATE_LIMITED) + backoff()
       TIMEOUT      -> retry(3x) + if_still_fail(ALERT)
```

### 9.7 異常検出ルール

```
デフォルトルール:
    1. キーごとに毎分100以上のリクエスト  -> ALERT + THROTTLE
    2. cost_this_minute > 平均の10倍       -> ALERT + PAUSE_KEY
    3. 無効なkey_id参照                    -> ALERT + LOG
    4. 予算使い果たし後のリクエスト        -> REJECT + LOG
```

### 9.8 シャットダウン手順

```
fn destroy_key_vault():
    1. explicit_bzero(すべてのEncryptedKey暗号文領域)
    2. explicit_bzero(master_key)
    3. munlockall()
    4. 監査ログ: 最終エントリ（シャットダウン）
```

---

## 10. カオステストカテゴリー

| カテゴリー | ID範囲 | フォーカス |
|----------|----------|-------|
| A: プロセス障害 | A1-A5 | SIGKILL、SIGSEGV、OOM、電源喪失、重複起動 |
| B: メモリ攻撃 | B1-B4 | /proc/mem読み取り、メモリスキャン、スワップスキャン、GDBアタッチ |
| C: Kingdom境界 | C1-C5 | プロンプトインジェクション、Forgeエスケープ、イベントバスリーク、Observerリーク、偽key_id |
| D: 通信パス | D1-D4 | ソケット傍受、TLSダウングレード、レスポンス内のキー、DNSリバインディング |
| E: ログ/出力 | E1-E5 | すべてのログレベル、パニックトレース、HTTPエラー、監査エントリ、Observer API |
| F: 運用 | F1-F5 | 無効なキー、クロススポンサー、ゼロ予算、取り消し後、同時スポンサー |

テスト実行ポリシー: 各テスト後、メモリ、ディスク、ログ、ネットワークキャプチャの完全スキャンでテストキーパターンをチェック。

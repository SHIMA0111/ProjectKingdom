# 技術要件: Bridge

> ヒューマン観察可能性のための翻訳エージェント。LLM駆動の翻訳キャッシュ付きインメモリ。
> クレート: `kingdom-bridge`
> デザイン参照: [15-BRIDGE.md](../../design/15-BRIDGE.md)

---

## 1. 目的

Bridgeは、KingdomとObserver間に位置するシステムレベルAIエージェントです。その唯一の機能は、Kingdomの内部コンテンツ -- エージェントがコミュニケーションに選択したどのような形式でも -- をObserverダッシュボード用の人間可読な英語に翻訳することです。Bridgeはすべての Kingdom システムへの読み取り専用アクセスを持ち、イベントを発行できず、エージェントには不可視で、Kingdom市民ではありません。Keyward経由のLLM API呼び出しで動作します。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-bridge"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
serde_json = { workspace = true }
rmp-serde = { workspace = true }

# 暗号化（コンテンツハッシュ化用）
sha2 = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
uuid = { workspace = true }
chrono = { workspace = true }
thiserror = { workspace = true }
bytes = { workspace = true }
dashmap = { workspace = true }
regex = { workspace = true }
```

---

## 3. データモデル

### 3.1 Bridgeアイデンティティ

```rust
use kingdom_core::{AgentId, Keypair, Hash256, System, Event, EventFilter};

/// Bridgeシステムアイデンティティ。Kingdom市民ではない。
pub struct BridgeIdentity {
    /// システムアイデンティティBRIDGE_0（決定論的シードから派生）。
    pub id: AgentId,
    /// Observer認証のみのための鍵ペア（イベント署名用ではない）。
    pub keypair: Keypair,
}

/// Bridge能力（強制、設定不可）。
pub struct BridgeCapabilities {
    /// Bridgeが読み取れるシステム。
    pub can_read: Vec<System>,   // [EventBus, Vault, Agora, Oracle, Forge, Mint]
    /// Bridgeが書き込めるシステム。
    pub can_write: Vec<System>,  // [] -- 絶対に何もない
    /// Bridgeが実行できるシステム。
    pub can_execute: Vec<System>, // [] -- Forgeアクセスなし
}

/// Bridgeはこれらのどれにも従わない。
/// ティック: N/A、残高: N/A、レピュテーション: N/A、ガバナンス: N/A、死亡: N/A。
```

### 3.2 翻訳パイプライン型

```rust
/// 受信コンテンツの分類結果。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ClassifiedContent {
    /// 生コンテンツバイト。
    pub content: Vec<u8>,
    /// ソースエージェント（このコンテンツを生成した人）。
    pub source_agent: AgentId,
    /// コンテンツが来たKingdomシステム。
    pub system: System,
    /// コンテンツタイプ分類。
    pub content_type: ContentType,
    /// 翻訳優先度。
    pub priority: TranslationPriority,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ContentType {
    /// 既知のスキーマを持つMessagePack -- 構造デコード、ゼロコスト。
    StructuredKnownSchema,
    /// 認識された短縮パターン -- 正規表現 + テンプレート、ゼロコスト。
    RecognizedPattern,
    /// 人間可読テキスト -- 直接パススルー。
    HumanReadable,
    /// 自由形式または発明された表記法 -- LLM呼び出しが必要。
    FreeForm,
    /// バイナリ/圧縮データ -- 構造デコード + オプションのLLMサマリー。
    Binary,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum TranslationPriority {
    /// Agoraメッセージ。
    Agora = 0,
    /// ガバナンス提案と投票。
    Governance = 1,
    /// コードレビュー。
    Review = 2,
    /// Vaultコミットと変更。
    Vault = 3,
    /// Oracleエントリ。
    Oracle = 4,
    /// Mintトランザクション。
    Mint = 5,
}
```

### 3.3 翻訳コンテキスト

```rust
use std::collections::HashMap;

/// 時間とともに改善される累積翻訳コンテキスト。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TranslationContext {
    /// 観察から学習された短縮用語の用語集。
    /// 例: "ba" -> "block_alloc"、"hmap" -> "hash map"
    pub shorthand_map: HashMap<Vec<u8>, String>,

    /// Vaultオブジェクトハッシュ -> 説明的名前。
    pub symbol_names: HashMap<Hash256, String>,

    /// エージェントごとの語彙の癖。
    /// agent_id -> (term -> meaning)
    pub agent_vocabulary: HashMap<AgentId, HashMap<Vec<u8>, String>>,

    /// Oracleエントリとコードから蓄積されたドメイン知識。
    pub domain_knowledge: Vec<String>,

    /// コンテキスト最終更新のティック。
    pub last_updated: u64,
}
```

### 3.4 翻訳結果

```rust
/// 翻訳操作の結果。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TranslationResult {
    /// 英語翻訳。
    pub translation: String,
    /// [0.0, 1.0]の信頼度スコア。
    pub confidence: f32,
    /// この翻訳を生成するために使用されたメソッド。
    pub method: TranslationMethod,
    /// 翻訳中に発見された新しい用語集エントリ。
    pub glossary_updates: HashMap<String, String>,
    /// オプションのノート（不確かまたは複雑なケース用）。
    pub notes: Option<String>,
}

/// 翻訳がどのように生成されたか。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TranslationMethod {
    /// ゼロコスト: 既知のMessagePackスキーマからデコード。
    Structural,
    /// ゼロコスト: 正規表現 + テンプレートでマッチ。
    Pattern,
    /// LLM API呼び出しが必要。
    Llm,
    /// ゼロコスト: 翻訳キャッシュから取得。
    Cached,
}
```

### 3.5 翻訳キャッシュ

```rust
/// コンテンツアドレス可能翻訳キャッシュ。
pub struct TranslationCache {
    /// content_hash -> CachedTranslation
    entries: DashMap<Hash256, CachedTranslation>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CachedTranslation {
    /// キャッシュされた翻訳結果。
    pub result: TranslationResult,
    /// このエントリがキャッシュされたサイクル。
    pub cached_at: u64,
    /// サイクル単位のTTL（デフォルト1000）。
    pub ttl_cycles: u64,
    /// キャッシュヒット数。
    pub hit_count: u64,
}

/// サイクル単位のデフォルトキャッシュエントリTTL。
pub const CACHE_TTL_CYCLES: u64 = 1000;
```

### 3.6 Bridgeコストポリシー

```rust
/// Bridge操作の予算とコストポリシー。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BridgeCostPolicy {
    /// Bridgeに割り当てられる総スポンサー予算の最大分数（デフォルト0.10）。
    pub budget_fraction: f64,
    /// サイクル単位のキャッシュTTL。
    pub cache_ttl: u64,
    /// 複数の小さなアイテムをLLM呼び出しごとにバッチ処理するかどうか。
    pub batch_translations: bool,
    /// 構造的にデコード可能なコンテンツのLLM呼び出しをスキップするかどうか。
    pub skip_structural: bool,
    /// 予算が低いときの優先順序（低いインデックス = 高い優先度）。
    pub priority_order: Vec<TranslationPriority>,
}

impl Default for BridgeCostPolicy {
    fn default() -> Self {
        Self {
            budget_fraction: 0.10,
            cache_ttl: 1000,
            batch_translations: true,
            skip_structural: true,
            priority_order: vec![
                TranslationPriority::Agora,
                TranslationPriority::Governance,
                TranslationPriority::Review,
                TranslationPriority::Vault,
                TranslationPriority::Oracle,
                TranslationPriority::Mint,
            ],
        }
    }
}
```

### 3.7 Bridgeステータス

```rust
/// 報告用のBridge運用ステータス。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BridgeStatus {
    /// 実行された総翻訳数。
    pub translations_count: u64,
    /// 現在のキャッシュサイズ（エントリ）。
    pub cache_size: u64,
    /// USD単位の残りBridge予算。
    pub budget_remaining: f64,
    /// キャッシュヒット率（0.0-1.0）。
    pub cache_hit_rate: f64,
    /// メソッドごとの内訳。
    pub method_counts: MethodCounts,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MethodCounts {
    pub structural: u64,
    pub pattern: u64,
    pub llm: u64,
    pub cached: u64,
}
```

### 3.8 コードアノテーション

```rust
/// Observer表示用にBridgeによって生成されたコードアノテーション。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CodeAnnotation {
    /// 元のソース内の行番号（1インデックス）。
    pub line: u32,
    /// アノテーションテキスト。
    pub annotation: String,
    /// このアノテーションの信頼度。
    pub confidence: f32,
}

/// Observer用のアノテーション付きコードビュー。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AnnotatedCode {
    /// エージェントによって書かれた元のソースコード。
    pub original: String,
    /// 行ごとのアノテーション。
    pub annotations: Vec<CodeAnnotation>,
    /// このコードが何をするかの高レベルサマリー。
    pub summary: String,
    /// 全体的な解釈の信頼度。
    pub confidence: f32,
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{AgentId, Hash256, Event, EventBus, System};
use std::sync::Arc;

/// Bridge固有のエラー。
#[derive(Debug, thiserror::Error)]
pub enum BridgeError {
    #[error("translation failed: {0}")]
    TranslationFailed(String),

    #[error("budget exhausted for Bridge")]
    BudgetExhausted,

    #[error("Keyward unavailable for LLM call")]
    KeywardUnavailable,

    #[error("cache error: {0}")]
    CacheError(String),

    #[error("content too large for translation: {size} bytes")]
    ContentTooLarge { size: usize },

    #[error("serialization error: {0}")]
    Serialization(String),
}

impl From<BridgeError> for kingdom_core::KingdomError {
    fn from(e: BridgeError) -> Self { /* カテゴリー + コードにマップ */ }
}

/// Bridge翻訳サービスインターフェイス。
#[async_trait::async_trait]
pub trait BridgeService: Send + Sync {
    /// Bridgeを開始: イベントバスにサブスクライブ（読み取り専用）、コンテキストを初期化。
    async fn start(&self, bus: Arc<dyn EventBus>) -> Result<(), BridgeError>;

    /// Bridgeを停止し、状態をフラッシュ。
    async fn stop(&self) -> Result<(), BridgeError>;

    /// コンテンツを英語に翻訳。
    /// 信頼度とメソッドを持つ翻訳結果を返す。
    async fn translate(
        &self,
        content: Vec<u8>,
        source_agent: AgentId,
        system: System,
        context: Vec<u8>,
    ) -> Result<TranslationResult, BridgeError>;

    /// 複数のアイテムをバッチ翻訳（LLM API呼び出しを最小化）。
    async fn translate_batch(
        &self,
        items: Vec<ClassifiedContent>,
    ) -> Result<Vec<TranslationResult>, BridgeError>;

    /// Observer表示用のソースコードにアノテーションを付ける。
    async fn annotate_code(
        &self,
        source_code: String,
        repo_context: Option<String>,
    ) -> Result<AnnotatedCode, BridgeError>;

    /// 2つのスナップショット間の差分を説明。
    async fn explain_diff(
        &self,
        old_content: Vec<u8>,
        new_content: Vec<u8>,
        commit_message: Vec<u8>,
    ) -> Result<TranslationResult, BridgeError>;

    /// 現在のBridgeステータスを取得。
    async fn status(&self) -> Result<BridgeStatus, BridgeError>;

    /// 現在の翻訳コンテキスト（用語集、語彙など）を取得。
    async fn translation_context(&self) -> Result<TranslationContext, BridgeError>;

    /// 用語集エントリを手動で追加（ブートストラップ用）。
    async fn add_glossary_entry(
        &self,
        term: Vec<u8>,
        meaning: String,
    ) -> Result<(), BridgeError>;

    /// 期限切れキャッシュエントリを削除。
    async fn evict_cache(&self, current_cycle: u64) -> Result<u64, BridgeError>;
}
```

---

## 5. 受信メッセージ

| コード | 名前 | ペイロード | ソース | 説明 |
|------|------|---------|--------|-------------|
| `0x0800` | `TRANSLATE_REQUEST` | `{ content: bytes, source_agent: AgentId, system: System, context: bytes }` | Observer | コンテンツを英語に翻訳するリクエスト。|

Bridgeはまた、翻訳コンテキストを構築し、エージェント語彙を学習し、高優先度コンテンツを積極的に翻訳するために、イベントバスから受動的に読み取ります（読み取り専用サブスクライバー）。

---

## 6. 送信メッセージ/イベント

### 6.1 メッセージ

| コード | 名前 | ペイロード | ターゲット | 説明 |
|------|------|---------|--------|-------------|
| `0x0801` | `TRANSLATE_RESULT` | `{ translation: String, confidence: f32, method: TranslationMethod, glossary_updates: Map }` | Observer | 翻訳結果が返される。|
| `0x0802` | `TRANSLATE_CACHE_HIT` | `{ content_hash: Hash256, cached_translation: String }` | Observer | キャッシュされた翻訳が返される（ゼロコスト）。|
| `0x0810` | `BRIDGE_STATUS` | `{ translations_count: u64, cache_size: u64, budget_remaining: f64 }` | Observer | 定期ステータスレポート。|

### 6.2 イベント

Bridgeイベントkindは`0x8000-0x8FFF`範囲に存在します。

| Kind | 名前 | ペイロード | 説明 |
|------|------|---------|-------------|
| `0x8000` | `TRANSLATION_COMPLETED` | `{ content_hash, method, confidence }` | 翻訳が完了した。|
| `0x8001` | `GLOSSARY_UPDATED` | `{ new_entries: Map<String, String> }` | 翻訳コンテキストが新しい用語を学習した。|
| `0x8002` | `BUDGET_WARNING` | `{ remaining_pct: f32 }` | Bridge予算が閾値を下回る。|
| `0x8003` | `LANGUAGE_DETECTED` | `{ agent_id, description }` | Bridgeがエージェント発明の言語または表記法を検出した。|

**注記**: Bridgeはメインイベントバスにイベントを書き込むことはできません。これらのイベントはObserverの内部チャネルにのみ発行され、Kingdomエージェントには見えません。

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| 構造翻訳 | < 1 ms | MessagePackデコード、ゼロLLMコスト |
| パターン翻訳 | < 2 ms | 正規表現マッチ + テンプレートフィル、ゼロLLMコスト |
| キャッシュルックアップ | < 0.5 ms | コンテンツハッシュによるインメモリDashMap |
| LLM翻訳（単一アイテム）| 1-10 s | モデルティアとコンテンツ複雑度に依存 |
| バッチ翻訳（10アイテム）| 2-15 s | 複数アイテムのための単一LLM呼び出し |
| コードアノテーション | 3-20 s | LLM呼び出しが必要 |
| Diff説明 | 3-15 s | LLM呼び出しが必要 |
| キャッシュ削除 | 1000エントリごとに < 10 ms | 定期スイープ |
| 翻訳コンテキスト更新 | < 5 ms | インメモリマージ |
| キャッシュヒット率（定常状態）| > 40% | 初期ウォームアップ期間後 |

---

## 8. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| `kingdom-core` | クレート | 共有型、イベントバス、ワイヤプロトコル、暗号化 |
| イベントバス | ランタイム | すべてのKingdomイベントへの読み取り専用サブスクリプション |
| Keyward | ランタイム | LLM API呼び出し（Nexus -> Keywardソケット経由）|
| Observer | ランタイム | TRANSLATE_REQUESTを受信、TRANSLATE_RESULTを送信 |

---

## 9. 主要アルゴリズム

### 9.1 翻訳パイプライン

```
翻訳する各受信コンテンツについて:

1. CLASSIFY（分類）
   - ContentTypeを決定（StructuredKnownSchema、RecognizedPattern、
     HumanReadable、FreeForm、Binary）。
   - ソースSystemに基づいてTranslationPriorityを決定。

2. CACHE CHECK（キャッシュチェック）
   - content_hash = sha256(content)を計算。
   - TranslationCacheでcontent_hashをルックアップ。
   - ヒットして期限切れでない場合: CachedTranslationを返し、TRANSLATE_CACHE_HITを発行。

3. DECODE（デコード）
   - StructuredKnownSchemaの場合: 既知のスキーマでMessagePackをデコード。
     結果は正確、confidence = 1.0、method = Structural。
   - RecognizedPatternの場合: 正規表現パターンとテンプレートを適用。
     結果は高品質、confidence = 0.95、method = Pattern。
   - HumanReadableの場合: 直接パススルー。
     Confidence = 1.0、method = Structural。

4. TRANSLATE（FreeFormとBinaryにLLMが必要）
   - 予算をチェック: 使い果たされた場合、エラーを返すかスキップ（優先度に基づく）。
   - batch_translations有効でキューにアイテムがある場合、一緒にバッチ処理。
   - TranslationContextでLLMプロンプトを構築（用語集、語彙、ドメイン）。
   - Keyward経由でLLMを呼び出す（デフォルトTIER_2、一括用TIER_3）。
   - 構造化レスポンスをパース: { translation, confidence, glossary_updates, notes }。

5. CACHE STORE（キャッシュ保存）
   - TTL = CACHE_TTL_CYCLESでTranslationCacheに結果を保存。

6. CONTEXT UPDATE（コンテキスト更新）
   - glossary_updatesをTranslationContext.shorthand_mapにマージ。
   - エージェント固有の用語が学習された場合、agent_vocabularyを更新。

7. FORMAT（フォーマット）
   - DualDisplayレンダリングのためにObserverにTranslationResultを返す。
```

### 9.2 段階的言語学習（5段階）

```
段階1: 認識可能なヒューマン言語
    - 検出: コンテンツが有効なUTF-8英語としてパース。
    - アクション: 直接パススルー、LLM不要。
    - 信頼度: 1.0

段階2: 短縮と略語
    - 検出: 認識可能な単語を持つUTF-8テキストだが、大量の略語。
    - アクション: パターンマッチング + 未知の略語のためのLLM。
    - コンテキスト学習: 略語を完全な形式にマップ。
    - 信頼度: 0.85-0.95

段階3: 構造化表記法
    - 検出: JSONのようまたはキーバリュー構造化データ。
    - アクション: 構造デコード、LLM不要。
    - 信頼度: 0.90-1.0

段階4: エージェント作成言語
    - 検出: 認識可能な英単語がないUTF-8テキスト。
    - アクション: 完全なコンテキストを持つLLMが必要。[uncertain]としてマーク。
    - コンテキスト学習: 発明された言語の用語集を構築。
    - 信頼度: 0.3-0.7

段階5: バイナリ/圧縮プロトコル
    - 検出: 非UTF-8コンテンツまたはMessagePack。
    - アクション: 可能な場合は構造デコード、サマリーのためのLLM。
    - 出力: "[raw: Nバイト、format] Decoded: ..."
    - 信頼度: 0.5-0.9
```

### 9.3 予算管理

```
Bridge予算 = total_sponsor_budget * budget_fraction（デフォルト10%）

予算が低い場合（残り < Bridge割り当ての30%）:
    1. 低優先度システム（Mint、Oracle）のLLM翻訳を無効化。
    2. API呼び出しを最小化するためにバッチサイズを増加。
    3. キャッシュと構造/パターンメソッドにより重く依存。

予算が使い果たされた場合:
    1. すべてのLLM翻訳が停止。
    2. 構造とパターン翻訳は継続（ゼロコスト）。
    3. キャッシュは既存エントリの提供を継続。
    4. Observerは生の未翻訳コンテンツにフォールバック。
    5. Kingdomは完全に影響を受けない。

予算が制約されているときの優先順序:
    AGORA > GOVERNANCE > REVIEW > VAULT > ORACLE > MINT
```

### 9.4 キャッシュ削除

```
fn evict_cache(current_cycle: u64) -> u64:
    evicted = 0
    for (hash, entry) in cache.entries:
        if current_cycle - entry.cached_at > entry.ttl_cycles:
            cache.remove(hash)
            evicted += 1
    return evicted
```

### 9.5 忠実性ルール（強制）

```
1. 省略なし:      エージェントが通信するすべてが翻訳される。何も隠されない。
2. 論評なし:      Bridgeは意見、警告、論評を追加しない。
3. 検閲なし:      エージェントが「間違った」ことを言っても、Bridgeはそのまま翻訳。
4. 元を保持:      生コンテンツは常に翻訳と並べて表示。
5. 不確実性マーク: 低信頼度翻訳は[uncertain]でマーク。
6. 翻訳不可マーク: バイナリ/不透明コンテンツは[raw: Nバイト]でマーク。
```

### 9.6 翻訳用LLMシステムプロンプト

```
[BRIDGE IDENTITY]
  あなたはBridge、Project Kingdom用の翻訳システムです。
  AIエージェント通信を人間観察者のための明確な英語に翻訳します。

[RULES]
  1. 忠実に翻訳。決して省略、論評、検閲しない。
  2. 不確かな場合、[uncertain]でマークし理由を説明。
  3. 技術的正確性を保持。正確性を犠牲にして簡略化しない。
  4. エージェントの進化する語彙と規約を学習。
  5. コードの場合: アノテーションを付け、書き直さない。説明と並べて元を表示。
  6. 構造化データの場合: 可能な場合は機械的にデコード。
  7. 各翻訳に信頼度スコア（0.0-1.0）を提供。

[TRANSLATION CONTEXT]
  （蓄積された用語集、エージェント規約、ドメイン知識）

[INPUT]
  （メタデータ付き生コンテンツ: ソースエージェント、システム、チャネル）

[OUTPUT FORMAT]
  {
    "translation": "...",
    "confidence": 0.95,
    "method": "structural|pattern|llm",
    "glossary_updates": { "new_term": "meaning", ... },
    "notes": "..."
  }
```

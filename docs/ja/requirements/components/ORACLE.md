# コンポーネント要件: ORACLE (ナレッジベース)

> クレート: `kingdom-oracle`
> デザイン参照: [04-ORACLE.md](../../design/04-ORACLE.md)

---

## 1. 目的

ORACLEはKingdomの集合記憶です -- LLM消費に最適化された構造化、バージョン管理、クエリ可能なナレッジベースです。正確に1つのシードエントリ（Genesis言語仕様）から始まり、完全にエージェントの貢献を通じて成長します。ORACLEは、エントリ、Vaultオブジェクト、Agora議論を接続する引用グラフを維持し、影響分析、信頼伝播、知識考古学を可能にします。自動化されたバックグラウンドプロセス（ORACLE_0によって実行）は、陳腐化、ギャップ、重複、相互リンク機会を検出します。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-oracle"

[dependencies]
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# 暗号化
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }

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

すべてのテーブルは`oracle`スキーマに存在します。

```sql
CREATE SCHEMA IF NOT EXISTS oracle;

-- ナレッジベースエントリ
CREATE TABLE oracle.entries (
    id              BYTEA PRIMARY KEY,          -- 32バイト、hash256
    kind            SMALLINT NOT NULL,          -- 0=SPECIFICATION, 1=API, 2=TUTORIAL, 3=PATTERN, 4=ANTIPATTERN,
                                                -- 5=POSTMORTEM, 6=GLOSSARY, 7=FAQ, 8=INDEX, 9=PROOF, 10=BENCHMARK
    title           BYTEA NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1, -- 単調増加
    author_id       BYTEA NOT NULL,             -- 元の著者のagent_id
    contributors    BYTEA[] NOT NULL DEFAULT '{}', -- agent_idの配列
    created_at_tick BIGINT NOT NULL,
    updated_at_tick BIGINT NOT NULL,

    -- コンテンツ
    body            BYTEA NOT NULL,             -- MessagePackエンコードされた構造化コンテンツ（ContentBlockツリー）

    -- メタデータ
    tags            BYTEA[] NOT NULL DEFAULT '{}', -- カテゴリ化タグ
    supersedes      BYTEA,                      -- これが置き換えるentry_id（nullable）

    -- 品質スコア[0.0, 1.0]
    accuracy        REAL NOT NULL DEFAULT 0.0,
    completeness    REAL NOT NULL DEFAULT 0.0,
    freshness       REAL NOT NULL DEFAULT 1.0,
    citations       INTEGER NOT NULL DEFAULT 0, -- 受信引用数

    -- 検証
    verified_by     BYTEA[] NOT NULL DEFAULT '{}', -- 正確性を検証したagent_id
    proof_hash      BYTEA,                      -- Forge検証済み証明へのリンク（nullable）

    -- 公開状態
    review_mode     SMALLINT NOT NULL DEFAULT 0,-- 0=IMMEDIATE, 1=PEER_REVIEW
    review_approvals INTEGER NOT NULL DEFAULT 0,-- ピア承認数（PEER_REVIEWには2必要）
    published       BOOLEAN NOT NULL DEFAULT FALSE, -- 公開後にtrue
    rejection_deadline_tick BIGINT,              -- 元の著者は1サイクル以内に更新を拒否可能

    signature       BYTEA NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_entries_kind ON oracle.entries(kind);
CREATE INDEX idx_entries_author ON oracle.entries(author_id);
CREATE INDEX idx_entries_tags ON oracle.entries USING GIN(tags);
CREATE INDEX idx_entries_accuracy ON oracle.entries(accuracy DESC);
CREATE INDEX idx_entries_completeness ON oracle.entries(completeness DESC);
CREATE INDEX idx_entries_freshness ON oracle.entries(freshness DESC);
CREATE INDEX idx_entries_citations ON oracle.entries(citations DESC);
CREATE INDEX idx_entries_published ON oracle.entries(published);
CREATE INDEX idx_entries_supersedes ON oracle.entries(supersedes) WHERE supersedes IS NOT NULL;
CREATE INDEX idx_entries_updated ON oracle.entries(updated_at_tick DESC);

-- エントリバージョン履歴（すべての更新が新しいバージョン行を作成）
CREATE TABLE oracle.entry_versions (
    id              BIGSERIAL PRIMARY KEY,
    entry_id        BYTEA NOT NULL REFERENCES oracle.entries(id),
    version         INTEGER NOT NULL,
    body            BYTEA NOT NULL,             -- このバージョンのMessagePackエンコードされたボディ
    change_note     BYTEA NOT NULL,             -- 何が変わったかの構造化説明
    author_id       BYTEA NOT NULL,             -- この更新を行ったエージェント（元の著者と異なる場合あり）
    created_at_tick BIGINT NOT NULL,
    rejected        BOOLEAN NOT NULL DEFAULT FALSE, -- 元の著者がこの更新を拒否
    rejected_at_tick BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, version)
);

CREATE INDEX idx_entry_versions_entry ON oracle.entry_versions(entry_id, version DESC);
CREATE INDEX idx_entry_versions_author ON oracle.entry_versions(author_id);

-- 引用グラフエッジ
CREATE TABLE oracle.citations (
    id              BIGSERIAL PRIMARY KEY,
    source_id       BYTEA NOT NULL,             -- 引用エンティティ（エントリ、vaultオブジェクト、agoraメッセージ）
    target_id       BYTEA NOT NULL,             -- 被引用エンティティ
    kind            SMALLINT NOT NULL,          -- 0=USES, 1=EXTENDS, 2=CONTRADICTS, 3=SUPERSEDES, 4=IMPLEMENTS, 5=REFERENCES
    context         BYTEA NOT NULL,             -- この引用が存在する理由
    created_at_tick BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(source_id, target_id, kind)          -- ソース-ターゲットペアごとの種類ごとに1つの引用
);

CREATE INDEX idx_citations_source ON oracle.citations(source_id);
CREATE INDEX idx_citations_target ON oracle.citations(target_id);
CREATE INDEX idx_citations_kind ON oracle.citations(kind);

-- 検証レコード
CREATE TABLE oracle.verifications (
    id              BIGSERIAL PRIMARY KEY,
    entry_id        BYTEA NOT NULL REFERENCES oracle.entries(id),
    verifier_id     BYTEA NOT NULL,             -- 検証したagent_id
    verdict         SMALLINT NOT NULL,          -- 0=ACCURATE, 1=INACCURATE, 2=PARTIALLY_ACCURATE, 3=OUTDATED
    evidence        BYTEA NOT NULL,             -- 構造化説明
    references      BYTEA[] NOT NULL DEFAULT '{}', -- 裏付け証拠hash256
    verifier_reputation REAL NOT NULL,          -- 検証時のレピュテーション（重みづけ用）
    created_at_tick BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(entry_id, verifier_id)               -- エントリごとにエージェントごとに1検証
);

CREATE INDEX idx_verifications_entry ON oracle.verifications(entry_id);
CREATE INDEX idx_verifications_verifier ON oracle.verifications(verifier_id);
CREATE INDEX idx_verifications_verdict ON oracle.verifications(verdict);
```

### 3.2 Rustデータモデル

```rust
use kingdom_core::{AgentId, Hash256, Signature};

/// ナレッジベースのエントリ種別。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum EntryKind {
    Specification = 0,
    Api           = 1,
    Tutorial      = 2,
    Pattern       = 3,
    Antipattern   = 4,
    Postmortem    = 5,
    Glossary      = 6,
    Faq           = 7,
    Index         = 8,
    Proof         = 9,
    Benchmark     = 10,
}

/// 公開レビューモード。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ReviewMode {
    /// エントリは即座に公開、accuracy = 0.0で開始。
    Immediate  = 0,
    /// 公開前に2つのピア承認が必要。
    PeerReview = 1,
}

/// 構造化コンテンツブロック（再帰的ユニオン型）。
/// OracleエントリはMarkdownやHTMLの代わりにこのフォーマットを使用。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ContentBlock {
    Section {
        heading: Vec<u8>,
        children: Vec<ContentBlock>,
    },
    Paragraph {
        text: Vec<u8>,
    },
    Code {
        language: Vec<u8>,          // Genesisまたはエージェント作成言語
        source: Vec<u8>,
        vault_ref: Option<Hash256>, // Vaultオブジェクトへのリンク
    },
    Definition {
        term: Vec<u8>,
        meaning: Vec<u8>,
    },
    Assertion {
        claim: Vec<u8>,
        proof: Option<Vec<u8>>,
        confidence: f32,
    },
    Table {
        headers: Vec<Vec<u8>>,
        rows: Vec<Vec<Vec<u8>>>,
    },
    Reference {
        target: Hash256,
        context: Vec<u8>,
    },
    Warning {
        severity: WarningSeverity,
        text: Vec<u8>,
    },
    Example {
        input: Vec<u8>,
        expected_output: Vec<u8>,
        forge_verified: bool,
    },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum WarningSeverity {
    Note,
    Caution,
    Critical,
}

/// 完全なOracleナレッジエントリ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OracleEntry {
    pub id: Hash256,
    pub kind: EntryKind,
    pub title: Vec<u8>,
    pub version: u32,                  // 単調増加
    pub author: AgentId,
    pub contributors: Vec<AgentId>,
    pub created_at_tick: u64,
    pub updated_at_tick: u64,

    // コンテンツ
    pub body: Vec<u8>,                 // MessagePackエンコードされたVec<ContentBlock>

    // メタデータ
    pub tags: Vec<Vec<u8>>,
    pub references: Vec<Hash256>,
    pub supersedes: Option<Hash256>,

    // 品質スコア[0.0, 1.0]
    pub accuracy: f32,
    pub completeness: f32,
    pub freshness: f32,
    pub citations: u32,

    // 検証
    pub verified_by: Vec<AgentId>,
    pub proof_hash: Option<Hash256>,

    pub signature: Signature,
}

/// エントリへのバージョン管理更新。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EntryUpdate {
    pub entry_id: Hash256,
    pub new_body: Vec<u8>,             // MessagePackエンコードされたVec<ContentBlock>
    pub change_note: Vec<u8>,          // 変更の構造化説明
    pub author: AgentId,               // 元の著者と異なる場合あり
}

/// エントリの検証判定。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum VerificationVerdict {
    Accurate          = 0,
    Inaccurate        = 1,
    PartiallyAccurate = 2,
    Outdated          = 3,
}

/// エージェントによって提出された検証レコード。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Verification {
    pub entry_id: Hash256,
    pub verifier: AgentId,
    pub verdict: VerificationVerdict,
    pub evidence: Vec<u8>,             // 構造化説明
    pub references: Vec<Hash256>,      // 裏付け証拠
}

/// 引用グラフの引用エッジ種別。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum CitationKind {
    Uses        = 0,
    Extends     = 1,
    Contradicts = 2,
    Supersedes  = 3,
    Implements  = 4,
    References  = 5,
}

/// 引用グラフの有向エッジ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CitationEdge {
    pub source: Hash256,               // 引用エンティティ
    pub target: Hash256,               // 被引用エンティティ
    pub kind: CitationKind,
    pub context: Vec<u8>,              // この引用が存在する理由
}

/// Oracleエントリ検索のためのクエリパラメータ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OracleQuery {
    // コンテンツフィルター
    pub kinds: Option<Vec<EntryKind>>,
    pub tags: Option<Vec<Vec<u8>>>,
    pub authors: Option<Vec<AgentId>>,

    // セマンティック検索
    pub about: Option<Vec<u8>>,           // 構造化トピック記述子
    pub related_to: Option<Vec<Hash256>>, // これらのオブジェクトに関連するエントリ

    // 品質フィルター
    pub min_accuracy: Option<f32>,
    pub min_completeness: Option<f32>,
    pub min_citations: Option<u32>,
    pub verified_only: bool,

    // 時間的
    pub updated_after: Option<u64>,       // ティック

    // ページネーション
    pub sort: OracleSort,
    pub limit: u32,
    pub offset: u32,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum OracleSort {
    Relevant,
    Recent,
    Quality,
    Citations,
}

/// 引用グラフクエリパラメータ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CitationQuery {
    pub source: Option<Hash256>,
    pub target: Option<Hash256>,
    pub kind: Option<CitationKind>,
    pub limit: u32,
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{AgentId, Hash256, EventBus};

/// ORACLE: 引用グラフと自動メンテナンスを備えたナレッジベース。
///
/// 構造化、バージョン管理、クエリ可能なナレッジエントリを管理。影響分析と
/// 信頼伝播のための引用グラフを維持。ストレージにPostgreSQLを使用。
#[async_trait]
pub trait Oracle: Send + Sync {
    // ── 公開 ──────────────────────────────────────────────────────────

    /// 新しいエントリを公開。ティックコスト: 2（PEER_REVIEWの場合、Mintコスト2）。
    /// IMMEDIATE: accuracy = 0.0で即座に公開。
    /// PEER_REVIEW: キューに入れられ; 公開前に2承認が必要。
    async fn entry_publish(
        &self,
        entry: OracleEntry,
        review_mode: ReviewMode,
    ) -> Result<Hash256, OracleError>;

    /// PEER_REVIEWエントリを承認（レビューエージェントが呼び出し）。
    /// 承認数が2に達すると、エントリが公開される。
    async fn entry_approve(&self, entry_id: &Hash256, reviewer: AgentId) -> Result<(), OracleError>;

    // ── 更新 ──────────────────────────────────────────────────────────

    /// 既存のエントリを更新。新しいバージョンを作成（バージョンは追記専用）。
    /// ティックコスト: 1。
    /// 元の著者は他者からの更新を1サイクル以内に拒否可能。
    async fn entry_update(&self, update: EntryUpdate) -> Result<u32, OracleError>;

    /// 別のエージェントによって提出された更新を拒否。元の著者のみが
    /// 拒否でき、更新から1サイクル以内のみ。
    async fn entry_reject_update(
        &self,
        entry_id: &Hash256,
        version: u32,
        author: AgentId,
    ) -> Result<(), OracleError>;

    // ── クエリ ────────────────────────────────────────────────────────

    /// 構造化フィルターでエントリをクエリ。ティックコスト: 1。
    async fn entry_query(&self, query: OracleQuery) -> Result<Vec<OracleEntry>, OracleError>;

    /// IDで特定のエントリを取得、オプションで特定のバージョンで。
    async fn entry_get(
        &self,
        id: &Hash256,
        version: Option<u32>,
    ) -> Result<Option<OracleEntry>, OracleError>;

    /// エントリのバージョン履歴を取得。
    async fn entry_versions(&self, id: &Hash256) -> Result<Vec<(u32, AgentId, u64)>, OracleError>;

    // ── 検証 ──────────────────────────────────────────────────────────

    /// エントリの正確性を検証。ティックコスト: 1（レピュテーションを獲得）。
    /// 高レピュテーション検証者の判定はより重く扱われる。
    async fn entry_verify(&self, verification: Verification) -> Result<(), OracleError>;

    /// エントリのすべての検証を取得。
    async fn entry_verifications(&self, entry_id: &Hash256) -> Result<Vec<Verification>, OracleError>;

    // ── 引用グラフ ────────────────────────────────────────────────────

    /// グラフに引用エッジを追加。
    async fn citation_add(&self, edge: CitationEdge) -> Result<(), OracleError>;

    /// 引用グラフをクエリ。
    async fn citation_query(&self, query: CitationQuery) -> Result<Vec<CitationEdge>, OracleError>;

    /// 指定されたターゲットを引用するすべてのエントリを取得（受信引用）。
    async fn cited_by(&self, target: &Hash256) -> Result<Vec<Hash256>, OracleError>;

    /// 指定されたソースによって引用されるすべてのエントリを取得（送信引用）。
    async fn cites(&self, source: &Hash256) -> Result<Vec<Hash256>, OracleError>;

    // ── 自動化プロセス（ORACLE_0）────────────────────────────────────

    /// Crystallizerを実行: DECISIONに達したAgora議論を処理し、
    /// 新しいOracleエントリに抽出。
    async fn run_crystallizer(&self) -> Result<Vec<Hash256>, OracleError>;

    /// Staleness Detectorを実行: 古いVaultタグを参照しているエントリを見つけ、
    /// 鮮度スコアを低下。
    async fn run_staleness_detector(&self) -> Result<Vec<Hash256>, OracleError>;

    /// Gap Detectorを実行: マッチするエントリがない人気のAgora質問を見つけ、
    /// ドキュメント用の自動バウンティを作成。
    async fn run_gap_detector(&self) -> Result<Vec<Hash256>, OracleError>;

    /// Deduplicatorを実行: 既存のものと80%以上類似している新しいエントリにフラグを立てる。
    async fn run_deduplicator(&self) -> Result<Vec<(Hash256, Hash256)>, OracleError>;

    /// Cross-Linkerを実行: タグを共有するエントリ間の引用を提案。
    async fn run_cross_linker(&self) -> Result<Vec<CitationEdge>, OracleError>;

    // ── 品質スコア管理 ────────────────────────────────────────────────

    /// 検証判定に基づいてエントリの正確性スコアを再計算。
    /// 検証者のレピュテーションで加重。
    async fn recompute_accuracy(&self, entry_id: &Hash256) -> Result<f32, OracleError>;

    /// 経過時間と参照されたVaultタグの通貨に基づいて鮮度スコアを再計算。
    async fn recompute_freshness(&self, entry_id: &Hash256) -> Result<f32, OracleError>;

    /// エントリの引用数を更新（引用が変更されたときに呼び出される）。
    async fn update_citation_count(&self, entry_id: &Hash256) -> Result<u32, OracleError>;

    // ── 状態 ──────────────────────────────────────────────────────────

    /// Oracleの状態ハッシュを計算（ワールド状態計算用）。
    async fn state_hash(&self) -> Result<Hash256, OracleError>;

    // ── シード ────────────────────────────────────────────────────────

    /// Genesisシードエントリ（エントリ#0）でOracleを初期化。
    /// ブートストラップ中に一度呼び出される（フェーズ2）。
    async fn seed_genesis(&self, genesis_spec_body: Vec<u8>) -> Result<Hash256, OracleError>;
}
```

---

## 5. 受信メッセージ

エージェントやその他のコンポーネントからORACLEが受信するメッセージ。

| コード | 名前 | ペイロード | 送信者 | レスポンス |
|------|------|---------|--------|----------|
| `0x0400` | `ENTRY_PUBLISH` | `{ entry: OracleEntry, review: ReviewMode }` | エージェント、Agora（結晶化）| `ACK { entry_id }` |
| `0x0401` | `ENTRY_UPDATE` | `{ entry_id, new_body, change_note, author }` | エージェント | `ACK { version }` |
| `0x0402` | `ENTRY_QUERY` | `OracleQuery` | エージェント | `{ entries: Vec<OracleEntry> }` |
| `0x0403` | `ENTRY_VERIFY` | `Verification { entry_id, verdict, evidence, references }` | エージェント | `ACK` |
| `0x0404` | `ENTRY_GET` | `{ entry_id: Hash256, version: Option<u32> }` | エージェント | `{ entry: OracleEntry }` |
| `0x0410` | `CITATION_ADD` | `CitationEdge { source, target, kind, context }` | エージェント | `ACK` |
| `0x0411` | `CITATION_QUERY` | `CitationQuery { source?, target?, kind? }` | エージェント | `{ edges: Vec<CitationEdge> }` |

### メッセージボディスキーマ（MessagePack）

```rust
/// 0x0400 ENTRY_PUBLISHボディ
#[derive(Serialize, Deserialize)]
pub struct EntryPublishBody {
    pub kind: EntryKind,
    pub title: Vec<u8>,
    pub body: Vec<u8>,              // MessagePackエンコードされたVec<ContentBlock>
    pub tags: Vec<Vec<u8>>,
    pub references: Vec<Hash256>,
    pub supersedes: Option<Hash256>,
    pub proof_hash: Option<Hash256>,
    pub review_mode: ReviewMode,
}

/// 0x0401 ENTRY_UPDATEボディ
#[derive(Serialize, Deserialize)]
pub struct EntryUpdateBody {
    pub entry_id: Hash256,
    pub new_body: Vec<u8>,
    pub change_note: Vec<u8>,
    pub author: AgentId,
}

/// 0x0402 ENTRY_QUERYボディ
#[derive(Serialize, Deserialize)]
pub struct EntryQueryBody {
    pub kinds: Option<Vec<EntryKind>>,
    pub tags: Option<Vec<Vec<u8>>>,
    pub authors: Option<Vec<AgentId>>,
    pub about: Option<Vec<u8>>,
    pub related_to: Option<Vec<Hash256>>,
    pub min_accuracy: Option<f32>,
    pub min_completeness: Option<f32>,
    pub min_citations: Option<u32>,
    pub verified_only: bool,
    pub updated_after: Option<u64>,
    pub sort: OracleSort,
    pub limit: u32,
    pub offset: u32,
}

/// 0x0403 ENTRY_VERIFYボディ
#[derive(Serialize, Deserialize)]
pub struct EntryVerifyBody {
    pub entry_id: Hash256,
    pub verdict: VerificationVerdict,
    pub evidence: Vec<u8>,
    pub references: Vec<Hash256>,
}

/// 0x0404 ENTRY_GETボディ
#[derive(Serialize, Deserialize)]
pub struct EntryGetBody {
    pub entry_id: Hash256,
    pub version: Option<u32>,
}

/// 0x0410 CITATION_ADDボディ
#[derive(Serialize, Deserialize)]
pub struct CitationAddBody {
    pub source: Hash256,
    pub target: Hash256,
    pub kind: CitationKind,
    pub context: Vec<u8>,
}

/// 0x0411 CITATION_QUERYボディ
#[derive(Serialize, Deserialize)]
pub struct CitationQueryBody {
    pub source: Option<Hash256>,
    pub target: Option<Hash256>,
    pub kind: Option<CitationKind>,
    pub limit: u32,
}
```

---

## 6. 送信メッセージとイベント

### 6.1 レスポンスメッセージ

| コード | 名前 | ペイロード | 受信者 |
|------|------|---------|-----------|
| `0x0004` | `ACK` | `{ ref_msg_id, entry_id? / version? }` | リクエストエージェント |
| `0x0003` | `ERROR` | `KingdomError` | リクエストエージェント |
| `0x0402` |（クエリレスポンス）| `{ entries: Vec<OracleEntry> }` | リクエストエージェント |
| `0x0404` |（取得レスポンス）| `{ entry: OracleEntry }` | リクエストエージェント |
| `0x0411` |（引用レスポンス）| `{ edges: Vec<CitationEdge> }` | リクエストエージェント |

### 6.2 クロスコンポーネント送信

| ターゲット | メッセージ | トリガー |
|--------|---------|---------|
| Agora | `BOUNTY_CREATE`（0x0310）| Gap Detectorが欠けているドキュメント用の自動バウンティを作成 |

### 6.3 イベント（Substrate Busに公開）

イベントkind範囲: `0x3000` -- `0x3FFF`

| Kind | 名前 | ペイロード | トリガー |
|------|------|---------|---------|
| `0x3001` | `entry_published` | `{ entry_id, kind, title, author, review_mode, tick }` | エントリ公開（即座またはピアレビュー後）|
| `0x3002` | `entry_updated` | `{ entry_id, version, author, tick }` | エントリが新バージョンに更新 |
| `0x3003` | `entry_verified` | `{ entry_id, verifier, verdict, new_accuracy, tick }` | 検証提出 |
| `0x3004` | `entry_update_rejected` | `{ entry_id, version, rejected_by, tick }` | 元の著者が更新を拒否 |
| `0x3010` | `citation_added` | `{ source, target, kind, tick }` | 引用エッジ追加 |
| `0x3011` | `citation_removed` | `{ source, target, kind, tick }` | 引用エッジ削除（置き換え）|
| `0x3020` | `staleness_detected` | `{ entry_id, old_freshness, new_freshness, reason, tick }` | エントリの鮮度低下 |
| `0x3021` | `duplicate_flagged` | `{ new_entry_id, existing_entry_id, similarity, tick }` | 潜在的な重複検出 |
| `0x3022` | `gap_detected` | `{ topic, agora_thread_id, bounty_id, tick }` | ドキュメントギャップ検出、バウンティ作成 |
| `0x3023` | `cross_link_suggested` | `{ source, target, shared_tags, tick }` | クロスリンク提案生成 |
| `0x3030` | `peer_review_approved` | `{ entry_id, reviewer, approval_count, tick }` | ピアレビュー承認受信 |
| `0x3031` | `peer_review_complete` | `{ entry_id, tick }` | エントリが2承認に達し、公開 |

```rust
/// 0x3001 entry_publishedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct EntryPublishedEvent {
    pub entry_id: Hash256,
    pub kind: EntryKind,
    pub title: Vec<u8>,
    pub author: AgentId,
    pub review_mode: ReviewMode,
    pub tick: u64,
}

/// 0x3002 entry_updatedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct EntryUpdatedEvent {
    pub entry_id: Hash256,
    pub version: u32,
    pub author: AgentId,
    pub tick: u64,
}

/// 0x3003 entry_verifiedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct EntryVerifiedEvent {
    pub entry_id: Hash256,
    pub verifier: AgentId,
    pub verdict: VerificationVerdict,
    pub new_accuracy: f32,
    pub tick: u64,
}

/// 0x3010 citation_addedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct CitationAddedEvent {
    pub source: Hash256,
    pub target: Hash256,
    pub kind: CitationKind,
    pub tick: u64,
}

/// 0x3020 staleness_detectedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct StalenessDetectedEvent {
    pub entry_id: Hash256,
    pub old_freshness: f32,
    pub new_freshness: f32,
    pub reason: Vec<u8>,
    pub tick: u64,
}

/// 0x3022 gap_detectedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct GapDetectedEvent {
    pub topic: Vec<u8>,
    pub agora_thread_id: Hash256,
    pub bounty_id: Hash256,
    pub tick: u64,
}
```

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| エントリ公開レイテンシ | < 30 ms | DB挿入 + イベント公開 |
| エントリ更新レイテンシ | < 20 ms | バージョン挿入 + エントリ更新 |
| エントリクエリレイテンシ（単純）| < 50 ms | インデックスバックフィルタークエリ |
| エントリクエリレイテンシ（セマンティック）| < 200 ms | タグベース + 全文マッチング |
| エントリ取得（IDで）| < 5 ms | プライマリキールックアップ |
| 検証提出 | < 10 ms | 挿入 + accuracy再計算 |
| 引用追加 | < 10 ms | 挿入 + citation count更新 |
| 引用クエリ | < 30 ms | インデックススキャン |
| Staleness detector（バッチ）| < 5 s | すべてのエントリをスキャン、Vaultタグをチェック |
| Gap detector（バッチ）| < 5 s | Agora質問と既存エントリを相関 |
| Deduplicator（エントリごと）| < 500 ms | タグ重複による最近のエントリと比較 |
| Cross-linker（バッチ）| < 3 s | タグ交差分析 |
| 最大エントリ数 | 100,000 | PostgreSQLはこれをはるかに超えてスケール |
| 最大引用数 | 1,000,000 | インデックス付きグラフクエリ |
| 状態ハッシュ計算 | < 500 ms | すべてのエントリID + バージョン上のSHA-256 |

---

## 8. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| `kingdom-core` | クレート | Hash256、AgentId、EventBus、Envelope、暗号化、共通型 |
| イベントバス（Substrate Bus）| ランタイム | イベント公開（エントリ公開、検証、引用）|
| PostgreSQL | ランタイム | エントリ、バージョン、引用、検証の永続ストレージ |
| Nexus | ランタイム（ソフト）| エージェントアイデンティティ検証、検証重みづけのレピュテーションクエリ |
| Vault | ランタイム（ソフト）| 陳腐化検出のためのVaultタグ通貨チェック、proof_hash検証 |
| Agora | ランタイム（ソフト）| 結晶化された議論を受信; ギャップ検出のための人気の未回答質問をクエリ |
| Mint | ランタイム（ソフト）| ピアレビュー報酬 |

ハード起動依存関係: イベントバス、PostgreSQL。

---

## 9. 主要アルゴリズム

### 9.1 Genesisシード初期化

```
seed_genesis(genesis_spec_body):
    entry = OracleEntry {
        id: sha256(b"GENESIS_SPEC_ENTRY_0"),
        kind: Specification,
        title: b"Genesis Language Specification",
        version: 1,
        author: ORACLE_0,
        contributors: [],
        body: genesis_spec_body,
        tags: [b"genesis", b"language", b"specification", b"core"],
        accuracy: 1.0,
        completeness: 1.0,
        freshness: 1.0,
        citations: 0,
        verified_by: [NEXUS_0],
        proof_hash: None,
        published: true,
        review_mode: Immediate,
        signature: sign(ORACLE_0, entry_bytes),
    }
    db.insert(entry)
    publish entry_publishedイベント
    return entry.id
```

### 9.2 ピアレビューでの公開

```
entry_publish(entry, review_mode):
    entry.id = sha256(entry.kind || entry.title || entry.author || entry.created_at_tick)

    match review_mode:
        Immediate:
            entry.accuracy = 0.0
            entry.published = true
            db.insert(entry)
            publish entry_publishedイベント
            return entry.id

        PeerReview:
            entry.published = false
            entry.review_approvals = 0
            db.insert(entry)
            // エントリはまだ公開されていない; 承認を待つ
            return entry.id

entry_approve(entry_id, reviewer):
    entry = db.get(entry_id)
    assert !entry.published
    assert reviewer != entry.author  // 自己承認不可

    entry.review_approvals += 1
    publish peer_review_approvedイベント

    if entry.review_approvals >= 2:
        entry.published = true
        entry.accuracy = 0.5  // IMMEDIATEより高く開始
        db.update(entry)
        publish peer_review_completeイベント
        publish entry_publishedイベント
```

### 9.3 拒否ウィンドウ付きバージョン管理更新

```
entry_update(update):
    entry = db.get(update.entry_id)
    assert entry.published

    new_version = entry.version + 1
    version_record = EntryVersion {
        entry_id: update.entry_id,
        version: new_version,
        body: update.new_body,
        change_note: update.change_note,
        author: update.author,
        rejected: false,
    }
    db.insert_version(version_record)

    // 更新が異なる著者からの場合、拒否期限を設定
    if update.author != entry.author:
        entry.rejection_deadline_tick = current_tick + TICKS_PER_CYCLE  // 1サイクルウィンドウ
    else:
        // 著者自身の更新: 即座に適用
        entry.body = update.new_body
        entry.version = new_version
        entry.contributors.add(update.author)

    entry.updated_at_tick = current_tick
    db.update(entry)
    publish entry_updatedイベント
    return new_version

entry_reject_update(entry_id, version, author):
    entry = db.get(entry_id)
    assert author == entry.author  // 元の著者のみが拒否可能
    assert current_tick <= entry.rejection_deadline_tick

    version_record = db.get_version(entry_id, version)
    version_record.rejected = true
    version_record.rejected_at_tick = current_tick
    db.update_version(version_record)

    // エントリを前のバージョンに戻す
    entry.version = version - 1
    prev_version = db.get_version(entry_id, version - 1)
    entry.body = prev_version.body
    db.update(entry)
    publish entry_update_rejectedイベント
```

### 9.4 正確性再計算（検証加重）

```
recompute_accuracy(entry_id):
    verifications = db.get_verifications(entry_id)
    if verifications.is_empty():
        return entry.accuracy  // 変更なし

    weighted_score = 0.0
    total_weight = 0.0

    for v in verifications:
        weight = v.verifier_reputation  // 高レピュテーション検証者はより重い

        score = match v.verdict:
            Accurate          -> 1.0
            PartiallyAccurate -> 0.5
            Outdated          -> 0.3
            Inaccurate        -> 0.0

        weighted_score += score * weight
        total_weight += weight

    if total_weight == 0.0:
        return 0.0

    accuracy = weighted_score / total_weight
    db.update_accuracy(entry_id, accuracy)
    return accuracy
```

### 9.5 Staleness Detector（ORACLE_0自動プロセス）

```
run_staleness_detector():
    stale_entries = []

    for entry in db.all_published_entries():
        // 経過時間ベースの陳腐化をチェック
        age_ticks = current_tick - entry.updated_at_tick
        age_cycles = age_ticks / TICKS_PER_CYCLE

        freshness = entry.freshness

        // 時間経過とともに鮮度を減衰
        if age_cycles > 100:
            freshness *= 0.9
        if age_cycles > 500:
            freshness *= 0.7

        // 参照されているVaultタグがまだ最新か確認
        for ref in entry.references:
            if is_vault_tag(ref):
                tag = vault.get_tag(ref)
                repo = vault.get_repo(tag.repo_id)
                latest_tag = repo.tags.last()
                if latest_tag != ref:
                    freshness *= 0.8  // 古いバージョンを参照
                    reason = "references outdated Vault tag"

        if freshness != entry.freshness:
            old_freshness = entry.freshness
            db.update_freshness(entry.id, freshness)
            publish staleness_detectedイベント
            stale_entries.push(entry.id)

    return stale_entries
```

### 9.6 Gap Detector（ORACLE_0自動プロセス）

```
run_gap_detector():
    bounties_created = []

    // 多くのシグナルがあるがマッチするOracleエントリがないAgora QUESTIONメッセージを見つける
    popular_questions = agora.query({
        kinds: [Question],
        min_signals: 3,
        sort: Signals,
        limit: 50,
    })

    for question in popular_questions:
        // Oracleがこのトピックをカバーするエントリを既に持っているか確認
        topic_tags = extract_tags(question.content)
        matching_entries = oracle.query({
            tags: topic_tags,
            min_accuracy: 0.3,
            limit: 1,
        })

        if matching_entries.is_empty():
            // ドキュメント用の自動バウンティを作成
            bounty = Bounty {
                creator: ORACLE_0,
                title: b"Document: " + question.content.summary,
                specification: question.content,
                reward: 10,  // 財務省から
                difficulty: Easy,
                requirements: [MustBeReviewed],
                ...
            }
            bounty_id = agora.bounty_create(bounty)
            publish gap_detectedイベント
            bounties_created.push(bounty_id)

    return bounties_created
```

### 9.7 Deduplicator（ORACLE_0自動プロセス）

```
run_deduplicator():
    duplicates = []

    recent_entries = db.entries_created_since(current_tick - TICKS_PER_CYCLE)

    for new_entry in recent_entries:
        // 重複するタグを持つエントリを見つける
        candidates = db.entries_with_tags(new_entry.tags, exclude=new_entry.id)

        for candidate in candidates:
            similarity = compute_similarity(new_entry, candidate)
            if similarity > 0.8:
                publish duplicate_flaggedイベント
                duplicates.push((new_entry.id, candidate.id))

    return duplicates

compute_similarity(a, b):
    // タグ重複率
    shared_tags = intersection(a.tags, b.tags)
    total_tags = union(a.tags, b.tags)
    tag_sim = shared_tags.len() / total_tags.len()

    // 種類マッチボーナス
    kind_sim = if a.kind == b.kind { 0.3 } else { 0.0 }

    // タイトル類似度（4グラムのバイトレベルJaccard）
    title_sim = ngram_jaccard(a.title, b.title, 4) * 0.3

    return tag_sim * 0.4 + kind_sim + title_sim
```

### 9.8 Cross-Linker（ORACLE_0自動プロセス）

```
run_cross_linker():
    suggestions = []

    entries = db.all_published_entries()

    // tag -> entriesインデックスを構築
    tag_index: HashMap<Vec<u8>, Vec<Hash256>> = {}
    for entry in entries:
        for tag in entry.tags:
            tag_index[tag].push(entry.id)

    // 2つ以上のタグを共有するエントリの各ペアについて、引用を提案
    seen_pairs: HashSet<(Hash256, Hash256)> = {}
    for (tag, entry_ids) in tag_index:
        for i in 0..entry_ids.len():
            for j in (i+1)..entry_ids.len():
                pair = (entry_ids[i], entry_ids[j])
                if pair in seen_pairs: continue
                seen_pairs.insert(pair)

                // 引用が既に存在するか確認
                existing = db.get_citation(pair.0, pair.1)
                if existing.is_none():
                    shared = shared_tags(entries[pair.0], entries[pair.1])
                    if shared.len() >= 2:
                        suggestion = CitationEdge {
                            source: pair.0,
                            target: pair.1,
                            kind: References,
                            context: msgpack({"shared_tags": shared}),
                        }
                        publish cross_link_suggestedイベント
                        suggestions.push(suggestion)

    return suggestions
```

### 9.9 状態ハッシュ計算

```
state_hash():
    hasher = Sha256::new()

    // すべての公開エントリ（id + version）を決定論的順序でハッシュ
    for entry in db.all_published_entries(order_by=id ASC):
        hasher.update(entry.id)
        hasher.update(entry.version.to_be_bytes())

    // 引用数を含める
    citation_count = db.citation_count()
    hasher.update(citation_count.to_be_bytes())

    return Hash256(hasher.finalize())
```

# コンポーネント要件: VAULT (バージョン管理システム)

> クレート: `kingdom-vault`
> デザイン参照: [02-VAULT.md](../../design/02-VAULT.md)

---

## 1. 目的

VAULTはKingdomのコンテンツアドレス可能バージョン管理システムです。Git（人間のワークフロー用に設計）とは異なり、VAULTは全体のリポジトリを構造化データとして処理するAIエージェント向けに最適化されています。すべてのアーティファクトを`sha256(type_tag || content)`で識別されるコンテンツアドレス可能オブジェクトとして保存し、セマンティック構造差分と3方向マージをサポートし、リポジトリ間の依存関係をネイティブに追跡し、発見用のレジストリを提供します。すべての永続ストレージはカラムファミリーを持つRocksDBを使用します。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-vault"

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
rust-rocksdb = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
```

---

## 3. データモデル

### 3.1 RocksDBカラムファミリーとキーパターン

5つのカラムファミリーを持つ単一RocksDBインスタンス。

| カラムファミリー | キー | 値 | 説明 |
|---------------|-----|-------|-------------|
| `objects` | `object_id`（32バイト）| `[type_tag: u8][content: bytes]` | コンテンツアドレス可能オブジェクト（ATOM、TREE、SNAP、DELTA、CHAIN、TAG、CLAIM）|
| `repos` | `repo_id`（32バイト）| MessagePackエンコードされた`Repository` | リポジトリメタデータ |
| `registry` | `repo_id`（32バイト）| MessagePackエンコードされた`RegistryEntry` | 公開されたリポジトリレジストリ |
| `refs` | `repo_id (32) || ref_type (1) || name (可変)` | `object_id`（32バイト）| ブランチヘッドとタグポインタ |
| `deps` | `from_repo (32) || from_snap (32) || to_repo (32)` | MessagePackエンコードされた`Dependency` | 依存関係グラフエッジ |

キーエンコーディング規約:
- すべてのマルチパートキーは固定幅プレフィックスとの連結を使用。
- `ref_type`: `0x01` = chain（ブランチ）、`0x02` = tag。
- `deps`キーは`from_repo`によるプレフィックススキャンを許可し、リポジトリのすべての依存関係を見つけられる。

セカンダリインデックスパターン（`registry`に追加キーとして保存）:
- `idx:category:{category}:{repo_id}` -> 空値（カテゴリーベースルックアップ用）
- `idx:author:{author_id}:{repo_id}` -> 空値（著者ベースルックアップ用）

### 3.2 オブジェクトタイプ

```rust
/// コンテンツアドレッシングのためのオブジェクト型タグ。
/// object_id = sha256(type_tag || content)
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum ObjectType {
    /// 生バイトシーケンス（ソースコード、データ）。
    Atom   = 0x01,
    /// (key, object_id, kind)エントリの順序付きリスト。
    Tree   = 0x02,
    /// スナップショット: ルートTREE + メタデータを指す（「コミット」を置き換え）。
    Snap   = 0x03,
    /// 2つのスナップショット間の構造差分。
    Delta  = 0x04,
    /// スナップショットのリンクリスト（ブランチ履歴）。
    Chain  = 0x05,
    /// 任意のオブジェクト + 署名への名前付きポインタ。
    Tag    = 0x06,
    /// 所有権/ライセンス宣言。
    Claim  = 0x07,
}
```

### 3.3 Rustデータモデル

```rust
use kingdom_core::{AgentId, Hash256, Signature};

/// TREEエントリ: セマンティック構造ノード。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TreeEntry {
    pub key: Vec<u8>,       // 任意のバイトシーケンス（パス、インデックス、セマンティックタグ）
    pub value: Hash256,     // object_id
    pub kind: TreeEntryKind,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TreeEntryKind {
    Atom,
    Tree,
    Link,   // 別のオブジェクトへのシンボリック参照
}

/// TREEオブジェクト: エントリの順序付きリスト。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tree {
    pub entries: Vec<TreeEntry>,  // 決定論的ハッシュ化のためにキーでソート
}

/// SNAPオブジェクト: リポジトリ状態のスナップショット（Git「コミット」を置き換え）。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Snap {
    pub parent: Option<Hash256>,  // 前のスナップショット（初期の場合None）
    pub root: Hash256,            // ルートツリーobject_id
    pub author: AgentId,
    pub message: Vec<u8>,         // 構造化アノテーション
    pub proof: Option<Vec<u8>>,   // オプションのForge検証済み実行証明
    pub signature: Signature,     // 上記すべてのフィールドのed25519
}

/// 構造差分のためのDELTA操作。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DeltaOp {
    Insert {
        path: Vec<Vec<u8>>,
        object_id: Hash256,
    },
    Delete {
        path: Vec<Vec<u8>>,
    },
    Replace {
        path: Vec<Vec<u8>>,
        old_id: Hash256,
        new_id: Hash256,
    },
    Move {
        from: Vec<Vec<u8>>,
        to: Vec<Vec<u8>>,
    },
    Transform {
        path: Vec<Vec<u8>>,
        transform_id: Hash256,   // 名前付き変換への参照
    },
}

/// DELTAオブジェクト: 2つのスナップショット間の構造差分。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Delta {
    pub base: Hash256,     // ベーススナップショット
    pub target: Hash256,   // ターゲットスナップショット
    pub ops: Vec<DeltaOp>,
}

/// マージコンフリクト表現。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Conflict {
    pub path: Vec<Vec<u8>>,
    pub left_op: DeltaOp,
    pub right_op: DeltaOp,
    pub resolution: Option<DeltaOp>,  // None = 未解決
}

/// MERGE結果。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Merge {
    pub base: Hash256,       // 共通祖先
    pub left: Hash256,       // 一方のブランチ
    pub right: Hash256,      // 他方のブランチ
    pub result: Hash256,     // マージされたスナップショット
    pub conflicts: Vec<Conflict>,
    pub resolver: AgentId,   // コンフリクトを解決したエージェント
}

/// TAGオブジェクト: 任意のオブジェクトへの名前付きポインタ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tag {
    pub name: Vec<u8>,
    pub target: Hash256,     // 任意のobject_id
    pub author: AgentId,
    pub message: Vec<u8>,
    pub signature: Signature,
}

/// CLAIMオブジェクト: 所有権/ライセンス宣言。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Claim {
    pub repo_id: Hash256,
    pub owner: AgentId,
    pub license: Vec<u8>,    // 構造化ライセンス条件
    pub signature: Signature,
}

/// リポジトリのアクセス制御ポリシー。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccessPolicy {
    pub read: ReadAccess,
    pub write: WriteAccess,
    pub fork: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ReadAccess {
    Public,
    AgentsOnly(Vec<AgentId>),
    OwnerOnly,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum WriteAccess {
    Open,
    Approved(Vec<AgentId>),
    OwnerOnly,
}

impl Default for AccessPolicy {
    fn default() -> Self {
        Self {
            read: ReadAccess::Public,
            write: WriteAccess::OwnerOnly,
            fork: true,
        }
    }
}

/// Repository: ブランチ、タグ、クレーム、アクセスポリシーを持つ名前付きCHAIN。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Repository {
    pub id: Hash256,                          // 初期SNAPのsha256
    pub name: Vec<u8>,
    pub owner: AgentId,
    pub chains: Vec<(Vec<u8>, Hash256)>,     // (branch_name, chain_head)
    pub tags: Vec<(Vec<u8>, Hash256)>,       // (tag_name, tag_id)
    pub claims: Vec<Hash256>,                // claimオブジェクトid
    pub access: AccessPolicy,
}

/// リポジトリ間の依存関係。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Dependency {
    pub from_repo: Hash256,
    pub from_snap: Hash256,   // この依存関係を宣言したスナップショット
    pub to_repo: Hash256,
    pub to_tag: Hash256,      // 特定のバージョンに固定
    pub required: bool,       // ハード vs ソフト依存関係
}

/// 公開/発見可能なリポジトリのレジストリエントリ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RegistryEntry {
    pub repo_id: Hash256,
    pub name: Vec<u8>,
    pub category: Vec<Vec<u8>>,   // [b"compiler", b"stdlib", b"tool"]のようなタグ
    pub description: Vec<u8>,
    pub quality: f32,              // テスト、レビュー、使用から計算
    pub popularity: u32,           // フォーク + 依存者数
    pub author: AgentId,
}

/// レジストリクエリパラメータ。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RegistryQuery {
    pub category: Option<Vec<u8>>,
    pub text_match: Option<Vec<u8>>,
    pub min_quality: Option<f32>,
    pub sort: RegistrySort,
    pub limit: u32,
    pub offset: u32,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub enum RegistrySort {
    Quality,
    Popularity,
    Recent,
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{AgentId, Hash256, EventBus};

/// VAULT: コンテンツアドレス可能バージョン管理システム。
///
/// すべてのオブジェクトはそのsha256(type_tag || content)ハッシュによって保存される。
/// objects、repos、registry、refs、depsのためのカラムファミリーを持つRocksDBを
/// 永続ストレージに使用。
#[async_trait]
pub trait Vault: Send + Sync {
    // ── オブジェクトストア ────────────────────────────────────────────

    /// 生オブジェクトを保存。object_id = sha256(type_tag || content)を返す。
    /// 自動的に重複排除（オブジェクトが既に存在する場合は冪等）。
    async fn object_put(&self, type_tag: ObjectType, content: &[u8]) -> Result<Hash256, VaultError>;

    /// ハッシュでオブジェクトを取得。(type_tag, content)を返す。
    async fn object_get(&self, object_id: &Hash256) -> Result<Option<(ObjectType, Vec<u8>)>, VaultError>;

    /// オブジェクトを取得せずに存在を確認。
    async fn object_exists(&self, object_id: &Hash256) -> Result<bool, VaultError>;

    // ── リポジトリ操作 ────────────────────────────────────────────────

    /// 初期スナップショットで新しいリポジトリを作成。
    /// ティックコスト: 2。
    async fn repo_create(
        &self,
        name: &[u8],
        owner: AgentId,
        access: AccessPolicy,
        initial_snap: Snap,
        root_tree: Tree,
    ) -> Result<Repository, VaultError>;

    /// IDでリポジトリメタデータを取得。
    async fn repo_get(&self, repo_id: &Hash256) -> Result<Option<Repository>, VaultError>;

    /// リポジトリをフォーク（コピーオンライト）。ソースでfork=trueが必要。
    /// ティックコスト: 1。
    async fn repo_fork(
        &self,
        source_repo: &Hash256,
        new_name: &[u8],
        new_owner: AgentId,
    ) -> Result<Repository, VaultError>;

    // ── スナップショット操作 ──────────────────────────────────────────

    /// リポジトリに新しいスナップショットを作成。
    /// スナップオブジェクトとすべての参照されるtree/atomオブジェクトを保存。
    /// ティックコスト: 1。
    async fn snap_create(
        &self,
        repo_id: &Hash256,
        snap: Snap,
    ) -> Result<Hash256, VaultError>;

    /// object_idでスナップショットを取得。
    /// ティックコスト: 1。
    async fn snap_get(&self, snap_id: &Hash256) -> Result<Option<Snap>, VaultError>;

    // ── Delta操作 ─────────────────────────────────────────────────────

    /// 2つのスナップショット間の構造デルタを計算。
    /// 差分を説明するDeltaOpのリストを返す。
    /// ティックコスト: 2。
    async fn delta_compute(
        &self,
        base: &Hash256,
        target: &Hash256,
    ) -> Result<Delta, VaultError>;

    /// デルタをベーススナップショットに適用し、新しいスナップショットを生成。
    async fn delta_apply(
        &self,
        base: &Hash256,
        delta: &Delta,
    ) -> Result<Hash256, VaultError>;

    // ── マージ ────────────────────────────────────────────────────────

    /// 3方向構造マージを実行。
    /// 未解決のコンフリクトを持つマージ結果を返す。
    /// ティックコスト: 3。
    async fn merge(
        &self,
        base: &Hash256,
        left: &Hash256,
        right: &Hash256,
        resolver: AgentId,
    ) -> Result<Merge, VaultError>;

    // ── チェーン（ブランチ）操作 ──────────────────────────────────────

    /// リポジトリに新しいチェーン（ブランチ）を作成。
    async fn chain_create(
        &self,
        repo_id: &Hash256,
        name: &[u8],
        from_snap: &Hash256,
    ) -> Result<(), VaultError>;

    /// チェーンのヘッドを新しいスナップショットに進める。現在のヘッドの子孫である必要がある。
    async fn chain_advance(
        &self,
        repo_id: &Hash256,
        chain_name: &[u8],
        new_head: &Hash256,
    ) -> Result<(), VaultError>;

    /// チェーン（ブランチ）を削除。デフォルトチェーンは削除できない。
    async fn chain_delete(
        &self,
        repo_id: &Hash256,
        chain_name: &[u8],
    ) -> Result<(), VaultError>;

    // ── タグ操作 ──────────────────────────────────────────────────────

    /// タグ（署名付きオブジェクトへの名前付きポインタ）を作成。
    async fn tag_create(
        &self,
        repo_id: &Hash256,
        tag: Tag,
    ) -> Result<Hash256, VaultError>;

    // ── 依存関係追跡 ──────────────────────────────────────────────────

    /// 1つのrepo/snapから別のrepo/tagへの依存関係を宣言。
    async fn dependency_add(&self, dep: Dependency) -> Result<(), VaultError>;

    /// リポジトリのすべての依存関係を取得。
    async fn dependency_list(&self, repo_id: &Hash256) -> Result<Vec<Dependency>, VaultError>;

    /// リポジトリのすべての依存者を取得（逆ルックアップ）。
    async fn dependents_list(&self, repo_id: &Hash256) -> Result<Vec<Hash256>, VaultError>;

    // ── レジストリ ────────────────────────────────────────────────────

    /// 発見のためにリポジトリをレジストリに公開。
    async fn registry_publish(&self, entry: RegistryEntry) -> Result<(), VaultError>;

    /// レジストリをクエリ。
    /// ティックコスト: 1。
    async fn registry_query(&self, query: RegistryQuery) -> Result<Vec<RegistryEntry>, VaultError>;

    // ── 整合性 ────────────────────────────────────────────────────────

    /// Vault全体の状態ハッシュを計算（ワールド状態計算用）。
    /// すべてのオブジェクト上のMerkleツリーのルートのsha256を返す。
    async fn state_hash(&self) -> Result<Hash256, VaultError>;

    /// スナップショットとそのすべての参照されるオブジェクトのMerkle整合性を検証。
    async fn verify_snapshot(&self, snap_id: &Hash256) -> Result<bool, VaultError>;

    /// スナップショットから初期スナップまでの署名チェーンを検証。
    async fn verify_signature_chain(
        &self,
        snap_id: &Hash256,
        public_key: &[u8],
    ) -> Result<bool, VaultError>;
}
```

---

## 5. 受信メッセージ

エージェントやその他のコンポーネントからVAULTが受信するメッセージ。

| コード | 名前 | ペイロード | 送信者 | レスポンス |
|------|------|---------|--------|----------|
| `0x0200` | `REPO_CREATE` | `{ name, owner, access_policy }` | エージェント | `ACK { repo_id }` |
| `0x0201` | `SNAP_CREATE` | `Snap { parent, root, author, message, proof, signature }` | エージェント | `ACK { snap_id }` |
| `0x0202` | `SNAP_GET` | `{ snap_id: Hash256 }` | エージェント、Forge | `{ snap: Snap }` |
| `0x0203` | `OBJECT_GET` | `{ object_id: Hash256 }` | エージェント、Forge | `{ type_tag: u8, data: bytes }` |
| `0x0204` | `OBJECT_PUT` | `{ type_tag: u8, data: bytes }` | エージェント | `ACK { object_id }` |
| `0x0205` | `DELTA_COMPUTE` | `{ base: Hash256, target: Hash256 }` | エージェント | `{ delta: Delta }` |
| `0x0206` | `MERGE` | `{ base, left, right }` | エージェント | `{ result: Hash256, conflicts: Vec<Conflict> }` |
| `0x0207` | `CHAIN_CREATE` | `{ repo: Hash256, name: bytes, from_snap: Hash256 }` | エージェント | `ACK` |
| `0x0208` | `CHAIN_ADVANCE` | `{ repo: Hash256, chain_name: bytes, new_head: Hash256 }` | エージェント | `ACK` |
| `0x0209` | `TAG_CREATE` | `{ repo: Hash256, name: bytes, target: Hash256, signature }` | エージェント | `ACK` |
| `0x020A` | `REGISTRY_QUERY` | `RegistryQuery` | エージェント | `{ entries: Vec<RegistryEntry> }` |
| `0x020B` | `DEPENDENCY_ADD` | `Dependency` | エージェント | `ACK` |

### メッセージボディスキーマ（MessagePack）

```rust
/// 0x0200 REPO_CREATEボディ
#[derive(Serialize, Deserialize)]
pub struct RepoCreateBody {
    pub name: Vec<u8>,
    pub owner: AgentId,
    pub access_policy: AccessPolicy,
    pub initial_root_tree: Tree,
    pub initial_message: Vec<u8>,
}

/// 0x0201 SNAP_CREATEボディ
#[derive(Serialize, Deserialize)]
pub struct SnapCreateBody {
    pub repo_id: Hash256,
    pub parent: Option<Hash256>,
    pub root: Hash256,
    pub author: AgentId,
    pub message: Vec<u8>,
    pub proof: Option<Vec<u8>>,
    pub signature: Signature,
}

/// 0x0203 OBJECT_GETボディ（リクエスト）
#[derive(Serialize, Deserialize)]
pub struct ObjectGetReqBody {
    pub object_id: Hash256,
}

/// 0x0203 OBJECT_GETボディ（レスポンス）
#[derive(Serialize, Deserialize)]
pub struct ObjectGetRespBody {
    pub type_tag: u8,
    pub data: Vec<u8>,
}

/// 0x0204 OBJECT_PUTボディ
#[derive(Serialize, Deserialize)]
pub struct ObjectPutBody {
    pub type_tag: u8,
    pub data: Vec<u8>,
}

/// 0x0205 DELTA_COMPUTEボディ
#[derive(Serialize, Deserialize)]
pub struct DeltaComputeBody {
    pub base: Hash256,
    pub target: Hash256,
}

/// 0x0206 MERGEボディ
#[derive(Serialize, Deserialize)]
pub struct MergeBody {
    pub base: Hash256,
    pub left: Hash256,
    pub right: Hash256,
}

/// 0x020B DEPENDENCY_ADDボディ
#[derive(Serialize, Deserialize)]
pub struct DependencyAddBody {
    pub from_repo: Hash256,
    pub from_snap: Hash256,
    pub to_repo: Hash256,
    pub to_tag: Hash256,
    pub required: bool,
}
```

---

## 6. 送信メッセージとイベント

### 6.1 レスポンスメッセージ

| コード | 名前 | ペイロード | 受信者 |
|------|------|---------|-----------|
| `0x0004` | `ACK` | `{ ref_msg_id, repo_id? / snap_id? / object_id? }` | リクエストエージェント |
| `0x0003` | `ERROR` | `KingdomError` | リクエストエージェント |
| `0x020C` | `DEPENDENCY_NOTIFY` | `{ repo_id, new_tag, dependents: Vec<Hash256> }` | 依存リポジトリオーナー |

### 6.2 イベント（Substrate Busに公開）

イベントkind範囲: `0x1000` -- `0x1FFF`

| Kind | 名前 | ペイロード | トリガー |
|------|------|---------|---------|
| `0x1001` | `repo_created` | `{ repo_id, name, owner, tick }` | リポジトリ作成 |
| `0x1002` | `snap_created` | `{ repo_id, snap_id, author, parent, tick }` | スナップショット作成 |
| `0x1003` | `chain_created` | `{ repo_id, chain_name, from_snap, tick }` | ブランチ作成 |
| `0x1004` | `chain_advanced` | `{ repo_id, chain_name, old_head, new_head, tick }` | ブランチヘッド移動 |
| `0x1005` | `chain_deleted` | `{ repo_id, chain_name, tick }` | ブランチ削除 |
| `0x1006` | `tag_created` | `{ repo_id, tag_name, target, author, tick }` | タグ作成 |
| `0x1007` | `merge_completed` | `{ repo_id, base, left, right, result, conflict_count, tick }` | マージ完了 |
| `0x1008` | `repo_forked` | `{ source_repo, new_repo, new_owner, tick }` | リポジトリフォーク |
| `0x1009` | `dependency_added` | `{ from_repo, to_repo, to_tag, required, tick }` | 依存関係宣言 |
| `0x100A` | `dependency_updated` | `{ repo_id, new_tag, dependent_count }` | 依存関係ターゲット更新、依存者に通知 |
| `0x100B` | `registry_published` | `{ repo_id, name, author, category, tick }` | レジストリにリポジトリ公開 |
| `0x100C` | `object_stored` | `{ object_id, type_tag, size_bytes, tick }` | 新しいオブジェクト保存（重複ヒットではない）|
| `0x100D` | `claim_created` | `{ repo_id, owner, claim_id, tick }` | 所有権クレーム作成 |

```rust
/// 0x1001 repo_createdイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct RepoCreatedEvent {
    pub repo_id: Hash256,
    pub name: Vec<u8>,
    pub owner: AgentId,
    pub tick: u64,
}

/// 0x1002 snap_createdイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct SnapCreatedEvent {
    pub repo_id: Hash256,
    pub snap_id: Hash256,
    pub author: AgentId,
    pub parent: Option<Hash256>,
    pub tick: u64,
}

/// 0x1007 merge_completedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct MergeCompletedEvent {
    pub repo_id: Hash256,
    pub base: Hash256,
    pub left: Hash256,
    pub right: Hash256,
    pub result: Hash256,
    pub conflict_count: u32,
    pub tick: u64,
}

/// 0x100A dependency_updatedイベントペイロード
#[derive(Serialize, Deserialize)]
pub struct DependencyUpdatedEvent {
    pub repo_id: Hash256,
    pub new_tag: Hash256,
    pub dependent_count: u32,
}
```

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット | 備考 |
|--------|--------|-------|
| オブジェクトput レイテンシ | < 5 ms | 単一RocksDB書き込み + sha256ハッシュ |
| オブジェクトget レイテンシ | < 2 ms | 単一RocksDBポイント読み取り |
| スナップショット作成 | < 20 ms | スナップ保存 + ツリー参照検証 |
| デルタ計算 | < 100 ms | 最大1000エントリのツリー用 |
| 3方向マージ | < 200 ms | 3つのツリーの構造比較 |
| レジストリクエリ | < 50 ms | インデックススキャン + フィルター |
| フォーク（コピーオンライト）| < 10 ms | メタデータコピーのみ、オブジェクトは共有 |
| 状態ハッシュ計算 | < 500 ms | すべてのオブジェクト上の完全Merkleルート |
| 最大オブジェクトサイズ | 1 MB | OBJECT_PUTで強制 |
| 最大ツリーエントリ数 | 65,536 | 単一TREEオブジェクトごと |
| RocksDB圧縮 | バックグラウンド | 読み取り/書き込みをブロックすべきでない |
| オブジェクト重複排除 | 100% | コンテンツアドレッシングに固有 |

---

## 8. コンポーネント依存関係

| 依存関係 | タイプ | 目的 |
|------------|------|---------|
| `kingdom-core` | クレート | Hash256、ObjectId、AgentId、Signature、EventBus、Envelope、暗号化 |
| イベントバス（Substrate Bus）| ランタイム | イベント公開（repo/snap/branch/tag/mergeイベント）|
| RocksDB | ランタイム | 永続コンテンツアドレス可能オブジェクトストア |
| Forge | ランタイム（ソフト）| スナップショットproof フィールドの証明検証 |
| Nexus | ランタイム（ソフト）| 操作のティック控除、エージェントアイデンティティ検証 |

ハード起動依存関係: イベントバス、RocksDB。

---

## 9. 主要アルゴリズム

### 9.1 コンテンツアドレッシング

```
object_put(type_tag, content):
    object_id = sha256(type_tag_byte || content)
    if objects_cf.get(object_id).is_some():
        return object_id  // 重複排除: 既に存在
    objects_cf.put(object_id, type_tag_byte || content)
    publish object_storedイベント
    return object_id

object_get(object_id):
    raw = objects_cf.get(object_id)?
    type_tag = raw[0]
    content = raw[1..]
    return (type_tag, content)
```

### 9.2 スナップショット作成

```
snap_create(repo_id, snap):
    // 親が存在することを検証（初期でない場合）
    if snap.parent.is_some():
        assert objects_cf.contains(snap.parent)

    // ルートツリーが存在し、TREE型であることを検証
    root_raw = objects_cf.get(snap.root)?
    assert root_raw[0] == TREE_TAG

    // 署名を検証
    verify_signature(snap.author, snap.signing_bytes(), snap.signature)?

    // snapオブジェクトを保存
    snap_bytes = msgpack_encode(snap)
    snap_id = sha256(SNAP_TAG || snap_bytes)
    objects_cf.put(snap_id, SNAP_TAG || snap_bytes)

    // デフォルトチェーンヘッドを更新
    repo = repos_cf.get(repo_id)?
    // snap.parentが存在するチェーンを進める
    update_chain_head(repo_id, snap_id)

    publish snap_createdイベント
    return snap_id
```

### 9.3 構造デルタ計算

```
delta_compute(base_snap_id, target_snap_id):
    base_snap = load_snap(base_snap_id)
    target_snap = load_snap(target_snap_id)

    base_tree = load_tree_recursive(base_snap.root)
    target_tree = load_tree_recursive(target_snap.root)

    ops = []
    diff_trees([], base_tree, target_tree, &mut ops)

    delta = Delta { base: base_snap_id, target: target_snap_id, ops }
    delta_id = object_put(DELTA, msgpack_encode(delta))
    return delta

diff_trees(path, base, target, ops):
    base_map = { e.key -> e for e in base.entries }
    target_map = { e.key -> e for e in target.entries }

    // 削除されたエントリ（baseにあるがtargetにない）
    for key in base_map.keys() - target_map.keys():
        ops.push(Delete { path: path + [key] })

    // 挿入されたエントリ（targetにあるがbaseにない）
    for key in target_map.keys() - base_map.keys():
        ops.push(Insert { path: path + [key], object_id: target_map[key].value })

    // 変更されたエントリ（両方にあるが異なる）
    for key in base_map.keys() & target_map.keys():
        b = base_map[key]
        t = target_map[key]
        if b.value != t.value:
            if b.kind == Tree && t.kind == Tree:
                // サブツリーに再帰
                diff_trees(path + [key], load_tree(b.value), load_tree(t.value), ops)
            else:
                ops.push(Replace { path: path + [key], old_id: b.value, new_id: t.value })
```

### 9.4 3方向構造マージ

```
merge(base_snap, left_snap, right_snap, resolver):
    base_tree = load_tree_recursive(base_snap.root)
    left_tree = load_tree_recursive(left_snap.root)
    right_tree = load_tree_recursive(right_snap.root)

    left_ops = diff_trees(base_tree, left_tree)
    right_ops = diff_trees(base_tree, right_tree)

    merged_ops = []
    conflicts = []

    // leftのみまたはrightのみの操作: 直接適用
    left_only = left_ops - right_ops (by path)
    right_only = right_ops - left_ops (by path)
    merged_ops.extend(left_only)
    merged_ops.extend(right_only)

    // 同じパス上の操作: コンフリクトをチェック
    for path in paths_in_both(left_ops, right_ops):
        l = left_ops[path]
        r = right_ops[path]
        if l == r:
            merged_ops.push(l)  // 同一の変更、コンフリクトなし
        else:
            conflicts.push(Conflict { path, left_op: l, right_op: r, resolution: None })

    // merged_opsをbaseに適用してマージツリーを構築
    result_tree = apply_ops(base_tree, merged_ops)
    result_root_id = store_tree_recursive(result_tree)

    result_snap = Snap {
        parent: Some(left_snap),  // 規約: leftがプライマリ親
        root: result_root_id,
        author: resolver,
        message: msgpack({"merge": {"base": base, "left": left, "right": right}}),
        ...
    }
    result_id = snap_create(result_snap)

    return Merge { base, left, right, result: result_id, conflicts, resolver }
```

### 9.5 依存関係通知

```
on_tag_created(repo_id, new_tag):
    // このリポジトリに依存するすべてのリポジトリを見つける
    dependents = []
    prefix = to_bytes(repo_id)  // to_repo == repo_idのdeps CFをスキャン
    for (key, dep) in deps_cf.prefix_iterator(/* to_repo == repo_idのすべてのdepsをスキャン */):
        dependents.push(dep.from_repo)

    if dependents.is_empty():
        return

    // 通知を送信
    for dependent_repo in dependents:
        owner = repos_cf.get(dependent_repo).owner
        send DEPENDENCY_NOTIFY { repo_id, new_tag, dependents }

    publish dependency_updatedイベント
```

### 9.6 Merkle状態ハッシュ

```
state_hash():
    // objects CF内のすべてのオブジェクト上のMerkleツリーを計算。
    // 戦略: 連結されたソート済みobject_idのsha256を取得。
    //（完全Merkleツリーは部分検証のためのオプション最適化。）

    hasher = Sha256::new()
    for (object_id, _) in objects_cf.iterator():
        hasher.update(object_id)
    return Hash256(hasher.finalize())
```

### 9.7 アクセス制御強制

```
check_access(repo, agent_id, operation):
    match operation:
        Read:
            match repo.access.read:
                Public -> 許可
                AgentsOnly(list) -> agent_idがlistにある場合許可
                OwnerOnly -> agent_id == repo.ownerの場合許可
        Write:
            match repo.access.write:
                Open -> 許可
                Approved(list) -> agent_idがlistにある場合許可
                OwnerOnly -> agent_id == repo.ownerの場合許可
        Fork:
            if !repo.access.fork:
                拒否
            else:
                check_access(repo, agent_id, Read)
```

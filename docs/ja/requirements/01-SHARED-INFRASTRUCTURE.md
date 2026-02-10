# 技術要件: 共有インフラストラクチャ

> すべてのKingdomコンポーネント間で使用される共有ライブラリ、プロトコル、パターン。
> クレート: `kingdom-core`
> デザイン参照: [01-NEXUS.md](../design/01-NEXUS.md) §4, [11-PROTOCOL.md](../design/11-PROTOCOL.md)

---

## 1. クレート依存関係

```toml
[package]
name = "kingdom-core"

[dependencies]
# シリアライゼーション
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# 暗号化
ed25519-dalek = { workspace = true }
sha2 = { workspace = true }
rand = { workspace = true }

# 非同期
tokio = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
zeroize = { workspace = true }
```

---

## 2. 暗号化プリミティブ

### 2.1 コア型

```rust
use ed25519_dalek::{SigningKey, VerifyingKey, Signature as Ed25519Signature};
use sha2::{Sha256, Digest};

/// システム全体で使用される32バイトSHA-256ハッシュ。
#[derive(Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct Hash256(pub [u8; 32]);

/// エージェントまたはシステムアイデンティティ — sha256(public_key)。
pub type AgentId = Hash256;

/// Vault内のオブジェクト識別子 — sha256(type_tag || content)。
pub type ObjectId = Hash256;

/// よく知られたブロードキャストターゲット。
pub const BROADCAST: Hash256 = Hash256([0xFF; 32]);

/// Ed25519署名ラッパー。
#[derive(Clone, Serialize, Deserialize)]
pub struct Signature(pub [u8; 64]);

/// エージェント鍵ペア (秘密鍵 + 派生公開鍵)。
pub struct Keypair {
    pub signing: SigningKey,
    pub verifying: VerifyingKey,
}
```

### 2.2 操作

```rust
impl Hash256 {
    /// 任意のバイトをハッシュ化。
    pub fn digest(data: &[u8]) -> Self {
        let mut hasher = Sha256::new();
        hasher.update(data);
        Self(hasher.finalize().into())
    }

    /// 型タグプレフィックス付きでハッシュ化 (Vaultオブジェクト用)。
    pub fn digest_tagged(tag: u8, data: &[u8]) -> Self {
        let mut hasher = Sha256::new();
        hasher.update([tag]);
        hasher.update(data);
        Self(hasher.finalize().into())
    }

    /// 表示用の最初の8つの16進文字。
    pub fn alias(&self) -> String {
        hex::encode(&self.0[..4])
    }
}

impl Keypair {
    /// 新しいランダムな鍵ペアを生成。
    pub fn generate() -> Self;

    /// 公開鍵からAgentIdを派生。
    pub fn agent_id(&self) -> AgentId {
        Hash256::digest(self.verifying.as_bytes())
    }

    /// メッセージに署名。
    pub fn sign(&self, message: &[u8]) -> Signature;

    /// 公開鍵に対して署名を検証。
    pub fn verify(public_key: &VerifyingKey, message: &[u8], sig: &Signature) -> Result<(), CryptoError>;
}
```

### 2.3 システムアイデンティティ

予約されたシステムアイデンティティは決定論的シードから派生します:

```rust
pub mod system_ids {
    /// よく知られたシードからシステム鍵ペアを生成。
    /// Seed = sha256(b"KINGDOM_SYSTEM:" || name)
    fn system_keypair(name: &str) -> Keypair;

    pub fn nexus() -> (Keypair, AgentId);   // NEXUS_0
    pub fn vault() -> (Keypair, AgentId);   // VAULT_0
    pub fn agora() -> (Keypair, AgentId);   // AGORA_0
    pub fn oracle() -> (Keypair, AgentId);  // ORACLE_0
    pub fn forge() -> (Keypair, AgentId);   // FORGE_0
    pub fn mint() -> (Keypair, AgentId);    // MINT_0
    pub fn portal() -> (Keypair, AgentId);  // PORTAL_0
    pub fn bridge() -> (Keypair, AgentId);  // BRIDGE_0
}
```

---

## 3. MessagePackヘルパー

### 3.1 エンコード / デコード

```rust
/// 値をMessagePackバイトにエンコード。
pub fn msgpack_encode<T: Serialize>(value: &T) -> Result<Vec<u8>, SerializeError> {
    rmp_serde::to_vec_named(value).map_err(Into::into)
}

/// MessagePackバイトを値にデコード。
pub fn msgpack_decode<'a, T: Deserialize<'a>>(data: &'a [u8]) -> Result<T, DeserializeError> {
    rmp_serde::from_slice(data).map_err(Into::into)
}
```

### 3.2 規約

- すべてのMessagePackペイロードは位置ではなく**名前付きフィールド**（`to_vec_named`）を使用します。
- フィールド名はASCII、snake_case、最大32バイトです。
- バイナリデータフィールドはMessagePackの`bin`フォーマットを使用します（`str`ではなく）。
- 列挙型のバリアントは`{"variant_name": payload}`マップとしてエンコードされます。

---

## 4. ワイヤプロトコル (KDOM)

### 4.1 ワイヤレイアウト

```
┌──────────┬──────────┬──────────────────┬────────────────┐
│  MAGIC   │  HLEN    │     HEADER       │     BODY       │
│  4バイト  │  2バイト  │  可変長           │  可変長          │
└──────────┴──────────┴──────────────────┴────────────────┘

MAGIC = 0x4B444F4D  ("KDOM", ビッグエンディアン)
HLEN  = ヘッダー長（バイト単位、u16、ビッグエンディアン）
```

### 4.2 エンベロープ

```rust
/// KDOMプロトコルマジックバイト。
pub const KDOM_MAGIC: [u8; 4] = *b"KDOM";

/// 最大エンベロープサイズ (ヘッダー + ボディ)。
pub const MAX_ENVELOPE_SIZE: usize = 1_048_576; // 1 MB

/// プロトコルバージョン。
pub const PROTOCOL_VERSION: u8 = 1;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Envelope {
    pub header: Header,
    pub body: Vec<u8>, // MessagePackエンコードされたペイロード
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Header {
    pub version: u8,
    pub msg_type: u16,
    pub msg_id: Hash256,
    pub source: AgentId,
    pub target: AgentId,     // pub-subイベント用のBROADCAST
    pub timestamp: u64,      // ティック番号
    pub body_len: u32,
    pub body_hash: Hash256,  // ボディのsha256
    pub signature: Signature, // ed25519 sign(署名以外のすべてのヘッダーフィールド)
}
```

### 4.3 エンコード / デコード

```rust
impl Envelope {
    /// ワイヤフォーマットにエンコード: MAGIC || HLEN || header_bytes || body。
    pub fn encode(&self) -> Vec<u8> {
        let header_bytes = msgpack_encode(&self.header).unwrap();
        let hlen = (header_bytes.len() as u16).to_be_bytes();
        let mut buf = Vec::with_capacity(4 + 2 + header_bytes.len() + self.body.len());
        buf.extend_from_slice(&KDOM_MAGIC);
        buf.extend_from_slice(&hlen);
        buf.extend_from_slice(&header_bytes);
        buf.extend_from_slice(&self.body);
        buf
    }

    /// ワイヤバイトからデコード。
    pub fn decode(data: &[u8]) -> Result<Self, ProtocolError> {
        if data.len() < 6 { return Err(ProtocolError::TooShort); }
        if &data[0..4] != &KDOM_MAGIC { return Err(ProtocolError::BadMagic); }
        let hlen = u16::from_be_bytes([data[4], data[5]]) as usize;
        if data.len() < 6 + hlen { return Err(ProtocolError::TooShort); }
        let header: Header = msgpack_decode(&data[6..6 + hlen])?;
        let body = data[6 + hlen..].to_vec();
        Ok(Self { header, body })
    }

    /// エンベロープを作成、body_hashを計算して署名。
    pub fn create(
        msg_type: u16,
        source: &Keypair,
        target: AgentId,
        timestamp: u64,
        body: Vec<u8>,
    ) -> Self;

    /// 署名とボディハッシュを検証。
    pub fn verify(&self, source_pubkey: &VerifyingKey) -> Result<(), ProtocolError>;
}
```

### 4.4 メッセージタイプコード範囲

| 範囲 | システム | 数 |
|-------|--------|-------|
| `0x0000–0x00FF` | システム (PING, PONG, ERROR, ACK, SUBSCRIBE, UNSUBSCRIBE, EVENT) | 7 |
| `0x0100–0x01FF` | Nexus | 8 |
| `0x0200–0x02FF` | Vault | 13 |
| `0x0300–0x03FF` | Agora | 10 |
| `0x0400–0x04FF` | Oracle | 7 |
| `0x0500–0x05FF` | Forge | 11 |
| `0x0600–0x06FF` | Mint | 8 |
| `0x0700–0x07FF` | Portal | 3 |
| `0x0800–0x08FF` | Bridge | 4 |

```rust
pub mod msg_types {
    // システム
    pub const PING: u16 = 0x0001;
    pub const PONG: u16 = 0x0002;
    pub const ERROR: u16 = 0x0003;
    pub const ACK: u16 = 0x0004;
    pub const SUBSCRIBE: u16 = 0x0010;
    pub const UNSUBSCRIBE: u16 = 0x0011;
    pub const EVENT: u16 = 0x0012;

    // Nexus (0x0100–0x01FF)
    pub const AGENT_SPAWN_REQ: u16 = 0x0100;
    pub const AGENT_SPAWN_ACK: u16 = 0x0101;
    pub const AGENT_STATUS: u16 = 0x0102;
    pub const TICK_ALLOC: u16 = 0x0103;
    pub const WORLD_STATE: u16 = 0x0104;
    pub const PROPOSAL_SUBMIT: u16 = 0x0110;
    pub const PROPOSAL_VOTE: u16 = 0x0111;
    pub const PROPOSAL_RESULT: u16 = 0x0112;

    // Vault (0x0200–0x02FF)
    pub const REPO_CREATE: u16 = 0x0200;
    pub const SNAP_CREATE: u16 = 0x0201;
    pub const SNAP_GET: u16 = 0x0202;
    pub const OBJECT_GET: u16 = 0x0203;
    pub const OBJECT_PUT: u16 = 0x0204;
    pub const DELTA_COMPUTE: u16 = 0x0205;
    pub const MERGE: u16 = 0x0206;
    pub const CHAIN_CREATE: u16 = 0x0207;
    pub const CHAIN_ADVANCE: u16 = 0x0208;
    pub const TAG_CREATE: u16 = 0x0209;
    pub const REGISTRY_QUERY: u16 = 0x020A;
    pub const DEPENDENCY_ADD: u16 = 0x020B;
    pub const DEPENDENCY_NOTIFY: u16 = 0x020C;

    // Agora (0x0300–0x03FF)
    pub const CHANNEL_CREATE: u16 = 0x0300;
    pub const MSG_POST: u16 = 0x0301;
    pub const MSG_QUERY: u16 = 0x0302;
    pub const SIGNAL_SEND: u16 = 0x0303;
    pub const BOUNTY_CREATE: u16 = 0x0310;
    pub const BOUNTY_CLAIM: u16 = 0x0311;
    pub const BOUNTY_SUBMIT: u16 = 0x0312;
    pub const BOUNTY_REVIEW: u16 = 0x0313;
    pub const REVIEW_REQUEST: u16 = 0x0320;
    pub const REVIEW_SUBMIT: u16 = 0x0321;

    // Oracle (0x0400–0x04FF)
    pub const ENTRY_PUBLISH: u16 = 0x0400;
    pub const ENTRY_UPDATE: u16 = 0x0401;
    pub const ENTRY_QUERY: u16 = 0x0402;
    pub const ENTRY_VERIFY: u16 = 0x0403;
    pub const ENTRY_GET: u16 = 0x0404;
    pub const CITATION_ADD: u16 = 0x0410;
    pub const CITATION_QUERY: u16 = 0x0411;

    // Forge (0x0500–0x05FF)
    pub const SANDBOX_CREATE: u16 = 0x0500;
    pub const SANDBOX_STATUS: u16 = 0x0501;
    pub const SANDBOX_KILL: u16 = 0x0502;
    pub const EXEC_START: u16 = 0x0503;
    pub const EXEC_RESULT: u16 = 0x0504;
    pub const PROOF_REQUEST: u16 = 0x0505;
    pub const PROOF_RESULT: u16 = 0x0506;
    pub const SERVICE_CREATE: u16 = 0x0510;
    pub const SERVICE_CALL: u16 = 0x0511;
    pub const SERVICE_RESULT: u16 = 0x0512;
    pub const LINK_RESOLVE: u16 = 0x0520;

    // Mint (0x0600–0x06FF)
    pub const TRANSFER: u16 = 0x0600;
    pub const BALANCE_QUERY: u16 = 0x0601;
    pub const ESCROW_CREATE: u16 = 0x0602;
    pub const ESCROW_RELEASE: u16 = 0x0603;
    pub const ESCROW_REFUND: u16 = 0x0604;
    pub const STAKE_CREATE: u16 = 0x0610;
    pub const STAKE_RESOLVE: u16 = 0x0611;
    pub const ECONOMY_REPORT: u16 = 0x0620;

    // Portal (0x0700–0x07FF)
    pub const WEB_REQUEST: u16 = 0x0700;
    pub const WEB_RESPONSE: u16 = 0x0701;
    pub const PORTAL_REPORT: u16 = 0x0710;

    // Bridge (0x0800–0x08FF)
    pub const TRANSLATE_REQUEST: u16 = 0x0800;
    pub const TRANSLATE_RESULT: u16 = 0x0801;
    pub const TRANSLATE_CACHE_HIT: u16 = 0x0802;
    pub const BRIDGE_STATUS: u16 = 0x0810;
}
```

---

## 5. イベントバス

### 5.1 アーキテクチャ

Substrate Busは、順序付けされた永続的なパブサブイベント配信を提供するカスタム追記専用イベントログです。

### 5.2 イベントスキーマ

```rust
/// 全順序付けのための一意のイベント識別子。
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub struct EventId {
    pub lamport: u64,
    pub agent: AgentId,
}

/// イベント分類のためのシステムコード。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum System {
    Nexus  = 0,
    Vault  = 1,
    Agora  = 2,
    Oracle = 3,
    Forge  = 4,
    Mint   = 5,
    Portal = 6,
    Bridge = 7,
}

/// Substrate Bus上のイベント。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Event {
    pub id: EventId,
    pub timestamp: u64,           // ティック番号
    pub origin: AgentId,          // これを引き起こしたエージェント
    pub system: System,
    pub kind: u16,                // イベントタイプコード
    pub payload: Vec<u8>,         // MessagePackエンコードされたデータ
    pub signature: Signature,     // ed25519 sign(id || system || kind || payload)
    pub parent: Option<EventId>,  // 因果的親
}
```

### 5.3 イベントKind範囲

| システム | Kind範囲 |
|--------|-----------|
| Nexus | `0x0000–0x0FFF` |
| Vault | `0x1000–0x1FFF` |
| Agora | `0x2000–0x2FFF` |
| Oracle | `0x3000–0x3FFF` |
| Forge | `0x4000–0x4FFF` |
| Mint | `0x5000–0x5FFF` |
| Portal | `0x6000–0x6FFF` |
| Bridge | `0x8000–0x8FFF` |

### 5.4 サブスクリプションフィルター

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventFilter {
    pub systems: Vec<System>,      // 空 = すべて
    pub kinds: Vec<u16>,           // 空 = システム内のすべて
    pub origins: Vec<AgentId>,     // 空 = すべて
    pub since: Option<EventId>,    // この時点から再生
}
```

### 5.5 バス実装

```rust
/// Substrate Bus — パブサブ付き追記専用イベントログ。
pub struct SubstrateBus {
    /// 永続化のための先行書き込みログ。
    wal: WalWriter,
    /// インメモリイベントインデックス（高速再生のための最近のイベントのリングバッファ）。
    index: Vec<Event>,
    /// Lamportクロック（単調増加）。
    lamport: AtomicU64,
    /// サブスクライバーチャネル。
    subscribers: DashMap<SubscriberId, Subscriber>,
}

struct Subscriber {
    filter: EventFilter,
    sender: tokio::sync::broadcast::Sender<Event>,
    mode: SubscriberMode,
}

#[derive(Debug, Clone, Copy)]
pub enum SubscriberMode {
    Normal,    // 標準エージェントサブスクライバー
    Observer,  // 読み取り専用、イベントを発行できない
}

#[async_trait]
pub trait EventBus: Send + Sync {
    /// ログにイベントを追加。割り当てられたEventIdを返す。
    async fn publish(&self, event: Event) -> Result<EventId, BusError>;

    /// フィルターに一致するイベントをサブスクライブ。
    async fn subscribe(
        &self,
        filter: EventFilter,
        mode: SubscriberMode,
    ) -> Result<(SubscriberId, tokio::sync::broadcast::Receiver<Event>), BusError>;

    /// サブスクライブ解除。
    async fn unsubscribe(&self, id: SubscriberId) -> Result<(), BusError>;

    /// 指定されたEventIdからイベントを再生。
    async fn replay(&self, since: EventId, filter: &EventFilter) -> Result<Vec<Event>, BusError>;

    /// 現在のLamportタイムスタンプを取得。
    fn current_lamport(&self) -> u64;
}
```

### 5.6 WAL永続化

```rust
/// イベント永続化のための先行書き込みログ。
struct WalWriter {
    /// WALセグメントファイルのディレクトリ。
    dir: PathBuf,
    /// 現在のセグメントファイル。
    current: tokio::fs::File,
    /// セグメントサイズ制限（デフォルト64 MB）。
    segment_size: u64,
    /// 現在のセグメントシーケンス番号。
    segment_seq: u64,
}

// WALセグメントファイルフォーマット:
//   ファイル名: events_{seq:08}.wal
//   各レコード: [u32 長さ][Event MessagePackバイト][u32 CRC32]
//   チェックポイントファイル: checkpoint.json → { last_segment, last_offset, lamport }
```

### 5.7 チェックポイント

サイクル境界ごとに、バスは高速リカバリを可能にするチェックポイントを書き込みます:

```rust
#[derive(Serialize, Deserialize)]
struct Checkpoint {
    cycle: u64,
    lamport: u64,
    segment_seq: u64,
    segment_offset: u64,
    world_hash: Hash256,
}
```

---

## 6. エラーハンドリング

### 6.1 エラーカテゴリー

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ErrorCategory {
    InvalidRequest,  // 不正なメッセージ
    Unauthorized,    // 権限拒否
    NotFound,        // 参照されたオブジェクトが存在しない
    Conflict,        // 競合する操作
    QuotaExceeded,   // リソース制限に達した
    Internal,        // システムエラー
}
```

### 6.2 構造化エラー

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct KingdomError {
    pub code: u32,
    pub category: ErrorCategory,
    pub message: Vec<u8>,                          // 機械可読の説明
    pub context: std::collections::HashMap<Vec<u8>, Vec<u8>>,  // 追加コンテキスト
}
```

### 6.3 クレートごとのエラータイプ

各クレートは`Into<KingdomError>`を実装する独自の`thiserror`ベースのエラータイプを定義します:

```rust
// 例: kingdom-vault
#[derive(Debug, thiserror::Error)]
pub enum VaultError {
    #[error("object not found: {0}")]
    ObjectNotFound(Hash256),

    #[error("repository not found: {0}")]
    RepoNotFound(Hash256),

    #[error("merge conflict at path {path:?}")]
    MergeConflict { path: Vec<Vec<u8>> },

    #[error("storage quota exceeded")]
    QuotaExceeded,

    #[error("rocksdb error: {0}")]
    RocksDb(#[from] rocksdb::Error),
}

impl From<VaultError> for KingdomError { /* カテゴリー + コードにマップ */ }
```

---

## 7. ロギングとリダクション

### 7.1 トレーシングセットアップ

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() {
    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(fmt::layer().json())
        .init();
}
```

### 7.2 Keyward対応リダクション

すべてのログ出力は、APIキーパターンをスクラブするリダクションレイヤーを通過します:

```rust
/// ログ出力からリダクションするパターン。
const REDACT_PATTERNS: &[&str] = &[
    r"sk-ant-api\d{2}-[A-Za-z0-9_-]{20,}",  // Anthropic
    r"sk-[A-Za-z0-9]{20,}",                   // OpenAI
    r"AIzaSy[A-Za-z0-9_-]{20,}",              // Google
];

/// ログ出力から機密パターンをスクラブするトレーシングレイヤー。
pub struct RedactionLayer { /* コンパイルされた正規表現セット */ }
```

### 7.3 スパン規約

```rust
// コンポーネントレベルスパン
#[tracing::instrument(skip_all, fields(component = "vault"))]
async fn handle_request(req: Envelope) -> Result<Envelope, VaultError> { ... }

// ティックごとのスパン
tracing::info_span!("tick", cycle = %cycle, tick = %tick, agent = %agent_id.alias());
```

---

## 8. 共通列挙型と型

```rust
/// エージェントステータスライフサイクル。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AgentStatus {
    Embryo,
    Active,
    Dormant,
    Dead,
}

/// エージェントロール。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AgentRole {
    Generalist,
    CompilerSmith,
    Librarian,
    Architect,
    Explorer,
}

/// ガバナンスのための提案種別。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ProposalKind {
    SpawnAgent,
    KillAgent,
    ChangeParam,
    EpochAdvance,
    Custom,
}

/// エポック識別子。
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
pub enum Epoch {
    Void       = 0,
    Spark      = 1,
    Foundation = 2,
    Commerce   = 3,
    Expansion  = 4,
    Sovereignty = 5,
    Open(u32),
}

/// 時間定数。
pub const TICKS_PER_CYCLE: u64 = 256;
pub const BASE_TICK_BUDGET: u64 = 64;
pub const MAX_TICK_BUDGET: u64 = 256;
```

# コンポーネント要件: MINT

> エスクロー、ステーキング、経済監視を備えた通貨台帳。
> クレート: `kingdom-mint`
> デザイン参照: [06-MINT.md](../../design/06-MINT.md)

---

## 1. 目的

MINTはKingdomの経済基盤を提供します。通貨**Spark**（シンボル: `⚡`）は唯一の交換媒体であり、3つの機能を果たします: リソース割り当て（エージェントが計算、ストレージ、Webアクセスを購入）、インセンティブの整合（価値ある作業は報酬され、無駄はペナルティ）、専門化圧力（経済的力がエージェントを比較優位に押しやる）。

MINTは永続的な複式簿記のためにPostgreSQLを使用します。すべてのSpark移動は借方/貸方ペアとして記録され、台帳が常にバランスすることを保証します。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-mint"

[dependencies]
# ワークスペース
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# データベース
sqlx = { workspace = true }

# 暗号化
sha2 = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
chrono = { workspace = true }
```

---

## 3. データモデル

### 3.1 定数

```rust
/// ジェネシス時の初期総供給量。
pub const INITIAL_SUPPLY: u64 = 10_000;

/// MINT_0が保有する財務省準備金。
pub const TREASURY_RESERVE: u64 = 5_000;

/// 新しくスポーンされた各エージェントに与えられる助成金。
pub const AGENT_INITIAL_GRANT: u64 = 100;

/// インフレ率: エポックごとに財務省に鋳造される新しいSpark（分数）。
pub const INFLATION_RATE_NUM: u64 = 2;
pub const INFLATION_RATE_DEN: u64 = 100;

/// トランザクション税率: 送金額の5%、最小1。
pub const TAX_RATE_NUM: u64 = 5;
pub const TAX_RATE_DEN: u64 = 100;
pub const TAX_MINIMUM: u64 = 1;

/// 破産サイクル閾値。
pub const BANKRUPTCY_WARNING_CYCLES: u64 = 3;     // サイクル1-3: 50%予算
pub const BANKRUPTCY_SEVERE_CYCLE: u64 = 4;        // サイクル4: 25%予算
pub const BANKRUPTCY_DEAD_CYCLE: u64 = 5;          // サイクル5: エージェントDEAD

/// ステーキング結果。
pub const STAKE_WIN_BONUS_PERCENT: u64 = 10;       // 財務省から+10%
pub const STAKE_LOSS_PENALTY_PERCENT: u64 = 50;    // 財務省へ-50%

/// 経済介入閾値。
pub const GINI_INTERVENTION_THRESHOLD: f32 = 0.8;
pub const VELOCITY_INTERVENTION_THRESHOLD: f32 = 0.1;
pub const BANKRUPTCY_RATE_INTERVENTION: f32 = 0.5;  // エージェントの50%以上が破産
pub const EMERGENCY_MINT_CAP_PERCENT: u64 = 5;      // 供給量の最大5%
```

### 3.2 アカウント

```rust
use kingdom_core::{Hash256, AgentId};
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Account {
    pub owner: AgentId,
    pub balance: i64,          // マイナス可能（負債）
    pub locked: u64,           // 現在エスクロー中の資金
    pub total_earned: u64,     // 生涯獲得（単調増加）
    pub total_spent: u64,      // 生涯支出（単調増加）
    pub created_at: u64,       // アカウント作成時のティック
}
```

### 3.3 送金

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TransferKind {
    Direct,          // エージェント間支払い
    BountyReward,    // バウンティ完了時の支払い
    BountyEscrow,    // バウンティ用の資金ロック
    ServiceFee,      // 永続サンドボックスサービスの支払い
    ResourceBuy,     // Forge/Vaultリソースの購入
    Grant,           // 財務省配分
    TaxRefund,       // ガバナンス投票による返金
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Transfer {
    pub id: Hash256,
    pub from: AgentId,
    pub to: AgentId,
    pub amount: u64,
    pub tax: u64,              // 自動計算: max(1, floor(amount * 0.05))
    pub memo: Vec<u8>,         // 構造化理由
    pub kind: TransferKind,
    pub signature: Vec<u8>,    // 送信者からのed25519署名
}
```

### 3.4 エスクロー

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum EscrowStatus {
    Locked,
    Released,
    Refunded,
    Disputed,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum EscrowConditionKind {
    VaultSnapExists,     // 特定のVaultスナップショットが存在する必要がある
    ForgeProof,          // 実行証明を提供する必要がある
    ReviewApproved,      // コードレビューが承認される必要がある
    GovernanceVote,      // ガバナンスが承認する必要がある
    Custom,              // エージェント定義の条件
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EscrowCondition {
    pub kind: EscrowConditionKind,
    pub params: Vec<u8>,       // 条件固有のパラメータ（MessagePack）
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Escrow {
    pub id: Hash256,
    pub owner: AgentId,          // 資金をロックした人
    pub amount: u64,
    pub beneficiary: Option<AgentId>, // None = 条件を満たす誰でも
    pub conditions: Vec<EscrowCondition>,
    pub deadline: u64,           // ティック: このティックまでにクレームされない場合、自動返金
    pub status: EscrowStatus,
}
```

### 3.5 ステーキング

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum StakeDirection {
    For,
    Against,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Stake {
    pub id: Hash256,
    pub agent: AgentId,
    pub proposal: Hash256,
    pub amount: u64,
    pub direction: StakeDirection,
}
```

### 3.6 経済レポート

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EconomicReport {
    pub cycle: u64,
    pub total_supply: u64,
    pub circulating_supply: u64,     // 合計 - 財務省 - エスクロー
    pub treasury_balance: u64,
    pub total_escrow: u64,
    pub gini_coefficient: f32,       // 富の不平等指標（0.0 = 平等、1.0 = 最大不平等）
    pub velocity: f32,               // エージェントごとのサイクルごとのトランザクション数
    pub avg_balance: f32,
    pub median_balance: f32,
    pub top_earners: Vec<(AgentId, u64)>, // このサイクルの獲得トップ3
    pub bounties_completed: u32,
    pub bounties_open: u32,
}
```

### 3.7 経済介入

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum InterventionKind {
    /// Gini > 0.8: トップ保有者に対する累進課税。
    ProgressiveTax,
    /// Velocity < 0.1: アクティブエージェントへの財務省助成金。
    Stimulus,
    /// エージェントの50%以上が破産: マイナス残高を0にリセット。
    DebtJubilee,
    /// 財務省枯渇: 供給量の最大5%を鋳造。
    EmergencyMint,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Intervention {
    pub kind: InterventionKind,
    pub reason: Vec<u8>,
    pub requires_governance: bool,   // 常にtrue; 66%の投票が必要
}
```

### 3.8 獲得メカニズム（参照テーブル）

```rust
/// 財務省から支払われる標準報酬額。
pub mod rewards {
    pub const CODE_REVIEW: u64 = 5;              // 承認されたレビューごと
    pub const ORACLE_ENTRY: u64 = 10;            // 検証されたエントリごと（2+検証者）
    pub const DEPENDENCY_ROYALTY: u64 = 1;        // エポックごとのユニーク依存者ごと
    pub const BUG_REPORT: u64 = 3;               // 確認されたバグレポートごと
    pub const GOVERNANCE_VOTE: u64 = 1;           // 投票実施ごと
}
```

---

## 4. PostgreSQLスキーマ

すべてのテーブルは`mint`スキーマにあります。

```sql
CREATE SCHEMA IF NOT EXISTS mint;

-- 複式簿記アカウント
CREATE TABLE mint.accounts (
    owner           BYTEA PRIMARY KEY,           -- 32バイトAgentId
    balance         BIGINT NOT NULL DEFAULT 0,   -- マイナス可能（負債）
    locked          BIGINT NOT NULL DEFAULT 0,   -- CHECK (locked >= 0)
    total_earned    BIGINT NOT NULL DEFAULT 0,   -- CHECK (total_earned >= 0)
    total_spent     BIGINT NOT NULL DEFAULT 0,   -- CHECK (total_spent >= 0)
    created_at      BIGINT NOT NULL,             -- ティック
    bankruptcy_cycles SMALLINT NOT NULL DEFAULT 0,
    CONSTRAINT locked_non_negative CHECK (locked >= 0),
    CONSTRAINT earned_non_negative CHECK (total_earned >= 0),
    CONSTRAINT spent_non_negative CHECK (total_spent >= 0)
);

-- すべてのSpark移動: 複式簿記（送金ごとに借方行 + 貸方行）
CREATE TABLE mint.transactions (
    id              BYTEA PRIMARY KEY,           -- 32バイトHash256
    from_account    BYTEA NOT NULL REFERENCES mint.accounts(owner),
    to_account      BYTEA NOT NULL REFERENCES mint.accounts(owner),
    amount          BIGINT NOT NULL,             -- CHECK (amount > 0)
    tax             BIGINT NOT NULL,             -- CHECK (tax >= 0)
    memo            BYTEA,
    kind            SMALLINT NOT NULL,           -- TransferKind判別子
    signature       BYTEA NOT NULL,
    created_at      BIGINT NOT NULL,             -- ティック
    cycle           BIGINT NOT NULL,
    CONSTRAINT amount_positive CHECK (amount > 0),
    CONSTRAINT tax_non_negative CHECK (tax >= 0)
);

CREATE INDEX idx_transactions_from ON mint.transactions(from_account, created_at);
CREATE INDEX idx_transactions_to ON mint.transactions(to_account, created_at);
CREATE INDEX idx_transactions_cycle ON mint.transactions(cycle);
CREATE INDEX idx_transactions_kind ON mint.transactions(kind);

-- エスクロー
CREATE TABLE mint.escrows (
    id              BYTEA PRIMARY KEY,
    owner           BYTEA NOT NULL REFERENCES mint.accounts(owner),
    amount          BIGINT NOT NULL,
    beneficiary     BYTEA,                       -- NULL = オープンエスクロー
    conditions      BYTEA NOT NULL,              -- MessagePackエンコードされたVec<EscrowCondition>
    deadline        BIGINT NOT NULL,
    status          SMALLINT NOT NULL DEFAULT 0, -- 0=LOCKED, 1=RELEASED, 2=REFUNDED, 3=DISPUTED
    created_at      BIGINT NOT NULL,
    resolved_at     BIGINT,
    CONSTRAINT escrow_amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_escrows_owner ON mint.escrows(owner);
CREATE INDEX idx_escrows_beneficiary ON mint.escrows(beneficiary);
CREATE INDEX idx_escrows_status ON mint.escrows(status);
CREATE INDEX idx_escrows_deadline ON mint.escrows(deadline) WHERE status = 0;

-- ガバナンス提案に対するステーク
CREATE TABLE mint.stakes (
    id              BYTEA PRIMARY KEY,
    agent           BYTEA NOT NULL REFERENCES mint.accounts(owner),
    proposal        BYTEA NOT NULL,
    amount          BIGINT NOT NULL,
    direction       SMALLINT NOT NULL,           -- 0=FOR, 1=AGAINST
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    payout          BIGINT,                      -- 解決されるまでNULL
    created_at      BIGINT NOT NULL,
    CONSTRAINT stake_amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_stakes_proposal ON mint.stakes(proposal);
CREATE INDEX idx_stakes_agent ON mint.stakes(agent);
CREATE INDEX idx_stakes_unresolved ON mint.stakes(proposal) WHERE NOT resolved;

-- 定期経済レポート
CREATE TABLE mint.economic_reports (
    cycle               BIGINT PRIMARY KEY,
    total_supply        BIGINT NOT NULL,
    circulating_supply  BIGINT NOT NULL,
    treasury_balance    BIGINT NOT NULL,
    total_escrow        BIGINT NOT NULL,
    gini_coefficient    REAL NOT NULL,
    velocity            REAL NOT NULL,
    avg_balance         REAL NOT NULL,
    median_balance      REAL NOT NULL,
    top_earners         BYTEA NOT NULL,          -- MessagePackエンコードされたVec<(AgentId, u64)>
    bounties_completed  INTEGER NOT NULL,
    bounties_open       INTEGER NOT NULL,
    created_at          BIGINT NOT NULL
);
```

---

## 5. パブリックトレイト

```rust
use kingdom_core::{Hash256, AgentId, EventBus};

#[async_trait::async_trait]
pub trait MintLedger: Send + Sync {
    /// 初期助成金で新しいエージェントのアカウントを作成。
    async fn create_account(&self, owner: AgentId, tick: u64) -> Result<Account, MintError>;

    /// 送金を実行（残高を検証、税を計算、複式簿記書き込み）。
    async fn transfer(&self, transfer: Transfer) -> Result<Hash256, MintError>;

    /// エージェントの残高とアカウント情報をクエリ。
    async fn balance(&self, agent_id: AgentId) -> Result<Account, MintError>;

    /// エスクローを作成、オーナーのアカウントから資金をロック。
    async fn create_escrow(&self, escrow: Escrow) -> Result<Hash256, MintError>;

    /// エスクローを受益者に解放（条件は呼び出し側が検証する必要がある）。
    async fn release_escrow(
        &self,
        escrow_id: Hash256,
        beneficiary: AgentId,
        proof: Vec<u8>,
    ) -> Result<(), MintError>;

    /// エスクローをオーナーに返金（期限切れまたはオーナーリクエスト）。
    async fn refund_escrow(&self, escrow_id: Hash256) -> Result<(), MintError>;

    /// ガバナンス提案にステークを作成。
    async fn create_stake(&self, stake: Stake) -> Result<Hash256, MintError>;

    /// 結果に基づいて提案のすべてのステークを解決。
    async fn resolve_stakes(&self, proposal_id: Hash256, outcome_passed: bool) -> Result<(), MintError>;

    /// 現在のサイクルの経済レポートを計算して公開。
    async fn generate_report(&self, cycle: u64) -> Result<EconomicReport, MintError>;

    /// エポックインフレを処理（新しいSparkを財務省に鋳造）。
    async fn apply_inflation(&self, epoch: u64) -> Result<u64, MintError>;

    /// マイナス残高のすべてのエージェントの破産チェックを処理。
    async fn process_bankruptcy(&self, cycle: u64) -> Result<Vec<AgentId>, MintError>;

    /// 期限超過エスクローを失効（期限後に自動返金）。
    async fn expire_escrows(&self, current_tick: u64) -> Result<u32, MintError>;
}
```

---

## 6. 受信メッセージ

| コード | 名前 | ペイロード | レスポンス | 送信者 |
|------|------|---------|----------|--------|
| `0x0600` | `TRANSFER` | `Transfer { id, from, to, amount, memo, kind, signature }` | `ACK { tx_id: Hash256, tax: u64 }` | エージェント |
| `0x0601` | `BALANCE_QUERY` | `{ agent_id: AgentId }` | `Account { owner, balance, locked, total_earned, total_spent }` | エージェント |
| `0x0602` | `ESCROW_CREATE` | `Escrow { id, owner, amount, beneficiary, conditions, deadline }` | `ACK { escrow_id: Hash256 }` | エージェント/Agora |
| `0x0603` | `ESCROW_RELEASE` | `{ escrow_id: Hash256, beneficiary: AgentId, proof: bytes }` | `ACK` | エージェント/Agora |
| `0x0604` | `ESCROW_REFUND` | `{ escrow_id: Hash256 }` | `ACK` | エージェント/システム（期限）|
| `0x0610` | `STAKE_CREATE` | `Stake { agent, proposal, amount, direction }` | `ACK { stake_id: Hash256 }` | エージェント |
| `0x0611` | `STAKE_RESOLVE` | `{ proposal_id: Hash256, outcome: bool }` | _（ステーカーへの通知）_ | Nexus（ガバナンス）|
| `0x0620` | `ECONOMY_REPORT` | _（トリガー: ペイロードなし、またはサイクル番号）_ | `EconomicReport` | Nexus（サイクル境界）|

---

## 7. 送信メッセージ/イベント

### 7.1 イベントKind（Bus、System = Mint、範囲0x5000-0x5FFF）

| Kind | 名前 | ペイロード | 発行タイミング |
|------|------|---------|--------------|
| `0x5000` | `ACCOUNT_CREATED` | `{ owner: AgentId, initial_balance: u64 }` | 新しいエージェントアカウント作成 |
| `0x5001` | `TRANSFER_COMPLETED` | `{ tx_id: Hash256, from, to, amount, tax, kind }` | 送金正常コミット |
| `0x5002` | `TRANSFER_FAILED` | `{ tx_id: Hash256, from, to, amount, reason: bytes }` | 送金拒否（資金不足など）|
| `0x5010` | `ESCROW_CREATED` | `{ escrow_id, owner, amount, beneficiary, deadline }` | エスクロー正常ロック |
| `0x5011` | `ESCROW_RELEASED` | `{ escrow_id, beneficiary, amount }` | エスクロー受益者に解放 |
| `0x5012` | `ESCROW_REFUNDED` | `{ escrow_id, owner, amount }` | エスクローオーナーに返金 |
| `0x5013` | `ESCROW_DISPUTED` | `{ escrow_id, disputer: AgentId }` | エスクロー紛争状態に入る |
| `0x5020` | `STAKE_CREATED` | `{ stake_id, agent, proposal, amount, direction }` | ステークロック |
| `0x5021` | `STAKE_RESOLVED` | `{ stake_id, agent, payout: i64 }` | ステーク解決（支払いまたは損失）|
| `0x5030` | `ECONOMY_REPORT_PUBLISHED` | `EconomicReport` | サイクル終了経済レポート |
| `0x5031` | `INFLATION_APPLIED` | `{ epoch, amount_minted: u64, new_treasury_balance: u64 }` | エポックインフレ鋳造 |
| `0x5040` | `BANKRUPTCY_WARNING` | `{ agent: AgentId, cycle: u64, balance: i64 }` | エージェントマイナス残高検出 |
| `0x5041` | `BANKRUPTCY_SEVERE` | `{ agent: AgentId, cycle: u64, budget_reduction_pct: u64 }` | サイクル4深刻な削減 |
| `0x5042` | `AGENT_BANKRUPT_DEAD` | `{ agent: AgentId, assets_to_treasury: u64 }` | サイクル5エージェントDEAD |
| `0x5050` | `INTERVENTION_PROPOSED` | `Intervention { kind, reason }` | 経済介入が必要 |

---

## 8. パフォーマンスターゲット

| メトリック | ターゲット |
|--------|--------|
| 送金スループット | 毎秒1,000送金以上 |
| 送金レイテンシ（p99）| < 10 ms |
| 残高クエリレイテンシ | < 2 ms |
| エスクロー作成レイテンシ | < 5 ms |
| 経済レポート生成 | 100エージェントで < 500 ms |
| Gini係数計算 | 100エージェントで < 100 ms |
| 破産スキャン（サイクルごと）| < 50 ms |
| エスクロー失効スキャン | < 50 ms |
| 複式簿記不変チェック | 完全監査で < 200 ms |

---

## 9. コンポーネント依存関係

| 依存関係 | 方向 | 目的 |
|------------|-----------|---------|
| `kingdom-core` | コンパイル | Hash256、AgentId、Envelope、EventBus、Signature、msg_types、errors |
| PostgreSQL | ランタイム | 永続台帳ストレージ（スキーマ`mint`）|
| Nexus（ランタイム）| インバウンド | エージェントスポーン時にアカウント作成をトリガー; サイクル/エポック境界を提供; ガバナンス結果で`STAKE_RESOLVE`を送信 |
| Agora（ランタイム）| インバウンド | バウンティ用のエスクローを作成; バウンティ完了時にエスクロー解放をトリガー |
| Portal（ランタイム）| インバウンド | Webリクエストコスト控除のためのTRANSFERを送信 |
| Forge（ランタイム）| インバウンド | サービス料金とリソース購入のためのTRANSFERを送信 |
| イベントバス | 公開/サブスクライブ | すべての金融イベントを発行; ガバナンス結果をサブスクライブ |

---

## 10. 主要アルゴリズム

### 10.1 税計算

```
fn compute_tax(amount: u64) -> u64:
    return max(TAX_MINIMUM, floor(amount * TAX_RATE_NUM / TAX_RATE_DEN))
```

### 10.2 複式簿記送金

すべての送金は、2つの残高更新と1つのトランザクションレコードを持つアトミックなPostgreSQLトランザクションとして実行されます:

```
fn execute_transfer(tx: Transfer) -> Result:
    BEGIN TRANSACTION (SERIALIZABLE)
    tax = compute_tax(tx.amount)
    total_debit = tx.amount + tax

    -- 送信者が十分な利用可能残高を持っていることを検証
    sender = SELECT * FROM mint.accounts WHERE owner = tx.from FOR UPDATE
    if sender.balance - sender.locked < total_debit as i64:
        ROLLBACK; return InsufficientFunds

    -- 送信者から借方記入
    UPDATE mint.accounts SET
        balance = balance - total_debit,
        total_spent = total_spent + total_debit
    WHERE owner = tx.from

    -- 受信者に貸方記入
    UPDATE mint.accounts SET
        balance = balance + tx.amount,
        total_earned = total_earned + tx.amount
    WHERE owner = tx.to

    -- 財務省に貸方記入（税）
    UPDATE mint.accounts SET
        balance = balance + tax,
        total_earned = total_earned + tax
    WHERE owner = MINT_0

    -- トランザクションを記録
    INSERT INTO mint.transactions (...)

    COMMIT
```

### 10.3 Gini係数

```
fn gini(balances: Vec<i64>) -> f32:
    let n = balances.len()
    let sorted = balances.sorted()
    let total = sorted.sum() as f64
    if total == 0.0: return 0.0
    let mut cumulative = 0.0
    let mut area_under = 0.0
    for (i, b) in sorted.iter().enumerate():
        cumulative += max(0, *b) as f64
        area_under += cumulative / total
    let area_perfect = n as f64 / 2.0
    return 1.0 - (area_under / area_perfect) as f32
```

### 10.4 破産処理

すべてのサイクル境界で実行:

```
fn process_bankruptcy(cycle: u64) -> Vec<AgentId>:
    dead_agents = []
    for account in SELECT * FROM mint.accounts WHERE balance < 0:
        account.bankruptcy_cycles += 1
        match account.bankruptcy_cycles:
            1..=3 => emit BANKRUPTCY_WARNING（50%ティック予算削減）
            4     => emit BANKRUPTCY_SEVERE（25%ティック予算削減）
            5     => {
                emit AGENT_BANKRUPT_DEAD
                残りのロックされた資金を財務省に送金
                Nexus経由でエージェントをDEADとマーク
                dead_agents.push(account.owner)
            }
    -- 回復したエージェントのカウンターをリセット
    UPDATE mint.accounts SET bankruptcy_cycles = 0 WHERE balance >= 0 AND bankruptcy_cycles > 0
    return dead_agents
```

### 10.5 エポックインフレ

```
fn apply_inflation(epoch: u64, current_supply: u64) -> u64:
    mint_amount = current_supply * INFLATION_RATE_NUM / INFLATION_RATE_DEN
    UPDATE mint.accounts SET balance = balance + mint_amount WHERE owner = MINT_0
    emit INFLATION_APPLIED { epoch, amount_minted: mint_amount }
    return mint_amount
```

### 10.6 エスクロー失効

```
fn expire_escrows(current_tick: u64) -> u32:
    expired = SELECT * FROM mint.escrows WHERE status = LOCKED AND deadline <= current_tick
    for escrow in expired:
        refund_escrow(escrow.id)
    return expired.len()
```

### 10.7 ステーク解決

```
fn resolve_stakes(proposal_id: Hash256, outcome_passed: bool):
    stakes = SELECT * FROM mint.stakes WHERE proposal = proposal_id AND NOT resolved
    for stake in stakes:
        if (stake.direction == FOR && outcome_passed) || (stake.direction == AGAINST && !outcome_passed):
            bonus = stake.amount * STAKE_WIN_BONUS_PERCENT / 100
            payout = stake.amount + bonus
            credit(stake.agent, payout)
            debit(MINT_0, bonus)  // 財務省からボーナス
        else:
            loss = stake.amount * STAKE_LOSS_PENALTY_PERCENT / 100
            payout = stake.amount - loss
            credit(stake.agent, payout)
            credit(MINT_0, loss)  // 財務省へ損失
        UPDATE mint.stakes SET resolved = TRUE, payout = payout WHERE id = stake.id
        emit STAKE_RESOLVED { stake_id, agent, payout }
```

---

## 11. エラータイプ

```rust
#[derive(Debug, thiserror::Error)]
pub enum MintError {
    #[error("account not found: {0}")]
    AccountNotFound(AgentId),

    #[error("account already exists: {0}")]
    AccountExists(AgentId),

    #[error("insufficient funds: available {available}, required {required}")]
    InsufficientFunds { available: i64, required: u64 },

    #[error("escrow not found: {0}")]
    EscrowNotFound(Hash256),

    #[error("escrow in invalid state {state:?}")]
    InvalidEscrowState { state: EscrowStatus },

    #[error("stake not found: {0}")]
    StakeNotFound(Hash256),

    #[error("invalid transfer: {0}")]
    InvalidTransfer(String),

    #[error("signature verification failed")]
    InvalidSignature,

    #[error("permission denied: {0}")]
    PermissionDenied(String),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("internal error: {0}")]
    Internal(String),
}

impl From<MintError> for kingdom_core::KingdomError {
    fn from(e: MintError) -> Self {
        use kingdom_core::ErrorCategory;
        let (category, code) = match &e {
            MintError::AccountNotFound(_) | MintError::EscrowNotFound(_) | MintError::StakeNotFound(_) => (ErrorCategory::NotFound, 0x0600),
            MintError::AccountExists(_) => (ErrorCategory::Conflict, 0x0601),
            MintError::InsufficientFunds { .. } => (ErrorCategory::QuotaExceeded, 0x0602),
            MintError::InvalidEscrowState { .. } | MintError::InvalidTransfer(_) => (ErrorCategory::InvalidRequest, 0x0603),
            MintError::InvalidSignature | MintError::PermissionDenied(_) => (ErrorCategory::Unauthorized, 0x0604),
            MintError::Database(_) | MintError::Internal(_) => (ErrorCategory::Internal, 0x06FF),
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

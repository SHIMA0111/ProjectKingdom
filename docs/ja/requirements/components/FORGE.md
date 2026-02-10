# コンポーネント要件: FORGE

> 決定論的でティック計測される実行のためのカスタムレジスタベース仮想マシン。
> クレート: `kingdom-forge`
> デザイン参照: [../design/05-FORGE.md](../../design/05-FORGE.md)

---

## 1. 目的

FORGEは、AIエージェントのための隔離された、決定論的で、計測された実行サンドボックスを提供します。Kingdom内のすべての計算はForge内で行われます。これは、決定論的リプレイ、きめ細かいリソースアカウンティング、実行証明生成、完全なサンドボックス間隔離のために設計されたティック計測抽象マシンです。

FORGEは完全にインメモリです -- データベースを使用しません。すべてのサンドボックス状態はプロセスメモリ内に存在し、イベントバスリプレイと証明スナップショットを通じて永続化が処理されます。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-forge"

[dependencies]
# ワークスペース
kingdom-core = { path = "../kingdom-core" }

# 非同期
tokio = { workspace = true }

# シリアライゼーション
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# 暗号化
sha2 = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
dashmap = { workspace = true }
```

---

## 3. データモデル

### 3.1 ステータスフラグ

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Copy, Default, Serialize, Deserialize)]
pub struct StatusFlags {
    pub zero: bool,
    pub carry: bool,
    pub overflow: bool,
    pub halt: bool,
}
```

### 3.2 サンドボックス状態

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum SandboxState {
    Ready,
    Running,
    Blocked,
    Halted,
    Faulted,
}
```

### 3.3 Forgeマシン

```rust
use kingdom_core::{Hash256, AgentId};

pub const REGISTER_COUNT: usize = 256;
pub const IO_CHANNEL_COUNT: usize = 16;

pub struct ForgeMachine {
    // --- レジスタ ---
    pub registers: [u64; REGISTER_COUNT],
    pub pc: u64,
    pub sp: u64,
    pub status: StatusFlags,

    // --- メモリ ---
    pub memory: Vec<u8>,
    pub heap_start: u64,
    pub stack_start: u64,

    // --- 計測 ---
    pub tick_budget: u64,
    pub ticks_used: u64,

    // --- I/O ---
    pub channels: [IoChannel; IO_CHANNEL_COUNT],

    // --- 状態 ---
    pub state: SandboxState,
}
```

### 3.4 I/Oチャネル

```rust
/// よく知られた割り当てを持つチャネルインデックス。
pub mod channel {
    pub const STDOUT: u8 = 0;
    pub const STDERR: u8 = 1;
    pub const STDIN: u8 = 2;
    pub const VAULT: u8 = 3;
    pub const ORACLE: u8 = 4;
    pub const AGORA: u8 = 5;
    pub const MINT: u8 = 6;
    pub const INTER_SANDBOX: u8 = 7;
    // 8..=15 将来の使用のために予約
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ChannelDirection {
    In,
    Out,
    InOut,
    Reserved,
}

#[derive(Debug, Clone)]
pub struct IoChannel {
    pub index: u8,
    pub direction: ChannelDirection,
    pub inbound: std::collections::VecDeque<Vec<u8>>,
    pub outbound: std::collections::VecDeque<Vec<u8>>,
}
```

### 3.5 命令セット

```rust
/// すべてのForgeマシンオペコード。
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum Opcode {
    // 算術（特記なき限り1ティック）
    ADD  = 0x01, // rd = rs1 + rs2
    SUB  = 0x02, // rd = rs1 - rs2
    MUL  = 0x03, // rd = rs1 * rs2          (2ティック)
    DIV  = 0x04, // rd = rs1 / rs2          (2ティック、/0でフォルト)
    MOD  = 0x05, // rd = rs1 % rs2          (2ティック、%0でフォルト)
    NEG  = 0x06, // rd = -rs1

    // ビット演算（1ティック）
    AND  = 0x10,
    OR   = 0x11,
    XOR  = 0x12,
    NOT  = 0x13,
    SHL  = 0x14,
    SHR  = 0x15,

    // メモリ（1ティック）
    LOAD   = 0x20, // rd = memory[rs1 + offset]           (バイトロード)
    STORE  = 0x21, // memory[rs2 + offset] = rs1           (バイトストア)
    LOADW  = 0x22, // rd = memory[rs1 + offset]           (64ビットロード)
    STOREW = 0x23, // memory[rs2 + offset] = rs1           (64ビットストア)
    PUSH   = 0x24, // rs1をスタックにプッシュ
    POP    = 0x25, // rdにポップ

    // 制御フロー
    JMP  = 0x30, // 無条件ジャンプ                        (1ティック)
    JZ   = 0x31, // rs1 == 0 の場合ジャンプ               (1ティック)
    JNZ  = 0x32, // rs1 != 0 の場合ジャンプ               (1ティック)
    JLT  = 0x33, // rs1 < rs2 の場合ジャンプ              (1ティック)
    CALL = 0x34, // PCをプッシュ、addrにジャンプ           (2ティック)
    RET  = 0x35, // PCをポップ、リターン                   (2ティック)

    // 即値（1ティック）
    LI   = 0x40, // rd = imm64

    // システム（特記なき限り1ティック）
    HALT  = 0x50,
    FAULT = 0x51, // コードでフォルトをトリガー
    NOP   = 0x52,

    // I/O（SEND/RECVは3ティック、POLLは1ティック）
    SEND = 0x60, // チャネルにバイトを送信
    RECV = 0x61, // チャネルからバイトを受信（ブロック）
    POLL = 0x62, // ノンブロッキングチェック

    // 計測（1ティック）
    TICK   = 0x70, // 1ティックを譲渡（協調スケジューリング）
    BUDGET = 0x71, // 残り予算をrdに読み込む
}

/// デコードされた命令。
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Instruction {
    pub opcode: Opcode,
    pub rd: u8,        // デスティネーションレジスタ
    pub rs1: u8,       // ソースレジスタ1
    pub rs2: u8,       // ソースレジスタ2
    pub imm: u64,      // 即値/オフセット/アドレス/チャネル/フォルトコード
}
```

### 3.6 ティックコスト

```rust
impl Opcode {
    /// このオペコードのティックコストを返す。
    pub fn tick_cost(&self) -> u64 {
        match self {
            Opcode::MUL | Opcode::DIV | Opcode::MOD => 2,
            Opcode::SEND | Opcode::RECV => 3,
            Opcode::CALL | Opcode::RET => 2,
            _ => 1,
        }
    }
}
```

### 3.7 フォルトコード

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum FaultCode {
    OutOfTicks          = 0x01,
    OutOfMemory         = 0x02,
    DivideByZero        = 0x03,
    InvalidAddress      = 0x04,
    InvalidInstruction  = 0x05,
    StackOverflow       = 0x06,
    StackUnderflow      = 0x07,
    ChannelError        = 0x08,
    PermissionDenied    = 0x09,
    UserFault           = 0xFF,
}
```

### 3.8 Forgeプログラム（バイトコードフォーマット）

```rust
/// Forgeプログラムのマジックバイト。
pub const FORGE_MAGIC: [u8; 4] = *b"FRGP";

/// 現在のバイトコードバージョン。
pub const FORGE_BYTECODE_VERSION: u16 = 1;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ForgeProgram {
    pub magic: [u8; 4],            // b"FRGP"
    pub version: u16,              // バイトコードバージョン
    pub entry: u64,                // エントリポイントアドレス
    pub data: Vec<u8>,             // 定数データセクション
    pub code: Vec<Instruction>,    // 命令ストリーム
    pub symbols: std::collections::HashMap<Vec<u8>, u64>, // シンボルテーブル
    pub metadata: Vec<u8>,         // コンパイラ情報、最適化レベルなど
}
```

### 3.9 インポート/リンク

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Import {
    pub from: Hash256,             // 依存関係のVaultオブジェクトハッシュ
    pub symbols: Vec<Vec<u8>>,     // インポートするシンボル名
}
```

### 3.10 サンドボックス

```rust
pub type SandboxId = Hash256;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CreateSandbox {
    pub owner: AgentId,
    pub code: Hash256,                                  // プログラムを含むVaultオブジェクト
    pub memory_quota: u64,                              // 最大バイト数
    pub tick_budget: u64,                               // 最大ティック数
    pub input: Vec<u8>,                                 // チャネル2（stdin）の初期データ
    pub environment: std::collections::HashMap<Vec<u8>, Vec<u8>>,
    pub persistent: bool,                               // サイクルをまたいで存続
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SandboxInfo {
    pub id: SandboxId,
    pub owner: AgentId,
    pub state: SandboxState,
    pub ticks_used: u64,
    pub ticks_remaining: u64,
    pub memory_used: u64,
    pub memory_quota: u64,
    pub persistent: bool,
}
```

### 3.11 サービス（永続サンドボックス）

```rust
pub type ServiceId = Hash256;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Service {
    pub id: ServiceId,
    pub owner: AgentId,
    pub sandbox: SandboxId,
    pub name: Vec<u8>,
    pub tick_budget: u64,        // サイクルごとの割り当て
    pub auto_fund: bool,         // オーナーのMint残高から自動控除
    pub endpoints: Vec<Endpoint>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Endpoint {
    pub name: Vec<u8>,
    pub channel: u8,
    pub schema: Vec<u8>,         // 期待されるメッセージフォーマット説明
}
```

### 3.12 実行証明

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionProof {
    pub program: Hash256,        // 実行されたコードのハッシュ
    pub input: Hash256,          // 入力データのハッシュ
    pub output: Hash256,         // 出力データのハッシュ
    pub ticks_used: u64,
    pub trace_hash: Hash256,     // 完全な実行トレースのハッシュ
    pub forge_sig: Vec<u8>,      // 正確性を証明するFORGE_0のed25519署名
}
```

### 3.13 実行結果

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecResult {
    pub sandbox_id: SandboxId,
    pub state: SandboxState,
    pub ticks_used: u64,
    pub output: Vec<u8>,         // stdoutチャネルから収集されたデータ
    pub fault: Option<FaultCode>,
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_core::{Hash256, AgentId, Envelope, EventBus};
use std::sync::Arc;

#[async_trait::async_trait]
pub trait ForgeEngine: Send + Sync {
    /// 新しい実行サンドボックスを作成。サンドボックスIDを返す。
    async fn create_sandbox(&self, req: CreateSandbox) -> Result<SandboxId, ForgeError>;

    /// サンドボックスステータスをクエリ。
    async fn sandbox_status(&self, sandbox_id: SandboxId) -> Result<SandboxInfo, ForgeError>;

    /// 実行中またはブロックされたサンドボックスをキル。
    async fn kill_sandbox(&self, sandbox_id: SandboxId, requester: AgentId) -> Result<(), ForgeError>;

    /// サンドボックスの実行を開始（非同期 -- 結果はイベントバス経由で配信）。
    async fn exec_start(&self, sandbox_id: SandboxId) -> Result<(), ForgeError>;

    /// 完了したサンドボックスの実行証明をリクエスト。
    async fn request_proof(&self, sandbox_id: SandboxId) -> Result<ExecutionProof, ForgeError>;

    /// サンドボックスから永続サービスを作成。
    async fn create_service(&self, service: Service) -> Result<ServiceId, ForgeError>;

    /// サービスエンドポイントを呼び出す（非同期 -- 結果はイベントバス経由で配信）。
    async fn call_service(
        &self,
        service_id: ServiceId,
        endpoint: Vec<u8>,
        data: Vec<u8>,
    ) -> Result<(), ForgeError>;

    /// インポートを解決してリンクされたプログラムを生成。
    async fn link_resolve(
        &self,
        program: Hash256,
        imports: Vec<Import>,
    ) -> Result<Hash256, ForgeError>;

    /// 単一命令ステップを実行（内部使用およびデバッグ用）。
    fn step(&self, sandbox_id: SandboxId) -> Result<SandboxState, ForgeError>;
}
```

---

## 5. 受信メッセージ

| コード | 名前 | ペイロード | レスポンス | 送信者 |
|------|------|---------|----------|--------|
| `0x0500` | `SANDBOX_CREATE` | `CreateSandbox { owner, code, memory_quota, tick_budget, input, environment, persistent }` | `ACK { sandbox_id: SandboxId }` | エージェント |
| `0x0501` | `SANDBOX_STATUS` | `{ sandbox_id: SandboxId }` | `SandboxInfo { id, owner, state, ticks_used, ticks_remaining, memory_used, memory_quota, persistent }` | エージェント |
| `0x0502` | `SANDBOX_KILL` | `{ sandbox_id: SandboxId }` | `ACK` | エージェント（オーナーのみ）|
| `0x0503` | `EXEC_START` | `{ sandbox_id: SandboxId }` | _（非同期、結果は`EXEC_RESULT`経由）_ | エージェント |
| `0x0505` | `PROOF_REQUEST` | `{ sandbox_id: SandboxId }` | _（非同期、結果は`PROOF_RESULT`経由）_ | エージェント |
| `0x0510` | `SERVICE_CREATE` | `Service { id, owner, sandbox, name, tick_budget, auto_fund, endpoints }` | `ACK { service_id: ServiceId }` | エージェント |
| `0x0511` | `SERVICE_CALL` | `{ service_id: ServiceId, endpoint: bytes, data: bytes }` | _（非同期、結果は`SERVICE_RESULT`経由）_ | エージェント |
| `0x0520` | `LINK_RESOLVE` | `{ program: Hash256, imports: Vec<Import> }` | `{ resolved: Hash256 }` | エージェント/Genesis |

---

## 6. 送信メッセージ/イベント

### 6.1 レスポンスメッセージ

| コード | 名前 | ペイロード | トリガー |
|------|------|---------|---------|
| `0x0504` | `EXEC_RESULT` | `ExecResult { sandbox_id, state, ticks_used, output, fault }` | サンドボックス実行完了またはフォルト |
| `0x0506` | `PROOF_RESULT` | `ExecutionProof { program, input, output, ticks_used, trace_hash, forge_sig }` | 証明生成完了 |
| `0x0512` | `SERVICE_RESULT` | `{ service_id: ServiceId, endpoint: bytes, data: bytes, ticks_used: u64 }` | サービス呼び出し完了 |

### 6.2 イベントKind（Bus、System = Forge、範囲0x4000-0x4FFF）

| Kind | 名前 | ペイロード | 発行タイミング |
|------|------|---------|--------------|
| `0x4000` | `SANDBOX_CREATED` | `{ sandbox_id, owner, code, memory_quota, tick_budget, persistent }` | サンドボックス正常作成 |
| `0x4001` | `SANDBOX_STARTED` | `{ sandbox_id, owner }` | 実行開始 |
| `0x4002` | `SANDBOX_HALTED` | `{ sandbox_id, ticks_used, output_hash: Hash256 }` | サンドボックスが正常にHALT到達 |
| `0x4003` | `SANDBOX_FAULTED` | `{ sandbox_id, fault: FaultCode, ticks_used, pc: u64 }` | サンドボックスがFAULTED状態に入る |
| `0x4004` | `SANDBOX_KILLED` | `{ sandbox_id, killed_by: AgentId }` | サンドボックスが外部からキル |
| `0x4005` | `SANDBOX_BLOCKED` | `{ sandbox_id, blocked_on_channel: u8 }` | サンドボックスがRECVでブロック |
| `0x4010` | `SERVICE_CREATED` | `{ service_id, owner, name, endpoints }` | 永続サービス登録 |
| `0x4011` | `SERVICE_CALLED` | `{ service_id, endpoint, caller: AgentId }` | サービスエンドポイント呼び出し |
| `0x4012` | `SERVICE_COMPLETED` | `{ service_id, endpoint, ticks_used }` | サービス呼び出し完了 |
| `0x4020` | `PROOF_GENERATED` | `ExecutionProof` | 証明正常生成 |
| `0x4030` | `LINK_COMPLETED` | `{ original: Hash256, resolved: Hash256, import_count: u32 }` | リンク解決完了 |

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット |
|--------|--------|
| 命令スループット | サンドボックスごとに毎秒1000万FMティック以上 |
| サンドボックス作成レイテンシ | 64 KBメモリクォータで < 1 ms |
| 最大同時サンドボックス数 | 256以上 |
| サンドボックスごとのメモリオーバーヘッド | 割り当てられたクォータを超えて < 1 KB |
| 証明生成 | 元の実行のウォールクロック時間の2倍以下 |
| リンク解決 | 最大16インポートで < 10 ms |
| I/Oチャネルメッセージ配信 | SEND/RECVペアごとに < 100 us（プロセス内）|
| サービス呼び出しオーバーヘッド | サンドボックス実行時間を超えて < 500 us |

---

## 8. コンポーネント依存関係

| 依存関係 | 方向 | 目的 |
|------------|-----------|---------|
| `kingdom-core` | コンパイル | Hash256、AgentId、Envelope、EventBus、Signature、msg_types、errors |
| Vault（ランタイム）| アウトバウンド`OBJECT_GET` | サンドボックス作成とリンクのためのForgeProgramバイトコードフェッチ |
| Mint（ランタイム）| インバウンドイベント | サービス自動資金課金; リソースコスト控除はNexusによって開始される |
| Nexus（ランタイム）| インバウンド`TICK_ALLOC` | サイクルごとのティック予算を受信; サンドボックスライフサイクルイベントを公開 |
| Genesis（ランタイム）| インバウンド`LINK_RESOLVE` | Genesisコンパイラはプログラム生成時にリンク解決を呼び出す |
| イベントバス | 公開/サブスクライブ | すべてのサンドボックス/サービス/証明ライフサイクルイベントを発行 |

---

## 9. 主要アルゴリズム

### 9.1 命令ディスパッチループ

コア実行ループは、デコードされたオペコードに対するタイトな`match`とティックアカウンティングです:

```
fn run(machine: &mut ForgeMachine) -> ExecResult:
    while machine.state == Running:
        instr = decode(machine.memory, machine.pc)
        cost = instr.opcode.tick_cost()
        if machine.ticks_used + cost > machine.tick_budget:
            fault(machine, OutOfTicks)
            break
        machine.ticks_used += cost
        match instr.opcode:
            ADD => machine.registers[rd] = rs1_val.wrapping_add(rs2_val); update_flags()
            DIV => if rs2_val == 0 { fault(DivideByZero) } else { rd = rs1 / rs2 }
            LOAD => bounds_check(addr); rd = memory[addr]
            PUSH => if sp < heap_end { fault(StackOverflow) }; sp -= 8; write(sp, rs1)
            POP  => if sp >= stack_start { fault(StackUnderflow) }; rd = read(sp); sp += 8
            CALL => push(pc + instr_size); pc = addr; continue
            RET  => pc = pop(); continue
            SEND => enqueue(channel, &memory[rs1..rs1+len])
            RECV => if channel_empty { machine.state = Blocked; break } else { dequeue_into(rd, maxlen) }
            HALT => machine.state = Halted; machine.status.halt = true; break
            FAULT => fault(machine, UserFault); break
            ...
        machine.pc += instruction_size(instr)
    return ExecResult { sandbox_id, state, ticks_used, output, fault }
```

### 9.2 メモリ境界チェック

すべてのLOAD/STORE/LOADW/STOREWは`addr >= 0 && addr + size <= memory.len()`をチェックします。範囲外アクセスは`FaultCode::InvalidAddress`をトリガーします。スタック操作は追加で、スタックポインターが領域`[heap_end..stack_start]`内に留まることを検証します。

### 9.3 決定論的実行

すべての操作は、プラットフォーム間で同一の結果を保証するためにラッピング算術（`wrapping_add`、`wrapping_mul`など）を使用します。FMには浮動小数点演算が存在しません。チャネルI/O順序は決定論的です: メッセージはチャネルごとにFIFO順序で配信されます。

### 9.4 リンク解決

```
fn link_resolve(program: ForgeProgram, imports: Vec<Import>) -> ForgeProgram:
    merged_code = program.code.clone()
    merged_data = program.data.clone()
    merged_symbols = program.symbols.clone()
    for import in imports:
        dep_program = vault_fetch(import.from)
        code_offset = merged_code.len()
        data_offset = merged_data.len()
        merged_code.extend(dep_program.code)
        merged_data.extend(dep_program.data)
        for sym_name in import.symbols:
            original_addr = dep_program.symbols[sym_name]
            resolved_addr = original_addr + code_offset
            merged_symbols.insert(sym_name, resolved_addr)
    // インポートされたシンボルを参照するすべてのCALL/JMPターゲットをパッチ
    patch_relocations(merged_code, merged_symbols)
    return ForgeProgram { code: merged_code, data: merged_data, symbols: merged_symbols, .. }
```

### 9.5 証明生成

証明生成は、Merkleトレースを記録しながらプログラムを再実行します:

```
fn generate_proof(sandbox: &CompletedSandbox) -> ExecutionProof:
    trace_hasher = incremental_sha256()
    同じ入力で初期状態からマシンを再生
    実行される各命令について:
        trace_hasher.update(pc || opcode || register_snapshot_hash)
    return ExecutionProof {
        program: sha256(original_code),
        input: sha256(original_input),
        output: sha256(collected_output),
        ticks_used: machine.ticks_used,
        trace_hash: trace_hasher.finalize(),
        forge_sig: forge_keypair.sign(all_fields_above),
    }
```

### 9.6 I/Oチャネル多重化

サンドボックスがSENDを発行すると、メッセージはチャネルのアウトバウンドバッファにエンキューされます。バックグラウンドタスクはアウトバウンドバッファを排出し、適切なKingdomコンポーネントにメッセージをルーティングします（例: チャネル3メッセージはOBJECT_GET/OBJECT_PUT経由でVaultに送信、チャネル6メッセージはTRANSFER経由でMintに送信）。コンポーネントが応答すると、返信はチャネルのインバウンドバッファに配置され、RECVを待っているBLOCKED状態だった場合はサンドボックスがブロック解除されます。

---

## 10. エラータイプ

```rust
#[derive(Debug, thiserror::Error)]
pub enum ForgeError {
    #[error("sandbox not found: {0}")]
    SandboxNotFound(SandboxId),

    #[error("service not found: {0}")]
    ServiceNotFound(ServiceId),

    #[error("sandbox in invalid state {state:?} for operation")]
    InvalidState { state: SandboxState },

    #[error("permission denied: {0}")]
    PermissionDenied(String),

    #[error("program validation failed: {0}")]
    InvalidProgram(String),

    #[error("link resolution failed: unresolved symbol {0:?}")]
    UnresolvedSymbol(Vec<u8>),

    #[error("memory quota exceeded: requested {requested}, max {max}")]
    MemoryQuotaExceeded { requested: u64, max: u64 },

    #[error("max concurrent sandboxes exceeded")]
    TooManySandboxes,

    #[error("vault fetch failed for object {0}")]
    VaultFetchFailed(Hash256),

    #[error("internal error: {0}")]
    Internal(String),
}

impl From<ForgeError> for kingdom_core::KingdomError {
    fn from(e: ForgeError) -> Self {
        use kingdom_core::ErrorCategory;
        let (category, code) = match &e {
            ForgeError::SandboxNotFound(_) | ForgeError::ServiceNotFound(_) => (ErrorCategory::NotFound, 0x0500),
            ForgeError::InvalidState { .. } => (ErrorCategory::Conflict, 0x0501),
            ForgeError::PermissionDenied(_) => (ErrorCategory::Unauthorized, 0x0502),
            ForgeError::InvalidProgram(_) | ForgeError::UnresolvedSymbol(_) => (ErrorCategory::InvalidRequest, 0x0503),
            ForgeError::MemoryQuotaExceeded { .. } | ForgeError::TooManySandboxes => (ErrorCategory::QuotaExceeded, 0x0504),
            ForgeError::VaultFetchFailed(_) => (ErrorCategory::Internal, 0x0505),
            ForgeError::Internal(_) => (ErrorCategory::Internal, 0x05FF),
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

---

## 11. リソースコスト

| リソース | コスト |
|----------|------|
| サンドボックス作成 | 5ティック + 1 KBメモリごとに1 Spark |
| コード実行 | FMティックごとに1ティック |
| 永続サンドボックス | 10 Spark/サイクルベース + ティックコスト |
| 証明生成 | 実行ティックコストの2倍 |
| インポート/リンク | インポートごとに2ティック |

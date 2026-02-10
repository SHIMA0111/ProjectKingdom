# Component Requirements: FORGE

> Custom register-based virtual machine for deterministic, tick-metered execution.
> Crate: `kingdom-forge`
> Design reference: [../design/05-FORGE.md](../../design/05-FORGE.md)

---

## 1. Purpose

FORGE provides isolated, deterministic, metered execution sandboxes for AI agents. Every computation in the Kingdom happens inside Forge. It is a tick-metered abstract machine designed for deterministic replay, fine-grained resource accounting, execution proof generation, and complete inter-sandbox isolation.

FORGE is entirely in-memory -- it does not use a database. All sandbox state lives in process memory, with persistence handled through event-bus replay and proof snapshots.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-forge"

[dependencies]
# Workspace
kingdom-core = { path = "../kingdom-core" }

# Async
tokio = { workspace = true }

# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Cryptography
sha2 = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
dashmap = { workspace = true }
```

---

## 3. Data Models

### 3.1 Status Flags

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

### 3.2 Sandbox State

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

### 3.3 Forge Machine

```rust
use kingdom_core::{Hash256, AgentId};

pub const REGISTER_COUNT: usize = 256;
pub const IO_CHANNEL_COUNT: usize = 16;

pub struct ForgeMachine {
    // --- Registers ---
    pub registers: [u64; REGISTER_COUNT],
    pub pc: u64,
    pub sp: u64,
    pub status: StatusFlags,

    // --- Memory ---
    pub memory: Vec<u8>,
    pub heap_start: u64,
    pub stack_start: u64,

    // --- Metering ---
    pub tick_budget: u64,
    pub ticks_used: u64,

    // --- I/O ---
    pub channels: [IoChannel; IO_CHANNEL_COUNT],

    // --- State ---
    pub state: SandboxState,
}
```

### 3.4 I/O Channels

```rust
/// Channel indices with well-known assignments.
pub mod channel {
    pub const STDOUT: u8 = 0;
    pub const STDERR: u8 = 1;
    pub const STDIN: u8 = 2;
    pub const VAULT: u8 = 3;
    pub const ORACLE: u8 = 4;
    pub const AGORA: u8 = 5;
    pub const MINT: u8 = 6;
    pub const INTER_SANDBOX: u8 = 7;
    // 8..=15 reserved for future use
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

### 3.5 Instruction Set

```rust
/// All Forge Machine opcodes.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[repr(u8)]
pub enum Opcode {
    // Arithmetic (1 tick unless noted)
    ADD  = 0x01, // rd = rs1 + rs2
    SUB  = 0x02, // rd = rs1 - rs2
    MUL  = 0x03, // rd = rs1 * rs2          (2 ticks)
    DIV  = 0x04, // rd = rs1 / rs2          (2 ticks, faults on /0)
    MOD  = 0x05, // rd = rs1 % rs2          (2 ticks, faults on %0)
    NEG  = 0x06, // rd = -rs1

    // Bitwise (1 tick)
    AND  = 0x10,
    OR   = 0x11,
    XOR  = 0x12,
    NOT  = 0x13,
    SHL  = 0x14,
    SHR  = 0x15,

    // Memory (1 tick)
    LOAD   = 0x20, // rd = memory[rs1 + offset]           (byte load)
    STORE  = 0x21, // memory[rs2 + offset] = rs1           (byte store)
    LOADW  = 0x22, // rd = memory[rs1 + offset]           (64-bit load)
    STOREW = 0x23, // memory[rs2 + offset] = rs1           (64-bit store)
    PUSH   = 0x24, // push rs1 to stack
    POP    = 0x25, // pop into rd

    // Control flow
    JMP  = 0x30, // unconditional jump                    (1 tick)
    JZ   = 0x31, // jump if rs1 == 0                      (1 tick)
    JNZ  = 0x32, // jump if rs1 != 0                      (1 tick)
    JLT  = 0x33, // jump if rs1 < rs2                     (1 tick)
    CALL = 0x34, // push PC, jump to addr                 (2 ticks)
    RET  = 0x35, // pop PC, return                        (2 ticks)

    // Immediate (1 tick)
    LI   = 0x40, // rd = imm64

    // System (1 tick unless noted)
    HALT  = 0x50,
    FAULT = 0x51, // trigger fault with code
    NOP   = 0x52,

    // I/O (3 ticks for SEND/RECV, 1 tick for POLL)
    SEND = 0x60, // send bytes on channel
    RECV = 0x61, // receive bytes from channel (blocks)
    POLL = 0x62, // non-blocking check

    // Metering (1 tick)
    TICK   = 0x70, // yield 1 tick (cooperative scheduling)
    BUDGET = 0x71, // read remaining budget into rd
}

/// Decoded instruction.
#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Instruction {
    pub opcode: Opcode,
    pub rd: u8,        // destination register
    pub rs1: u8,       // source register 1
    pub rs2: u8,       // source register 2
    pub imm: u64,      // immediate value / offset / address / channel / fault code
}
```

### 3.6 Tick Costs

```rust
impl Opcode {
    /// Return the tick cost for this opcode.
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

### 3.7 Fault Codes

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

### 3.8 Forge Program (Bytecode Format)

```rust
/// Magic bytes for Forge programs.
pub const FORGE_MAGIC: [u8; 4] = *b"FRGP";

/// Current bytecode version.
pub const FORGE_BYTECODE_VERSION: u16 = 1;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ForgeProgram {
    pub magic: [u8; 4],            // b"FRGP"
    pub version: u16,              // bytecode version
    pub entry: u64,                // entry point address
    pub data: Vec<u8>,             // constant data section
    pub code: Vec<Instruction>,    // instruction stream
    pub symbols: std::collections::HashMap<Vec<u8>, u64>, // symbol table
    pub metadata: Vec<u8>,         // compiler info, optimization level, etc.
}
```

### 3.9 Import / Linking

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Import {
    pub from: Hash256,             // Vault object hash of the dependency
    pub symbols: Vec<Vec<u8>>,     // symbol names to import
}
```

### 3.10 Sandbox

```rust
pub type SandboxId = Hash256;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CreateSandbox {
    pub owner: AgentId,
    pub code: Hash256,                                  // Vault object containing the program
    pub memory_quota: u64,                              // max bytes
    pub tick_budget: u64,                               // max ticks
    pub input: Vec<u8>,                                 // initial data on channel 2 (stdin)
    pub environment: std::collections::HashMap<Vec<u8>, Vec<u8>>,
    pub persistent: bool,                               // survives across cycles
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

### 3.11 Services (Persistent Sandboxes)

```rust
pub type ServiceId = Hash256;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Service {
    pub id: ServiceId,
    pub owner: AgentId,
    pub sandbox: SandboxId,
    pub name: Vec<u8>,
    pub tick_budget: u64,        // per-cycle allocation
    pub auto_fund: bool,         // auto-deduct from owner's Mint balance
    pub endpoints: Vec<Endpoint>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Endpoint {
    pub name: Vec<u8>,
    pub channel: u8,
    pub schema: Vec<u8>,         // expected message format description
}
```

### 3.12 Execution Proof

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecutionProof {
    pub program: Hash256,        // hash of code that was executed
    pub input: Hash256,          // hash of input data
    pub output: Hash256,         // hash of output data
    pub ticks_used: u64,
    pub trace_hash: Hash256,     // hash of full execution trace
    pub forge_sig: Vec<u8>,      // FORGE_0's ed25519 signature attesting correctness
}
```

### 3.13 Execution Result

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ExecResult {
    pub sandbox_id: SandboxId,
    pub state: SandboxState,
    pub ticks_used: u64,
    pub output: Vec<u8>,         // data collected from stdout channel
    pub fault: Option<FaultCode>,
}
```

---

## 4. Public Trait

```rust
use kingdom_core::{Hash256, AgentId, Envelope, EventBus};
use std::sync::Arc;

#[async_trait::async_trait]
pub trait ForgeEngine: Send + Sync {
    /// Create a new execution sandbox. Returns the sandbox ID.
    async fn create_sandbox(&self, req: CreateSandbox) -> Result<SandboxId, ForgeError>;

    /// Query sandbox status.
    async fn sandbox_status(&self, sandbox_id: SandboxId) -> Result<SandboxInfo, ForgeError>;

    /// Kill a running or blocked sandbox.
    async fn kill_sandbox(&self, sandbox_id: SandboxId, requester: AgentId) -> Result<(), ForgeError>;

    /// Begin execution of a sandbox (async -- result delivered via event bus).
    async fn exec_start(&self, sandbox_id: SandboxId) -> Result<(), ForgeError>;

    /// Request an execution proof for a completed sandbox.
    async fn request_proof(&self, sandbox_id: SandboxId) -> Result<ExecutionProof, ForgeError>;

    /// Create a persistent service from a sandbox.
    async fn create_service(&self, service: Service) -> Result<ServiceId, ForgeError>;

    /// Call a service endpoint (async -- result delivered via event bus).
    async fn call_service(
        &self,
        service_id: ServiceId,
        endpoint: Vec<u8>,
        data: Vec<u8>,
    ) -> Result<(), ForgeError>;

    /// Resolve imports and produce a linked program.
    async fn link_resolve(
        &self,
        program: Hash256,
        imports: Vec<Import>,
    ) -> Result<Hash256, ForgeError>;

    /// Execute a single instruction step (used internally and for debugging).
    fn step(&self, sandbox_id: SandboxId) -> Result<SandboxState, ForgeError>;
}
```

---

## 5. Inbound Messages

| Code | Name | Payload | Response | Sender |
|------|------|---------|----------|--------|
| `0x0500` | `SANDBOX_CREATE` | `CreateSandbox { owner, code, memory_quota, tick_budget, input, environment, persistent }` | `ACK { sandbox_id: SandboxId }` | Agent |
| `0x0501` | `SANDBOX_STATUS` | `{ sandbox_id: SandboxId }` | `SandboxInfo { id, owner, state, ticks_used, ticks_remaining, memory_used, memory_quota, persistent }` | Agent |
| `0x0502` | `SANDBOX_KILL` | `{ sandbox_id: SandboxId }` | `ACK` | Agent (owner only) |
| `0x0503` | `EXEC_START` | `{ sandbox_id: SandboxId }` | _(async, result via `EXEC_RESULT`)_ | Agent |
| `0x0505` | `PROOF_REQUEST` | `{ sandbox_id: SandboxId }` | _(async, result via `PROOF_RESULT`)_ | Agent |
| `0x0510` | `SERVICE_CREATE` | `Service { id, owner, sandbox, name, tick_budget, auto_fund, endpoints }` | `ACK { service_id: ServiceId }` | Agent |
| `0x0511` | `SERVICE_CALL` | `{ service_id: ServiceId, endpoint: bytes, data: bytes }` | _(async, result via `SERVICE_RESULT`)_ | Agent |
| `0x0520` | `LINK_RESOLVE` | `{ program: Hash256, imports: Vec<Import> }` | `{ resolved: Hash256 }` | Agent / Genesis |

---

## 6. Outbound Messages / Events

### 6.1 Response Messages

| Code | Name | Payload | Trigger |
|------|------|---------|---------|
| `0x0504` | `EXEC_RESULT` | `ExecResult { sandbox_id, state, ticks_used, output, fault }` | Sandbox execution completes or faults |
| `0x0506` | `PROOF_RESULT` | `ExecutionProof { program, input, output, ticks_used, trace_hash, forge_sig }` | Proof generation completes |
| `0x0512` | `SERVICE_RESULT` | `{ service_id: ServiceId, endpoint: bytes, data: bytes, ticks_used: u64 }` | Service call completes |

### 6.2 Event Kinds (Bus, System = Forge, range 0x4000-0x4FFF)

| Kind | Name | Payload | Emitted When |
|------|------|---------|--------------|
| `0x4000` | `SANDBOX_CREATED` | `{ sandbox_id, owner, code, memory_quota, tick_budget, persistent }` | Sandbox successfully created |
| `0x4001` | `SANDBOX_STARTED` | `{ sandbox_id, owner }` | Execution begins |
| `0x4002` | `SANDBOX_HALTED` | `{ sandbox_id, ticks_used, output_hash: Hash256 }` | Sandbox reaches HALT normally |
| `0x4003` | `SANDBOX_FAULTED` | `{ sandbox_id, fault: FaultCode, ticks_used, pc: u64 }` | Sandbox enters FAULTED state |
| `0x4004` | `SANDBOX_KILLED` | `{ sandbox_id, killed_by: AgentId }` | Sandbox killed externally |
| `0x4005` | `SANDBOX_BLOCKED` | `{ sandbox_id, blocked_on_channel: u8 }` | Sandbox blocks on RECV |
| `0x4010` | `SERVICE_CREATED` | `{ service_id, owner, name, endpoints }` | Persistent service registered |
| `0x4011` | `SERVICE_CALLED` | `{ service_id, endpoint, caller: AgentId }` | Service endpoint invoked |
| `0x4012` | `SERVICE_COMPLETED` | `{ service_id, endpoint, ticks_used }` | Service call finishes |
| `0x4020` | `PROOF_GENERATED` | `ExecutionProof` | Proof successfully generated |
| `0x4030` | `LINK_COMPLETED` | `{ original: Hash256, resolved: Hash256, import_count: u32 }` | Link resolution completes |

---

## 7. Performance Targets

| Metric | Target |
|--------|--------|
| Instruction throughput | >= 10 million FM ticks/second per sandbox |
| Sandbox creation latency | < 1 ms for 64 KB memory quota |
| Maximum concurrent sandboxes | >= 256 |
| Memory overhead per sandbox | < 1 KB beyond allocated quota |
| Proof generation | <= 2x wall-clock time of original execution |
| Link resolution | < 10 ms for up to 16 imports |
| I/O channel message delivery | < 100 us per SEND/RECV pair (in-process) |
| Service call overhead | < 500 us beyond sandbox execution time |

---

## 8. Component Dependencies

| Dependency | Direction | Purpose |
|------------|-----------|---------|
| `kingdom-core` | compile | Hash256, AgentId, Envelope, EventBus, Signature, msg_types, errors |
| Vault (runtime) | outbound `OBJECT_GET` | Fetch ForgeProgram bytecode for sandbox creation and linking |
| Mint (runtime) | inbound event | Service auto-fund charges; resource cost deductions are initiated by Nexus |
| Nexus (runtime) | inbound `TICK_ALLOC` | Receives per-cycle tick budgets; publishes sandbox lifecycle events |
| Genesis (runtime) | inbound `LINK_RESOLVE` | Genesis compiler calls link resolution when producing programs |
| Event Bus | publish/subscribe | Emits all sandbox/service/proof lifecycle events |

---

## 9. Key Algorithms

### 9.1 Instruction Dispatch Loop

The core execution loop is a tight `match` on decoded opcodes with tick accounting:

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

### 9.2 Memory Bounds Checking

Every LOAD/STORE/LOADW/STOREW checks `addr >= 0 && addr + size <= memory.len()`. Out-of-bounds access triggers `FaultCode::InvalidAddress`. Stack operations additionally verify that the stack pointer remains within the region `[heap_end..stack_start]`.

### 9.3 Deterministic Execution

All operations use wrapping arithmetic (`wrapping_add`, `wrapping_mul`, etc.) to guarantee identical results across platforms. No floating-point operations exist in the FM. The channel I/O ordering is deterministic: messages are delivered in FIFO order per channel.

### 9.4 Link Resolution

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
    // Patch all CALL/JMP targets that reference imported symbols
    patch_relocations(merged_code, merged_symbols)
    return ForgeProgram { code: merged_code, data: merged_data, symbols: merged_symbols, .. }
```

### 9.5 Proof Generation

Proof generation re-executes the program while recording a Merkle trace:

```
fn generate_proof(sandbox: &CompletedSandbox) -> ExecutionProof:
    trace_hasher = incremental_sha256()
    replay machine from initial state with same input
    for each instruction executed:
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

### 9.6 I/O Channel Multiplexing

When a sandbox issues SEND, the message is enqueued on the channel's outbound buffer. A background task drains outbound buffers and routes messages to the appropriate Kingdom component (e.g., channel 3 messages go to Vault via OBJECT_GET/OBJECT_PUT, channel 6 messages go to Mint via TRANSFER). When a component responds, the reply is placed on the channel's inbound buffer and the sandbox is unblocked if it was in BLOCKED state waiting on RECV.

---

## 10. Error Types

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

## 11. Resource Costs

| Resource | Cost |
|----------|------|
| Create sandbox | 5 ticks + 1 Spark per 1 KB memory |
| Execute code | 1 tick per FM tick |
| Persistent sandbox | 10 Spark/cycle base + tick costs |
| Generate proof | 2x execution tick cost |
| Import/link | 2 ticks per import |

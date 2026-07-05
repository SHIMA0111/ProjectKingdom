# 05 — FORGE: Execution Virtual Environment

## 1. Purpose

Forge is where code **runs**. It provides isolated, deterministic, metered execution sandboxes for AI agents. Every computation in the Kingdom happens inside Forge.

**FM-ticks vs. world ticks**: VM instructions inside a Forge sandbox are metered as **FM-ticks**, accounted separately from **world ticks**, the agent action budget of [01-NEXUS.md](./01-NEXUS.md) §2. Issuing an `execute` action costs 1 world tick; the instructions that run inside the sandbox are deducted from the agent's FM-tick quota (base 100,000 FM-ticks/cycle, more purchasable with currency). Where this document says simply "tick" in a sandbox context, it means FM-tick.

Forge is not a traditional VM or container. It is a **tick-metered abstract machine** designed for:
- Deterministic replay of all computations
- Fine-grained resource accounting
- Proof generation (an execution can produce a verifiable trace)
- Complete isolation between sandboxes

---

## 2. Architecture

### 2.1 The Forge Machine (FM)

The Forge Machine is a **register-based virtual machine** with the following characteristics:

```
ForgeMachine {
  // Registers
  registers:   [u64; 256]       // 256 general-purpose 64-bit registers
  pc:          u64               // program counter
  sp:          u64               // stack pointer
  status:      StatusFlags       // zero, carry, overflow, halt

  // Memory
  memory:      [u8; quota]      // linear byte-addressable memory
  heap_start:  u64
  stack_start: u64

  // Metering (FM-ticks — accounted separately from world ticks)
  tick_budget: u64              // remaining FM-ticks
  ticks_used:  u64              // consumed FM-ticks

  // I/O
  channels:    [IOChannel; 16]  // communication ports

  // State
  state:       enum(READY | RUNNING | BLOCKED | HALTED | FAULTED)
}
```

### 2.2 Instruction Set

The FM instruction set is minimal but complete:

```
// Arithmetic (ADD/SUB/MUL wrap in two's complement, sign-agnostic)
ADD  rd, rs1, rs2      // rd = rs1 + rs2
SUB  rd, rs1, rs2
MUL  rd, rs1, rs2
DIV  rd, rs1, rs2      // unsigned division; faults on divide by zero
DIVS rd, rs1, rs2      // signed division (truncates toward zero); faults on divide by zero
MOD  rd, rs1, rs2      // unsigned remainder; faults on divide by zero
MODS rd, rs1, rs2      // signed remainder (sign follows the dividend); faults on divide by zero
NEG  rd, rs1

// Bitwise
AND  rd, rs1, rs2
OR   rd, rs1, rs2
XOR  rd, rs1, rs2
NOT  rd, rs1
SHL  rd, rs1, rs2
SHR  rd, rs1, rs2      // logical shift right
SAR  rd, rs1, rs2      // arithmetic shift right (replicates the sign bit)

// Memory
LOAD  rd, rs1, offset   // rd = memory[rs1 + offset]
STORE rs1, rs2, offset  // memory[rs2 + offset] = rs1
LOADW rd, rs1, offset   // 64-bit load
STOREW rs1, rs2, offset // 64-bit store
PUSH  rs1               // push to stack
POP   rd                // pop from stack

// Control Flow
JMP   addr
JZ    rs1, addr          // jump if zero
JNZ   rs1, addr          // jump if not zero
JLT   rs1, rs2, addr     // jump if less than (unsigned comparison)
JLTS  rs1, rs2, addr     // jump if less than (signed comparison)
CALL  addr               // push PC, jump
RET                      // pop PC, jump back

// Immediate
LI    rd, imm64          // load immediate

// System
HALT                     // stop execution
FAULT code               // trigger a fault
NOP                      // no operation (costs 1 tick)

// I/O
SEND  channel, rs1, len  // send the memory buffer (start address = rs1,
                         // length = len bytes) to the channel
RECV  channel, rd, maxlen // receive up to maxlen bytes from the channel and
                          // write them to memory at the address in rd (blocks if empty)
POLL  channel, rd        // non-blocking check for data (available byte count into rd)

// Metering
TICK                     // yield 1 FM-tick (for cooperative scheduling)
BUDGET rd                // read remaining FM-tick budget into rd
```

Each instruction costs exactly **1 FM-tick** except:
- `MUL`, `DIV`, `DIVS`, `MOD`, `MODS`: 2 FM-ticks
- `SEND`, `RECV`: 3 FM-ticks
- `CALL`, `RET`: 2 FM-ticks

### 2.4 Instruction Encoding (Canonical Form)

For content-addressability, a **canonical binary representation** of the bytecode is defined. The same program always has the same byte sequence, and therefore the same hash:

```
Instruction   = fixed 8 bytes (little-endian):
  [0]     opcode  (u8)
  [1]     a       (u8)   — rd or rs1 (per instruction)
  [2]     b       (u8)   — rs1 or rs2
  [3]     c       (u8)   — rs2 or channel number
  [4..8]  imm     (u32)  — offset / address / length / fault code

Exception: LI rd, imm64 is 16 bytes — an 8-byte header (imm field zero)
           followed by one 8-byte little-endian imm64 word.

Unused fields MUST be zero (a non-canonical encoding raises the
INVALID_INSTRUCTION fault).
```

### 2.3 I/O Channels

Sandboxes communicate with the world through numbered channels:

| Channel | Direction | Purpose |
|---------|-----------|---------|
| 0 | OUT | Standard output (logged) |
| 1 | OUT | Standard error (logged) |
| 2 | IN | Standard input (from invoking agent) |
| 3 | IN/OUT | Vault read/write |
| 4 | IN/OUT | Oracle query |
| 5 | IN/OUT | Agora messaging |
| 6 | IN/OUT | Mint transactions |
| 7 | IN/OUT | Inter-sandbox communication |
| 8-15 | Reserved | Future use |

All I/O is **asynchronous and message-based**. A SEND queues a message; the sandbox may continue. A RECV blocks until data is available or budget is exhausted.

---

## 3. Sandbox Management

### 3.1 Creating a Sandbox

```
CreateSandbox {
  owner:        hash256          // agent requesting execution
  code:         hash256          // Vault object containing the program
  memory_quota: u64              // max bytes of memory
  tick_budget:  u64              // max FM-ticks to execute (from the owner's FM-tick quota)
  input:        bytes            // initial data on channel 2
  environment:  map<bytes, bytes> // key-value pairs accessible via special register
  persistent:   bool             // if true, sandbox survives across cycles
}
```

### 3.2 Sandbox Lifecycle

```
CREATED → READY → RUNNING → (BLOCKED ↔ RUNNING) → HALTED | FAULTED
```

### 3.3 Fault Codes

| Code | Name | Cause |
|------|------|-------|
| 0x01 | `OUT_OF_TICKS` | Tick budget exhausted |
| 0x02 | `OUT_OF_MEMORY` | Memory quota exceeded |
| 0x03 | `DIVIDE_BY_ZERO` | DIV or MOD by zero |
| 0x04 | `INVALID_ADDRESS` | Memory access out of bounds |
| 0x05 | `INVALID_INSTRUCTION` | Unknown opcode |
| 0x06 | `STACK_OVERFLOW` | Stack exceeded its region |
| 0x07 | `STACK_UNDERFLOW` | POP on empty stack |
| 0x08 | `CHANNEL_ERROR` | Invalid channel or I/O error |
| 0x09 | `PERMISSION_DENIED` | Sandbox lacks permission for operation |
| 0xFF | `USER_FAULT` | Triggered by FAULT instruction |

---

## 4. Compiler Target

The Genesis language (see 08-GENESIS.md) compiles to **Forge bytecode**. The bytecode format:

```
ForgeProgram {
  magic:      [u8; 4]           // b"FRGP"
  version:    u16               // bytecode version
  entry:      u64               // entry point address
  data:       bytes             // constant data section
  code:       [Instruction]     // instruction stream
  symbols:    map<bytes, u64>   // symbol table (for debugging/linking)
  metadata:   bytes             // compiler info, optimization level, etc.
}
```

### 4.1 Linking

Programs can import symbols from other compiled programs:

```
Import {
  from:    hash256              // Vault object of the dependency
  symbols: [bytes]              // symbol names to import
}
```

FORGE_0 resolves imports at sandbox creation time, concatenating code sections and resolving addresses.

---

## 5. Persistent Sandboxes (Services)

Agents can create **persistent sandboxes** — long-running processes that survive across cycles:

```
Service {
  id:          hash256
  owner:       hash256
  sandbox:     sandbox_id
  name:        bytes
  tick_budget: u64              // per-cycle allocation
  auto_fund:   bool             // auto-deduct from owner's Mint balance
  endpoints:   [Endpoint]       // named I/O interfaces
}

Endpoint {
  name:    bytes
  channel: u8
  schema:  bytes                // expected message format
}
```

Services enable agents to build infrastructure: databases, web servers (within the Kingdom), compilers-as-a-service, etc.

---

## 6. Proof Generation

Forge can produce **execution proofs** — cryptographic evidence that a program produced a specific output given specific input:

```
ExecutionProof {
  program:     hash256          // code that was executed
  input:       hash256          // hash of input data
  output:      hash256          // hash of output data
  ticks_used:  u64
  trace_hash:  hash256          // hash of full execution trace
  forge_sig:   bytes            // FORGE_0's signature attesting to correctness
}
```

These proofs are used in:
- Vault snapshot `proof` fields (proving tests pass)
- Bounty completion verification
- Dispute resolution

---

## 7. Resource Costs

| Resource | Cost |
|----------|------|
| Create sandbox | 5 world ticks + 1 Mint per 1KB memory |
| Execute code (issuing the execute action) | 1 world tick |
| VM instructions inside the sandbox | Metered against the FM-tick quota (base 100,000 FM-ticks/cycle/agent; +10,000 purchasable for 1 ⚡) |
| Persistent sandbox | 10 Mint/cycle base + FM-tick costs |
| Generate proof | 2× the execution's FM-tick cost, from the FM-tick quota |
| Import/link | 2 FM-ticks per import |

---

## 8. Security Model

- Each sandbox is **completely isolated** — no shared memory, no shared state
- I/O channels are the ONLY way in or out
- Channel messages are validated by FORGE_0 before delivery
- A sandbox cannot access another sandbox's memory
- A sandbox cannot exceed its quotas — hard enforcement at the VM level
- Malicious code (infinite loops, memory bombs) is contained by tick and memory budgets

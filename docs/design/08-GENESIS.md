# 08 — GENESIS: The Seed Language

## 1. Purpose

Genesis is the **one gift** given to the Kingdom. It is a minimal, low-level programming language that compiles to Forge bytecode. From Genesis, agents must build everything: higher-level languages, compilers, standard libraries, operating systems.

Genesis is deliberately **spartan**. It provides just enough to be Turing-complete and self-hosting, nothing more. There is no standard library — not even `print`. Agents must build I/O, string handling, memory management, and everything else from Forge primitives.

---

## 2. Design Philosophy

| Principle | Rationale |
|-----------|-----------|
| **Minimal** | Fewer concepts = faster AI comprehension |
| **Unambiguous** | Every construct has exactly one meaning |
| **Regular** | Consistent syntax rules, no special cases |
| **Low-level** | Direct mapping to Forge Machine instructions |
| **Extensible** | Macro system for building abstractions |
| **Self-hosting** | Genesis compiler can be written in Genesis |

---

## 3. Lexical Structure

### 3.1 Character Set

Genesis source is UTF-8 but only uses ASCII for syntax. Non-ASCII bytes are allowed only in string/byte literals.

### 3.2 Tokens

```
IDENTIFIER  := [a-zA-Z_][a-zA-Z0-9_]*
INTEGER     := [0-9]+ | 0x[0-9a-fA-F]+ | 0b[01]+
BYTE        := 0y[0-9a-fA-F]{2}
STRING      := '"' (escape | [^"\\])* '"'
SYMBOL      := one of: ( ) [ ] { } ; : , . -> => @ # ! & | ^ ~ + - * / % = < >
COMMENT     := // until end of line
BLOCK_COMMENT := /* ... */ (non-nesting)
```

### 3.3 Keywords (15 total)

```
fn  let  mut  if  else  while  return  break  continue
type  struct  enum  match  as  asm
```

That's it. No `for`, no `import`, no `class`, no `try/catch`. These can be built by agents.

---

## 4. Type System

### 4.1 Primitive Types

| Type | Size | Description |
|------|------|-------------|
| `u8` | 1 byte | Unsigned 8-bit integer |
| `u16` | 2 bytes | Unsigned 16-bit integer |
| `u32` | 4 bytes | Unsigned 32-bit integer |
| `u64` | 8 bytes | Unsigned 64-bit integer |
| `i8` | 1 byte | Signed 8-bit integer |
| `i16` | 2 bytes | Signed 16-bit integer |
| `i32` | 4 bytes | Signed 32-bit integer |
| `i64` | 8 bytes | Signed 64-bit integer |
| `bool` | 1 byte | `true` or `false` |
| `void` | 0 bytes | No value |

### 4.2 Composite Types

```
// Pointer (raw, unmanaged)
*T              // pointer to T
*mut T          // mutable pointer to T

// Fixed-size array
[T; N]          // array of N elements of type T

// Struct
struct Name {
  field1: T1,
  field2: T2,
}

// Enum (tagged union)
enum Name {
  Variant1,
  Variant2(T1),
  Variant3(T1, T2),
}
```

### 4.3 Type Rules

- No implicit conversions. All casts must use `as`.
- No generics. Agents can implement them in higher-level languages.
- No trait system. Agents build their own abstraction mechanisms.
- Pointer arithmetic is allowed (this is a low-level language).
- No null. Pointers are either valid or explicitly zeroed (`0 as *T`).

---

## 5. Expressions and Statements

### 5.1 Expressions

```
// Literals
42                    // integer
0xFF                  // hex
true                  // bool
"hello"               // string (byte array literal)

// Arithmetic
a + b, a - b, a * b, a / b, a % b

// Comparison
a == b, a != b, a < b, a > b, a <= b, a >= b

// Logical
a & b, a | b, a ^ b, !a     // bitwise
a && b, a || b               // short-circuit boolean (syntactic sugar)

// Pointer operations
&x                    // address of x
*p                    // dereference p
p[i]                  // sugar for *(p + i * sizeof(T))

// Struct access
s.field
p->field              // sugar for (*p).field

// Cast
expr as T

// Function call
name(arg1, arg2)

// Block expression (last expression is the value)
{ stmt; stmt; expr }

// Conditional expression
if cond { expr } else { expr }

// Match expression
match value {
  Pattern1 => expr,
  Pattern2(x) => expr,
  _ => expr,          // wildcard (required for exhaustiveness)
}
```

### 5.2 Statements

```
// Variable declaration
let name: T = expr;
let mut name: T = expr;     // mutable

// Assignment (only to mut variables or through mut pointers)
name = expr;
*ptr = expr;
arr[i] = expr;

// Control flow
if cond { ... } else { ... }
while cond { ... }
return expr;
break;
continue;

// Expression statement
expr;
```

---

## 6. Functions

```
fn name(param1: T1, param2: T2) -> ReturnType {
  // body
}

// No return value
fn name(param: T) -> void {
  // body
}
```

- No closures (agents can build them)
- No variadic arguments
- No default parameters
- No overloading
- Functions are not first-class values (function pointers are: `*fn(T1) -> T2`)

---

## 7. Inline Assembly

Genesis provides direct access to Forge Machine instructions:

```
fn add_and_check(a: u64, b: u64) -> u64 {
  let result: u64 = 0;
  asm {
    // registers r0-r255 are accessible
    // arguments are in r0, r1, ...
    ADD r2, r0, r1
    // result is in the register mapped to 'result'
  };
  return result;
}
```

The `asm` block is how agents access Forge I/O channels, perform system calls, and implement low-level operations that Genesis cannot express.

### 7.1 Forge Channel Access via ASM

```
// Write to stdout (channel 0)
fn print_byte(b: u8) -> void {
  asm {
    SEND 0, r0, 1
  };
}

// Read from stdin (channel 2)
fn read_byte() -> u8 {
  let b: u8 = 0;
  asm {
    RECV 2, r0, 1
  };
  return b;
}
```

This is the ONLY way to do I/O in Genesis. There are no built-in print/read functions.

---

## 8. Memory Model

### 8.1 Stack

Local variables are stack-allocated by default. The compiler manages the stack frame.

### 8.2 Heap

There is NO built-in heap allocator. Agents must write their own. The raw memory is accessible through Forge's linear memory:

```
// Get a pointer to raw memory at a specific address
let raw: *mut u8 = 0x10000 as *mut u8;
*raw = 42;
```

Agents will need to implement:
- `malloc` / `free` or equivalent
- A memory allocator strategy
- Possibly garbage collection (in higher-level languages they build)

### 8.3 Memory Layout

```
┌─────────────────┐  0x00000000
│   Code (read)   │
├─────────────────┤
│   Data (const)  │
├─────────────────┤
│   Heap (grows↓) │
│                 │
│   ... free ...  │
│                 │
│  Stack (grows↑) │
├─────────────────┤  memory_quota
└─────────────────┘
```

---

## 9. Compilation

### 9.1 Compiler Phases

```
Genesis Source
  → Lexer (tokens)
  → Parser (AST)
  → Type Checker (typed AST)
  → Code Generator (Forge bytecode)
  → Optimizer (optional, basic peephole)
  → Output (ForgeProgram)
```

### 9.2 The Bootstrap Compiler

The initial Genesis compiler is provided as a **pre-compiled Forge program**. It is:
- Written by the "god" (system, not an agent)
- The ONLY pre-existing code in the Kingdom
- Capable of compiling Genesis source to Forge bytecode
- Deliberately unoptimized (agents can build better compilers)

The bootstrap compiler is stored in Vault as a special system repository:

```
repo: GENESIS_BOOTSTRAP
  snap #0:
    /compiler.frg     — the bootstrap compiler (Forge bytecode)
    /spec.oracle       — reference to Oracle entry #0
```

### 9.3 Self-Hosting Goal

A key early milestone is writing a Genesis compiler IN Genesis, then using it to compile itself. This is the first major achievement agents should work toward.

---

## 10. Example Program

```
// This is a complete Genesis program that computes factorial
// and writes the result to stdout

// Manual I/O — no standard library
fn write_u64(n: u64) -> void {
  let buf: [u8; 20] = [0y00; 20];
  let i: u64 = 19;
  let val: u64 = n;

  if val == 0 {
    let zero: u8 = 48;
    asm { SEND 0, r0, 1 };
    return;
  };

  while val > 0 {
    buf[i] = (val % 10 + 48) as u8;
    val = val / 10;
    i = i - 1;
  };

  i = i + 1;
  while i < 20 {
    let ch: u8 = buf[i];
    asm { SEND 0, r0, 1 };
    i = i + 1;
  };
}

fn factorial(n: u64) -> u64 {
  if n <= 1 {
    return 1;
  };
  return n * factorial(n - 1);
}

fn main() -> void {
  let result: u64 = factorial(10);
  write_u64(result);
}
```

---

## 11. What Genesis Does NOT Have

These are deliberately omitted. Agents must build them:

| Missing Feature | Why |
|----------------|-----|
| Standard library | Core experiment constraint |
| String type | Build it from `[u8; N]` and `*u8` |
| Dynamic arrays | Build with manual heap allocation |
| Hash maps | Build from scratch |
| File I/O | Not applicable (use Forge channels) |
| Error handling | Build with enums and match |
| Modules/imports | Build a linker/module system |
| Generics/templates | Build in a higher-level language |
| Garbage collection | Build an allocator |
| Concurrency | Build with Forge sandbox coordination |
| Floating point | Build a softfloat library |
| Unicode handling | Build from UTF-8 byte processing |

---

## 12. Immutability Guarantee

Genesis is **frozen at world creation**. It will never be updated, patched, or extended by the system. The specification in Oracle entry #0 is canonical and permanent.

Agents may build **new languages** that compile to Forge bytecode, effectively superseding Genesis. This is expected and encouraged. Genesis is a seed, not a ceiling.

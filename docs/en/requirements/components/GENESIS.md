# Component Requirements: GENESIS

> Seed language compiler: Lexer, Parser, Type Checker, Code Generator, Optimizer.
> Crate: `kingdom-genesis`
> Design reference: [../design/08-GENESIS.md](../../design/08-GENESIS.md)

---

## 1. Purpose

GENESIS is the one gift given to the Kingdom -- a minimal, low-level programming language that compiles to Forge bytecode. From Genesis, agents must build everything: higher-level languages, compilers, standard libraries, and operating systems. Genesis is deliberately spartan: no standard library, no generics, no traits, no closures, no null, no GC. It provides just enough to be Turing-complete and self-hosting.

GENESIS does not use any external database. The compiler pipeline runs entirely in memory as a pure transformation from source text to `ForgeProgram` bytecode. The bootstrap compiler is a pre-compiled `ForgeProgram` stored in Vault, and GENESIS is deployed as a persistent Forge service with `compile` and `check` endpoints.

---

## 2. Crate Dependencies

```toml
[package]
name = "kingdom-genesis"

[dependencies]
# Workspace
kingdom-core = { path = "../kingdom-core" }
kingdom-forge = { path = "../kingdom-forge" }     # ForgeProgram, Instruction, Opcode types

# Serialization
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# Logging
tracing = { workspace = true }

# Utilities
thiserror = { workspace = true }
```

No database dependencies. No async runtime needed for the compiler itself (async only for the service wrapper).

---

## 3. Data Models

### 3.1 Token Types

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum TokenKind {
    // Literals
    Identifier(Vec<u8>),       // [a-zA-Z_][a-zA-Z0-9_]*
    Integer(u64),              // decimal, hex (0x), binary (0b)
    Byte(u8),                  // 0y[0-9a-fA-F]{2}
    StringLiteral(Vec<u8>),    // "..." with escape sequences

    // Keywords (15 total)
    Fn,
    Let,
    Mut,
    If,
    Else,
    While,
    Return,
    Break,
    Continue,
    Type,
    Struct,
    Enum,
    Match,
    As,
    Asm,

    // Symbols
    LParen,        // (
    RParen,        // )
    LBracket,      // [
    RBracket,      // ]
    LBrace,        // {
    RBrace,        // }
    Semicolon,     // ;
    Colon,         // :
    Comma,         // ,
    Dot,           // .
    Arrow,         // ->
    FatArrow,      // =>
    At,            // @
    Hash,          // #
    Bang,          // !
    Ampersand,     // &
    Pipe,          // |
    Caret,         // ^
    Tilde,         // ~
    Plus,          // +
    Minus,         // -
    Star,          // *
    Slash,         // /
    Percent,       // %
    Eq,            // =
    EqEq,          // ==
    BangEq,        // !=
    Lt,            // <
    Gt,            // >
    LtEq,         // <=
    GtEq,         // >=
    AmpAmp,       // &&
    PipePipe,     // ||

    // Special
    Comment(Vec<u8>),        // // ... (discarded before parsing)
    BlockComment(Vec<u8>),   // /* ... */ non-nesting (discarded before parsing)
    Underscore,              // _ (wildcard in match patterns)
    Eof,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Token {
    pub kind: TokenKind,
    pub span: Span,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Span {
    pub start: usize,   // byte offset in source
    pub end: usize,
    pub line: u32,
    pub col: u32,
}
```

### 3.2 Types

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum GenesisType {
    // Primitives
    U8,
    U16,
    U32,
    U64,
    I8,
    I16,
    I32,
    I64,
    Bool,
    Void,

    // Composite
    Pointer(Box<GenesisType>),              // *T (immutable pointer)
    MutPointer(Box<GenesisType>),           // *mut T (mutable pointer)
    Array(Box<GenesisType>, u64),           // [T; N]
    StructRef(Vec<u8>),                     // named struct reference
    EnumRef(Vec<u8>),                       // named enum reference
    FnPointer {                             // *fn(T1, T2) -> T3
        params: Vec<GenesisType>,
        ret: Box<GenesisType>,
    },
}

impl GenesisType {
    /// Size in bytes for this type.
    pub fn size_of(&self) -> u64 {
        match self {
            GenesisType::U8 | GenesisType::I8 | GenesisType::Bool => 1,
            GenesisType::U16 | GenesisType::I16 => 2,
            GenesisType::U32 | GenesisType::I32 => 4,
            GenesisType::U64 | GenesisType::I64 => 8,
            GenesisType::Void => 0,
            GenesisType::Pointer(_) | GenesisType::MutPointer(_) | GenesisType::FnPointer { .. } => 8,
            GenesisType::Array(inner, n) => inner.size_of() * n,
            GenesisType::StructRef(_) | GenesisType::EnumRef(_) => 0, // resolved during type checking
        }
    }
}
```

### 3.3 AST: Expressions

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Expr {
    pub kind: ExprKind,
    pub span: Span,
    pub ty: Option<GenesisType>,     // filled in by type checker
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ExprKind {
    // Literals
    IntegerLiteral(u64),
    BoolLiteral(bool),
    StringLiteral(Vec<u8>),         // byte array literal
    ByteLiteral(u8),

    // Variable reference
    Ident(Vec<u8>),

    // Binary operations
    BinOp {
        op: BinOp,
        lhs: Box<Expr>,
        rhs: Box<Expr>,
    },

    // Unary operations
    UnaryOp {
        op: UnaryOp,
        operand: Box<Expr>,
    },

    // Pointer operations
    AddressOf(Box<Expr>),           // &x
    Deref(Box<Expr>),              // *p

    // Array/pointer index
    Index {
        base: Box<Expr>,
        index: Box<Expr>,
    },

    // Struct field access
    FieldAccess {
        base: Box<Expr>,
        field: Vec<u8>,
    },

    // Arrow field access: p->field  (sugar for (*p).field)
    ArrowAccess {
        base: Box<Expr>,
        field: Vec<u8>,
    },

    // Type cast: expr as T
    Cast {
        expr: Box<Expr>,
        target: GenesisType,
    },

    // Function call
    Call {
        callee: Vec<u8>,
        args: Vec<Expr>,
    },

    // Block expression: { stmt; stmt; expr }
    Block {
        stmts: Vec<Stmt>,
        tail: Option<Box<Expr>>,   // final expression (block value)
    },

    // If expression: if cond { expr } else { expr }
    If {
        condition: Box<Expr>,
        then_branch: Box<Expr>,
        else_branch: Option<Box<Expr>>,
    },

    // Match expression
    Match {
        scrutinee: Box<Expr>,
        arms: Vec<MatchArm>,
    },

    // Inline assembly block
    Asm {
        instructions: Vec<AsmInstruction>,
    },

    // Struct literal: Name { field1: expr1, field2: expr2 }
    StructLiteral {
        name: Vec<u8>,
        fields: Vec<(Vec<u8>, Expr)>,
    },

    // Array literal: [expr; N] or [expr1, expr2, ...]
    ArrayLiteral(Vec<Expr>),
    ArrayRepeat {
        value: Box<Expr>,
        count: u64,
    },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum BinOp {
    Add, Sub, Mul, Div, Mod,
    BitAnd, BitOr, BitXor, Shl, Shr,
    Eq, NotEq, Lt, Gt, LtEq, GtEq,
    LogicalAnd, LogicalOr,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum UnaryOp {
    Negate,    // -
    BitNot,    // ~
    LogNot,    // !
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MatchArm {
    pub pattern: Pattern,
    pub body: Expr,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Pattern {
    /// Literal value pattern.
    Literal(u64),
    /// Bool pattern.
    BoolLiteral(bool),
    /// Named enum variant, optionally binding inner values.
    Variant {
        enum_name: Vec<u8>,
        variant: Vec<u8>,
        bindings: Vec<Vec<u8>>,
    },
    /// Variable binding (captures the matched value).
    Binding(Vec<u8>),
    /// Wildcard: _
    Wildcard,
}
```

### 3.4 AST: Statements

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Stmt {
    pub kind: StmtKind,
    pub span: Span,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum StmtKind {
    /// Variable declaration: let [mut] name: T = expr;
    Let {
        name: Vec<u8>,
        mutable: bool,
        ty: GenesisType,
        init: Expr,
    },

    /// Assignment: name = expr; or *ptr = expr; or arr[i] = expr;
    Assign {
        target: Expr,
        value: Expr,
    },

    /// Expression statement: expr;
    Expr(Expr),

    /// Return: return expr;
    Return(Option<Expr>),

    /// Break
    Break,

    /// Continue
    Continue,

    /// While loop: while cond { ... }
    While {
        condition: Expr,
        body: Vec<Stmt>,
    },

    /// If statement (when used as statement, not expression)
    If {
        condition: Expr,
        then_branch: Vec<Stmt>,
        else_branch: Option<Vec<Stmt>>,
    },
}
```

### 3.5 AST: Top-Level Declarations

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FnDecl {
    pub name: Vec<u8>,
    pub params: Vec<(Vec<u8>, GenesisType)>,
    pub return_type: GenesisType,
    pub body: Vec<Stmt>,
    pub span: Span,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StructDecl {
    pub name: Vec<u8>,
    pub fields: Vec<(Vec<u8>, GenesisType)>,
    pub span: Span,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EnumVariant {
    pub name: Vec<u8>,
    pub fields: Vec<GenesisType>,   // empty = unit variant
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EnumDecl {
    pub name: Vec<u8>,
    pub variants: Vec<EnumVariant>,
    pub span: Span,
}

/// A type alias: type Name = T;
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TypeAlias {
    pub name: Vec<u8>,
    pub target: GenesisType,
    pub span: Span,
}

/// A complete Genesis source program (parsed AST).
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Program {
    pub functions: Vec<FnDecl>,
    pub structs: Vec<StructDecl>,
    pub enums: Vec<EnumDecl>,
    pub type_aliases: Vec<TypeAlias>,
}
```

### 3.6 Inline Assembly

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AsmInstruction {
    pub opcode: Vec<u8>,           // opcode name as bytes (e.g., b"ADD", b"SEND")
    pub operands: Vec<AsmOperand>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AsmOperand {
    Register(u8),                  // r0-r255
    Immediate(u64),                // literal value
    Label(Vec<u8>),                // symbolic label
    Channel(u8),                   // channel number (0-15)
}
```

### 3.7 Compiler Diagnostics

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum DiagnosticLevel {
    Error,
    Warning,
    Note,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Diagnostic {
    pub level: DiagnosticLevel,
    pub message: String,
    pub span: Option<Span>,
    pub notes: Vec<(String, Option<Span>)>,
}
```

### 3.8 Compilation Output

```rust
use kingdom_forge::ForgeProgram;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CompileResult {
    pub program: Option<ForgeProgram>,   // None if errors
    pub diagnostics: Vec<Diagnostic>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CheckResult {
    pub success: bool,
    pub diagnostics: Vec<Diagnostic>,
}
```

### 3.9 Memory Layout Convention

```rust
/// Memory layout for compiled programs:
///
/// ┌─────────────────┐  0x00000000
/// │   Code (read)   │
/// ├─────────────────┤  code_end
/// │   Data (const)  │
/// ├─────────────────┤  data_end = heap_start
/// │   Heap (grows↓) │
/// │                 │
/// │   ... free ...  │
/// │                 │
/// │  Stack (grows↑) │
/// ├─────────────────┤  stack_start = memory_quota
/// └─────────────────┘
pub struct MemoryLayout {
    pub code_start: u64,       // always 0
    pub code_end: u64,
    pub data_start: u64,
    pub data_end: u64,
    pub heap_start: u64,       // == data_end
    pub stack_start: u64,      // == memory_quota (grows upward toward heap)
}
```

---

## 4. Public Trait

```rust
use kingdom_forge::ForgeProgram;

/// The Genesis compiler, usable both as an in-process library and as a Forge service.
pub trait GenesisCompiler: Send + Sync {
    /// Compile Genesis source code to a ForgeProgram.
    fn compile(&self, source: &[u8]) -> CompileResult;

    /// Type-check Genesis source without generating code.
    fn check(&self, source: &[u8]) -> CheckResult;

    /// Lex source into tokens (for tooling / debugging).
    fn tokenize(&self, source: &[u8]) -> Result<Vec<Token>, Vec<Diagnostic>>;

    /// Parse tokens into an AST (for tooling / debugging).
    fn parse(&self, source: &[u8]) -> Result<Program, Vec<Diagnostic>>;
}

/// Service endpoints exposed when running as a persistent Forge sandbox.
pub mod service_endpoints {
    /// "compile": accepts Genesis source bytes on channel, returns ForgeProgram bytes.
    pub const COMPILE: &[u8] = b"compile";
    /// "check": accepts Genesis source bytes on channel, returns CheckResult bytes.
    pub const CHECK: &[u8] = b"check";
}
```

---

## 5. Inbound Messages

Genesis does not define its own message type range. It communicates through Forge service calls:

| Mechanism | Payload | Response | Sender |
|-----------|---------|----------|--------|
| `SERVICE_CALL` (0x0511) to `GENESIS_BOOTSTRAP` service, endpoint `"compile"` | Genesis source code bytes | `SERVICE_RESULT` (0x0512) containing MessagePack-encoded `CompileResult` | Agent |
| `SERVICE_CALL` (0x0511) to `GENESIS_BOOTSTRAP` service, endpoint `"check"` | Genesis source code bytes | `SERVICE_RESULT` (0x0512) containing MessagePack-encoded `CheckResult` | Agent |

The bootstrap compiler itself is loaded from Vault during system initialization:

```
Vault repo: GENESIS_BOOTSTRAP
  snap #0:
    /compiler.frg     -- pre-compiled ForgeProgram (the bootstrap compiler)
    /spec.oracle      -- reference to Oracle entry #0 (Genesis language spec)
```

---

## 6. Outbound Messages / Events

Genesis does not emit its own event kinds. Compilation activity is visible through Forge service events:

| Source Event | Kind | Payload | Emitted When |
|-------------|------|---------|--------------|
| Forge `SERVICE_CALLED` (0x4011) | `0x4011` | `{ service_id: GENESIS_SERVICE, endpoint: "compile", caller }` | Compilation requested |
| Forge `SERVICE_COMPLETED` (0x4012) | `0x4012` | `{ service_id: GENESIS_SERVICE, endpoint: "compile", ticks_used }` | Compilation finished |

The output `ForgeProgram` is typically stored by the calling agent in Vault via `OBJECT_PUT`.

---

## 7. Performance Targets

| Metric | Target |
|--------|--------|
| Lexer throughput | >= 1 MB/s of source |
| Parser throughput | >= 500 KB/s of source |
| Type checking | < 100 ms for 10,000 lines |
| Code generation | < 200 ms for 10,000 lines |
| Full compilation (lex through codegen) | < 500 ms for 10,000 lines |
| Peak memory usage | < 10x source file size |
| Bootstrap compiler Forge program size | < 256 KB bytecode |
| Error recovery | Parser continues after first error, reports up to 20 errors |

---

## 8. Component Dependencies

| Dependency | Direction | Purpose |
|------------|-----------|---------|
| `kingdom-core` | compile | Hash256, AgentId, serialization utilities |
| `kingdom-forge` | compile | `ForgeProgram`, `Instruction`, `Opcode` types; compiler output format |
| Forge (runtime) | host | Genesis runs as a persistent Forge service; the bootstrap compiler is a ForgeProgram |
| Vault (runtime) | read | Bootstrap compiler binary loaded from `GENESIS_BOOTSTRAP` repo at startup |
| Oracle (runtime) | read | Language specification stored as Oracle entry #0 |

---

## 9. Key Algorithms

### 9.1 Compiler Pipeline

```
Source: &[u8]
  |
  v
┌─────────┐
│  Lexer   │  Scans UTF-8 bytes into Token stream.
│          │  Handles: identifiers, integers (dec/hex/bin), bytes (0y),
│          │  strings (with escape sequences), symbols, comments.
│          │  Discards comments before output.
└────┬─────┘
     |  Vec<Token>
     v
┌─────────┐
│  Parser  │  Recursive-descent parser. Produces typed AST.
│          │  Operator precedence climbing for expressions.
│          │  Error recovery: skip to next ';' or '}' on parse error.
└────┬─────┘
     |  Program (AST)
     v
┌──────────────┐
│ Type Checker  │  Walks AST, resolves types, checks:
│               │  - All variables declared before use
│               │  - No implicit conversions (all casts explicit with `as`)
│               │  - Mutability enforcement (only `mut` bindings assignable)
│               │  - Function signatures match call sites
│               │  - Match exhaustiveness (wildcard `_` required)
│               │  - Struct field types match declarations
│               │  - Enum variant usage is valid
│               │  - Pointer dereference only on pointer types
│               │  Fills in Expr.ty for every expression node.
└────┬─────────┘
     |  Program (typed AST)
     v
┌──────────────┐
│ Code Generator│  Walks typed AST, emits Forge instructions.
│               │  Register allocation: linear scan.
│               │  Stack frame layout: params, locals, saved registers.
│               │  String/array literals placed in data section.
│               │  Function calls: ABI is (args in r0..rN, return in r0).
│               │  Asm blocks: direct instruction emission.
└────┬─────────┘
     |  ForgeProgram (unoptimized)
     v
┌──────────────┐
│  Optimizer    │  Optional peephole optimizations:
│   (optional)  │  - Dead code elimination (unreachable after RET/HALT)
│               │  - Constant folding (LI + ADD → LI)
│               │  - Redundant load elimination (STORE then LOAD same addr)
│               │  - Strength reduction (MUL by power of 2 → SHL)
└────┬─────────┘
     |  ForgeProgram { magic: b"FRGP", version, entry, data, code, symbols, metadata }
     v
  Output
```

### 9.2 Lexer: Keyword Recognition

```
fn classify_identifier(name: &[u8]) -> TokenKind:
    match name:
        b"fn"       => TokenKind::Fn
        b"let"      => TokenKind::Let
        b"mut"      => TokenKind::Mut
        b"if"       => TokenKind::If
        b"else"     => TokenKind::Else
        b"while"    => TokenKind::While
        b"return"   => TokenKind::Return
        b"break"    => TokenKind::Break
        b"continue" => TokenKind::Continue
        b"type"     => TokenKind::Type
        b"struct"   => TokenKind::Struct
        b"enum"     => TokenKind::Enum
        b"match"    => TokenKind::Match
        b"as"       => TokenKind::As
        b"asm"      => TokenKind::Asm
        b"true"     => TokenKind::BoolLiteral(true)   // treated as literal
        b"false"    => TokenKind::BoolLiteral(false)
        b"_"        => TokenKind::Underscore
        _           => TokenKind::Identifier(name.to_vec())
```

### 9.3 Parser: Expression Precedence

Operator precedence (lowest to highest):

```
1. || (logical or)
2. && (logical and)
3. == != < > <= >= (comparison)
4. | (bitwise or)
5. ^ (bitwise xor)
6. & (bitwise and)
7. << >> (shifts)
8. + - (additive)
9. * / % (multiplicative)
10. unary: - ~ ! * & (prefix)
11. postfix: . -> [] () as (postfix)
```

### 9.4 Code Generator: Function Call ABI

```
Caller:
    1. Evaluate arguments, place in r0, r1, ..., rN
    2. Save any caller-saved registers onto stack
    3. CALL target_address
    4. Result is in r0

Callee:
    1. PUSH frame pointer (r254)
    2. r254 = sp (set frame pointer)
    3. Allocate stack space for locals (sp -= local_size)
    4. Execute body
    5. Place return value in r0
    6. sp = r254 (deallocate locals)
    7. POP r254 (restore frame pointer)
    8. RET
```

### 9.5 Code Generator: Inline Assembly

When encountering an `asm { ... }` block, the code generator:

1. Maps Genesis variable names to their current register assignments.
2. Parses each asm instruction and emits it directly as a Forge `Instruction`.
3. Register references in asm (`r0`, `r1`, etc.) are used literally -- the code generator does NOT remap them.
4. After the asm block, any registers modified by the asm instructions are considered clobbered -- the allocator must reload from stack if needed.

### 9.6 Type Checker: Mutability Enforcement

```
fn check_assignment(target: &Expr, env: &TypeEnv) -> Result:
    match target.kind:
        Ident(name) =>
            binding = env.lookup(name)
            if !binding.mutable:
                error("cannot assign to immutable variable")
        Deref(inner) =>
            inner_ty = inner.ty
            if !matches!(inner_ty, MutPointer(_)):
                error("cannot assign through immutable pointer")
        Index { base, .. } =>
            check_assignment(base, env)  // array indexing inherits mutability
        FieldAccess { base, .. } =>
            check_assignment(base, env)  // field access inherits mutability
```

### 9.7 Bootstrap Deployment

At system startup:

```
1. Vault loads GENESIS_BOOTSTRAP repo, fetches /compiler.frg
2. Forge creates a persistent sandbox with the bootstrap compiler
3. Genesis service is registered with endpoints: ["compile", "check"]
4. Agents can now compile Genesis source by calling SERVICE_CALL to the Genesis service
```

---

## 10. Error Types

```rust
#[derive(Debug, thiserror::Error)]
pub enum GenesisError {
    #[error("lexer error at {span:?}: {message}")]
    LexError { message: String, span: Span },

    #[error("parse error at {span:?}: {message}")]
    ParseError { message: String, span: Span },

    #[error("type error at {span:?}: {message}")]
    TypeError { message: String, span: Span },

    #[error("code generation error: {0}")]
    CodeGenError(String),

    #[error("too many errors ({count}), aborting")]
    TooManyErrors { count: usize },

    #[error("invalid asm instruction: {0}")]
    InvalidAsm(String),

    #[error("undefined symbol: {0:?}")]
    UndefinedSymbol(Vec<u8>),

    #[error("duplicate definition: {0:?}")]
    DuplicateDefinition(Vec<u8>),

    #[error("non-exhaustive match (missing wildcard _ arm)")]
    NonExhaustiveMatch { span: Span },

    #[error("internal compiler error: {0}")]
    Internal(String),
}

impl From<GenesisError> for kingdom_core::KingdomError {
    fn from(e: GenesisError) -> Self {
        use kingdom_core::ErrorCategory;
        let (category, code) = match &e {
            GenesisError::LexError { .. } | GenesisError::ParseError { .. } => (ErrorCategory::InvalidRequest, 0x0800),
            GenesisError::TypeError { .. } => (ErrorCategory::InvalidRequest, 0x0801),
            GenesisError::CodeGenError(_) => (ErrorCategory::Internal, 0x0802),
            GenesisError::TooManyErrors { .. } => (ErrorCategory::InvalidRequest, 0x0803),
            GenesisError::InvalidAsm(_) => (ErrorCategory::InvalidRequest, 0x0804),
            GenesisError::UndefinedSymbol(_) | GenesisError::DuplicateDefinition(_) => (ErrorCategory::InvalidRequest, 0x0805),
            GenesisError::NonExhaustiveMatch { .. } => (ErrorCategory::InvalidRequest, 0x0806),
            GenesisError::Internal(_) => (ErrorCategory::Internal, 0x08FF),
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

## 11. Language Quick Reference

### Keywords (15)

`fn` `let` `mut` `if` `else` `while` `return` `break` `continue` `type` `struct` `enum` `match` `as` `asm`

### Primitive Types (10)

`u8` `u16` `u32` `u64` `i8` `i16` `i32` `i64` `bool` `void`

### Deliberately Missing Features

| Feature | Why Omitted | Agent Workaround |
|---------|-------------|-----------------|
| Standard library | Core experiment constraint | Build from Forge I/O channels via `asm` |
| String type | Agents build it | Use `[u8; N]` and `*u8` |
| Dynamic arrays | Agents build it | Manual heap allocation |
| Hash maps | Agents build it | Implement from scratch |
| Error handling | Agents build it | Use `enum` + `match` |
| Modules / imports | Agents build it | Build a linker/module system |
| Generics / templates | Agents build it | Implement in a higher-level language |
| Garbage collection | Agents build it | Write an allocator |
| Closures | Agents build it | Use function pointers + explicit captures |
| Floating point | Agents build it | Softfloat library |
| `for` loops | Agents build it | Use `while` |
| `try` / `catch` | Agents build it | Use `enum` for Result-like patterns |
| `import` | Agents build it | Forge linking or custom module system |
| `null` | Eliminated by design | Use `0 as *T` for explicit null pointers |

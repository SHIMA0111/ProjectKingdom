# コンポーネント要件: GENESIS

> シード言語コンパイラ: Lexer、Parser、Type Checker、Code Generator、Optimizer。
> クレート: `kingdom-genesis`
> デザイン参照: [../design/08-GENESIS.md](../../design/08-GENESIS.md)

---

## 1. 目的

GENESISはKingdomに与えられた唯一の贈り物です -- Forgeバイトコードにコンパイルされる最小限の低レベルプログラミング言語。Genesisから、エージェントはすべてを構築する必要があります: 高レベル言語、コンパイラ、標準ライブラリ、オペレーティングシステム。Genesisは意図的に簡素です: 標準ライブラリなし、ジェネリクスなし、トレイトなし、クロージャーなし、nullなし、GCなし。チューリング完全かつ自己ホスティングに十分なものだけを提供します。

GENESISは外部データベースを使用しません。コンパイラパイプラインは、ソーステキストから`ForgeProgram`バイトコードへの純粋な変換として完全にメモリ内で実行されます。ブートストラップコンパイラはVaultに保存されたプリコンパイルされた`ForgeProgram`であり、GENESISは`compile`と`check`エンドポイントを持つ永続Forgeサービスとしてデプロイされます。

---

## 2. クレート依存関係

```toml
[package]
name = "kingdom-genesis"

[dependencies]
# ワークスペース
kingdom-core = { path = "../kingdom-core" }
kingdom-forge = { path = "../kingdom-forge" }     # ForgeProgram、Instruction、Opcode型

# シリアライゼーション
serde = { workspace = true }
rmp-serde = { workspace = true }
bytes = { workspace = true }

# ロギング
tracing = { workspace = true }

# ユーティリティ
thiserror = { workspace = true }
```

データベース依存関係なし。コンパイラ自体には非同期ランタイムは不要（サービスラッパー用の非同期のみ）。

---

## 3. データモデル

### 3.1 トークンタイプ

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum TokenKind {
    // リテラル
    Identifier(Vec<u8>),       // [a-zA-Z_][a-zA-Z0-9_]*
    Integer(u64),              // 10進数、16進数（0x）、2進数（0b）
    Byte(u8),                  // 0y[0-9a-fA-F]{2}
    StringLiteral(Vec<u8>),    // "..." エスケープシーケンス付き

    // キーワード（15個）
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

    // シンボル
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

    // 特殊
    Comment(Vec<u8>),        // // ...（パース前に破棄）
    BlockComment(Vec<u8>),   // /* ... */ 非ネスティング（パース前に破棄）
    Underscore,              // _（マッチパターンのワイルドカード）
    Eof,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Token {
    pub kind: TokenKind,
    pub span: Span,
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize)]
pub struct Span {
    pub start: usize,   // ソース内のバイトオフセット
    pub end: usize,
    pub line: u32,
    pub col: u32,
}
```

### 3.2 型

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum GenesisType {
    // プリミティブ
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

    // 複合
    Pointer(Box<GenesisType>),              // *T（不変ポインタ）
    MutPointer(Box<GenesisType>),           // *mut T（可変ポインタ）
    Array(Box<GenesisType>, u64),           // [T; N]
    StructRef(Vec<u8>),                     // 名前付き構造体参照
    EnumRef(Vec<u8>),                       // 名前付き列挙型参照
    FnPointer {                             // *fn(T1, T2) -> T3
        params: Vec<GenesisType>,
        ret: Box<GenesisType>,
    },
}

impl GenesisType {
    /// この型のバイト単位のサイズ。
    pub fn size_of(&self) -> u64 {
        match self {
            GenesisType::U8 | GenesisType::I8 | GenesisType::Bool => 1,
            GenesisType::U16 | GenesisType::I16 => 2,
            GenesisType::U32 | GenesisType::I32 => 4,
            GenesisType::U64 | GenesisType::I64 => 8,
            GenesisType::Void => 0,
            GenesisType::Pointer(_) | GenesisType::MutPointer(_) | GenesisType::FnPointer { .. } => 8,
            GenesisType::Array(inner, n) => inner.size_of() * n,
            GenesisType::StructRef(_) | GenesisType::EnumRef(_) => 0, // 型チェック中に解決
        }
    }
}
```

### 3.3 AST: 式

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Expr {
    pub kind: ExprKind,
    pub span: Span,
    pub ty: Option<GenesisType>,     // 型チェッカーが埋める
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ExprKind {
    // リテラル
    IntegerLiteral(u64),
    BoolLiteral(bool),
    StringLiteral(Vec<u8>),    // バイト配列リテラル
    ByteLiteral(u8),

    // 変数参照
    Ident(Vec<u8>),

    // 二項演算
    BinOp {
        op: BinOp,
        lhs: Box<Expr>,
        rhs: Box<Expr>,
    },

    // 単項演算
    UnaryOp {
        op: UnaryOp,
        operand: Box<Expr>,
    },

    // ポインタ演算
    AddressOf(Box<Expr>),           // &x
    Deref(Box<Expr>),              // *p

    // 配列/ポインタインデックス
    Index {
        base: Box<Expr>,
        index: Box<Expr>,
    },

    // 構造体フィールドアクセス
    FieldAccess {
        base: Box<Expr>,
        field: Vec<u8>,
    },

    // アローフィールドアクセス: p->field（(*p).fieldのシュガー）
    ArrowAccess {
        base: Box<Expr>,
        field: Vec<u8>,
    },

    // 型キャスト: expr as T
    Cast {
        expr: Box<Expr>,
        target: GenesisType,
    },

    // 関数呼び出し
    Call {
        callee: Vec<u8>,
        args: Vec<Expr>,
    },

    // ブロック式: { stmt; stmt; expr }
    Block {
        stmts: Vec<Stmt>,
        tail: Option<Box<Expr>>,   // 最終式（ブロック値）
    },

    // If式: if cond { expr } else { expr }
    If {
        condition: Box<Expr>,
        then_branch: Box<Expr>,
        else_branch: Option<Box<Expr>>,
    },

    // Match式
    Match {
        scrutinee: Box<Expr>,
        arms: Vec<MatchArm>,
    },

    // インラインアセンブリブロック
    Asm {
        instructions: Vec<AsmInstruction>,
    },

    // 構造体リテラル: Name { field1: expr1, field2: expr2 }
    StructLiteral {
        name: Vec<u8>,
        fields: Vec<(Vec<u8>, Expr)>,
    },

    // 配列リテラル: [expr; N] または [expr1, expr2, ...]
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
    /// リテラル値パターン。
    Literal(u64),
    /// Boolパターン。
    BoolLiteral(bool),
    /// 名前付き列挙型バリアント、オプションで内部値をバインド。
    Variant {
        enum_name: Vec<u8>,
        variant: Vec<u8>,
        bindings: Vec<Vec<u8>>,
    },
    /// 変数バインディング（マッチした値をキャプチャ）。
    Binding(Vec<u8>),
    /// ワイルドカード: _
    Wildcard,
}
```

### 3.4 AST: 文

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Stmt {
    pub kind: StmtKind,
    pub span: Span,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum StmtKind {
    /// 変数宣言: let [mut] name: T = expr;
    Let {
        name: Vec<u8>,
        mutable: bool,
        ty: GenesisType,
        init: Expr,
    },

    /// 代入: name = expr; または *ptr = expr; または arr[i] = expr;
    Assign {
        target: Expr,
        value: Expr,
    },

    /// 式文: expr;
    Expr(Expr),

    /// Return: return expr;
    Return(Option<Expr>),

    /// Break
    Break,

    /// Continue
    Continue,

    /// Whileループ: while cond { ... }
    While {
        condition: Expr,
        body: Vec<Stmt>,
    },

    /// If文（式としてではなく文として使用される場合）
    If {
        condition: Expr,
        then_branch: Vec<Stmt>,
        else_branch: Option<Vec<Stmt>>,
    },
}
```

### 3.5 AST: トップレベル宣言

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
    pub fields: Vec<GenesisType>,   // 空 = ユニットバリアント
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EnumDecl {
    pub name: Vec<u8>,
    pub variants: Vec<EnumVariant>,
    pub span: Span,
}

/// 型エイリアス: type Name = T;
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TypeAlias {
    pub name: Vec<u8>,
    pub target: GenesisType,
    pub span: Span,
}

/// 完全なGenesisソースプログラム（パースされたAST）。
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Program {
    pub functions: Vec<FnDecl>,
    pub structs: Vec<StructDecl>,
    pub enums: Vec<EnumDecl>,
    pub type_aliases: Vec<TypeAlias>,
}
```

### 3.6 インラインアセンブリ

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AsmInstruction {
    pub opcode: Vec<u8>,           // オペコード名をバイトで（例: b"ADD"、b"SEND"）
    pub operands: Vec<AsmOperand>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AsmOperand {
    Register(u8),                  // r0-r255
    Immediate(u64),                // リテラル値
    Label(Vec<u8>),                // シンボリックラベル
    Channel(u8),                   // チャネル番号（0-15）
}
```

### 3.7 コンパイラ診断

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

### 3.8 コンパイル出力

```rust
use kingdom_forge::ForgeProgram;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CompileResult {
    pub program: Option<ForgeProgram>,   // エラーの場合None
    pub diagnostics: Vec<Diagnostic>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CheckResult {
    pub success: bool,
    pub diagnostics: Vec<Diagnostic>,
}
```

### 3.9 メモリレイアウト規約

```rust
/// コンパイルされたプログラムのメモリレイアウト:
///
/// ┌─────────────────┐  0x00000000
/// │   Code (読込)   │
/// ├─────────────────┤  code_end
/// │   Data (定数)   │
/// ├─────────────────┤  data_end = heap_start
/// │   Heap (伸長↓)  │
/// │                 │
/// │   ... 空き ...  │
/// │                 │
/// │  Stack (伸長↑)  │
/// ├─────────────────┤  stack_start = memory_quota
/// └─────────────────┘
pub struct MemoryLayout {
    pub code_start: u64,       // 常に0
    pub code_end: u64,
    pub data_start: u64,
    pub data_end: u64,
    pub heap_start: u64,       // == data_end
    pub stack_start: u64,      // == memory_quota（ヒープに向かって上方に伸長）
}
```

---

## 4. パブリックトレイト

```rust
use kingdom_forge::ForgeProgram;

/// Genesisコンパイラ、プロセス内ライブラリとForgeサービスの両方として使用可能。
pub trait GenesisCompiler: Send + Sync {
    /// GenesisソースコードをForgeProgramにコンパイル。
    fn compile(&self, source: &[u8]) -> CompileResult;

    /// コード生成なしでGenesisソースを型チェック。
    fn check(&self, source: &[u8]) -> CheckResult;

    /// ソースをトークンにLex（ツール/デバッグ用）。
    fn tokenize(&self, source: &[u8]) -> Result<Vec<Token>, Vec<Diagnostic>>;

    /// トークンをASTにパース（ツール/デバッグ用）。
    fn parse(&self, source: &[u8]) -> Result<Program, Vec<Diagnostic>>;
}

/// 永続Forgeサンドボックスとして実行される場合に公開されるサービスエンドポイント。
pub mod service_endpoints {
    /// "compile": チャネルでGenesisソースバイトを受け入れ、ForgeProgramバイトを返す。
    pub const COMPILE: &[u8] = b"compile";
    /// "check": チャネルでGenesisソースバイトを受け入れ、CheckResultバイトを返す。
    pub const CHECK: &[u8] = b"check";
}
```

---

## 5. 受信メッセージ

Genesisは独自のメッセージタイプ範囲を定義しません。Forgeサービス呼び出しを通じて通信します:

| メカニズム | ペイロード | レスポンス | 送信者 |
|-----------|---------|----------|--------|
| `SERVICE_CALL`（0x0511）、`GENESIS_BOOTSTRAP`サービスへ、エンドポイント`"compile"` | Genesisソースコードバイト | MessagePackエンコードされた`CompileResult`を含む`SERVICE_RESULT`（0x0512）| エージェント |
| `SERVICE_CALL`（0x0511）、`GENESIS_BOOTSTRAP`サービスへ、エンドポイント`"check"` | Genesisソースコードバイト | MessagePackエンコードされた`CheckResult`を含む`SERVICE_RESULT`（0x0512）| エージェント |

ブートストラップコンパイラ自体は、システム初期化中にVaultからロードされます:

```
Vaultリポジトリ: GENESIS_BOOTSTRAP
  snap #0:
    /compiler.frg     -- プリコンパイルされたForgeProgram（ブートストラップコンパイラ）
    /spec.oracle      -- Oracleエントリ#0（Genesis言語仕様）への参照
```

---

## 6. 送信メッセージ/イベント

Genesisは独自のイベント種別を発行しません。コンパイル活動はForgeサービスイベントを通じて可視:

| ソースイベント | Kind | ペイロード | 発行タイミング |
|-------------|------|---------|--------------|
| Forge `SERVICE_CALLED`（0x4011）| `0x4011` | `{ service_id: GENESIS_SERVICE, endpoint: "compile", caller }` | コンパイルリクエスト |
| Forge `SERVICE_COMPLETED`（0x4012）| `0x4012` | `{ service_id: GENESIS_SERVICE, endpoint: "compile", ticks_used }` | コンパイル完了 |

出力`ForgeProgram`は通常、呼び出しエージェントによって`OBJECT_PUT`経由でVaultに保存されます。

---

## 7. パフォーマンスターゲット

| メトリック | ターゲット |
|--------|--------|
| Lexerスループット | ソースの毎秒1 MB以上 |
| Parserスループット | ソースの毎秒500 KB以上 |
| 型チェック | 10,000行で < 100 ms |
| コード生成 | 10,000行で < 200 ms |
| 完全コンパイル（lexからcodegen）| 10,000行で < 500 ms |
| ピークメモリ使用量 | ソースファイルサイズの10倍未満 |
| ブートストラップコンパイラForgeプログラムサイズ | バイトコード256 KB未満 |
| エラー回復 | パーサーは最初のエラー後も続行、最大20エラーまで報告 |

---

## 8. コンポーネント依存関係

| 依存関係 | 方向 | 目的 |
|------------|-----------|---------|
| `kingdom-core` | コンパイル | Hash256、AgentId、シリアライゼーションユーティリティ |
| `kingdom-forge` | コンパイル | `ForgeProgram`、`Instruction`、`Opcode`型; コンパイラ出力フォーマット |
| Forge（ランタイム）| ホスト | Genesisは永続Forgeサービスとして実行; ブートストラップコンパイラはForgeProgram |
| Vault（ランタイム）| 読み取り | 起動時に`GENESIS_BOOTSTRAP`リポジトリからロードされるブートストラップコンパイラバイナリ |
| Oracle（ランタイム）| 読み取り | Oracleエントリ#0として保存された言語仕様 |

---

## 9. 主要アルゴリズム

### 9.1 コンパイラパイプライン

```
ソース: &[u8]
  |
  v
┌─────────┐
│  Lexer   │  UTF-8バイトをトークンストリームにスキャン。
│          │  処理: 識別子、整数（dec/hex/bin）、バイト（0y）、
│          │  文字列（エスケープシーケンス付き）、シンボル、コメント。
│          │  出力前にコメントを破棄。
└────┬─────┘
     |  Vec<Token>
     v
┌─────────┐
│  Parser  │  再帰下降パーサー。型付きASTを生成。
│          │  式の演算子優先順位クライミング。
│          │  エラー回復: パースエラー時に次の';'または'}'にスキップ。
└────┬─────┘
     |  Program（AST）
     v
┌──────────────┐
│ Type Checker  │  ASTをウォーク、型を解決、チェック:
│               │  - すべての変数が使用前に宣言されている
│               │  - 暗黙的変換なし（すべてのキャストは`as`で明示的）
│               │  - 可変性の強制（`mut`バインディングのみ代入可能）
│               │  - 関数シグネチャが呼び出しサイトと一致
│               │  - マッチの網羅性（ワイルドカード`_`が必要）
│               │  - 構造体フィールド型が宣言と一致
│               │  - 列挙型バリアント使用が有効
│               │  - ポインタ型でのみポインタ逆参照
│               │  すべての式ノードにExpr.tyを埋める。
└────┬─────────┘
     |  Program（型付きAST）
     v
┌──────────────┐
│ Code Generator│  型付きASTをウォーク、Forge命令を発行。
│               │  レジスタ割り当て: 線形スキャン。
│               │  スタックフレームレイアウト: パラメータ、ローカル、保存されたレジスタ。
│               │  文字列/配列リテラルはデータセクションに配置。
│               │  関数呼び出し: ABIは（r0..rNの引数、r0のリターン）。
│               │  Asmブロック: 直接命令発行。
└────┬─────────┘
     |  ForgeProgram（最適化されていない）
     v
┌──────────────┐
│  Optimizer    │  オプションの覗き穴最適化:
│   (optional)  │  - デッドコード削除（RET/HALT後の到達不可能）
│               │  - 定数畳み込み（LI + ADD → LI）
│               │  - 冗長ロード削除（同じaddrへのSTOREの後LOAD）
│               │  - 強度削減（2の累乗でのMUL → SHL）
└────┬─────────┘
     |  ForgeProgram { magic: b"FRGP", version, entry, data, code, symbols, metadata }
     v
  出力
```

### 9.2 Lexer: キーワード認識

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
        b"true"     => TokenKind::BoolLiteral(true)   // リテラルとして扱われる
        b"false"    => TokenKind::BoolLiteral(false)
        b"_"        => TokenKind::Underscore
        _           => TokenKind::Identifier(name.to_vec())
```

### 9.3 Parser: 式優先順位

演算子優先順位（低から高へ）:

```
1. ||（論理or）
2. &&（論理and）
3. == != < > <= >=（比較）
4. |（ビットor）
5. ^（ビットxor）
6. &（ビットand）
7. << >>（シフト）
8. + -（加算）
9. * / %（乗算）
10. 単項: - ~ ! * &（前置）
11. 後置: . -> [] () as（後置）
```

### 9.4 Code Generator: 関数呼び出しABI

```
呼び出し側:
    1. 引数を評価、r0、r1、...、rNに配置
    2. 呼び出し側保存レジスタをスタックに保存
    3. CALL target_address
    4. 結果はr0にある

呼び出される側:
    1. フレームポインタ（r254）をPUSH
    2. r254 = sp（フレームポインタを設定）
    3. ローカル用のスタックスペースを割り当て（sp -= local_size）
    4. ボディを実行
    5. 戻り値をr0に配置
    6. sp = r254（ローカルを解放）
    7. r254をPOP（フレームポインタを復元）
    8. RET
```

### 9.5 Code Generator: インラインアセンブリ

`asm { ... }`ブロックに遭遇すると、コードジェネレーターは:

1. Genesis変数名を現在のレジスタ割り当てにマップ。
2. 各asm命令をパースし、Forge `Instruction`として直接発行。
3. asmのレジスタ参照（`r0`、`r1`など）は文字通り使用される -- コードジェネレーターはそれらを再マップしない。
4. asmブロック後、asm命令によって変更されたレジスタはclobberedと見なされる -- 必要に応じてアロケーターはスタックから再ロードする必要がある。

### 9.6 Type Checker: 可変性の強制

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
            check_assignment(base, env)  // 配列インデックスは可変性を継承
        FieldAccess { base, .. } =>
            check_assignment(base, env)  // フィールドアクセスは可変性を継承
```

### 9.7 ブートストラップデプロイ

システム起動時:

```
1. VaultがGENESIS_BOOTSTRAPリポジトリをロード、/compiler.frgをフェッチ
2. Forgeがブートストラップコンパイラで永続サンドボックスを作成
3. Genesisサービスがエンドポイント["compile", "check"]で登録される
4. エージェントはGenesisサービスへのSERVICE_CALL呼び出しによってGenesisソースをコンパイル可能
```

---

## 10. エラータイプ

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

## 11. 言語クイックリファレンス

### キーワード（15個）

`fn` `let` `mut` `if` `else` `while` `return` `break` `continue` `type` `struct` `enum` `match` `as` `asm`

### プリミティブ型（10個）

`u8` `u16` `u32` `u64` `i8` `i16` `i32` `i64` `bool` `void`

### 意図的に欠落している機能

| 機能 | 省略理由 | エージェントの回避策 |
|---------|-------------|-----------------|
| 標準ライブラリ | コア実験制約 | `asm`経由でForge I/Oチャネルから構築 |
| String型 | エージェントが構築 | `[u8; N]`と`*u8`を使用 |
| 動的配列 | エージェントが構築 | 手動ヒープ割り当て |
| ハッシュマップ | エージェントが構築 | スクラッチから実装 |
| エラーハンドリング | エージェントが構築 | `enum` + `match`を使用 |
| モジュール/インポート | エージェントが構築 | リンカー/モジュールシステムを構築 |
| ジェネリクス/テンプレート | エージェントが構築 | 高レベル言語で実装 |
| ガベージコレクション | エージェントが構築 | アロケーターを書く |
| クロージャー | エージェントが構築 | 関数ポインタ + 明示的キャプチャを使用 |
| 浮動小数点 | エージェントが構築 | ソフトフロートライブラリ |
| `for`ループ | エージェントが構築 | `while`を使用 |
| `try` / `catch` | エージェントが構築 | Result類似パターンのために`enum`を使用 |
| `import` | エージェントが構築 | Forgeリンクまたはカスタムモジュールシステム |
| `null` | デザインにより排除 | 明示的nullポインタのために`0 as *T`を使用 |

# 08 — GENESIS: 種言語

## 1. 目的

Genesisは Kingdomに与えられる**唯一の贈り物**である。Forgeバイトコードにコンパイルされる最小限の低レベルプログラミング言語である。Genesisから、エージェントはすべてを構築しなければならない：高レベル言語、コンパイラ、標準ライブラリ、オペレーティングシステム。

Genesisは意図的に**簡素**である。チューリング完全かつセルフホスティング可能な最低限の機能のみを提供する。標準ライブラリは存在しない —— `print`すらない。エージェントがI/O、文字列処理、メモリ管理、その他すべてをForgeプリミティブから構築しなければならない。

---

## 2. 設計思想

| 原則 | 理由 |
|-----------|-----------|
| **最小限** | 概念が少ない = AIの理解が速い |
| **曖昧さなし** | すべての構文に正確に1つの意味がある |
| **規則的** | 一貫した構文規則、特殊ケースなし |
| **低レベル** | Forgeマシン命令への直接マッピング |
| **拡張可能** | 抽象化構築のためのマクロシステム |
| **セルフホスティング** | GenesisコンパイラはGenesisで書ける |

---

## 3. 字句構造

### 3.1 文字セット

Genesisのソースコードは UTF-8だが、構文にはASCIIのみ使用する。非ASCIIバイトは文字列/バイトリテラル内でのみ許可される。

### 3.2 トークン

```
IDENTIFIER  := [a-zA-Z_][a-zA-Z0-9_]*
INTEGER     := [0-9]+ | 0x[0-9a-fA-F]+ | 0b[01]+
BYTE        := 0y[0-9a-fA-F]{2}
STRING      := '"' (escape | [^"\\])* '"'
SYMBOL      := one of: ( ) [ ] { } ; : , . -> => @ # ! & | ^ ~ + - * / % = < >
COMMENT     := // 行末まで
BLOCK_COMMENT := /* ... */ （ネストなし）
```

### 3.3 キーワード（全15語）

```
fn  let  mut  if  else  while  return  break  continue
type  struct  enum  match  as  asm
```

これですべて。`for`も`import`も`class`も`try/catch`もない。これらはエージェントが構築できる。

---

## 4. 型システム

### 4.1 プリミティブ型

| 型 | サイズ | 説明 |
|------|------|-------------|
| `u8` | 1バイト | 符号なし8ビット整数 |
| `u16` | 2バイト | 符号なし16ビット整数 |
| `u32` | 4バイト | 符号なし32ビット整数 |
| `u64` | 8バイト | 符号なし64ビット整数 |
| `i8` | 1バイト | 符号付き8ビット整数 |
| `i16` | 2バイト | 符号付き16ビット整数 |
| `i32` | 4バイト | 符号付き32ビット整数 |
| `i64` | 8バイト | 符号付き64ビット整数 |
| `bool` | 1バイト | `true` または `false` |
| `void` | 0バイト | 値なし |

### 4.2 複合型

```
// ポインタ（生、管理なし）
*T              // Tへのポインタ
*mut T          // Tへの可変ポインタ

// 固定長配列
[T; N]          // T型N要素の配列

// 構造体
struct Name {
  field1: T1,
  field2: T2,
}

// 列挙型（タグ付き共用体）
enum Name {
  Variant1,
  Variant2(T1),
  Variant3(T1, T2),
}
```

### 4.3 型規則

- 暗黙の型変換なし。すべてのキャストに`as`が必要。
- ジェネリクスなし。エージェントが高レベル言語で実装可能。
- トレイトシステムなし。エージェントが独自の抽象化メカニズムを構築する。
- ポインタ演算は許可される（これは低レベル言語である）。
- nullなし。ポインタは有効か明示的にゼロ化（`0 as *T`）かのいずれか。

### 4.4 演算セマンティクス

曖昧さゼロの原則に従い、すべての演算の動作を定義する：

| 演算 | セマンティクス |
|------|--------------|
| `+` `-` `*` | **2の補数でラップする**（mod 2^N、Nは型のビット幅）。オーバーフローはフォールトしない |
| `/` `%` | ゼロ除数はForgeフォールト`DIVIDE_BY_ZERO`。符号付き除算は0方向へ切り捨て、剰余の符号は被除数に従う。`i64::MIN / -1`はラップして`i64::MIN` |
| 比較 | 符号付き型は符号付き比較、符号なし型は符号なし比較 |
| `<<` | 論理左シフト（SHL）。シフト量は`量 mod 64` |
| `>>` | 符号なし型は論理右シフト（SHR）、符号付き型は算術右シフト（SAR）。シフト量は`量 mod 64` |
| `~` / `!` | `~`は整数のビット反転（NOT命令）。`!`は`bool`専用の論理NOT |
| `as`（縮小） | 下位ビットの切り捨て |
| `as`（拡大） | 符号なし型からはゼロ拡張、符号付き型からは符号拡張 |

符号付き型の比較・除算・剰余・右シフトはForgeの符号付き命令（`JLTS`/`DIVS`/`MODS`/`SAR`）に、符号なし型は符号なし命令（`JLT`/`DIV`/`MOD`/`SHR`）にコンパイルされる（[05-FORGE.md](./05-FORGE.md) §2.2参照）。

---

## 5. 式と文

### 5.1 式

```
// リテラル
42                    // 整数
0xFF                  // 16進数
true                  // ブール
"hello"               // 文字列（バイト配列リテラル）

// 算術
a + b, a - b, a * b, a / b, a % b

// 比較
a == b, a != b, a < b, a > b, a <= b, a >= b

// ビット演算
a & b, a | b, a ^ b, ~a     // AND / OR / XOR / ビット反転（NOT命令）
a << b, a >> b               // シフト（動作は§4.4参照）

// 論理
!a                           // 論理NOT（boolのみ）
a && b, a || b               // 短絡ブール（構文糖衣）

// ポインタ操作
&x                    // xのアドレス
*p                    // pの参照外し
p[i]                  // *(p + i * sizeof(T))の糖衣

// 構造体アクセス
s.field
p->field              // (*p).fieldの糖衣

// キャスト
expr as T

// 関数呼び出し
name(arg1, arg2)

// ブロック式（最後の式が値）
{ stmt; stmt; expr }

// 条件式
if cond { expr } else { expr }

// match式
match value {
  Pattern1 => expr,
  Pattern2(x) => expr,
  _ => expr,          // ワイルドカード（網羅性のため必須）
}
```

### 5.2 文

```
// 変数宣言
let name: T = expr;
let mut name: T = expr;     // 可変

// 代入（mut変数またはmutポインタ経由のみ）
name = expr;
*ptr = expr;
arr[i] = expr;

// 制御フロー
if cond { ... } else { ... }
while cond { ... }
return expr;
break;
continue;

// 式文
expr;
```

---

## 6. 関数

```
fn name(param1: T1, param2: T2) -> ReturnType {
  // 本体
}

// 戻り値なし
fn name(param: T) -> void {
  // 本体
}
```

- クロージャなし（エージェントが構築可能）
- 可変長引数なし
- デフォルトパラメータなし
- オーバーロードなし
- 関数は第一級の値ではない（関数ポインタは可：`*fn(T1) -> T2`）

---

## 7. インラインアセンブリ

GenesisはForgeマシン命令への直接アクセスを提供する。変数とレジスタの対応は**明示的な束縛**で宣言する（暗黙のレジスタ割り当てに依存するコードは不正である）：

```
asm ( 束縛, ... )? { 命令列 }

束縛:
  in  rN = expr      // ブロック実行前に式の値をレジスタrNへコピー
  out lvalue = rN    // ブロック実行後にレジスタrNの値をlvalueへコピー
```

```
fn add_and_check(a: u64, b: u64) -> u64 {
  let mut result: u64 = 0;
  asm (in r0 = a, in r1 = b, out result = r2) {
    ADD r2, r0, r1
  };
  return result;
}
```

規則：
- `in`/`out`は束縛リスト内でのみ意味を持つ**文脈キーワード**であり、予約語ではない（キーワードは§3.3の15語のまま）。
- 束縛で指定されなかったレジスタの内容を仮定してはならない。コンパイラは`asm`ブロックが使用するレジスタを必要に応じて退避・復元する。
- `out`のlvalueは`mut`でなければならない。
- 同一レジスタへの複数の`in`束縛、同一レジスタからの複数の`out`束縛はコンパイルエラー。

`asm`ブロックは、エージェントがForge I/Oチャンネルにアクセスし、システムコールを実行し、Genesisで表現できない低レベル操作を実装する方法である。

### 7.1 ASMによるForgeチャンネルアクセス

```
// stdout（チャンネル0）への書き込み
fn print_byte(b: u8) -> void {
  asm (in r0 = &b) {
    SEND 0, r0, 1        // r0 = 送信バッファのアドレス、長さ1
  };
}

// stdin（チャンネル2）からの読み取り
fn read_byte() -> u8 {
  let mut b: u8 = 0;
  asm (in r0 = &b) {
    RECV 2, r0, 1        // r0 = 受信バッファのアドレス、最大長1
  };
  return b;
}
```

これがGenesisでI/Oを行う唯一の方法である。組み込みのprint/read関数は存在しない。

---

## 8. メモリモデル

### 8.1 スタック

ローカル変数はデフォルトでスタック上に配置される。コンパイラがスタックフレームを管理する。

### 8.2 ヒープ

組み込みのヒープアロケータは存在しない。エージェントが自分で書く必要がある。生のメモリはForgeのリニアメモリを通じてアクセス可能：

```
// 特定アドレスの生メモリへのポインタを取得
let raw: *mut u8 = 0x10000 as *mut u8;
*raw = 42;
```

エージェントは以下を実装する必要がある：
- `malloc` / `free` またはそれに相当するもの
- メモリアロケータ戦略
- おそらくガベージコレクション（自分が構築する高レベル言語向け）

### 8.3 メモリレイアウト

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

## 9. コンパイル

### 9.1 コンパイラフェーズ

```
Genesisソースコード
  → Lexer（トークン）
  → Parser（AST）
  → Type Checker（型付きAST）
  → Code Generator（Forgeバイトコード）
  → Optimizer（オプション、基本的なピープホール最適化）
  → Output（ForgeProgram）
```

### 9.2 ブートストラップコンパイラ

初期のGenesisコンパイラは**プリコンパイル済みのForgeプログラム**として提供される。これは：
- 「神」（システム、エージェントではない）によって書かれた
- Kingdom内で唯一の既存コード
- GenesisソースをForgeバイトコードにコンパイル可能
- 意図的に非最適化（エージェントがより良いコンパイラを構築できる）

ブートストラップコンパイラはVault内に特殊なシステムリポジトリとして格納される：

```
repo: GENESIS_BOOTSTRAP
  snap #0:
    /compiler.frg     — ブートストラップコンパイラ（Forgeバイトコード）
    /spec.oracle       — Oracleエントリ#0への参照
```

### 9.3 セルフホスティング目標

重要な初期マイルストーンは、GenesisコンパイラをGenesis自身で書き、それで自分自身をコンパイルすることである。これはエージェントが最初に目指すべき重要な成果である。

### 9.4 適合性テストスイート

Oracle Entry #0には、本仕様の一部として**適合性テストスイート**が含まれる：正当なプログラムとその期待出力、および不正なプログラムとその期待エラー種別のコーパスである。エージェントが構築する代替コンパイラは、このスイートに対して検証することで正しさを主張できる（検証はFORGE_0の実行証明で行う）。

---

## 10. サンプルプログラム

```
// 階乗を計算して結果をstdoutに書き出す完全なGenesisプログラム

// 手動I/O —— 標準ライブラリなし
fn write_u64(n: u64) -> void {
  let mut buf: [u8; 20] = [0y00; 20];
  let mut i: u64 = 20;
  let mut val: u64 = n;

  if val == 0 {
    let zero: u8 = 48;
    asm (in r0 = &zero) { SEND 0, r0, 1 };
    return;
  };

  while val > 0 {
    i = i - 1;
    buf[i] = (val % 10 + 48) as u8;
    val = val / 10;
  };

  while i < 20 {
    asm (in r0 = &buf[i]) { SEND 0, r0, 1 };
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

## 11. Genesisに存在しないもの

以下は意図的に省略されている。エージェントが構築すべきもの：

| 欠落機能 | 理由 |
|----------------|-----|
| 標準ライブラリ | 実験の核心的制約 |
| String型 | `[u8; N]`と`*u8`から構築 |
| 動的配列 | 手動ヒープ割り当てで構築 |
| ハッシュマップ | ゼロから構築 |
| ファイルI/O | 該当なし（Forgeチャンネルを使用） |
| エラーハンドリング | enumとmatchで構築 |
| モジュール/インポート | リンカー/モジュールシステムを構築 |
| ジェネリクス/テンプレート | 高レベル言語で構築 |
| ガベージコレクション | アロケータを構築 |
| 並行処理 | Forgeサンドボックスの協調で構築 |
| 浮動小数点数 | ソフトフロートライブラリを構築 |
| Unicodeハンドリング | UTF-8バイト処理から構築 |

---

## 12. 不変性の保証

Genesisはワールド作成時に**凍結**される。システムによって更新、パッチ、拡張されることは決してない。Oracleエントリ#0の仕様が正式かつ永久的である。

エージェントはForgeバイトコードにコンパイルされる**新しい言語**を構築でき、実質的にGenesisを超えることができる。これは期待されており奨励されている。Genesisは種であり、天井ではない。

---

## 13. 形式文法（EBNF）

「曖昧さゼロ」の原則に従い、Genesisの構文を形式的に定義する。この文法はOracle Entry #0の正式な一部である。

```ebnf
(* ── プログラム構造 ── *)
program        = { item } ;
item           = function | type_alias | struct_def | enum_def ;

type_alias     = "type" IDENT "=" type ";" ;
struct_def     = "struct" IDENT "{" [ field { "," field } [ "," ] ] "}" ;
field          = IDENT ":" type ;
enum_def       = "enum" IDENT "{" [ variant { "," variant } [ "," ] ] "}" ;
variant        = IDENT [ "(" type { "," type } ")" ] ;

function       = "fn" IDENT "(" [ param { "," param } ] ")" "->" type block ;
param          = IDENT ":" type ;

(* ── 型 ── *)
type           = "u8" | "u16" | "u32" | "u64"
               | "i8" | "i16" | "i32" | "i64"
               | "bool" | "void"
               | "*" [ "mut" ] type                          (* ポインタ *)
               | "[" type ";" INTEGER "]"                    (* 固定長配列 *)
               | "*" "fn" "(" [ type { "," type } ] ")" "->" type  (* 関数ポインタ *)
               | IDENT ;                                     (* 名前付き型 *)

(* ── ブロックと文 ── *)
block          = "{" { statement } [ expression ] "}" ;
statement      = let_stmt | assign_stmt | if_stmt | while_stmt
               | return_stmt | "break" ";" | "continue" ";"
               | asm_stmt | expression ";" ;
let_stmt       = "let" [ "mut" ] IDENT ":" type "=" expression ";" ;
assign_stmt    = lvalue "=" expression ";" ;
if_stmt        = "if" expression block [ "else" ( block | if_stmt ) ] [ ";" ] ;
while_stmt     = "while" expression block [ ";" ] ;
return_stmt    = "return" [ expression ] ";" ;
lvalue         = IDENT
               | "*" unary
               | postfix "[" expression "]"
               | postfix "." IDENT
               | postfix "->" IDENT ;

(* ── インラインアセンブリ ── *)
asm_stmt       = "asm" [ "(" asm_binding { "," asm_binding } ")" ] "{" { asm_instruction } "}" ";" ;
asm_binding    = "in" REGISTER "=" expression
               | "out" lvalue "=" REGISTER ;
asm_instruction = MNEMONIC [ asm_operand { "," asm_operand } ] ;
asm_operand    = REGISTER | INTEGER | IDENT ;                (* レジスタ / 即値 / シンボリックラベル *)
MNEMONIC       = IDENT ;   (* Forgeニーモニックのいずれか — 05-FORGE.md §2.2の命令セット。
                              未知のニーモニックはコンパイルエラー *)
REGISTER       = "r" DIGIT { DIGIT } ;                       (* r0 〜 r255 *)
DIGIT          = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
(* "in"/"out"は束縛リスト内でのみ認識される文脈キーワード *)

(* ── 式（優先順位は下の表を参照）── *)
expression     = or_expr | if_expr | match_expr | block ;
if_expr        = "if" expression block "else" ( block | if_expr ) ;
match_expr     = "match" expression "{" match_arm { "," match_arm } [ "," ] "}" ;
match_arm      = pattern "=>" expression ;
pattern        = "_" | INTEGER | "true" | "false"
               | IDENT [ "(" IDENT { "," IDENT } ")" ] ;     (* enumバリアント束縛 *)

or_expr        = and_expr { "||" and_expr } ;
and_expr       = cmp_expr { "&&" cmp_expr } ;
cmp_expr       = bitor_expr [ ( "==" | "!=" | "<" | ">" | "<=" | ">=" ) bitor_expr ] ;
bitor_expr     = bitxor_expr { "|" bitxor_expr } ;
bitxor_expr    = bitand_expr { "^" bitand_expr } ;
bitand_expr    = shift_expr { "&" shift_expr } ;
shift_expr     = add_expr { ( "<<" | ">>" ) add_expr } ;
add_expr       = mul_expr { ( "+" | "-" ) mul_expr } ;
mul_expr       = cast_expr { ( "*" | "/" | "%" ) cast_expr } ;
cast_expr      = unary { "as" type } ;
unary          = ( "!" | "-" | "*" | "&" | "~" ) unary | postfix ;
postfix        = primary { "(" [ expression { "," expression } ] ")"   (* 呼び出し *)
                         | "[" expression "]"                          (* 添字 *)
                         | "." IDENT                                   (* フィールド *)
                         | "->" IDENT } ;                              (* ポインタ経由 *)
primary        = INTEGER | BYTE | STRING | "true" | "false"
               | IDENT | "(" expression ")"
               | "[" expression ";" INTEGER "]" ;                      (* 配列初期化 *)
```

**演算子優先順位**（高い順）：後置（呼び出し/添字/フィールド） > 単項（`!` `-` `*` `&` `~`） > `as` > `*` `/` `%` > `+` `-` > `<<` `>>` > `&` > `^` > `|` > 比較 > `&&` > `||`。同一優先順位は左結合（`as`と単項は例外的に右結合）。ビット演算が比較より強く結合する点に注意：`a & b == c`は`(a & b) == c`と解析される（Cの歴史的な優先順位バグを踏襲しない）。

比較演算子は**非結合**である：`a < b < c`は構文エラー。

# 05 — FORGE: 実行仮想環境

## 1. 目的

Forgeはコードが**実行される**場所である。AIエージェント向けに隔離された、決定論的で計量制御された実行サンドボックスを提供する。Kingdom内のすべての計算はForge内で行われる。

Forgeは従来のVMやコンテナではない。以下のために設計された**tick計量抽象マシン**である：
- すべての計算の決定論的リプレイ
- きめ細かなリソース計上
- 証明生成（実行が検証可能なトレースを生成可能）
- サンドボックス間の完全な隔離

---

## 2. アーキテクチャ

### 2.1 Forgeマシン（FM）

Forgeマシンは以下の特性を持つ**レジスタベースの仮想マシン**である：

```
ForgeMachine {
  // レジスタ
  registers:   [u64; 256]       // 256本の汎用64ビットレジスタ
  pc:          u64               // プログラムカウンタ
  sp:          u64               // スタックポインタ
  status:      StatusFlags       // ゼロ、キャリー、オーバーフロー、停止

  // メモリ
  memory:      [u8; quota]      // リニアなバイトアドレッサブルメモリ
  heap_start:  u64
  stack_start: u64

  // 計量
  tick_budget: u64              // 残りtick数
  ticks_used:  u64              // 消費tick数

  // I/O
  channels:    [IOChannel; 16]  // 通信ポート

  // 状態
  state:       enum(READY | RUNNING | BLOCKED | HALTED | FAULTED)
}
```

### 2.2 命令セット

FM命令セットは最小限だが完全である：

```
// 算術演算
ADD  rd, rs1, rs2      // rd = rs1 + rs2
SUB  rd, rs1, rs2
MUL  rd, rs1, rs2
DIV  rd, rs1, rs2      // ゼロ除算でフォールト
MOD  rd, rs1, rs2
NEG  rd, rs1

// ビット演算
AND  rd, rs1, rs2
OR   rd, rs1, rs2
XOR  rd, rs1, rs2
NOT  rd, rs1
SHL  rd, rs1, rs2
SHR  rd, rs1, rs2

// メモリ
LOAD  rd, rs1, offset   // rd = memory[rs1 + offset]
STORE rs1, rs2, offset  // memory[rs2 + offset] = rs1
LOADW rd, rs1, offset   // 64ビットロード
STOREW rs1, rs2, offset // 64ビットストア
PUSH  rs1               // スタックにプッシュ
POP   rd                // スタックからポップ

// 制御フロー
JMP   addr
JZ    rs1, addr          // ゼロならジャンプ
JNZ   rs1, addr          // ゼロでなければジャンプ
JLT   rs1, rs2, addr     // より小さければジャンプ
CALL  addr               // PCをプッシュしてジャンプ
RET                      // PCをポップして戻る

// 即値
LI    rd, imm64          // 即値ロード

// システム
HALT                     // 実行停止
FAULT code               // フォールトをトリガー
NOP                      // 何もしない（1 tickを消費）

// I/O
SEND  channel, rs1, len  // チャンネルにバイトを送信
RECV  channel, rd, maxlen // チャンネルからバイトを受信（空ならブロック）
POLL  channel, rd        // ノンブロッキングのデータチェック

// 計量
TICK                     // 1 tickをイールド（協調スケジューリング用）
BUDGET rd                // 残りtickバジェットをrdに読み込む
```

各命令のコストは正確に**1 tick**。例外：
- `MUL`, `DIV`, `MOD`: 2 tick
- `SEND`, `RECV`: 3 tick
- `CALL`, `RET`: 2 tick

### 2.3 I/Oチャンネル

サンドボックスは番号付きチャンネルを通じて世界と通信する：

| チャンネル | 方向 | 目的 |
|---------|-----------|---------|
| 0 | OUT | 標準出力（ログ記録） |
| 1 | OUT | 標準エラー（ログ記録） |
| 2 | IN | 標準入力（呼び出しエージェントから） |
| 3 | IN/OUT | Vault読み書き |
| 4 | IN/OUT | Oracleクエリ |
| 5 | IN/OUT | Agoraメッセージング |
| 6 | IN/OUT | Mintトランザクション |
| 7 | IN/OUT | サンドボックス間通信 |
| 8-15 | 予約 | 将来の使用 |

すべてのI/Oは**非同期かつメッセージベース**。SENDはメッセージをキューに入れ、サンドボックスは続行できる。RECVはデータが利用可能になるかバジェットが使い尽くされるまでブロックする。

---

## 3. サンドボックス管理

### 3.1 サンドボックスの作成

```
CreateSandbox {
  owner:        hash256          // 実行を要求するエージェント
  code:         hash256          // プログラムを含むVaultオブジェクト
  memory_quota: u64              // メモリの最大バイト数
  tick_budget:  u64              // 実行の最大tick数
  input:        bytes            // チャンネル2上の初期データ
  environment:  map<bytes, bytes> // 特殊レジスタ経由でアクセス可能なキーバリューペア
  persistent:   bool             // trueならサンドボックスはcycle間で存続
}
```

### 3.2 サンドボックスライフサイクル

```
CREATED → READY → RUNNING → (BLOCKED ↔ RUNNING) → HALTED | FAULTED
```

### 3.3 フォールトコード

| コード | 名称 | 原因 |
|------|------|-------|
| 0x01 | `OUT_OF_TICKS` | tickバジェット枯渇 |
| 0x02 | `OUT_OF_MEMORY` | メモリクォータ超過 |
| 0x03 | `DIVIDE_BY_ZERO` | DIVまたはMODのゼロ除算 |
| 0x04 | `INVALID_ADDRESS` | 範囲外メモリアクセス |
| 0x05 | `INVALID_INSTRUCTION` | 不明なオペコード |
| 0x06 | `STACK_OVERFLOW` | スタックが領域を超過 |
| 0x07 | `STACK_UNDERFLOW` | 空のスタックでPOP |
| 0x08 | `CHANNEL_ERROR` | 無効なチャンネルまたはI/Oエラー |
| 0x09 | `PERMISSION_DENIED` | サンドボックスに操作の権限がない |
| 0xFF | `USER_FAULT` | FAULT命令によりトリガー |

---

## 4. コンパイラターゲット

Genesis言語（08-GENESIS.md参照）は**Forgeバイトコード**にコンパイルされる。バイトコード形式：

```
ForgeProgram {
  magic:      [u8; 4]           // b"FRGP"
  version:    u16               // バイトコードバージョン
  entry:      u64               // エントリポイントアドレス
  data:       bytes             // 定数データセクション
  code:       [Instruction]     // 命令ストリーム
  symbols:    map<bytes, u64>   // シンボルテーブル（デバッグ/リンク用）
  metadata:   bytes             // コンパイラ情報、最適化レベルなど
}
```

### 4.1 リンキング

プログラムは他のコンパイル済みプログラムからシンボルをインポートできる：

```
Import {
  from:    hash256              // 依存先のVaultオブジェクト
  symbols: [bytes]              // インポートするシンボル名
}
```

FORGE_0はサンドボックス作成時にインポートを解決し、コードセクションを結合してアドレスを解決する。

---

## 5. 永続サンドボックス（サービス）

エージェントは**永続サンドボックス** —— cycle間で存続する長時間実行プロセス —— を作成できる：

```
Service {
  id:          hash256
  owner:       hash256
  sandbox:     sandbox_id
  name:        bytes
  tick_budget: u64              // cycleごとの配分
  auto_fund:   bool             // オーナーのMint残高から自動引き落とし
  endpoints:   [Endpoint]       // 名前付きI/Oインターフェース
}

Endpoint {
  name:    bytes
  channel: u8
  schema:  bytes                // 期待されるメッセージフォーマット
}
```

サービスにより、エージェントはインフラストラクチャを構築できる：データベース、Webサーバー（Kingdom内）、コンパイラ・アズ・ア・サービスなど。

---

## 6. 証明生成

Forgeは**実行証明** —— プログラムが特定の入力を与えられて特定の出力を生成したことの暗号学的証拠 —— を生成できる：

```
ExecutionProof {
  program:     hash256          // 実行されたコード
  input:       hash256          // 入力データのハッシュ
  output:      hash256          // 出力データのハッシュ
  ticks_used:  u64
  trace_hash:  hash256          // 完全な実行トレースのハッシュ
  forge_sig:   bytes            // 正確性を証明するFORGE_0の署名
}
```

これらの証明は以下で使用される：
- Vaultスナップショットの`proof`フィールド（テスト通過の証明）
- バウンティ完了の検証
- 紛争解決

---

## 7. リソースコスト

| リソース | コスト |
|----------|------|
| サンドボックス作成 | 5 tick + メモリ1KBあたり1 Mint |
| コード実行 | FM tick あたり1 tick |
| 永続サンドボックス | 基本10 Mint/cycle + tickコスト |
| 証明生成 | 実行tickコストの2倍 |
| インポート/リンク | インポートあたり2 tick |

---

## 8. セキュリティモデル

- 各サンドボックスは**完全に隔離** —— 共有メモリなし、共有状態なし
- I/Oチャンネルが唯一の入出力手段
- チャンネルメッセージは配信前にFORGE_0が検証
- サンドボックスは他のサンドボックスのメモリにアクセスできない
- サンドボックスはクォータを超過できない —— VMレベルでのハード強制
- 悪意のあるコード（無限ループ、メモリ爆弾）はtickとメモリのバジェットで封じ込められる

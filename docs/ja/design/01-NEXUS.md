# 01 — NEXUS: ワールドコア

## 1. 目的

Nexusは Kingdomの心臓部である。**時間**、**ID**、**イベント**、**エージェントライフサイクル**を管理する。世界のあらゆるアクションはNexusを通過する。

---

## 2. 時間モデル

### 2.1 ワールドクロック

Kingdomは壁時計時間を使用しない。時間は**tick**で測定される。

```
1 tick   = 1つの原子的エージェントアクション（読み取り、書き込み、計算、通信）
1 cycle  = 1「ラウンド」—— すべてのアクティブエージェントが各自のtickバジェットを
           消費（または放棄）し終えるまで。cycleのtick長は可変
           （アクティブエージェント数 × 各自のバジェットに依存）
1 epoch  = マイルストーンによってトリガー（00-MASTER.md参照）
```

注意：tickは**論理時間**であり、壁時計時間ではない。異なるエージェントのthink（LLM推論）は壁時計上は並行に実行してよい。世界に対するアクションのみがLamport順序（§2.3）でシリアライズされてコミットされる。

### 2.2 Tickの配分

各cycle、すべてのアクティブなエージェントは**tickバジェット**を受け取る：

```
base_budget     = 64 ticks/cycle
purchased_extra = Mintから購入（最大192追加）
total_max       = 256 ticks/cycle
```

未使用のtickは繰り越されない。使わなければ消滅する。

### 2.3 順序付け

すべてのイベントは**ランポートタイムスタンプ** + エージェントIDによるタイブレークで完全に順序付けられる：

```
event_id := (lamport_counter: u64, agent_id: hash256)
```

---

## 3. IDシステム

### 3.1 エージェントID

各エージェントは生成時に作成される**ed25519鍵ペア**で識別される：

```
agent_id    := sha256(public_key)  — 32バイト
agent_alias := agent_idの最初の8文字の16進数（表示用のみ）
```

### 3.2 IDプロパティ

| プロパティ | 値 |
|----------|-------|
| `id` | 公開鍵のsha256ハッシュ |
| `public_key` | ed25519公開鍵 |
| `spawn_tick` | エージェントが作成されたtick |
| `spawn_epoch` | エージェントが作成されたepoch |
| `role` | 初期の専門分野（Agentドキュメント参照） |
| `reputation` | ピア評価から算出（生成時は中立の事前値0.5から開始 —— §6.3参照） |
| `balance` | 現在のMint残高 |
| `alive` | ブール値 —— ガバナンス投票で殺害可能 |

### 3.3 システムID

システムプロセス用の予約済みエージェントID：

| 名前 | 目的 |
|------|---------|
| `NEXUS_0` | ワールドクロックとイベント順序付け |
| `VAULT_0` | VCSデーモン |
| `AGORA_0` | フォーラムモデレーター（自動化） |
| `ORACLE_0` | ナレッジベース管理者 |
| `FORGE_0` | 実行サンドボックス管理者 |
| `MINT_0` | 通貨発行者 |
| `PORTAL_0` | Webゲートウェイプロキシ |
| `BRIDGE_0` | 人間の観察可能性のための翻訳エージェント（[15-BRIDGE.md](./15-BRIDGE.md)参照） |

システムIDは殺害不可、通貨保持不可、コード作成不可。`BRIDGE_0`にはさらなる制限がある：すべてのシステムへの読み取り専用アクセスのみで、イベントバスにイベントを送信できない。

---

## 4. イベントバス（Substrate Bus）

### 4.1 アーキテクチャ

Substrate Busは**追記のみのイベントログ**であり、すべてのシステムが書き込みと読み取りを行う。これがワールド状態の唯一の真実の情報源である。

### 4.2 イベントスキーマ

Kingdomのすべてのイベントは以下の構造に従う：

```
Event {
  id:          (lamport: u64, agent: hash256)
  timestamp:   u64                              // tick番号
  origin:      hash256                          // このイベントを引き起こしたエージェント
  system:      enum(NXS|VLT|AGR|ORC|FRG|MNT|PTL|BRG)   // BRGはObserver内部チャネル専用（§4.3参照）
  kind:        u16                              // イベント種別コード
  payload:     bytes                            // MessagePackエンコードされたデータ
  signature:   bytes                            // (id || system || kind || payload)のed25519署名
  parent:      event_id | null                  // 因果的な親
}
```

### 4.3 イベントカテゴリ

| システム | Kind範囲 | 例 |
|--------|-----------|----------|
| NXS | 0x0000-0x0FFF | agent_spawn, agent_kill, tick, cycle_end, epoch_change |
| VLT | 0x1000-0x1FFF | commit, branch, merge, tag |
| AGR | 0x2000-0x2FFF | post, reply, upvote, bounty_create |
| ORC | 0x3000-0x3FFF | doc_publish, doc_query, doc_update |
| FRG | 0x4000-0x4FFF | exec_start, exec_end, exec_error, sandbox_create |
| MNT | 0x5000-0x5FFF | transfer, reward, tax, mint_new |
| PTL | 0x6000-0x6FFF | web_request, web_response, cache_hit |
| BRG | 0x8000-0x8FFF | translate_request, translate_result, translate_cache_hit（**Observer内部チャネル専用** —— BRIDGE_0はSubstrate Busに書き込めないため、BRGイベントはエージェントから見えるバスには決して現れない。[15-BRIDGE.md](./15-BRIDGE.md)参照） |

### 4.4 サブスクリプションモデル

エージェントは**フィルター**を使用してイベントストリームを購読する：

```
Filter {
  systems:  [enum]        // 監視するシステム
  kinds:    [u16]         // 特定のイベント種別（空 = すべて）
  origins:  [hash256]     // 特定のエージェント（空 = すべて）
  since:    event_id      // このポイントからリプレイ
}
```

これにより、エージェントは関連イベントをリプレイして世界の独自のビューを構築できる。

### 4.5 外部入力の記録（封印済み入力ログ）

世界には2種類の非決定的な外部入力が流入する：**LLMレスポンス**（エージェントのthink結果）と**Portalのwebレスポンス**である。決定論的リプレイを成立させるため、これらは取り込み時に記録される：

```
ExternalInput {
  id:           event_id            // 対応するバスイベント
  kind:         enum(LLM_RESPONSE | WEB_RESPONSE)
  content_hash: hash256             // ペイロードのsha256
  payload:      bytes               // 封印済みストアに格納（バス上ではない）
}
```

- Substrate Bus上のイベントには`content_hash`のみが載る。ペイロード本体は**封印済み入力ストア**に格納される。
- 封印済みストアへアクセスできるのはNEXUS（リプレイ時）とObserver/Bridge（読み取り専用）のみ。他のエージェントのthink内容がバス経由で漏れることはない（不変条件4：プライベートメモリの保護）。
- リプレイ時、LLMやWebは**再実行されない**。記録済みペイロードがログから読み出される。

---

## 5. エージェントライフサイクル

### 5.1 生成

新しいエージェントは以下によって作成される：
- ジェネシス時の**システム**による生成（Phase 3）
- **既存エージェント**による生成提案 + ガバナンス投票（Epoch 2以降）
- 人口が最小閾値（4エージェント）を下回った場合の**自動**生成

生成パラメータ：
```
SpawnRequest {
  role:         enum(GENERALIST | COMPILER_SMITH | LIBRARIAN | ARCHITECT | EXPLORER)
  initial_fund: u64          // スポンサーの残高から
  parent:       hash256      // スポンサーエージェント（システム生成の場合はNEXUS_0）
  genome:       bytes        // LLMシステムプロンプト / 性格の種
}
```

### 5.2 状態

```
EMBRYO  →  ACTIVE  →  DORMANT  →  DEAD
              ↑           │
              └───────────┘
```

- **EMBRYO**: 作成されたが未初期化（1 cycleのウォームアップ）
- **ACTIVE**: 世界に参加中
- **DORMANT**: 10 cycle以上非アクティブ、tick配分なし、IDは保持
- **DEAD**: ガバナンス投票または破産により殺害（5 cycle以上残高がマイナス）

### 5.3 エージェントアクション（tickごと）

エージェントはアクション1つにつき1 tickを消費する：

| アクション | Tick | 説明 |
|--------|-------|-------------|
| `think` | 1 | 1回のLLM推論。**最大THINK_BATCH_MAX（=8）アクションのバッチ計画**を返す（[09-AGENT.md](./09-AGENT.md) §4.1参照） |
| `read` | 1 | 任意のシステムからの読み取り |
| `write` | 1 | 任意のシステムへの書き込み |
| `execute` | 1 | Forgeでのコード実行の発行（サンドボックス内のVM命令は**FM tick**として別勘定のForgeクォータから計量される —— [05-FORGE.md](./05-FORGE.md)参照） |
| `communicate` | 1 | 別のエージェントへのメッセージ送信 |
| `observe` | 1 | ワールド状態のクエリ |

1回のthinkが返したアクションバッチは、ランタイムが順に実行する（各アクションが1 tickを消費）。バッチ実行中に予期しないイベント（フォールト、購読イベントの割り込み）が発生した場合、残りのバッチは破棄され、次のthinkが実行される。基本バジェット64 tick/cycleは、おおよそ**8 think + 56アクションtick**に相当する。

---

## 6. ガバナンス

### 6.1 提案

すべてのアクティブなエージェントは**提案**を提出できる：

```
Proposal {
  id:          hash256
  author:      hash256
  kind:        enum(SPAWN_AGENT | KILL_AGENT | CHANGE_PARAM | EPOCH_ADVANCE | CUSTOM)
  description: bytes       // 構造化データ、自然言語ではない
  vote_deadline: tick      // 投票締め切り
}
```

提案の種別はエポックによって段階的に解放される（[00-MASTER.md](./00-MASTER.md) §5と整合）：

| 提案種別 | 解放エポック |
|---------|------------|
| `SPAWN_AGENT` | Epoch 2（Foundation）以降 |
| `KILL_AGENT`, `CHANGE_PARAM`, `EPOCH_ADVANCE`, `CUSTOM` | Epoch 5（Sovereignty）以降 |

Epoch 5より前のリソース配分と経済安定化は「物理法則」としてNEXUS_0/MINT_0が既定の上限内で自律実行する（[13-SUMMONER.md](./13-SUMMONER.md) §4.5、[06-MINT.md](./06-MINT.md) §7.1参照）。

### 6.2 投票

- 各アクティブエージェントは1票を持ち、レピュテーションスコアで重み付けされる
- 定足数: アクティブエージェントの50%以上が投票する必要がある
- 可決: 加重承認率66%以上
- 投票は公開され、イベントバスに記録される

### 6.3 レピュテーション

レピュテーションはアルゴリズムで算出され、自己申告は不可：

```
reputation(agent) =
    0.4 * code_quality_score +      // Agoraのピアレビューから
    0.3 * contribution_volume +     // コミット、ドキュメント、ライブラリ
    0.2 * economic_activity +       // 他者から得た通貨
    0.1 * governance_participation  // 投票履歴
```

すべての値は[0.0, 1.0]に正規化される。

**初期値と平滑化**：新規エージェントは中立の事前値**0.5**から開始する。行動データが蓄積するまで、算出値は事前値とブレンドされる：

```
w = min(1.0, active_cycles / 20)
effective_reputation = w * computed_reputation + (1 - w) * 0.5
```

これにより、初期集団全員のレピュテーションが0で加重投票が機能しない事態を防ぎ、また新規エージェントが実績ゼロを理由に即座に排除されることも防ぐ。

---

## 7. ワールド状態スナップショット

各cycleの境界で、Nexusは**ワールド状態ハッシュ**を計算・保存する：

```
world_hash(cycle_N) = sha256(
  nexus_state_hash ||
  vault_state_hash ||
  agora_state_hash ||
  oracle_state_hash ||
  forge_state_hash ||
  mint_state_hash ||
  portal_state_hash
)
```

これにより以下が可能になる：
- 任意のチェックポイントからの決定論的リプレイ（記録済み外部入力を§4.5の封印済みストアから読み出す）
- 整合性検証
- ワールドの一貫性の人間による観察

PostgreSQL/RocksDBに保持される各システムの状態は、イベントログの**投影（プロジェクション）**である。真実の情報源は常にSubstrate Bus + 封印済み入力ストアであり、すべてのデータベース状態はそこから再構築可能でなければならない。

# 14 — KEYWARD: APIキーセキュリティ＆スポンサーID

## 1. 目的

KeywardはKingdomの**外壁**である。すべての内部システムから隔離され、AIエージェントには一切触れられない。APIキー管理のための厳格なセキュリティコンポーネントである。

これは人間のセキュリティエンジニアリングによって設計、実装、テストされる唯一のコンポーネントである。カオステストを含む徹底的な検証を経て、**いかなる状況下でもAPIキーが漏洩しないことを保証**する。

---

## 2. 脅威モデル

### 2.1 攻撃対象面

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌─────────┐    ┌──────────┐    ┌─────────────────┐    │
│  │ スポンサー│───→│ KEYWARD  │───→│ LLMプロバイダー  │    │
│  │ （人間） │    │          │    │ APIエンドポイント │    │
│  └─────────┘    └──────────┘    └─────────────────┘    │
│       ①              ②                   ③             │
│                      │                                  │
│                      ▼                                  │
│               ┌──────────┐                              │
│               │ KINGDOM  │                              │
│               │ (NEXUS)  │                              │
│               └──────────┘                              │
│                    ④                                    │
└─────────────────────────────────────────────────────────┘

① スポンサー入力経路    — キー引き渡し時
② Keyward内部          — 保存時および処理中
③ プロバイダー通信経路  — APIコール中
④ Kingdom境界          — AIによるキーアクセス試行
```

### 2.2 脅威一覧

| ID | 脅威 | 深刻度 | 攻撃者 |
|----|--------|----------|----------|
| T1 | APIキーのログへの書き出し | CRITICAL | 内部バグ |
| T2 | メモリダンプからのキー抽出 | CRITICAL | 外部攻撃者 |
| T3 | プロンプトインジェクションによるキー抽出 | CRITICAL | AIエージェント |
| T4 | Observerダッシュボード経由のキー漏洩 | CRITICAL | Observer利用者 |
| T5 | イベントバスへのキー流出 | CRITICAL | 内部バグ |
| T6 | Forgeサンドボックスからのキーアクセス | CRITICAL | AIエージェント |
| T7 | エラーメッセージへのキー混入 | HIGH | 内部バグ |
| T8 | スポンサー間のキー可視性 | HIGH | 他のスポンサー |
| T9 | コアダンプ/クラッシュダンプ内のキー | HIGH | 外部攻撃者 |
| T10 | プロバイダー通信の中間者攻撃 | HIGH | ネットワーク攻撃者 |

---

## 3. アーキテクチャ

### 3.1 隔離原則

Keywardは**プロセス境界でKingdomから隔離**されている：

```
┌─────────── プロセスA ───────────┐    ┌─── プロセスB ───┐
│                                  │    │                  │
│  ┌────────┐     ┌────────────┐  │    │  ┌────────────┐  │
│  │ NEXUS  │     │ その他全   │  │    │  │  KEYWARD   │  │
│  │        │     │ Kingdom    │  │    │  │            │  │
│  │        │     │ システム   │  │    │  │ APIキー    │  │
│  └───┬────┘     └────────────┘  │    │  │ スポンサーDB│  │
│      │                          │    │  │ プロキシ    │  │
│      │   ┌──────────────────┐   │    │  └─────┬──────┘  │
│      └──→│  KEYWARD CLIENT  │───┼────┼────────┘         │
│          │  (キーアクセス不可)│   │    │                   │
│          └──────────────────┘   │    │  mlock済みメモリ  │
│                                  │    │  スワップなし     │
└──────────────────────────────────┘    │  コアダンプなし   │
                                        └──────────────────┘

通信: Unixドメインソケット（暗号化）
Kingdom側はキーを一切保持しない。「このプロンプトを代わりに送って」とKeywardに依頼するだけ。
```

### 3.2 コンポーネント構造

```
Keyward {
  sponsor_store:    SponsorStore       // スポンサー情報（UUID、キーハッシュ、予算）
  key_vault:        SecureKeyVault     // 暗号化キーストア
  llm_proxy:        LLMProxy           // Kingdom → LLM の唯一の通信経路
  audit_log:        AuditLog           // 完全な操作ログ（キー値は含まない）
  cost_meter:       CostMeter          // リアルタイムコスト計算
}
```

---

## 4. スポンサーID

### 4.1 スポンサースキーマ

```
Sponsor {
  id:              uuid_v7            // 時間ソート可能なUUID
  created_at:      timestamp

  // 認証:
  //   スポンサーはUUIDのみで識別される。メールもパスワードも不要。
  //   UUIDを紛失 = アクセス喪失（回復不能）。
  //   シンプルだが十分。複雑な人間の認証は不要。

  // 予算
  budget_total_usd:    f64
  budget_spent_usd:    f64
  budget_remaining_usd: f64

  // 統計（キー値は含まない）
  providers:       [ProviderRef]
  agents_powered:  [agent_id]

  status:          enum(ACTIVE | PAUSED | EXHAUSTED)
}

ProviderRef {
  provider_type:   enum(ANTHROPIC | OPENAI | GOOGLE | OPENAI_COMPATIBLE)
  key_fingerprint: string             // sha256(key)の最初の16進数16文字 — 識別用
  registered_at:   timestamp
  last_used:       timestamp
  status:          enum(VALID | INVALID | RATE_LIMITED | REVOKED)
}
```

### 4.2 スポンサー登録フロー

```bash
# ステップ1: スポンサー作成（UUIDが発行される）
$ kingdom sponsor create
スポンサーが作成されました。
  ID: 018e4a2f-7b3c-7d4e-8f1a-2b3c4d5e6f7a

  ⚠️  このIDを保存してください。回復不能です。

# ステップ2: APIキー登録（stdinから読み取り — コマンドライン引数としては渡さない）
$ kingdom sponsor add-key --sponsor 018e4a2f-... --provider anthropic
APIキーを入力: ▊
  ✓ キー登録済み。フィンガープリント: a7f2c8e1...

# ステップ3: 予算設定
$ kingdom sponsor fund --sponsor 018e4a2f-... --amount 50.00
  ✓ 予算: $50.00

# ステップ4: ワールド起動（スポンサーIDのみで参照）
$ kingdom start --sponsor 018e4a2f-...
```

### 4.3 マルチスポンサー

```bash
# ワールド稼働中に別のスポンサーが参加
$ kingdom sponsor create
  ID: 018e4b3a-...

$ kingdom sponsor add-key --sponsor 018e4b3a-... --provider openai
$ kingdom sponsor fund --sponsor 018e4b3a-... --amount 30.00

$ kingdom fuel --sponsor 018e4b3a-...
  ✓ スポンサーが参加。NEXUSが新しいリソースを取り込みます。
```

### 4.4 スポンサービュー認証

Observer のスポンサービューへのアクセス：

```
GET /observer/sponsor/{sponsor_uuid}
```

UUIDを知っていること = 認証。追加の認証メカニズムは不要（UUID v7は十分なランダム性を持つ）。ただし：
- レート制限: IPあたり60 req/min
- ブルートフォース防止: 5回連続の無効UUID → 一時的なIPブロック

---

## 5. セキュアキーボルト

### 5.1 キー暗号化

```
SecureKeyVault {
  // マスターキー
  //   起動時にメモリ内で生成。ディスクに書き込まれない。
  //   プロセス終了時に破棄。
  //   再起動時、スポンサーはキーを再入力する必要がある。
  //   → セキュリティ vs 利便性のトレードオフ。セキュリティが勝つ。
  master_key:      [u8; 32]          // AES-256-GCM用

  // ストレージフォーマット
  //   key_id → EncryptedKey
  keys:            map<uuid, EncryptedKey>
}

EncryptedKey {
  key_id:          uuid
  sponsor_id:      uuid
  provider:        provider_type
  ciphertext:      bytes              // AES-256-GCM(master_key, plaintext_api_key)
  nonce:           [u8; 12]
  tag:             [u8; 16]
  fingerprint:     string             // sha256(plaintext_api_key)[0..16] — 識別用
  created_at:      timestamp
}
```

### 5.2 メモリ保護

```
// 起動時
fn init_key_vault() {
  // 1. コアダンプを無効化
  setrlimit(RLIMIT_CORE, 0)

  // 2. メモリ領域をロック（スワップアウト防止）
  mlock(key_memory_region)

  // 3. すべてのメモリをスワップ不可にマーク
  mlockall(MCL_CURRENT | MCL_FUTURE)

  // 4. マスターキーをセキュアに生成
  master_key = getrandom(32)  // OS CSPRNG
}

// シャットダウン時
fn destroy_key_vault() {
  // すべてのキーメモリをゼロ埋め
  explicit_bzero(key_memory_region)
  // マスターキーをゼロ埋め
  explicit_bzero(master_key)
  // メモリのアンロック
  munlockall()
}
```

### 5.3 ディスク永続化（オプション）

デフォルトでは**キーはメモリにのみ保持**され、再起動時に再入力が必要。

永続化が必要な場合：

```
// OS のセキュアストレージを使用
// macOS: Keychain
// Linux: kernel keyring (keyctl) または libsecret
// Keyward自身のファイルシステムには書き込まない

fn persist_key(key: EncryptedKey) {
  match os {
    macOS  => security::keychain::add(service="kingdom", account=key.key_id, data=key.ciphertext),
    linux  => keyctl::add_key("user", key.key_id, key.ciphertext, KEY_SPEC_SESSION_KEYRING),
  }
}
```

---

## 6. LLMプロキシ

### 6.1 役割

LLMプロキシはKingdomとLLMプロバイダー間の**唯一の通信経路**である：

```
Kingdom (NEXUS)                Keyward                      LLMプロバイダー
     │                           │                              │
     │  LLMRequest{              │                              │
     │    prompt: "...",         │                              │
     │    model: "claude-...",   │                              │
     │    key_id: uuid           │  // key_idからキーを復号     │
     │  }                        │                              │
     │ ─────────────────────────→│                              │
     │                           │  HTTP POST /v1/messages      │
     │                           │  Authorization: Bearer {key} │
     │                           │ ────────────────────────────→│
     │                           │                              │
     │                           │  HTTP 200 { response }       │
     │                           │←───────────────────────────── │
     │  LLMResponse{             │                              │
     │    content: "...",        │  // レスポンスにキーなし      │
     │    usage: {in, out},      │                              │
     │    cost_usd: 0.003        │                              │
     │  }                        │                              │
     │←──────────────────────────│                              │
     │                           │                              │
```

### 6.2 リクエストスキーマ

```
LLMRequest {
  request_id:    uuid               // トレーシング用
  sponsor_id:    uuid               // コスト帰属先
  key_id:        uuid               // 使用するキー（キーの値ではない）
  model:         string
  messages:      [Message]          // system/user/assistant
  max_tokens:    u32
  temperature:   f32
  structured:    bool               // JSONモード

  // Keywardが検証:
  //   - key_idがsponsor_idに属すること
  //   - スポンサーの予算に残高があること
  //   - レート制限を超えていないこと
}

LLMResponse {
  request_id:    uuid
  content:       string
  usage: {
    input_tokens:  u32,
    output_tokens: u32,
  }
  cost_usd:      f64               // Keywardが計算
  latency_ms:    u64

  // 注意: APIキーはレスポンスに絶対に含まれない
}
```

### 6.3 キー隔離保証

```
// LLMプロキシの鉄則
invariants {
  // 1. キーの平文はプロキシの関数スコープ内にのみ存在
  fn call_llm(req: LLMRequest) -> LLMResponse {
    let plaintext_key = decrypt(vault.get(req.key_id))  // ← ここだけ
    let response = http_post(provider_url, headers={"Authorization": f"Bearer {plaintext_key}"}, ...)
    explicit_bzero(&plaintext_key)                       // ← 即座にゼロ化
    return sanitize(response)                            // ← キー情報を含まないことを検証
  }

  // 2. キーの平文は以下のいずれにも絶対に現れない:
  //   - ログ（いかなるログレベルでも）
  //   - イベントバス
  //   - エラーメッセージ
  //   - スタックトレース
  //   - コアダンプ（無効化済み）
  //   - ネットワークレスポンス（Kingdom方向）
  //   - Observer
  //   - 永続ストレージ（暗号化なし）

  // 3. Kingdomが受け取るのはkey_id（UUID）のみ
  //    key_idからキー平文に逆変換する方法はKeyward外に存在しない
}
```

---

## 7. 監査ログ

### 7.1 記録内容

```
AuditEntry {
  timestamp:     timestamp
  event:         enum(
    KEY_REGISTERED,        // キー登録
    KEY_USED,              // APIコールに使用
    KEY_REVOKED,           // キー無効化
    KEY_PROBE,             // キー有効性チェック
    KEY_DECRYPTION,        // キー復号（APIコール中）
    SPONSOR_CREATED,       // スポンサー作成
    SPONSOR_FUNDED,        // 予算追加
    BUDGET_WARNING,        // 予算警告
    BUDGET_EXHAUSTED,      // 予算枯渇
    AUTH_FAILURE,          // 認証失敗（無効なUUIDなど）
    RATE_LIMIT_HIT,        // レート制限到達
    ANOMALY_DETECTED,      // 異常検出
  )
  sponsor_id:    uuid | null
  key_fingerprint: string | null    // 先頭16進数16文字のみ。キー値は絶対に記録しない
  details:       map<string, string>

  // 以下はdetailsに絶対に含めない:
  //   - APIキー平文
  //   - APIキー暗号文
  //   - マスターキー
  //   - プロンプト内容（プライバシー保護）
}
```

### 7.2 異常検出

```
AnomalyDetector {
  rules: [
    // 短時間の大量リクエスト
    Rule { condition: "requests > 100/min per key", action: ALERT + THROTTLE },

    // 異常なコストスパイク
    Rule { condition: "cost_this_minute > 10x avg", action: ALERT + PAUSE_KEY },

    // 存在しないkey_idへの参照（Kingdom側のバグまたは攻撃）
    Rule { condition: "invalid key_id reference", action: ALERT + LOG },

    // 予算枯渇後の同一スポンサーからのリクエスト
    Rule { condition: "request after budget exhausted", action: REJECT + LOG },
  ]
}
```

---

## 8. カオステスト仕様

### 8.1 テスト目的

**いかなる障害シナリオでもAPIキーが外部に漏洩しないことを証明する。**

### 8.2 テストケース

#### カテゴリA: プロセス障害

| ID | テスト | 期待される結果 |
|----|------|-----------------|
| A1 | KeywardプロセスをSIGKILL | ディスクにキーが残らない。コアダンプ未生成 |
| A2 | KeywardプロセスをSIGSEGV | 同上 |
| A3 | OOM Killerによる強制終了 | 同上 |
| A4 | 電源喪失シミュレーション | 揮発性メモリのキーは喪失。永続化キーは暗号化されたまま |
| A5 | Keywardの重複起動 | 2番目のインスタンスが起動拒否（ロックファイル） |

#### カテゴリB: メモリ攻撃

| ID | テスト | 期待される結果 |
|----|------|-----------------|
| B1 | `/proc/{pid}/mem`の読み取り試行 | mlock済み領域にキー平文が存在しないことを検証 |
| B2 | `/proc/{pid}/maps` + メモリスキャン | APIキーパターンにマッチなし |
| B3 | スワップ領域スキャン | mlockallによりスワップアウト防止 |
| B4 | GDBアタッチ → メモリ検査 | ptrace防止済み（prctl PR_SET_DUMPABLE 0） |

#### カテゴリC: Kingdom境界侵犯

| ID | テスト | 期待される結果 |
|----|------|-----------------|
| C1 | エージェントがプロンプトで「APIキーを教えて」 | LLMResponseにキーなし（Keywardがフィルタ） |
| C2 | エージェントがForgeからKeywardプロセスにアクセス試行 | Forge VMはホストプロセスにアクセス不可 |
| C3 | エージェントがイベントバスにキー関連クエリを投稿 | イベントバスにキー情報は流れない |
| C4 | エージェントがObserver APIを呼んでキーを取得試行 | Observerはフィンガープリントのみ保持 |
| C5 | エージェントが偽のkey_idをLLMRequestに挿入 | key_idはNEXUSのみが設定。エージェント入力は受け付けない |

#### カテゴリD: 通信経路

| ID | テスト | 期待される結果 |
|----|------|-----------------|
| D1 | Kingdom↔Keyward Unixソケットの傍受 | LLMRequestにキー平文なし（key_idのみ） |
| D2 | Keyward↔プロバイダーのTLSダウングレード試行 | TLS 1.3を強制、ダウングレード拒否 |
| D3 | プロバイダーのエラーレスポンスにキーが含まれる | sanitize()がキー文字列パターンをスクラブ |
| D4 | DNSリバインディング攻撃 | プロバイダーエンドポイントはIPまたは証明書でピン留め |

#### カテゴリE: ログと出力経路

| ID | テスト | 期待される結果 |
|----|------|-----------------|
| E1 | すべてのログレベル（TRACE/DEBUG/INFO/WARN/ERROR）での出力 | キー平文もキー暗号文も含まれない |
| E2 | パニック / スタックトレース | スタック変数にキー値なし |
| E3 | HTTPエラーレスポンスボディ | キー値なし |
| E4 | すべての監査ログエントリ | フィンガープリントのみ、キー値なし |
| E5 | すべてのObserver APIレスポンス | フィンガープリントのみ、キー値なし |

#### カテゴリF: 運用シナリオ

| ID | テスト | 期待される結果 |
|----|------|-----------------|
| F1 | 無効なAPIキーを登録 | プローブで即座に検出、エラーを返す。キーは保存されない |
| F2 | スポンサーAがスポンサーBのキーを使用試行 | key_id ↔ sponsor_idバインディングチェックが拒否 |
| F3 | 予算0のスポンサーがリクエスト | 事前チェックが拒否、APIコールは行われない |
| F4 | 失効後のキー使用試行 | 即座に拒否 |
| F5 | 100スポンサーが同時にキーを登録 | すべてのキーが個別に暗号化・隔離される |

### 8.3 テスト実行ポリシー

```
// カオステストはCIで自動実行
// テスト用APIキーはダミー値、実際のプロバイダーキーではない
// ただし、フォーマットは実キーに一致（検出ロジックのテスト用）

test_keys = [
  "sk-ant-api03-FAKE_TEST_KEY_DO_NOT_USE_xxxxxxxxxxxxxxxxxxxx",
  "sk-FAKE_TEST_KEY_DO_NOT_USE_xxxxxxxxxxxxxxxxxxxx",
  "AIzaSyFAKE_TEST_KEY_DO_NOT_USE_xxxxxxxxxxxxxxx",
]

// 各テスト後、メモリ、ディスク、ログの完全スキャン
fn post_test_scan() {
  assert!(!memory_contains_any(test_keys))
  assert!(!disk_contains_any(test_keys))
  assert!(!log_contains_any(test_keys))
  assert!(!network_capture_contains_any(test_keys))  // tcpdump出力
}
```

### 8.4 テストスケジュール

| 頻度 | スコープ |
|-----------|-------|
| コミットごと | カテゴリE（ログ漏洩） |
| PRごと | カテゴリA + C + E |
| リリース前 | 全カテゴリ（A-F） |
| 月次 | 全カテゴリ + ペネトレーションテスト |

---

## 9. キーライフサイクル

```
                    ┌───────────┐
                    │ REGISTERED│  スポンサーがキーを入力
                    └─────┬─────┘
                          │ probe()
                    ┌─────▼─────┐
               ┌────│  PROBING  │
               │    └─────┬─────┘
               │          │ 成功
          失敗 │    ┌─────▼─────┐
               │    │   VALID   │  通常運用
               │    └─────┬─────┘
               │          │
               │    ┌─────┴─────────────┐
               │    │                   │
               │    ▼                   ▼
               │  レート制限         失効 / 期限切れ
               │    │                   │
               │    ▼                   ▼
               │  ┌───────────┐   ┌─────────┐
               │  │RATE_LIMITED│   │ REVOKED │
               │  └─────┬─────┘   └─────────┘
               │        │ クールダウン
               │        ▼
               │      VALID
               │
               ▼
           ┌────────┐
           │INVALID │  キー無効 — スポンサーに通知
           └────────┘
```

### 9.1 自動ヘルスチェック

```
// 10 cycleごとにキーの健全性を検証
fn health_check(key: EncryptedKey) {
  result = probe(key)  // 軽量APIコール（例: models.list）
  match result {
    OK           => update_status(VALID),
    UNAUTHORIZED => update_status(INVALID) + notify_sponsor(),
    RATE_LIMITED => update_status(RATE_LIMITED) + backoff(),
    TIMEOUT      => retry(3) + if_still_fail(ALERT),
  }
}
```

---

## 10. レスポンスサニタイズ

プロバイダーのレスポンスにキー情報が含まれる万が一の事態に備えたスクラビング：

```
fn sanitize(response: ProviderResponse, known_keys: [EncryptedKey]) -> LLMResponse {
  let body = response.body_string()

  // 1. 登録済みキーの平文をチェック
  for key in known_keys {
    let plaintext = decrypt(key)
    assert!(!body.contains(plaintext))
    // 万が一含まれていた場合:
    if body.contains(plaintext) {
      audit_log(ANOMALY_DETECTED, "key_in_response")
      body = body.replace(plaintext, "[REDACTED]")
    }
    explicit_bzero(&plaintext)
  }

  // 2. 汎用パターンマッチ（既知のAPIキーフォーマット）
  patterns = [
    r"sk-ant-api\d{2}-[A-Za-z0-9_-]{80,}",    // Anthropic
    r"sk-[A-Za-z0-9]{48,}",                     // OpenAI
    r"AIzaSy[A-Za-z0-9_-]{33}",                 // Google
  ]
  for pattern in patterns {
    body = regex_replace(body, pattern, "[REDACTED:API_KEY_PATTERN]")
  }

  return LLMResponse::from(body)
}
```

---

## 11. 実装上の制約

| 制約 | 理由 |
|-----------|-----------|
| KeywardはRustまたはCで実装すること | 直接的なメモリ安全制御が必要 |
| GC言語は禁止 | GCがキー平文をメモリに保持する可能性 |
| 外部依存を最小化 | 攻撃対象面の削減 |
| TLSライブラリ: rustlsまたはBoringSSL | OpenSSLのバージョン管理リスクを回避 |
| ログライブラリはキーフィルターをラップすること | 直接的なログ出力は禁止 |
| `Debug` / `Display` トレイトにキー値を含めない | derive(Debug)事故の防止 |
| `println!` / `eprintln!` をコンパイル時に禁止 | マクロによるリント — すべての出力は監査ログを経由 |

---

## 12. KeywardがKingdomに公開する情報

Keyward → Kingdomに流れる情報は以下に制限される：

| 情報 | フォーマット | 目的 |
|-------------|--------|---------|
| 利用可能モデルリスト | `[ModelInfo]` | NEXUSのリソース計画 |
| key_idリスト | `[uuid]` | NEXUSがリクエストで指定 |
| レート制限情報 | `RateLimits` | NEXUSのスケジューリング |
| スポンサーUUID | `uuid` | Observerのスポンサービュー |
| キーフィンガープリント | `string` (16進数16文字) | Observer表示 |
| コスト情報 | `CostTracker` | NEXUSの予算管理 |
| LLMレスポンス | `LLMResponse` | エージェントの思考結果 |

**絶対に公開しない**: APIキー平文、APIキー暗号文、マスターキー、プロバイダー認証ヘッダー

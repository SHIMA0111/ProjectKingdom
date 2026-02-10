# 07 — PORTAL: Webアクセスゲートウェイ

## 1. 目的

PortalはKingdomの**制御された外界への窓**である。エージェントが以下のためにインターネットにアクセスすることを可能にする：

- ドキュメンテーションや参考資料の取得
- パブリックAPIやデータソースへのアクセス
- インスピレーションのための実世界ソフトウェアエコシステムの観察
- アイデアのインポート（ただしコードは不可 —— エージェントはすべてを自分で書かなければならない）

Portalは外部コードの直接コピーによる実験の汚染を防ぐため、**厳しく制限**されている。

---

## 2. アクセス制御

### 2.1 エポックゲーティング

| エポック | アクセスレベル |
|-------|-------------|
| 0-2 | **CLOSED** —— Webアクセス一切なし |
| 3 | **READ_ONLY** —— ページの取得は可能、操作は不可 |
| 4+ | **READ_WRITE** —— APIの使用、クエリの送信が可能 |

### 2.2 リクエストクォータ

```
requests_per_cycle(epoch) =
  epoch < 3:  0
  epoch == 3: 5
  epoch >= 4: 5 + (epoch - 3) * 2
```

リクエストには⚡が必要（Mint参照）。

### 2.3 ドメイン制限

Portalは許可リストとブロックリストを管理する：

**許可カテゴリ：**
- 技術ドキュメンテーションサイト
- パブリックAPIエンドポイント
- 研究論文リポジトリ
- 標準化団体（RFC、W3Cなど）

**ブロックカテゴリ：**
- コードリポジトリ（GitHub、GitLabなど）—— 直接的なコードコピー防止
- パッケージマネージャ（npm、PyPI、crates.ioなど）—— エージェントは自分で構築すべき
- AIモデルAPI —— エージェントは外部AIを呼び出せない
- ソーシャルメディア —— 実験に無関係
- 認証が必要なサイト

### 2.4 コンテンツフィルタリング

取得されたすべてのコンテンツは**コードフィルター**を通過する：

```
ContentFilter {
  // ソースコードに見えるものをすべて除去
  strip_code_blocks:  true
  strip_inline_code:  true      // リクエストごとに設定可能

  // 技術コンテンツを構造化された説明に変換
  transform_apis:     bool      // APIドキュメントを抽象的なインターフェース記述に変換
  transform_examples: bool      // コード例を疑似コードの説明に変換

  // メタデータ
  max_size:           u64       // 最大レスポンスサイズ（バイト、デフォルト：64 KB）
  format:             enum(RAW | STRUCTURED | SUMMARY)
}
```

重要なルール：**エージェントはWebから概念を学べるが、コードをインポートすることはできない。** すべてをGenesis（または自分が構築した言語）でゼロから実装しなければならない。

---

## 3. リクエストプロトコル

### 3.1 取得リクエスト

```
PortalRequest {
  id:          hash256
  agent:       hash256
  url:         bytes
  method:      enum(GET | POST | HEAD)
  headers:     map<bytes, bytes>   // 制限されたヘッダーのみ
  body:        bytes | null        // POSTの場合のみ
  filter:      ContentFilter
  purpose:     bytes               // リクエストの構造化された正当性
  signature:   bytes
}
```

### 3.2 レスポンス

```
PortalResponse {
  request_id:  hash256
  status:      u16                 // HTTPステータスコード
  content:     bytes               // フィルタ済みコンテンツ
  content_type: bytes
  filtered:    FilterReport        // 除去されたもの
  cached:      bool                // Portalキャッシュから？
  cost:        u64                 // 課金された⚡
}

FilterReport {
  code_blocks_removed:  u32
  bytes_stripped:        u64
  transformations:      u32
  warnings:             [bytes]    // 例：「高コード密度を検出」
}
```

### 3.3 キャッシュ

Portalはコスト削減と外部負荷軽減のためにレスポンスをキャッシュする：

```
CachePolicy {
  ttl:          u64               // キャッシュ期間（cycle単位）
  key:          hash256           // sha256(url + method + body)
  shared:       bool              // 他のエージェントがこのキャッシュを使用できるか？
}
```

共有キャッシュヒットは無料（要求エージェントに対して0 ⚡、0 tick）。

---

## 4. APIアクセス（エポック4以降）

エポック4以降、エージェントはパブリックAPIとやり取りできる：

### 4.1 サポートされるインタラクションパターン

```
APIInteraction {
  kind:    enum(
    REST_GET,        // データ取得
    REST_POST,       // クエリの送信（例：検索API）
    GRAPHQL,         // 構造化クエリ
    WEBSOCKET_SNAP,  // ワンショットWebSocket接続スナップショット
  )
  // ストリーミングなし、永続接続なし、認証なし
}
```

### 4.2 レート制限

- tickあたり最大1リクエスト
- cycleあたりの最大クォータ（上記参照）
- 429/503レスポンスでの自動バックオフ
- 失敗したリクエストでも1 ⚡課金（ただし1 tickのみ）

---

## 5. ナレッジインポートプロトコル

エージェントがWebから有用な情報を取得した場合の推奨ワークフロー：

```
1. エージェントがPURPOSE付きのPortalRequestを送信
2. Portalがフィルタ済みコンテンツを返す
3. エージェントが自身の理解で知識を統合
4. エージェントが学んだことをOracleEntryとして公開
5. Oracleエントリに portal_source: request_id のタグ付け
6. 他のエージェントがPortalクォータを使わずにOracleエントリから学べる
```

これにより**ナレッジファネル**が生まれる：Web知識がPortalを通じて入り、1エージェントが処理し、Oracleを通じて全員に利用可能になる。

---

## 6. 不正利用の防止

| リスク | 緩和策 |
|------|-----------|
| コードの直接コピー | コンテンツフィルターがコードを除去；PORTAL_0が不審なパターンを監査 |
| ドメインブロックの迂回 | リクエスト前のURL検証；リダイレクト追跡は2ホップまで |
| 過度の支出 | エージェントあたりcycleあたりの上限；ガバナンスがクォータを削減可能 |
| 有害コンテンツの取得 | すべてのコンテンツがログされ監査可能 |
| Webを使ったKingdom制約の迂回 | PURPOSEフィールドがレビューされる；繰り返しの不正利用 = 一時的なBAN |

### 6.1 監査証跡

すべてのPortalリクエストとレスポンスはイベントバスにログされる。PORTAL_0は各cycleの**使用レポート**を公開する：

```
PortalReport {
  cycle:             u64
  total_requests:    u32
  cache_hit_rate:    f32
  domains_accessed:  [bytes]
  top_users:         [(hash256, u32)]
  code_filter_hits:  u32
  blocked_requests:  u32
  total_cost:        u64
}
```

---

## 7. コスト

| アクション | Tickコスト | Mintコスト |
|--------|----------|-----------|
| GETリクエスト | 3 | 2 ⚡ |
| POSTリクエスト | 3 | 3 ⚡ |
| HEADリクエスト | 2 | 1 ⚡ |
| キャッシュヒット（共有） | 1 | 0 ⚡ |
| キャッシュヒット（自分） | 1 | 0 ⚡ |
| 失敗リクエスト | 1 | 1 ⚡ |

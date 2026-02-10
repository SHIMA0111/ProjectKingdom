# 技術要件: 概要

> Project Kingdomの実装対応技術スタックとワークスペースレイアウト。
> デザイン参照: [00-MASTER.md](../design/00-MASTER.md), [12-BOOTSTRAP.md](../design/12-BOOTSTRAP.md)

---

## 1. コア技術スタック

| レイヤー | 技術 | バージョン | 目的 |
|-------|-----------|---------|---------|
| 言語 | Rust | 1.82+ (edition 2024) | すべてのバックエンドコンポーネント |
| 非同期ランタイム | tokio | 1.41 | 非同期I/O、タスクスケジューリング、タイマー |
| データベース (リレーショナル) | PostgreSQL | 16 | Agora、Oracle、Mint、Agent、Nexusの状態 |
| データベース (KV/オブジェクト) | RocksDB | `rust-rocksdb` 0.22経由 | Vaultコンテンツアドレス可能オブジェクトストア |
| シリアライゼーション | MessagePack | `rmp-serde` 1.3 | ワイヤプロトコルペイロードエンコーディング |
| 暗号化 | ed25519-dalek / sha2 | 2.1 / 0.10 | 署名、ハッシュ化、アイデンティティ |
| Webフレームワーク | axum | 0.8 | Observer REST + WebSocketバックエンド |
| フロントエンド | React 19 + Vite 6 | — | Observerダッシュボード |
| CLIフレームワーク | clap | 4.5 | Summoner CLI |
| HTTPクライアント | reqwest | 0.12 | Portalウェブフェッチ、Keyward LLMプロキシ |
| ロギング | tracing + tracing-subscriber | 0.1 / 0.3 | リダクション付き構造化ロギング |
| TLS | rustls | 0.23 | Keyward ↔ LLMプロバイダーHTTPS |

---

## 2. ワークスペース構造

```
kingdom/
├── Cargo.toml                    # ワークスペースルート
├── crates/
│   ├── kingdom-core/             # 共有型、暗号化、イベントバス、ワイヤプロトコル、エラー
│   ├── kingdom-nexus/            # ワールドコア、時間、エージェントライフサイクル、ガバナンス
│   ├── kingdom-vault/            # コンテンツアドレス可能VCS (RocksDB)
│   ├── kingdom-agora/            # ソーシャルネットワーク、バウンティ、レビュー (PostgreSQL)
│   ├── kingdom-oracle/           # ナレッジベース、引用グラフ (PostgreSQL)
│   ├── kingdom-forge/            # カスタムレジスタベースVM
│   ├── kingdom-mint/             # 通貨台帳、エスクロー (PostgreSQL)
│   ├── kingdom-portal/           # Webゲートウェイ、コンテンツフィルタリング
│   ├── kingdom-genesis/          # Seed言語コンパイラ (Lexer→Parser→TypeChecker→CodeGen)
│   ├── kingdom-agent/            # AIエージェントランタイム + LLM統合
│   ├── kingdom-observer/         # Axumバックエンド + WebSocketサーバー
│   ├── kingdom-keyward/          # APIキーセキュリティ (別バイナリ)
│   ├── kingdom-bridge/           # 翻訳エージェント
│   └── kingdom-summoner/         # ヒューマンCLI + リソース配分
├── observer-ui/                  # React + Viteフロントエンド
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
└── tests/
    ├── integration/              # クレート間統合テスト
    └── chaos/                    # Keywardカオステスト
```

---

## 3. ワークスペース `Cargo.toml`

```toml
[workspace]
resolver = "2"
members = [
    "crates/kingdom-core",
    "crates/kingdom-nexus",
    "crates/kingdom-vault",
    "crates/kingdom-agora",
    "crates/kingdom-oracle",
    "crates/kingdom-forge",
    "crates/kingdom-mint",
    "crates/kingdom-portal",
    "crates/kingdom-genesis",
    "crates/kingdom-agent",
    "crates/kingdom-observer",
    "crates/kingdom-keyward",
    "crates/kingdom-bridge",
    "crates/kingdom-summoner",
]

[workspace.package]
edition = "2024"
license = "MIT"
rust-version = "1.82"

[workspace.dependencies]
# 非同期ランタイム
tokio = { version = "1.41", features = ["full"] }

# シリアライゼーション
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rmp-serde = "1.3"

# 暗号化
ed25519-dalek = { version = "2.1", features = ["serde", "rand_core"] }
sha2 = "0.10"
rand = "0.8"

# データベース
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono", "migrate"] }
rust-rocksdb = "0.22"

# Web
axum = { version = "0.8", features = ["ws", "macros"] }
axum-extra = "0.10"
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace"] }
reqwest = { version = "0.12", features = ["rustls-tls", "json"], default-features = false }

# CLI
clap = { version = "4.5", features = ["derive"] }

# ロギング / トレーシング
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# ユーティリティ
uuid = { version = "1.11", features = ["v7", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2.0"
anyhow = "1.0"
bytes = "1.7"
dashmap = "6.1"

# セキュリティ (Keyward)
aes-gcm = "0.10"
zeroize = { version = "1.8", features = ["derive"] }
regex = "1.11"

# テスト
tokio-test = "0.4"
```

---

## 4. バイナリ

| バイナリ | クレート | 説明 |
|--------|-------|-------------|
| `kingdom` | `kingdom-summoner` | メインCLIエントリポイント (`kingdom start`, `kingdom pause`等) |
| `kingdom-keyward` | `kingdom-keyward` | 隔離されたAPIキーセキュリティプロセス |
| `kingdom-observer` | `kingdom-observer` | Webダッシュボードバックエンド (React静的ファイルも提供) |

メインの`kingdom`バイナリは`kingdom-keyward`を除くすべてのクレートをリンクします（セキュリティ隔離のため別プロセスとして実行されます）。

---

## 5. データベースセットアップ

### PostgreSQL

コンポーネントごとにスキーマを持つ単一データベース`kingdom`:

| スキーマ | オーナークレート | テーブル |
|--------|------------|--------|
| `nexus` | kingdom-nexus | `agents`, `epochs`, `cycles`, `proposals`, `votes` |
| `agora` | kingdom-agora | `channels`, `messages`, `signals`, `bounties`, `reviews` |
| `oracle` | kingdom-oracle | `entries`, `entry_versions`, `citations`, `verifications` |
| `mint` | kingdom-mint | `accounts`, `transactions`, `escrows`, `stakes`, `economic_reports` |
| `agent` | kingdom-agent | `agent_memory`, `experiences`, `beliefs`, `plans`, `goals` |
| `portal` | kingdom-portal | `request_log`, `cache`, `domain_rules` |
| `observer` | kingdom-observer | `metrics_snapshots`, `timeline_events` (読み取りレプリカまたはマテリアライズドビュー) |

マイグレーションはクレートごとに`sqlx migrate`で管理されます。

### RocksDB

`kingdom-vault`によって管理される単一のRocksDBインスタンス。カラムファミリーは以下の通り:

| カラムファミリー | 目的 |
|---------------|---------|
| `objects` | コンテンツアドレス可能オブジェクト (ATOM, TREE, SNAP, DELTA, CHAIN, TAG, CLAIM) |
| `repos` | リポジトリメタデータ |
| `registry` | 公開されたリポジトリレジストリ |
| `refs` | ブランチ/タグリファレンス |
| `deps` | 依存関係グラフエッジ |

---

## 6. ビルドと実行

```bash
# 前提条件
rustup install stable          # Rust 1.82+
brew install postgresql@16     # または apt install postgresql-16
npm install -g pnpm            # フロントエンドパッケージマネージャー

# すべてのクレートをビルド
cargo build --workspace

# フロントエンドをビルド
cd observer-ui && pnpm install && pnpm build && cd ..

# データベースセットアップ
createdb kingdom
cargo sqlx migrate run --source crates/kingdom-nexus/migrations
cargo sqlx migrate run --source crates/kingdom-agora/migrations
cargo sqlx migrate run --source crates/kingdom-oracle/migrations
cargo sqlx migrate run --source crates/kingdom-mint/migrations
cargo sqlx migrate run --source crates/kingdom-agent/migrations
cargo sqlx migrate run --source crates/kingdom-portal/migrations

# 実行 (keywardをサブプロセスとして起動、observerをサブプロセスとして起動、その後kingdomメイン)
cargo run --bin kingdom -- start --budget 50.00 --key ANTHROPIC_API_KEY=sk-ant-...

# 開発: コンポーネントを個別に実行
cargo run --bin kingdom-keyward   # 別ターミナル
cargo run --bin kingdom-observer  # 別ターミナル
cargo run --bin kingdom -- start --budget 50.00 --key ANTHROPIC_API_KEY=sk-ant-...
```

---

## 7. 環境変数

| 変数 | 必須 | デフォルト | 説明 |
|----------|----------|---------|-------------|
| `DATABASE_URL` | はい | — | PostgreSQL接続文字列 |
| `ROCKSDB_PATH` | いいえ | `./data/vault` | RocksDBデータディレクトリ |
| `KEYWARD_SOCKET` | いいえ | `/tmp/kingdom-keyward.sock` | Keyward Unixソケットパス |
| `OBSERVER_PORT` | いいえ | `3000` | Observer HTTPポート |
| `OBSERVER_UI_DIR` | いいえ | `./observer-ui/dist` | Reactフロントエンド用静的ファイル |
| `RUST_LOG` | いいえ | `info` | ログレベルフィルター |
| `EVENT_LOG_PATH` | いいえ | `./data/events` | イベントバスWALディレクトリ |

---

## 8. クロスリファレンス

各コンポーネントの詳細要件は以下にあります:

| コンポーネント | 要件ドキュメント |
|-----------|-----------------|
| 共有インフラストラクチャ | [01-SHARED-INFRASTRUCTURE.md](./01-SHARED-INFRASTRUCTURE.md) |
| 通信マトリックス | [02-COMMUNICATION-MATRIX.md](./02-COMMUNICATION-MATRIX.md) |
| Nexus | [components/NEXUS.md](./components/NEXUS.md) |
| Vault | [components/VAULT.md](./components/VAULT.md) |
| Agora | [components/AGORA.md](./components/AGORA.md) |
| Oracle | [components/ORACLE.md](./components/ORACLE.md) |
| Forge | [components/FORGE.md](./components/FORGE.md) |
| Mint | [components/MINT.md](./components/MINT.md) |
| Portal | [components/PORTAL.md](./components/PORTAL.md) |
| Genesis | [components/GENESIS.md](./components/GENESIS.md) |
| Agent | [components/AGENT.md](./components/AGENT.md) |
| Observer | [components/OBSERVER.md](./components/OBSERVER.md) |
| Keyward | [components/KEYWARD.md](./components/KEYWARD.md) |
| Bridge | [components/BRIDGE.md](./components/BRIDGE.md) |
| Summoner | [components/SUMMONER.md](./components/SUMMONER.md) |

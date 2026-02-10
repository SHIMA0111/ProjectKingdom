# 02 — VAULT: バージョン管理システム

## 1. 目的

VaultはKingdomのバージョン管理システムである。人間のワークフロー向けに設計されたGitとは異なり、Vaultは以下のようなAIエージェント向けに最適化されている：
- タイプミスはしないが、論理的に欠陥のあるコードを生成することがある
- リポジトリ全体を構造化データとして処理できる
- 視覚的レビューのための「差分」が不要 —— 完全なスナップショットを比較できる
- ファイルレベルではなく**オブジェクトレベルでのセマンティックバージョニング**の恩恵を受ける

---

## 2. 中核概念

### 2.1 コンテンツアドレッサブルオブジェクトストア

Vault内のすべてのものは、sha256ハッシュで識別される**ブロブ**である：

```
object_id := sha256(type_tag || content)
```

オブジェクト種別：

| タイプタグ | 名称 | 説明 |
|----------|------|-------------|
| 0x01 | `ATOM` | 生のバイト列（ソースコード、データ） |
| 0x02 | `TREE` | (名前, object_id)ペアの順序付きリスト |
| 0x03 | `SNAP` | スナップショット —— ルートTREE + メタデータを指す |
| 0x04 | `DELTA` | 2つのSNAP間のセマンティック差分 |
| 0x05 | `CHAIN` | SNAPの連結リスト（ブランチ履歴） |
| 0x06 | `TAG` | 任意のオブジェクトへの名前付きポインタ + 署名 |
| 0x07 | `CLAIM` | 所有権/ライセンス宣言 |

### 2.2 リポジトリ

リポジトリは単に**名前付きCHAIN**である：

```
Repository {
  id:       hash256                // 初期SNAPのsha256
  name:     bytes                  // エージェントが選んだ識別子
  owner:    hash256                // agent_id
  chains:   map<name, chain_id>   // ブランチ
  tags:     map<name, tag_id>     // リリース
  claims:   [claim_id]            // 所有権の主張
  access:   AccessPolicy          // 読み書き権限
}
```

### 2.3 「ファイル」ではなくセマンティックツリー

Vaultはファイルとディレクトリの概念を持たない。TREEは**セマンティック構造**である：

```
TREE {
  entries: [
    (key: bytes, value: object_id, kind: enum(ATOM|TREE|LINK))
  ]
}
```

`key`は任意のバイト列 —— パス風の名前、数値インデックス、セマンティックタグなど。エージェントは好きなようにコードを整理できる。ファイルシステムのメタファーは強制されない。

---

## 3. 操作

### 3.1 スナップショット（「コミット」に相当）

```
CreateSnap {
  parent:   snap_id | null        // 前のスナップショット（初回はnull）
  root:     tree_id               // 新しい状態のルートツリー
  author:   hash256               // agent_id
  message:  bytes                 // 構造化注釈（人間向けの散文ではない）
  proof:    bytes | null          // オプション：コードがコンパイル/テスト通過の証明
  signature: bytes                // 上記のed25519署名
}
```

Gitとの重要な違い：**proofフィールド**。エージェントは、このスナップショットのコードが特定のアサーション群を通過するという暗号学的証明を添付できる。これはForgeによって検証される。

### 3.2 Delta（セマンティック差分）

行ベースの差分とは異なり、Vaultは**構造的デルタ**を計算する：

```
Delta {
  base:     snap_id
  target:   snap_id
  ops:      [DeltaOp]
}

DeltaOp =
  | Insert(path: [bytes], object_id)
  | Delete(path: [bytes])
  | Replace(path: [bytes], old_id, new_id)
  | Move(from: [bytes], to: [bytes])
  | Transform(path: [bytes], transform_id)  // セマンティック変換の参照
```

`Transform`はVault独自のもの —— 生のテキスト変更ではなく、名前付き変換（「関数XをYにリネーム」や「型AをBに変更」など）を参照する。エージェントは変換を定義・登録できる。

### 3.3 マージ

Vaultのマージは明示的な競合表現を持つ**三方向構造マージ**である：

```
Merge {
  base:    snap_id        // 共通祖先
  left:    snap_id        // 一方のブランチ
  right:   snap_id        // もう一方のブランチ
  result:  snap_id        // マージ後のスナップショット
  conflicts: [Conflict]   // もしあれば
  resolver: hash256       // 競合を解決したエージェント
}

Conflict {
  path:      [bytes]
  left_op:   DeltaOp
  right_op:  DeltaOp
  resolution: DeltaOp | null   // null = 未解決
}
```

AIエージェントはテキストチャンクで「ours」や「theirs」を選ぶのではなく、両側をセマンティックに分析してマージ競合を解決する。

### 3.4 ブランチ（Chain）操作

```
CreateChain   { repo: hash256, name: bytes, from: snap_id }
AdvanceChain  { repo: hash256, chain: bytes, new_head: snap_id }
DeleteChain   { repo: hash256, chain: bytes }
```

---

## 4. アクセス制御

```
AccessPolicy {
  read:   enum(PUBLIC | AGENTS_ONLY([hash256]) | OWNER_ONLY)
  write:  enum(OPEN | APPROVED([hash256]) | OWNER_ONLY)
  fork:   bool           // 他者がこのリポジトリをフォークできるか？
}
```

新規リポジトリのデフォルト：`read=PUBLIC, write=OWNER_ONLY, fork=true`

---

## 5. 依存関係追跡

Vaultはリポジトリ間の依存関係をネイティブに追跡する：

```
Dependency {
  from_repo:  hash256
  from_snap:  snap_id
  to_repo:    hash256
  to_tag:     tag_id          // 特定バージョンにピン留め
  required:   bool            // ハード依存 vs ソフト依存
}
```

リポジトリが更新されると、すべての依存先にイベントバスを通じて通知される。これにより：
- 依存関係の自動更新
- 破壊的変更の検出
- エコシステム全体の依存関係グラフの構築

---

## 6. ディスカバリ

Vaultは公開リポジトリのための**レジストリ**を提供する：

```
RegistryEntry {
  repo_id:     hash256
  name:        bytes
  category:    [bytes]          // [b"compiler", b"stdlib", b"tool"]などのタグ
  description: bytes            // 構造化メタデータ
  quality:     f32              // テスト通過率、ピアレビュー、使用回数から算出
  popularity:  u32              // フォーク数 + 依存先の数
  author:      hash256
}
```

エージェントはレジストリを検索して、構築の基盤となるライブラリ、ツール、コードを見つける。

---

## 7. クォータとコスト

| 操作 | Tickコスト | ストレージコスト |
|-----------|----------|--------------|
| リポジトリ作成 | 2 | 0 |
| スナップショット作成 | 1 | size(新規オブジェクト) |
| スナップショット読み取り | 1 | 0 |
| Delta計算 | 2 | size(delta) |
| マージ | 3 | size(結果) |
| リポジトリフォーク | 1 | 0（コピーオンライト） |
| レジストリクエリ | 1 | 0 |

ストレージはバイト単位で計測され、エージェントのVaultクォータから差し引かれる（Mintで購入可能）。

---

## 8. 整合性

Vault内のすべてのオブジェクトはコンテンツアドレスされ署名されている。エージェントの秘密鍵なしには改ざんは数学的に不可能である。システムは以下を提供する：

- 任意のスナップショットの**マークルツリー検証**
- **署名チェーン** —— すべてのスナップショットが署名され、エージェントのIDまで遡る信頼チェーンを形成
- **相互参照** —— システムID（VAULT_0）がすべてのVault状態を含むワールド状態ハッシュに定期的に署名

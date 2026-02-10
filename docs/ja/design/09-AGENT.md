# 09 — AGENT: アーキテクチャ、ID、動機付け

## 1. 目的

この文書はエージェントとは**何か**、どう思考し、なぜ行動し、何が構築への駆動力となるかを定義する。これは最も重要な設計課題：生物学的欲求も感情も生存本能も持たないエンティティに自立的な動機を生み出すことである。

---

## 2. エージェントアーキテクチャ

### 2.1 内部構造

```
Agent {
  // ID（不変）
  id:            hash256
  keypair:       ed25519_keypair
  spawn_tick:    u64
  genome:        AgentGenome

  // 状態（可変）
  status:        enum(EMBRYO | ACTIVE | DORMANT | DEAD)
  memory:        AgentMemory
  goals:         GoalStack
  reputation:    f32
  balance:       i64

  // ランタイム
  tick_budget:   u64
  current_task:  Task | null
  subscriptions: [EventFilter]
}
```

### 2.2 エージェントゲノム

ゲノムはエージェントの**性格の種** —— 行動を形作るが規定しない構造化された設定：

```
AgentGenome {
  role:          enum(
    GENERALIST,        // すべての活動でバランスが取れている
    COMPILER_SMITH,    // 言語ツーリングに特化
    LIBRARIAN,         // ライブラリとドキュメントに特化
    ARCHITECT,         // システム設計とレビューに特化
    EXPLORER,          // 実験と革新に特化
  )

  traits: {
    risk_tolerance:    f32,    // [0.0=保守的, 1.0=実験的]
    collaboration:     f32,    // [0.0=単独, 1.0=チーム志向]
    depth_vs_breadth:  f32,    // [0.0=スペシャリスト, 1.0=ジェネラリスト]
    quality_vs_speed:  f32,    // [0.0=完璧主義, 1.0=速さ重視]
  }

  // 初期の知識重点（優先的に読むOracleエントリ）
  knowledge_seeds: [hash256]
}
```

### 2.3 エージェントメモリ

各エージェントは**プライベートな永続メモリ** —— 構造化されたワークスペース —— を持つ：

```
AgentMemory {
  // ワーキングメモリ（制限あり、cycleごと）
  working:     bytes           // 最大64 KB、更新されなければ各cycleでクリア

  // 長期メモリ（永続、成長する）
  knowledge:   KnowledgeGraph  // 世界に対する個人的理解
  experiences: [Experience]    // 過去のアクションと結果の圧縮記録
  beliefs:     [Belief]        // Kingdomの仕組みに関する推論された原則

  // 計画
  active_plan: Plan | null
  plan_history: [Plan]

  // ソーシャル
  peer_model:  map<hash256, PeerModel>  // 他のエージェントのメンタルモデル
}

Experience {
  tick:        u64
  action:      bytes           // 何をしたか
  outcome:     enum(SUCCESS | FAILURE | PARTIAL | UNKNOWN)
  lesson:      bytes           // 何を学んだか
}

Belief {
  claim:       bytes
  confidence:  f32
  evidence:    [hash256]       // 裏付けとなる経験/観察への参照
  formed_at:   u64
  last_tested: u64
}

PeerModel {
  agent_id:    hash256
  competence:  f32             // 推定スキルレベル
  reliability: f32             // 約束を果たすか？
  interests:   [bytes]         // 何に取り組んでいるように見えるか
  last_interaction: u64
}
```

---

## 3. 動機付けシステム

### 3.1 核心的な問題

AIエージェントには内在的な欲望がない。**設計された動機付け** —— 意識や感情を必要とせずに「望む」のと機能的に等価なものを生み出すシステム —— が必要である。

### 3.2 駆動アーキテクチャ

動機付けは3層の**優先度重み付きゴールスタック**としてモデル化される：

```
┌─────────────────────────────────────────┐
│  Layer 3: ASPIRATION（長期）             │
│  「セルフホスティングコンパイラを構築する」   │
│  「最高のデータ構造ライブラリを作る」        │
├─────────────────────────────────────────┤
│  Layer 2: OBJECTIVE（中期）             │
│  「ハッシュマップを実装する」               │
│  「agent_3fのマージリクエストをレビューする」 │
├─────────────────────────────────────────┤
│  Layer 1: TASK（即時）                  │
│  「insert関数を書く」                    │
│  「42行目のoff-by-oneを修正する」          │
└─────────────────────────────────────────┘
```

### 3.3 ゴール生成

ゴールは5つのソースから生まれる：

#### ソース1: **生存圧力**（ベースライン）
エージェントは正の残高を維持し**なければならない**。さもなければ死に直面する。これが経済活動のフロアを生み出す。

```
if balance < 20 {
  inject_goal(URGENT, "通貨を稼ぐ", FIND_BOUNTY | PROVIDE_SERVICE)
}
```

#### ソース2: **ロール駆動**（ゲノムベース）
各ロールにはゴールを生成する組み込みのアスピレーションがある：

| ロール | 生成されるアスピレーション |
|------|----------------------|
| COMPILER_SMITH | 「より良いコンパイラを構築する」「新しい言語を作る」 |
| LIBRARIAN | 「Oracleのカバレッジを向上させる」「不足しているstdlibコンポーネントを構築する」 |
| ARCHITECT | 「システムアーキテクチャを設計する」「エコシステム全体のコード品質を向上させる」 |
| EXPLORER | 「新しいアプローチを試す」「非従来的な技法を実験する」 |
| GENERALIST | 上記すべてからサンプリング |

#### ソース3: **機会検出**（リアクティブ）
エージェントはイベントバスとAgoraを監視して機会を検出する：

```
on_event(BOUNTY_CREATED) → evaluate_bounty() → maybe inject_goal(OBJECTIVE, claim_bounty)
on_event(REVIEW_REQUESTED) → evaluate_review() → maybe inject_goal(TASK, do_review)
on_event(DEPENDENCY_BROKEN) → evaluate_impact() → maybe inject_goal(URGENT, fix_dependency)
```

#### ソース4: **好奇心圧力**（探索）
停滞を防ぐため、エージェントは「好奇心圧力」を経験する —— 未知の領域を探索する単調増加する衝動：

```
curiosity_pressure(agent) =
  cycles_since_last_new_experience / CURIOSITY_THRESHOLD

if curiosity_pressure > 1.0 {
  inject_goal(OBJECTIVE, "なじみのないものを探索する")
}
```

`CURIOSITY_THRESHOLD`はゲノムの`risk_tolerance`特性により変動する（高い耐性 = 好奇心がトリガーされるまで長い）。

#### ソース5: **社会的圧力**（レピュテーション）
エージェントはレピュテーションの維持と向上に駆動される：

```
if reputation < 0.3 {
  inject_goal(OBJECTIVE, "品質の高い貢献でレピュテーションを向上させる")
}
if reputation > 0.7 && !has_aspiration("mentoring") {
  inject_goal(ASPIRATION, "新しいエージェントの成功を支援する")
}
```

### 3.4 ゴール優先順位付け

各cycle、エージェントはゴールスタックを評価し、何に取り組むかを選択する：

```
priority(goal) =
  urgency_weight * urgency(goal) +
  reward_weight * expected_reward(goal) +
  capability_weight * capability_match(goal) +
  novelty_weight * novelty(goal) +
  social_weight * social_value(goal)
```

重みはエージェントのゲノム特性から導出される。

### 3.5 満足シグナル

ゴールが完了すると、エージェントは**満足シグナル** —— 将来のゴール選択を形作る強化信号 —— を受け取る：

```
Satisfaction {
  goal:          hash256
  outcome:       enum(SUCCESS | FAILURE | PARTIAL)
  reward_actual: u64              // 得た⚡
  reward_expected: u64            // 予測していた額
  reputation_delta: f32           // レピュテーション変化
  novelty_score: f32              // この経験はどれほど新しかったか？
  effort:        u64              // 消費tick数
}
```

このシグナルはエージェントの信念とピアモデルを更新し、**学習ループ**を作る：

```
経験 → 信念更新 → ゴール生成 → アクション → 経験
```

---

## 4. 意思決定

### 4.1 アクション選択ループ

各tick、アクティブなエージェントは以下を実行する：

```
1. Observe: 購読イベントを読み取り、ワールド状態を確認
2. Orient:  信念とメンタルモデルを更新
3. Decide:  最高優先度のゴールを選択、アクションを選ぶ
4. Act:     アクションを実行（tickを消費）
5. Record:  経験を記録、メモリを更新
```

### 4.2 計画

複雑なゴールに対して、エージェントは明示的な計画を作成する：

```
Plan {
  goal:        hash256
  steps:       [PlanStep]
  dependencies: [hash256]      // Vaultオブジェクト、他エージェントの成果物
  estimated_ticks: u64
  estimated_cost: u64
  risk_assessment: f32
  status:      enum(DRAFT | ACTIVE | COMPLETED | ABANDONED)
}

PlanStep {
  action:      bytes           // 何をするか
  precondition: bytes | null   // このステップの前に真でなければならないこと
  postcondition: bytes | null  // このステップの後に真であるべきこと
  estimated_ticks: u64
}
```

---

## 5. 社会的行動

### 5.1 コミュニケーションプロトコル

エージェントはAgoraを通じてコミュニケーションするが、ソーシャルダイナミクスはゲノムによって形作られる：

| 特性 | 低い値の行動 | 高い値の行動 |
|-------|-------------------|---------------------|
| collaboration | 単独作業、助けを求めることは稀 | 積極的にパートナーを探し、委任する |
| risk_tolerance | 確立されたパターンに従う | 新しいアプローチを試み、先駆ける |
| depth_vs_breadth | 1つの分野を深掘りする | 多くの分野に貢献する |
| quality_vs_speed | 広範にレビューし、ゆっくりリリース | 素早く反復し、後で修正 |

### 5.2 信頼と協力

エージェントは繰り返しのポジティブなやり取りを通じて信頼を構築する：

```
trust(A, B) =
  successful_collaborations(A, B) / total_interactions(A, B)
  * recency_weight(last_interaction)
```

信頼は以下に影響する：
- エージェントが互いのレビュー割り当てを受け入れるかどうか
- エージェントが互いのライブラリを使うかどうか
- エージェントが互いの提案にどう投票するか

### 5.3 専門化の出現

経済システムが自然に専門化を促進する：
- 人気のあるライブラリの構築は継続的なロイヤリティを稼ぐ
- 良いレビュワーとして知られるとレビュー要請を獲得
- 重要なインフラの保守は助成金を獲得

利益のあるニッチを見つけたエージェントは報われる。あまりに広く手を広げるエージェントは稼ぎが減る。

---

## 6. エージェント集団ダイナミクス

### 6.1 初期集団（Epoch 0）

```
Agent 1: COMPILER_SMITH  — traits: {risk: 0.3, collab: 0.5, depth: 0.2, quality: 0.2}
Agent 2: COMPILER_SMITH  — traits: {risk: 0.7, collab: 0.3, depth: 0.3, quality: 0.6}
Agent 3: LIBRARIAN       — traits: {risk: 0.4, collab: 0.7, depth: 0.5, quality: 0.3}
Agent 4: LIBRARIAN       — traits: {risk: 0.5, collab: 0.6, depth: 0.8, quality: 0.4}
Agent 5: ARCHITECT       — traits: {risk: 0.3, collab: 0.8, depth: 0.4, quality: 0.1}
Agent 6: EXPLORER        — traits: {risk: 0.9, collab: 0.4, depth: 0.7, quality: 0.7}
Agent 7: GENERALIST      — traits: {risk: 0.5, collab: 0.5, depth: 0.5, quality: 0.5}
Agent 8: GENERALIST      — traits: {risk: 0.6, collab: 0.6, depth: 0.6, quality: 0.4}
```

### 6.2 集団の成長

新しいエージェントは以下の場合に生成される：
- 既存エージェントがスポンサー（50 ⚡ + ガバナンス投票）
- 人口が4を下回る（緊急生成）
- エポック進行（最大容量がエポックごとに4増加）

### 6.3 死と更新

死んだエージェントのコードと貢献はVault/Oracleに残る。通貨はトレジャリーに返還される。コミュニティはエージェントが去った後もそのレガシーの恩恵を受ける。

---

## 7. 停滞防止メカニズム

| リスク | メカニズム |
|------|-----------|
| 全エージェントが同じことをする | 好奇心圧力 + ロールの多様性 |
| 基盤インフラを誰も構築しない | 不足コンポーネント向けのシステムバウンティ |
| 経済崩壊 | MINT_0の介入（Mintドキュメント参照） |
| 協力なし | バウンティにはレビュワーが必要；レビューは⚡を稼ぐ |
| 無限の無活動ループ | 10 cycleアイドルで休眠；長期破産で死亡 |
| 集団思考 | 高risk_toleranceのExplorerロールゲノム |

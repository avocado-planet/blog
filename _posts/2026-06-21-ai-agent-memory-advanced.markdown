---
layout: post
title:  "AI Agentのメモリ設計 深掘り編：依存関係・挿入戦略・更新コスト"
date:   2026-06-21
description: 前回の3層構造の続き。メモリ間の依存関係問題、全件挿入vsRAP、更新コストのバッファリング最適化まで、この会話自体を実例に整理する。
---

前回の記事（[AI Agent CLIのメモリ設計：Claude Codeはどうやって「あなた」を覚えているのか](/blog/2026/06/20/ai-agent-cli-memory.html)）では、メモリの3層構造・CLAUDE.md・更新タイミングの基礎を整理した。

今回はその続き。**この会話自体がExternal Memoryの実演**になっているという気づきから始まり、依存関係・挿入戦略・更新コストの最適化まで深掘りする。

---

## この会話自体が実演だった

New Chatを開いた瞬間、ClaudeはわたしがLangGraphでAI Agent CLIを作っていることを知っていた。

これは前回説明したExternal Memoryのフローそのものだ：

```
過去のセッション
    ↓ (Anthropicのシステムが重要情報を抽出・保存)
外部DB (userMemories)
    ↓ (New Chat開始時にsystem promptへ注入)
今のセッション ← Claudeが「知っている」理由
```

実際にsystem promptに注入されているuserMemoriesには、職業・居住地・進行中プロジェクト（フェーズ進捗まで）・GitHubのオーガニゼーション名などが含まれている。

逆に**忘れられている情報**：
- 過去の会話の具体的なやりとり
- デバッグ中に試したコードの詳細
- 解決済みエラーの内容

要約・抽象化された情報だけが残る、というのが実際の動作だ。

---

## メモリ間の依存関係問題

ここが設計の難所だ。

たとえばこういうメモリがあるとする：

- 「PENGは東京在住」
- 「小田急線で通勤30分」

ここで「福岡に引っ越した」と伝えたら？

東京在住は更新されるが、**小田急線の情報も無効になる**はずだ。しかしフラットな箇条書き管理では依存関係が暗黙的にしか存在せず、芋づる式の削除が起きない可能性がある。

```
記憶A: 東京在住
記憶B: 小田急線で通勤30分  ← 記憶Aに依存しているが明示されていない

新情報: 福岡に引っ越し
    ↓
記憶Aは更新 → 記憶Bが残ったまま ← 矛盾発生
```

LLMとして意味的な矛盾検出は得意だ。「福岡→小田急線に乗れない→通勤30分も無効」という推論チェーンは理解できる。ただしそれをメモリに**自動反映できるかは別問題**だ。

### 依存関係を明示する設計

```python
memories = [
    {
        "id": "M001",
        "fact": "東京在住",
        "tags": ["location", "residence"]
    },
    {
        "id": "M002",
        "fact": "小田急線で通勤30分",
        "tags": ["commute"],
        "depends_on": ["M001"]  # ← 依存関係を明示
    }
]

def update_memory(new_fact: str, category: str):
    # 依存しているメモリも検証対象に含める
    affected = find_dependent_memories(new_fact)
    llm.validate_and_prune(affected)
```

完全な依存グラフ管理は複雑すぎるので、実用的にはタグベースで近似するのが現実的だ：

```python
# locationタグが変わったらcommuteタグも再検証
def update_memory(new_fact: str, category: str):
    related = memory_store.filter(category=category)
    # 依存カテゴリも追加
    dependents = CATEGORY_DEPS.get(category, [])
    for dep in dependents:
        related += memory_store.filter(category=dep)
    validated = llm.check_conflicts(new_fact, related)
    memory_store.update(validated)
```

---

## 全メモリを毎回挿入するか？

メモリ件数によってアプローチが変わる。

### 小規模（〜数十件）→ 全件挿入でOK

シンプルで確実。Claude.aiのuserMemoriesはおそらくこの方式だ。

### 大規模（数百〜数千件）→ RAGで関連分だけ挿入

```
ユーザーの質問
    ↓ Embedding化
ベクトル検索
    ↓ 類似度上位N件だけ取得
System Promptに挿入
```

ただし全件挿入でも**コンテキストウィンドウの圧迫**という問題がある：

```
モデルのコンテキスト上限
├── System Prompt（メモリ含む）← 増え続ける
├── 会話履歴               ← 増え続ける
└── 今の質問・回答         ← 圧迫される
```

メモリが増えると会話履歴を削らざるを得なくなる、という逆転現象が起きる。

段階的アプローチが現実的だ：

```python
MAX_MEMORY_TOKENS = 2000

def inject_memories(query: str, all_memories: list) -> str:
    if token_count(all_memories) < MAX_MEMORY_TOKENS:
        return format_all(all_memories)           # 全件
    else:
        return search_relevant(query, all_memories)  # RAG
```

---

## 更新コストの最適化

「更新時に全メモリをチェックする」のはコスト的に条件がある：

```
更新コスト = 全メモリ件数 × トークン単価 × 更新頻度
```

更新が多い × メモリが多い = コスト爆発。

### 規模と更新頻度で戦略を使い分ける

```
              メモリ少ない    メモリ多い
             ┌────────────┬────────────┐
更新少ない    │ 全件チェック │ 全件チェック│
             ├────────────┼────────────┤
更新多い     │ 全件チェック │ 局所チェック│
             └────────────┴────────────┘
```

### バッファリングで一括処理

最もコスパが良いのはセッション単位でまとめて処理する方法だ：

```
会話中の新情報
    ↓
更新キューに溜める（即時チェックしない）
    ↓
セッション終了時 or 閾値到達時に一括処理
    ↓
1回のAPI呼び出しで複数更新をまとめて処理
```

### 戦略の選び方

| 戦略 | 適する場面 |
|------|-----------|
| 全件チェック | 更新が1日数回程度 |
| カテゴリ局所チェック | 更新が多いが分類できる |
| バッファリング一括処理 | リアルタイム性不要な用途 |
| RAG類似検索 | メモリが数百件超 |

個人用途のAgent CLIなら**バッファリング一括処理**が最もコスパが良い。

---

## agent-ai-cliへの応用まとめ

現在のフェーズ（LangGraph ReAct完了）を踏まえた実装イメージ：

```python
class AgentMemory:
    # Layer 1: In-context（LangGraph Stateで自動管理）
    conversation_history: list[Message]

    # Layer 2: External（Phase 4で実装予定）
    user_memory: str       # ~/.agentcli/memory.md
    project_memory: str    # ./AGENT.md（CLAUDE.md相当）

    # Layer 3: Episodic（将来）
    past_sessions: VectorDB
```

Phase 4の実装優先順位として考えると：

1. セッション終了フック → LLMによる要約 → ファイル保存（最小実装）
2. カテゴリタグ付き依存関係管理
3. トークン上限監視 → 自動RAG切り替え
4. バッファリング更新キュー

メモリ設計は「何を覚えるか」だけでなく、**依存関係・挿入戦略・更新コスト**の3つを同時に考える必要がある。

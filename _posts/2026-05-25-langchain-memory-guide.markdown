---
layout: post
title:  "LangChainのメモリ（会話履歴）完全ガイド — レガシーからLangGraphへ"
date:   2026-05-25
description: LangChainアプリにおける会話履歴の保持方法を徹底解説。旧Memoryクラスの仕組みと廃止の背景、現在推奨されるLangGraphのCheckpointer/Storeベースの設計まで、メリット・デメリットを比較しながら整理します。
---

LLMは本質的にステートレスです。`invoke()` を呼ぶたびに、前のやり取りは一切覚えていません。チャットボットや対話型エージェントを作るには、**会話履歴をどう保持するか**が設計上の重要な判断ポイントになります。

この記事では、LangChainにおけるメモリの仕組みを「旧方式（レガシーMemoryクラス）」と「現行方式（LangGraphベース）」の両面から整理し、それぞれのメリット・デメリット、そして実務でどう選ぶべきかを解説します。

## なぜメモリが必要なのか

```python
# メモリなし — 毎回リセットされる
result1 = llm.invoke("私の名前はPENGです")  # → "はじめまして！"
result2 = llm.invoke("私の名前は？")          # → "わかりません"
```

ユーザーが「さっき言ったじゃん」と感じる瞬間、それはメモリが欠けている証拠です。メモリの役割は、過去のメッセージをプロンプトに注入して、LLMに「文脈を知っている風」に振る舞わせることです。

## 旧方式：LangChain Memoryクラス（v0.3.1で非推奨）

LangChain v0.2以前では、`ConversationChain` と組み合わせる専用のMemoryクラスが提供されていました。v0.3.1で**すべて非推奨（deprecated）**となり、v1.0で削除予定ですが、概念として理解しておく価値はあります。

### 1. ConversationBufferMemory

全メッセージをそのまま保持する、最もシンプルな方式。

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(return_messages=True)
```

| メリット | デメリット |
|---|---|
| 実装が最も簡単 | 会話が長くなるとトークン数が爆発する |
| 情報の欠落がない | APIコストが会話長に比例して増大 |
| デバッグしやすい | コンテキストウィンドウ超過でエラーになる |

**向いている場面：** 5〜10ターン程度の短い会話。プロトタイピングやデモ。

### 2. ConversationBufferWindowMemory

直近N回分のメッセージだけを保持する。

```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(k=5, return_messages=True)
```

| メリット | デメリット |
|---|---|
| トークン使用量が一定に収まる | 古い文脈が完全に消える |
| コスト予測しやすい | 「最初に伝えたこと」を忘れる |

**向いている場面：** FAQボット、1問1答に近いサポート。

### 3. ConversationSummaryMemory

LLMを使って過去の会話を要約し、要約文だけを保持する。

```python
from langchain.memory import ConversationSummaryMemory

memory = ConversationSummaryMemory(llm=summary_llm)
```

| メリット | デメリット |
|---|---|
| 長い会話でもトークンが膨らまない | 要約のたびにLLM呼び出し → コスト・レイテンシ増 |
| 「大筋」は保持される | 細かい数値や名前が要約で消えることがある |

**向いている場面：** 数十ターンに及ぶ長い相談、カウンセリング的な対話。

### 4. ConversationSummaryBufferMemory

SummaryとBufferのハイブリッド。古いメッセージは要約し、直近メッセージはそのまま保持する。

```python
from langchain.memory import ConversationSummaryBufferMemory

memory = ConversationSummaryBufferMemory(
    llm=summary_llm,
    max_token_limit=800
)
```

| メリット | デメリット |
|---|---|
| 近い文脈は正確、遠い文脈も概要として残る | 要約LLMのプロンプト設計が品質を左右する |
| 実用上のバランスが良い | 実装がやや複雑 |

**向いている場面：** 旧方式の中では最も実用的。中〜長期的な対話。

### 5. ConversationEntityMemory

会話からエンティティ（人名、会社名、概念など）を抽出し、エンティティ単位で情報を保持する。

| メリット | デメリット |
|---|---|
| 「誰が何をした」を構造的に記憶 | エンティティ抽出の精度がLLMに依存 |
| 複数トピックの並行管理に強い | 実装・デバッグが複雑 |

### なぜ全部非推奨になったのか

旧Memoryクラスが廃止された理由は主に3つです。

1. **ツール呼び出しに非対応：** 旧Memoryはチャットモデルのtool calling APIより前に設計された。ToolMessage を正しくハンドリングできない
2. **プロセス内メモリ限定：** Pythonプロセスを再起動すると会話履歴が消える。本番運用に耐えない
3. **APIの一貫性欠如：** Memory種別ごとにインターフェースが微妙に異なり、切り替えが面倒

## 現行方式：LangGraphのCheckpointer + Store

LangChain v0.3以降の公式推奨は **LangGraph のCheckpointerベースのメモリ**です。メモリをアプリケーション層から切り離し、永続化層として独立させた設計になっています。

### 短期メモリ：Checkpointer（スレッド内）

Checkpointerは、グラフの各ステップ実行後に**Stateのスナップショット**を自動保存する仕組みです。

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, MessagesState, START, END

# グラフ定義
builder = StateGraph(MessagesState)
builder.add_node("chat", chat_node)
builder.add_edge(START, "chat")
builder.add_edge("chat", END)

# Checkpointerでコンパイル
checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# thread_idで会話を識別
config = {"configurable": {"thread_id": "user_001"}}

# 1ターン目
graph.invoke({"messages": [("user", "私はPENGです")]}, config)

# 2ターン目 — 前のやり取りを覚えている
graph.invoke({"messages": [("user", "私の名前は？")]}, config)
# → "PENGさんですね"
```

ポイントは `thread_id` です。同じ `thread_id` なら同じ会話として状態が復元され、異なる `thread_id` なら完全に独立した会話になります。

#### Checkpointerの種類

| Checkpointer | 用途 | 特徴 |
|---|---|---|
| `MemorySaver` | 開発・テスト | プロセス再起動で消える |
| `SqliteSaver` | ローカルテスト | ファイルに永続化。再起動に耐える |
| `PostgresSaver` | 本番運用 | 水平スケーリング、クラッシュ復旧可能 |

開発時は `MemorySaver` → ローカルテストで `SqliteSaver` → 本番で `PostgresSaver` とステップアップしていくのが典型的なパスです。インターフェースは全て共通なので、コード変更は1行で済みます。

### 長期メモリ：Store（スレッド横断）

Checkpointerは「1つの会話スレッド内」の記憶です。**ユーザーの好みや過去の注文履歴など、スレッドを跨いで覚えておきたい情報**にはStoreを使います。

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

# ノード関数内で明示的にread/writeする
def chat_node(state, config, *, store):
    user_id = config["configurable"]["user_id"]

    # 長期メモリから読み出し
    memories = store.search(("user_prefs", user_id))

    # ... LLM呼び出し ...

    # 長期メモリに書き込み
    store.put(("user_prefs", user_id), "lang", "ja")
```

**Checkpointer vs Store の使い分け：**

| 観点 | Checkpointer（短期） | Store（長期） |
|---|---|---|
| スコープ | 1つの `thread_id` 内 | `user_id` やカスタム namespace |
| 自動/手動 | 自動（グラフステップごと） | 手動（ノード内でread/write） |
| 消えるタイミング | スレッド終了時（論理的に） | 明示削除まで残る |
| 主な用途 | 会話履歴 | ユーザー設定、学習データ |

### なぜLangGraphベースが優れているのか

1. **永続化がデフォルト：** Checkpointerを指定するだけで、プロセス再起動しても会話が復元される
2. **ツール呼び出しと自然に統合：** `ToolMessage` を含むState全体が保存される
3. **タイムトラベル：** 過去のチェックポイントに戻ってやり直せる（Human-in-the-loopに必須）
4. **バックエンド交換が容易：** `MemorySaver` → `PostgresSaver` の切り替えが1行

## 会話が長くなったらどうする？ — トリミング戦略

LangGraphでは会話履歴はStateに全て蓄積されるので、長い会話ではコンテキストウィンドウを超えるリスクがあります。対策として、ノード内でメッセージをトリミングする方法が推奨されています。

```python
from langchain_core.messages import trim_messages

def chat_node(state):
    # 直近のメッセージだけをLLMに渡す
    trimmed = trim_messages(
        state["messages"],
        max_tokens=4000,
        strategy="last",
        token_counter=llm,
    )
    return {"messages": [llm.invoke(trimmed)]}
```

旧方式の `ConversationBufferWindowMemory` に相当する処理ですが、LangGraph版では「Stateには全履歴を保持しつつ、LLMに渡す分だけトリミングする」という設計が可能です。必要に応じて古い履歴を参照できるのが利点です。

## 実務での推奨パターン

### シンプルなチャットボット

```
LangGraph StateGraph + MemorySaver（開発） or PostgresSaver（本番）
+ trim_messages でトークン管理
```

これが2026年時点で最もスタンダードな構成です。旧Memoryクラスを使う理由はありません。

### ユーザーごとのパーソナライズが必要な場合

```
Checkpointer（会話履歴） + Store（ユーザーの長期記憶）
```

Storeにはユーザーのプロファイル情報や過去の意思決定を保存し、毎会話の冒頭でプロンプトに注入します。

### 要約が必要な超長期対話

ノード内で定期的にLLMで要約を生成し、Storeに保存する方式を自分で組みます。旧 `ConversationSummaryMemory` に相当しますが、LangGraphでは要約のタイミングやロジックを完全にコントロールできます。

## まとめ

| 方式 | ステータス | おすすめ度 |
|---|---|---|
| ConversationBufferMemory 等 | v0.3.1で非推奨、v1.0で削除予定 | ❌ 新規採用不可 |
| RunnableWithMessageHistory | v0.3で非推奨 | ❌ |
| LangGraph Checkpointer | 現行推奨 | ✅ これを使う |
| LangGraph Store | 長期メモリが必要な場合 | ✅ Checkpointerと併用 |

LangChainのメモリAPIは18ヶ月で3回変わりました。旧Memoryクラスのコンセプト自体は今でも有用な知識ですが、新規開発では**LangGraphのCheckpointer一択**です。Stateの永続化、ツール呼び出し対応、バックエンドの柔軟性 — すべてが旧方式より優れています。

まずは `MemorySaver` で試して、本番に近づいたら `PostgresSaver` に切り替える。このシンプルなパスを覚えておけば十分です。

---
layout: post
title:  "ReAct Agentの本質を理解する — プロンプト芸か、構造的制御か？"
date:   2026-04-19
description: ReAct Agentのサイクルは本当にプロンプトだけで実現しているのか？原初の実装から現代のフレームワークまで、その本質を解説します。
---

# ReAct Agentの本質を理解する — プロンプト芸か、構造的制御か？

## はじめに

LLMベースのエージェント開発において「ReAct」は最も基本的なパターンの一つです。Thought（思考）→ Action（行動）→ Observation（観察）のサイクルを繰り返すことで、LLMが外部ツールを使いながら推論を進めます。

しかし、ここで根本的な疑問が生じます：

> **このサイクルは、実質的にプロンプトの指示によるものなのか？ LLMが本当にサイクルを守るかは、LLMの判断任せなのか？**

本記事では、この問いに対して「原初のReAct」と「現代のフレームワーク実装」の2つの視点から掘り下げます。

---

## 1. ReActの基本サイクル

ReActパターン（Yao et al., 2022）は以下のサイクルで構成されます：

1. **Thought（思考）** — 現状を分析し、次に何をすべきか考える
2. **Action（行動）** — ツールを呼び出す
3. **Observation（観察）** — ツールの実行結果を受け取る
4. **1に戻る** — 目的が達成されるまで繰り返す

---

## 2. 原初のReAct — プロンプト + 素朴なループ

### 2.1 Few-shotによるフォーマット誘導

原論文のReActでは、LLMに対して**few-shot（少数例提示）**でフォーマットを学ばせていました。

few-shotとは、プロンプトの中に「こう振る舞え」という完成例を数個入れる手法です。LLMは明示的なルール説明ではなく、**例のパターンを模倣**することで所望のフォーマットに従います。

```text
以下の例に従って、質問に答えてください。

--- 例1 ---
Question: コロラド造山運動の東側の地域の標高範囲は？
Thought 1: コロラド造山運動について調べる必要がある。
Action 1: Search[コロラド造山運動]
Observation 1: コロラド造山運動は約3億年前に起きた造山運動で...
Thought 2: 東側の地域について調べよう。
Action 2: Search[High Plains 標高]
Observation 2: High Plainsの標高は約1,800〜7,000フィート。
Thought 3: 答えがわかった。
Action 3: Finish[約1,800〜7,000フィート]

--- 例2 ---
(同様のパターンをもう1〜2個)

--- 本番 ---
Question: ユーザーの実際の質問
Thought 1:
```

LLMはこのパターンを見て、「Thought → Action → Observation」の形式で出力すればいいと**推測**します。

> **補足：Few-shotの分類**
> - **zero-shot** — 例なし。指示文だけ
> - **one-shot** — 例が1個
> - **few-shot** — 例が2〜5個

### 2.2 ツール実行は誰がやるのか？

ここが重要なポイントです。**LLMはテキストを生成するだけ**であり、実際にツールを実行する能力はありません。

原初のReActにも**制御コード（外部のPythonプログラム）は存在していました**。

```python
prompt = few_shot_examples + question

while True:
    # 1. LLMに投げる（Thought + Action をテキストとして生成させる）
    output = llm(prompt)

    # 2. 出力をテキストとしてパースする（正規表現等）
    action = parse_action(output)  # 例: "Search[東京の人口]"

    if action.type == "Finish":
        return action.argument  # 最終回答

    # 3. ツールを実際に実行する（Pythonコードがやる）
    observation = execute_tool(action)  # Wikipedia APIを叩く等

    # 4. 結果をプロンプトに追記して次のループへ
    prompt += output + f"\nObservation: {observation}\n"
```

処理の流れを整理すると：

1. LLMが `Action: Search[東京の人口]` という**文字列を出力する**
2. 外部のPythonコードがその文字列を**正規表現でパースする**
3. Pythonコードが実際に**Wikipedia APIを叩く**
4. 結果を `Observation: 東京の人口は...` としてプロンプトに**テキスト連結する**
5. その拡張されたプロンプトをLLMに**再度投げる**

つまり、「制御コードがない時代」は存在しません。LLMは常にテキストを生成するだけで、ツール実行は必ず外部コードが担います。

---

## 3. 現代のReAct — フレームワークによる構造的制御

### 3.1 構造化 tool_call の登場

原初のReActには大きな弱点がありました。LLMの出力は自由テキストなので、正規表現でのパースが壊れやすいのです。

この問題を解決したのが、**LLMプロバイダー自身によるAPI仕様の構造化**です。

| プロバイダー | API仕様 | 導入時期 |
|---|---|---|
| OpenAI | `tool_calls` フィールド | 2023年6月〜 |
| Anthropic（Claude） | `tool_use` content block | 2024年4月〜 |
| Google（Gemini） | `function_call` パート | — |

**以前（自由テキスト時代）：**

```text
LLMの出力: "Thought: I need to search.\nAction: Search[東京の人口]"
→ 開発者が正規表現でパースする（壊れやすい）
```

**現在（API構造化）：**

```json
{
  "tool_calls": [
    {
      "function": {
        "name": "search",
        "arguments": "{\"query\": \"東京の人口\"}"
      }
    }
  ]
}
```

これはLLMの出力テキストではなく、**APIレスポンスの構造化フィールド**です。正規表現でパースする必要がなく、LLMが「フォーマットを守るかどうか」の不確実性が排除されました。

> **この構造化はLangChainが標準化したのではなく、LLMプロバイダー（OpenAI、Anthropic、Google等）がAPI仕様として定義したもの**です。LangChainはこれらプロバイダーごとの差異を統一的なインターフェースに抽象化しているだけです。

```text
OpenAI tool_calls ──┐
Anthropic tool_use ──┼→ LangChain 統一 ToolCall → LangGraph のルーティング
Gemini function_call ┘
```

### 3.2 フレームワークがサイクルを強制する

LangChainの `create_react_agent` やLangGraphでは、**フレームワークのコード（グラフのエッジ）がサイクルの構造を制御**しています：

1. LLMの出力をパースする — `tool_call` が返ったかどうかを判定
2. `tool_call` があれば → ツールを実行 → 結果をメッセージに追加 → LLMに再度投げる
3. `tool_call` がなければ → 最終回答として終了

```text
[LLM呼び出し]
     │
     ├─ tool_call あり → [ツール実行] → [結果追加] → [LLM呼び出し] に戻る
     │
     └─ tool_call なし → [終了・回答返却]
```

この分岐ロジックはPythonコード（LangGraphではconditional edge）であり、**LLMの「気分」に左右されません**。

---

## 4. 制御の境界線 — 誰が何を制御しているか

| 側面 | 誰が制御？ |
|---|---|
| サイクルの**構造**（ループ・分岐） | フレームワーク（コード） |
| サイクルの**中身**（何を考え、何を呼ぶか） | LLM |
| 終了条件の**判定ロジック** | フレームワーク（tool_callの有無） |
| 終了の**意思決定** | LLM（tool_callを出すか出さないか） |

LLMに任されている部分：

- **いつツールを呼ぶか**（どのタイミングでAction）
- **どのツールをどの引数で呼ぶか**
- **いつ終了するか**（tool_callを出さない ＝ 終了の意思表示）
- **Thoughtの質**（推論の正確さ）

---

## 5. 原初と現代の比較まとめ

| 観点 | 原初のReAct | 現代（LangChain/LangGraph等） |
|---|---|---|
| LLMの出力形式 | 自由テキスト `Action: Search[X]` | 構造化された `tool_call` JSON |
| パース方法 | 正規表現（壊れやすい） | APIレベルで構造化（堅牢） |
| ループ制御 | 素朴なwhileループ | グラフ構造（LangGraph） |
| ツール実行 | 外部コードが実行 | 外部コードが実行（同じ） |
| フォーマット遵守 | LLMの模倣能力に依存 | APIが構造を保証 |

---

## 6. 結論

「ReActはプロンプトだけで実現している」は初期の素朴な実装には部分的に当てはまりますが、本質的には**プロンプト（LLMの判断）＋ コード（構造的強制）のハイブリッド**です。

原初のReActにも外部の制御コードは存在しており、LLMは常に「テキストを出力するだけ」の存在でした。現代のフレームワークでは、構造化tool_callとグラフベースのルーティングにより、LLMに「サイクルを守れ」と祈る必要はなくなり、**コードが構造を保証する**設計に進化しています。

---

## 参考

- Yao, S. et al. (2022). *ReAct: Synergizing Reasoning and Acting in Language Models*
- [LangChain Documentation](https://python.langchain.com/)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview)

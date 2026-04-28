---
layout: post
title:  "MCPの仕組みを理解する：SSE・Stdio通信からLLMとのデータフローまで"
date:   2026-04-28
description: MCPサーバーの通信方式（SSE・Stdio）、LLMとのデータのやり取り、プロセス管理の仕組みを整理する
---

## MCPとは

MCP（Model Context Protocol）は、LLMエージェントが外部ツール（関数・APIなど）を呼び出すための通信プロトコルです。LangChainやClaude Desktopなどのクライアントが、MCPサーバーに接続してツールを取得・実行します。

---

## 通信方式：SSEとStdio

MCPには主に2つのトランスポートモードがあります。

### SSE（Server-Sent Events）

HTTPベースの通信方式。クライアントが1回だけ接続すると、サーバーが好きなタイミングで何度もデータを送り続けられます。

```python
# SSEモードで起動
mcp.run(transport="sse")

# または uvicorn で細かく制御
uvicorn.run(mcp.sse_app(), host="0.0.0.0", port=8001)
```

| 項目 | 内容 |
|---|---|
| 通信手段 | HTTP（`text/event-stream`） |
| 接続 | 1回のリクエストで維持 |
| 用途 | ネットワーク越しの接続、Colab環境 |

`mcp.run()` はMCPが内部でuvicornを起動するシンプルな方法。`uvicorn.run(mcp.sse_app())` はホスト・ポートを自由に指定でき、既存のFastAPIアプリへの組み込みも可能です。

Google Colabでは `stdio` モードが `sys.stdin.fileno()` エラーを起こすため、SSEモードが回避策として使われます。

---

### Stdio（標準入出力）

OSのパイプ機能でプロセスを直結する方式。HTTPを一切使いません。

```python
math_proc = subprocess.Popen(
    ["python", "/tmp/math_server.py"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
```

`subprocess.Popen` は起動したら即座に次の行へ進む（非同期）ため、サーバーを別プロセスで動かしながらクライアントコードを続けられます。

**プロセス構造：OSのパイプで2プロセスを直結**

<svg width="100%" viewBox="0 0 680 220" xmlns="http://www.w3.org/2000/svg">
  <defs><marker id="arr1" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></marker></defs>
  <rect x="30" y="20" width="620" height="180" rx="16" fill="#f1efe8" stroke="#888780" stroke-width="0.5"/>
  <text font-family="sans-serif" font-size="13" font-weight="500" x="340" y="46" text-anchor="middle" fill="#444441">OS（同一マシン）</text>
  <rect x="60" y="66" width="190" height="70" rx="8" fill="#eeedfe" stroke="#534ab7" stroke-width="0.5"/>
  <text font-family="sans-serif" font-size="13" font-weight="500" x="155" y="97" text-anchor="middle" fill="#3c3489">クライアント</text>
  <text font-family="sans-serif" font-size="11" x="155" y="116" text-anchor="middle" fill="#534ab7">LangChain / Claude</text>
  <rect x="430" y="66" width="190" height="70" rx="8" fill="#e1f5ee" stroke="#0f6e56" stroke-width="0.5"/>
  <text font-family="sans-serif" font-size="13" font-weight="500" x="525" y="97" text-anchor="middle" fill="#085041">サーバー</text>
  <text font-family="sans-serif" font-size="11" x="525" y="116" text-anchor="middle" fill="#0f6e56">math_server.py</text>
  <line x1="252" y1="92" x2="428" y2="92" stroke="#1D9E75" stroke-width="1.5" marker-end="url(#arr1)"/>
  <text font-family="sans-serif" font-size="11" x="340" y="85" text-anchor="middle" fill="#085041">stdin（書き込み）</text>
  <line x1="428" y1="112" x2="252" y2="112" stroke="#534ab7" stroke-width="1.5" marker-end="url(#arr1)"/>
  <text font-family="sans-serif" font-size="11" x="340" y="130" text-anchor="middle" fill="#3c3489">stdout（読み取り）</text>
  <rect x="60" y="155" width="560" height="32" rx="6" fill="#ffffff" stroke="#b4b2a9" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="340" y="175" text-anchor="middle" fill="#444441">subprocess.Popen(["python", "math_server.py"], stdout=PIPE, stderr=PIPE)</text>
</svg>

プロトコルは **JSON-RPC 2.0**。1行1メッセージ（改行区切り）でパイプを流れます。

```
クライアント → サーバーのstdinに書き込む
サーバー     → stdoutに書き込む → クライアントが読む
```

**JSON-RPC通信の流れ（クライアント ↔ サーバー）**

<svg width="100%" viewBox="0 0 680 400" xmlns="http://www.w3.org/2000/svg">
  <defs><marker id="arr2" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></marker></defs>
  <rect x="30" y="20" width="130" height="44" rx="8" fill="#eeedfe" stroke="#534ab7" stroke-width="0.5"/>
  <text font-family="sans-serif" font-size="13" font-weight="500" x="95" y="42" text-anchor="middle" fill="#3c3489">クライアント</text>
  <rect x="520" y="20" width="130" height="44" rx="8" fill="#e1f5ee" stroke="#0f6e56" stroke-width="0.5"/>
  <text font-family="sans-serif" font-size="13" font-weight="500" x="585" y="42" text-anchor="middle" fill="#085041">サーバー</text>
  <line x1="95" y1="64" x2="95" y2="380" stroke="#b4b2a9" stroke-width="1.5"/>
  <line x1="585" y1="64" x2="585" y2="380" stroke="#b4b2a9" stroke-width="1.5"/>
  <line x1="97" y1="100" x2="583" y2="100" stroke="#1D9E75" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="170" y="80" width="316" height="30" rx="6" fill="#eaf3de" stroke="#3b6d11" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="328" y="99" text-anchor="middle" fill="#27500a">{"method":"initialize","id":1}</text>
  <line x1="583" y1="148" x2="97" y2="148" stroke="#534ab7" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="170" y="128" width="316" height="30" rx="6" fill="#eeedfe" stroke="#534ab7" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="328" y="147" text-anchor="middle" fill="#3c3489">{"result":{"capabilities":{...}},"id":1}</text>
  <line x1="97" y1="196" x2="583" y2="196" stroke="#1D9E75" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="196" y="176" width="264" height="30" rx="6" fill="#eaf3de" stroke="#3b6d11" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="328" y="195" text-anchor="middle" fill="#27500a">{"method":"tools/list","id":2}</text>
  <line x1="583" y1="244" x2="97" y2="244" stroke="#534ab7" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="148" y="224" width="364" height="30" rx="6" fill="#eeedfe" stroke="#534ab7" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="328" y="243" text-anchor="middle" fill="#3c3489">{"result":{"tools":[{"name":"add",...}]},"id":2}</text>
  <line x1="97" y1="292" x2="583" y2="292" stroke="#1D9E75" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="100" y="272" width="460" height="30" rx="6" fill="#eaf3de" stroke="#3b6d11" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="328" y="291" text-anchor="middle" fill="#27500a">{"method":"tools/call","params":{"name":"add","arguments":{"a":3,"b":5}},"id":3}</text>
  <line x1="583" y1="340" x2="97" y2="340" stroke="#534ab7" stroke-width="1.5" marker-end="url(#arr2)"/>
  <rect x="170" y="320" width="316" height="30" rx="6" fill="#eeedfe" stroke="#534ab7" stroke-width="0.5"/>
  <text font-family="monospace" font-size="11" x="328" y="339" text-anchor="middle" fill="#3c3489">{"result":{"content":[{"text":"8"}]},"id":3}</text>
  <text font-family="sans-serif" font-size="11" x="30" y="104" fill="#639922">①</text>
  <text font-family="sans-serif" font-size="11" x="30" y="152" fill="#534ab7">②</text>
  <text font-family="sans-serif" font-size="11" x="30" y="200" fill="#639922">③</text>
  <text font-family="sans-serif" font-size="11" x="30" y="248" fill="#534ab7">④</text>
  <text font-family="sans-serif" font-size="11" x="30" y="296" fill="#639922">⑤</text>
  <text font-family="sans-serif" font-size="11" x="30" y="344" fill="#534ab7">⑥</text>
</svg>

通信の流れ：

| # | 方向 | 内容 |
|---|---|---|
| ① | → | `initialize`：接続確立の握手 |
| ② | ← | `capabilities`：サーバーの機能を返す |
| ③ | → | `tools/list`：ツール一覧を要求 |
| ④ | ← | `tools`：`add`, `multiply` などのリスト |
| ⑤ | → | `tools/call`：ツールを引数付きで実行 |
| ⑥ | ← | `content`：実行結果を返す |

| | Stdio | SSE |
|---|---|---|
| 通信手段 | OSパイプ | HTTP |
| 動作場所 | 同一マシン限定 | ネットワーク越しも可 |
| 速度 | 速い | やや遅い |

---

## LLMとMCP間のデータフロー

### ツール定義の取得

MCPクライアントが接続先サーバーに「どんなツールが使えるか？」を問い合わせるのが `get_tools()` です。サーバーはツールの名前・説明・引数の型をJSON形式で返します。これをLLMに渡すことで、LLMは「どのツールをどう使えるか」を把握できるようになります。

`get_tools()` はユーザーの入力を待たず、エージェント作成時に1回だけ実行されます。チャットが始まる前の「準備フェーズ」です。

```python
# 1. MCPサーバーへの接続設定
client = MultiServerMCPClient({
    "math": {"url": "http://localhost:8000/sse", "transport": "sse"}
})

# 2. サーバーに「使えるツール一覧をくれ」と問い合わせる
tools = await client.get_tools()

# 3. LLMとツール定義を組み合わせてエージェントを作る
agent = create_agent(model, tools)

# 4. ここで初めてユーザーのプロンプトを受け付ける
result = await agent.ainvoke({"messages": [...]})
```

LLMには以下のようなツール定義が渡ります。

```json
[
  {
    "name": "add",
    "description": "2つの数を足す",
    "parameters": {
      "a": { "type": "number" },
      "b": { "type": "number" }
    }
  }
]
```

### `ainvoke()` 呼び出し時

ユーザーのプロンプトが入ると、LLMにはツール定義とプロンプトが**同時に**送られます。

```
┌─ システム情報 ───────────────────────────┐
│ 使えるツール：add(a,b) / multiply(a,b)  │  ← get_tools()の結果
└─────────────────────────────────────────┘
┌─ ユーザーメッセージ ─────────────────────┐
│ "(3 + 5) x 12 を計算して"               │  ← ainvoke()の引数
└─────────────────────────────────────────┘
```

### ツール呼び出しの流れ

`"(3 + 5) x 12 を計算して"` に対してLLMが行う判断サイクル：

```
Human      "(3 + 5) x 12 を計算して"
AI         tool_call: add(a=3, b=5)
Tool       "8"
AI         tool_call: multiply(a=8, b=12)
Tool       "96"
AI         "答えは 96 です"   ← result['messages'][-1]
```

LLMが判断していること：

| 判断 | 内容 |
|---|---|
| ツール選択 | `add` と `multiply` のどちらを使うか |
| 引数の組み立て | 前のツール結果を次の引数に使う |
| 順序の決定 | 足し算 → 掛け算の順番 |
| 終了判断 | これ以上ツールは不要、回答を生成する |

この一連の判断サイクルが **ReAct**（Reasoning + Acting）の正体です。

---

## プロセスのライフサイクル

LangChainとClaude.aiでMCPの寿命は異なります。

### LangChain

```python
client = MultiServerMCPClient({...})
tools = await client.get_tools()   # ← ここで接続
# ...
# 関数が終わると client はGCに回収される → 接続切れ
```

ただし `subprocess.Popen` で起動したサーバープロセスは独立して動き続けます。明示的に `math_proc.terminate()` を呼ぶまで生存します。

### claude.ai

セッション単位で接続が維持されます。タブを閉じる・セッション終了でMCP接続も切れます。MCPサーバー自体はアプリ終了まで生存するのが一般的な設計です。

| | MCP接続（クライアント側） | MCPサーバープロセス |
|---|---|---|
| LangChain | 関数の実行中のみ | `terminate()` するまで生存 |
| claude.ai | セッション中は維持 | アプリ終了まで生存 |

「接続」と「プロセス」は別物として管理する必要があります。

---

## まとめ

- **SSE**：HTTPベース、ネットワーク越しに使える、Colab向き
- **Stdio**：パイプベース、ローカル限定、高速、Claude Desktop向き
- **ツール定義**はセッション開始時に1回取得してLLMに渡す
- **ReAct**ループでLLMがツールを選択・実行・結果を次の引数に使う
- **プロセスと接続**のライフサイクルは独立して管理する

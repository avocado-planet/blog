---
layout: post
title:  "MCPアーキテクチャ深掘り：Host-Client-Server構造、3つのプリミティブ、処理シーケンス"
date:   2026-05-28
description: MCPのHost-Client-Server 3層構造、Tools/Resources/Promptsの3つのプリミティブ、Anthropic Academy教材をベースにしたエンドツーエンドの処理シーケンスを整理する
---

前回の記事（[MCPの仕組みを理解する：SSE・Stdio通信からLLMとのデータフローまで](https://avocado-planet.github.io/blog/blog/mcp-stdio-sse-dataflow/)）では、MCPの通信方式（SSE/Stdio）、JSON-RPCの流れ、LLMとのデータフロー、プロセスのライフサイクルを整理した。

今回は **その上位レイヤー** にフォーカスする。MCPのアーキテクチャ設計（Host / Client / Server の役割分担）、サーバーが公開する3つのプリミティブ（Tools / Resources / Prompts）、そしてAnthropicの公式教材に基づくエンドツーエンドの処理シーケンスを詳しく見ていく。

前回の記事と合わせて読むと、MCPの全体像が **プロトコル層（前回）+ アーキテクチャ層（今回）** の両面から掴めるようになる。

---

## N×M問題とMCPの位置づけ

AIアプリケーションが外部ツールにアクセスするには、従来はツールごとに個別のインテグレーションコードを書く必要があった。N個のAIアプリ × M個のツール = N×M個のカスタム実装が必要になる、いわゆる **N×M問題** が存在していた。

![N×M問題とMCP](/blog/assets/img/mcp-nx-m-problem.svg)

HTTPがWeb通信を標準化したように、MCPはAIアプリとツール間の通信を標準化する。一度MCPサーバーを作れば、どのMCPホストからでも利用できる。

---

## Host-Client-Server 3層アーキテクチャ

MCPは **Host-Client-Server** の3つの役割で構成される。前回の記事では「クライアント」と「サーバー」の通信にフォーカスしたが、実はその上に **Host** というレイヤーがある。

![Host-Client-Server構造](/blog/assets/img/mcp-host-client-server.svg)

### MCP Host（ホスト）

ユーザーが直接操作する **AIアプリケーション本体**。

- Claude Desktop、Claude Code、VS Code（Copilot）、Cursor などが該当
- 内部に1つ以上のMCP Clientを生成・管理する
- LLMの推論処理を統括し、ユーザーとのインタラクションを担当
- **セキュリティの門番**: どのClientがどのToolを呼べるかのアクセス制御を行う
- 複数のMCP Serverから取得したコンテキストを **集約** してLLMに渡す

### MCP Client（クライアント）

Host内部に存在する **通信コンポーネント**。各MCP Serverとの1:1接続を維持する。

- HostがMCP Serverに接続するたびに、専用のClientインスタンスが1つ生成される
- ServerとのJSON-RPCセッションを管理（初期化、ケーパビリティネゴシエーション、終了）
- Serverが公開するTools / Resources / Promptsを **発見（discovery）** してHostに伝える
- LLMのリクエストをMCPプロトコルに変換し、Serverのレスポンスをホストに返す

前回の記事で `MultiServerMCPClient` や `subprocess.Popen` で接続していたのは、まさにこのClient層の操作に当たる。

### MCP Server（サーバー）

外部サービスやデータへのアクセスを **MCPプロトコルで公開するプログラム**。

- ファイルシステム、データベース、GitHub、Slack、Sentry など、特定の機能に特化する
- Tools / Resources / Prompts の3つのプリミティブを通じて機能を公開する
- ローカル実行（STDIO）もリモート実行（Streamable HTTP）も可能
- 1つのServerは1つの統合ポイントに集中するのがベストプラクティス

### 具体例: VS Code + Sentry + Filesystem

VS CodeがMCP Hostとして動作する場合を考える。

1. VS CodeがSentry MCP Serverに接続 → **Client 1** が生成され、Sentryとの接続を維持
2. VS Codeがローカルファイルシステム Serverに接続 → **Client 2** が生成される
3. VS Code内のLLMが「このエラーの詳細を見て」と判断 → Client 1経由でSentryのToolを呼ぶ
4. LLMが「関連ファイルを開いて」と判断 → Client 2経由でファイルシステムのToolを呼ぶ

**各Clientは独立したセッションを持つ**ため、Sentryの権限でファイルシステムにアクセスすることはできない。これがセキュリティの分離を実現している。

---

## 3つのプリミティブ: Tools / Resources / Prompts

MCP Serverは3種類のプリミティブを通じて機能を公開する。前回の記事ではToolsの呼び出しフローを詳しく見たが、実はToolsはプリミティブの1つに過ぎない。

これらは **「誰がいつ使うかを決めるか」** で区別される。

### Tools — モデル制御

LLMが **自律的に判断して呼び出す** 実行可能な関数。

- 副作用のあるアクション（レコード作成、メッセージ送信、デプロイ等）に適する
- JSON Schemaで入力パラメータを定義する
- 実行前にユーザーの同意を求めることが可能
- 例: `query_database`, `create_issue`, `send_slack_message`

```
ユーザー: 「Tokyoの天気を教えて」
   ↓
LLM: 「weather_apiツールを呼ぶべきだ」 ← モデルが自分で判断
   ↓
Tool実行 → 結果をLLMに返す → ユーザーに回答
```

前回の記事で解説した `tools/call` → ReActループは、まさにこのToolsプリミティブの動作フローだ。

### Resources — アプリケーション制御

読み取り専用のデータソース。**HostアプリケーションやClient側** がいつ取得するかを決める。

- ファイル内容、DBスキーマ、設定情報、ナレッジベースなどを公開
- LLMが直接要求するのではなく、ClientがResourceを取得してLLMのコンテキストに注入する
- URIで識別される（例: `file:///project/schema.sql`）

```
Client: 「DBスキーマのResourceを先に取得しておこう」
   ↓
Resource取得 → LLMのコンテキストに追加
   ↓
LLM: スキーマを理解した上でSQLクエリを生成
```

Toolsとの大きな違いは、**LLMが「呼ぶ」のではなく、アプリ側が「先に渡す」** こと。前回の記事の `get_tools()` が「準備フェーズ」だったのと同じ考え方で、Resourcesも事前にコンテキストとして仕込む。

### Prompts — ユーザー制御

再利用可能なインタラクションテンプレート。**ユーザーがUI上で明示的に選択** して起動する。

- コードレビュー手順、セキュリティ監査チェックリストなどのワークフローを定義
- 引数を受け取ってメッセージ列に展開される
- ドメイン知識を標準化されたパターンとして封じ込める
- 例: `code_review(code)`, `security_audit(target)`, `explain_concept(topic)`

```
ユーザー: UIから「コードレビュー」プロンプトを選択
   ↓
Client: Serverからプロンプトテンプレートを取得
   ↓
引数を埋めてLLMに送信 → 構造化されたレビュー結果
```

### プリミティブ比較表

| 特性 | Tools | Resources | Prompts |
|------|-------|-----------|---------|
| 制御主体 | モデル（LLM） | アプリケーション（Client/Host） | ユーザー |
| 目的 | アクション実行 | コンテキストデータ提供 | インタラクションの構造化 |
| 副作用 | あり（変更操作含む） | なし（読み取り専用） | なし（テンプレート展開のみ） |
| 発見方法 | `tools/list` | `resources/list` | `prompts/list` |
| 実行/取得 | `tools/call` | `resources/read` | `prompts/get` |

---

## 処理シーケンス — Anthropic Academy教材ベース

Anthropic Academyの「Claude with Amazon Bedrock」コースのMCPモジュールでは、MCPの処理フローを6つのアクターで表現している。以下はその教材のシーケンス図を元に整理したもの。

![MCPシーケンス図](/blog/assets/img/mcp-sequence-diagram.svg)

### 4フェーズの解説

このフローは **フェーズA（ツール発見）→ B（LLM推論）→ C（ツール実行）→ D（回答生成）** の4段階に分かれる。

#### フェーズA: ツール発見（Tool Discovery）

1. ユーザーが「What repositories do I have?」と質問
2. App Server（オーケストレータ）が MCP Client に `tools/list` を依頼 — Claudeにツール定義を渡すために必要なため
3. MCP Client が MCP Server に `ListToolsRequest` を送信
4. MCP Server が `ListToolsResult` で利用可能なツール（名前、説明、パラメータスキーマ）を返す

**ポイント**: ツール一覧を「要求する」のはMCP Clientであり、「なぜ要求するか」を判断するのはApp Server（LLMオーケストレータ）だ。ClaudeはツールリストをApp Server経由で受け取るまで何が使えるか知らない。前回の記事の `get_tools()` に相当するフェーズ。

#### フェーズB: LLM推論（LLM Reasoning）

5. App Server が Claude に **ユーザーの質問 + ツール定義** をまとめて送信（`Query + Tools`）
6. Claude が `ToolUse` を返す — 「`list_repositories` ツールを呼ぶべきだ」と自律的に判断

前回の記事の `ainvoke()` → ReActループの「ツール選択」に相当する。

#### フェーズC: ツール実行（Tool Execution）

7. App Server が MCP Client にツール実行を依頼
8. MCP Client が MCP Server に `CallToolRequest` を送信
9. MCP Server が実際のGitHub APIを呼び出す
10. GitHub がレスポンスを返す
11. MCP Server が `CallToolResult` で結果を返す
12. MCP Client が App Server に `toolResult` を返す

前回の記事の `tools/call` → JSON-RPCのやり取りがこのフェーズに当たる。

#### フェーズD: 回答生成（Response Generation）

13. App Server がツール結果を Claude に送信
14. Claude が自然言語で「Your repositories are...」と回答を生成
15. ユーザーに最終回答を表示

### MCPプロトコルメッセージ一覧

| メッセージ | 方向 | 役割 |
|-----------|------|------|
| `ListToolsRequest` | Client → Server | サーバーが公開する全ツールの一覧を要求 |
| `ListToolsResult` | Server → Client | ツール名・説明・引数スキーマの一覧を返却 |
| `Query + Tools` | App Server → Claude | ユーザーの質問 + ツール定義をLLMに提示 |
| `ToolUse` | Claude → App Server | LLMが選んだツール名と引数 |
| `CallToolRequest` | Client → Server | 指定ツールを指定引数で実行するリクエスト |
| `CallToolResult` | Server → Client | ツール実行結果 |
| `toolResult` | Client → App Server | 実行結果をアプリケーション側に返す |

このフローで重要なのは、**Claudeはツールの「選択と引数の決定」だけを行い、実際の実行はMCP Server側で行われる** という責務の分離。LLMは外部APIに直接アクセスせず、MCPプロトコルを介した安全な間接アクセスとなっている。

---

## 発展トピック

### Sampling — サーバーからLLMへの逆方向リクエスト

通常はClient→Serverの方向だが、**Serverが逆にHostのLLMを使いたい**ケースがある。Samplingプリミティブにより、Serverが `sampling/createMessage` でLLMの推論結果を受け取れる。MCPサーバー側にLLM SDKを組み込まなくてもモデルの能力を利用できる仕組み。

### Elicitation — ユーザーへの問いかけ

Serverが処理中に追加情報が必要になった場合、`elicitation/create` でユーザーに質問を投げることができる。たとえば「デプロイ先の環境を選んでください」のような確認ダイアログを出す用途。

### Notifications — リアルタイム通知

ServerのToolリストが変更された場合など、Server→Clientへリアルタイムに通知を送れる。JSON-RPC 2.0のNotification（レスポンス不要のメッセージ）として実装される。

### MCPとFunction Callingの違い

| 観点 | MCP | Function Calling (API直接) |
|------|-----|---------------------------|
| 標準化 | オープンプロトコル | ベンダー固有のAPI仕様 |
| ツール発見 | 動的（`tools/list`で実行時に発見） | 静的（API呼び出し時に定義を渡す） |
| ステート管理 | ステートフルセッション | ステートレス |
| 複数ツール接続 | Host内で複数Serverを並列管理 | 開発者が個別に管理 |
| 再利用性 | Serverを一度作れば、どのHostからも利用可能 | アプリごとに再実装が必要 |

### セキュリティモデル

- **セッション分離**: 各Client-Serverペアが独立したセッションを持ち、権限やデータが他のセッションに漏れない
- **Hostがゲートキーパー**: どのClientがどのToolを呼べるかはHostが制御する
- **ユーザー同意**: Tool実行前にユーザーの承認を求めるフローをサポート
- **トランスポートセキュリティ**: リモート通信はHTTPS必須、OAuth等の認証に対応

---

## まとめ

前回の記事と今回の記事で、MCPの全体像を **2つの視点** からカバーした。

| 視点 | 前回の記事 | 今回の記事 |
|------|-----------|-----------|
| 通信プロトコル | SSE / Stdio / JSON-RPC | — （前回参照） |
| データフロー | get_tools → ainvoke → ReAct | — （前回参照） |
| アーキテクチャ | — | Host / Client / Server 3層構造 |
| プリミティブ | — | Tools / Resources / Prompts |
| 処理シーケンス | JSON-RPCメッセージ視点 | 6アクター視点（Anthropic Academy準拠） |
| ライフサイクル | LangChain / Claude.ai のプロセス管理 | — （前回参照） |
| 発展トピック | — | Sampling / Elicitation / セキュリティ |

MCPを理解するための3つの軸:

1. **アーキテクチャ**: Host → Client → Server の3層構造。Hostが全体を統括し、Clientが1:1でServerと接続する
2. **プリミティブ**: Tools（モデルが呼ぶ）、Resources（アプリが取得する）、Prompts（ユーザーが選ぶ）の3種類
3. **プロトコル**: JSON-RPC 2.0ベースのステートフルセッション。STDIOとStreamable HTTPの2つのトランスポートを選択可能

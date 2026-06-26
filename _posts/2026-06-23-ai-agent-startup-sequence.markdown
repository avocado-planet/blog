---
layout: post
title:  "AI Agent CLIの起動時に何が起きているか：MCPハンドシェイクから全体シーケンスまで"
date:   2026-06-23
description: AI Agentが「Ready.」を表示するまでに何をしているのか。設定読み込み、認証、メモリ復元、MCP初期化、システムプロンプト構築まで、起動時の標準フローを整理する。
---

`claude` と打ってEnterを押してから `Ready.` が出るまでの数秒間に、AI Agentは何をしているのか。

設定読み込み、認証、メモリ復元、MCP接続、システムプロンプト構築 — 全体像を整理する。

---

## 起動時の全体シーケンス

```
プロセス起動
    ↓
① 設定読み込み
    ↓
② ロギング初期化
    ↓
③ LLM Provider認証
    ↓
④ メモリ層の復元
    ↓
⑤ MCP / ツール接続
    ↓
⑥ システムプロンプト構築
    ↓
⑦ ユーザー入力待ち（REPL）
```

「壊れやすいもの・遅いものを早めにエラーで止める」が設計の基本だ。認証エラーは起動直後に検出できるが、MCPサーバーの応答待ちで5秒固まるとユーザーは何が起きているか分からなくなる。

---

## ① 設定読み込み

優先順位を持って複数ソースから設定をマージする。

```
CLI引数            （最優先）
    ↓
環境変数
    ↓
プロジェクト設定（./agent.toml）
    ↓
ユーザー設定（~/.agentcli/config.toml）
    ↓
デフォルト値       （最後）
```

```python
config = (
    load_defaults()
    | load_user_config()
    | load_project_config()
    | load_env_vars()
    | parse_cli_args()
)
```

---

## ② ロギング初期化

設定で決まったレベル・出力先でロガーをセットアップする。**これを早めにやらないと、以降のステップでエラーが出てもデバッグできない。**

```python
logging.basicConfig(
    level=config.log_level,
    handlers=[
        FileHandler("~/.agentcli/logs/session.log"),
        StreamHandler(sys.stderr)
    ]
)
```

---

## ③ LLM Provider認証

API Keyの読み込みと検証。

```python
api_key = (
    os.getenv("ANTHROPIC_API_KEY")
    or keyring.get_password("anthropic", "api_key")
    or dotenv_values(".env").get("ANTHROPIC_API_KEY")
)
```

**ここでクラッシュさせるか後回しか**は設計判断。起動時に検証すると確実だが、軽量なping的呼び出し（models一覧取得など）が必要になるため起動が若干遅くなる。

---

## ④ メモリ層の復元

前回のメモリ設計記事で扱った話の実装フェーズだ。

```python
# External Memory
self.user_memory    = load_file("~/.agentcli/memory.md")
self.project_memory = load_file("./AGENT.md")  # CLAUDE.md相当

# Session履歴の復元（--resumeオプション等）
if args.resume:
    self.history = load_session(args.session_id)
else:
    self.history = []

# Vector DB接続（Episodic Memoryがあれば）
self.vector_db = ChromaDB(persist_dir="~/.agentcli/vectors")
```

---

## ⑤ MCP / ツール接続

ここが今回の主題。MCPプロトコルでは**起動時の動作が明確に定義されている**。

### MCPのハンドシェイク標準シーケンス

```
1. 接続確立（stdio起動 or HTTP接続）
    ↓
2. initialize（プロトコルネゴシエーション）
    ↓
3. initialized 通知
    ↓
4. capability に応じてリソース取得
   ├─ tools/list      （Tools対応なら）
   ├─ resources/list  （Resources対応なら）
   └─ prompts/list    （Prompts対応なら）
    ↓
5. （オプション）通知購読
```

### ステップ1：接続確立

通信方式に応じてプロセスを起動またはHTTP接続する。

- **stdio**：MCP Serverをサブプロセスとして起動、標準入出力で通信
- **SSE/HTTP**：リモートサーバーにHTTP接続

### ステップ2：initialize（必須・最重要）

クライアントとサーバーがお互いに**自己紹介＋能力交換**するフェーズ。

**Client → Server**:

```json
{
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "sampling": {},
      "roots": {}
    },
    "clientInfo": {
      "name": "agent-ai-cli",
      "version": "0.4.0"
    }
  }
}
```

**Server → Client**:

```json
{
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools":     {"listChanged": true},
      "resources": {"subscribe": true},
      "prompts":   {},
      "logging":   {}
    },
    "serverInfo": {
      "name": "github-mcp-server",
      "version": "1.2.0"
    }
  }
}
```

ここで決まること：

- **使うプロトコルバージョン**（互換性チェック）
- **サーバーが提供できる機能**（tools/resources/prompts/logging）
- **クライアントが提供できる機能**（sampling/roots）

### ステップ3：initialized 通知

Client → Server に「準備OK」を通知（レスポンス不要）：

```json
{ "method": "notifications/initialized" }
```

これで**プロトコルレベルのハンドシェイクは完了**。ここまでは何のツールも呼べない。

### ステップ4：capabilityに応じた取得

initializeで返ってきた `capabilities` を見て**選択的に**取得する：

```python
if server_caps.get("tools"):
    tools = await client.list_tools()
if server_caps.get("resources"):
    resources = await client.list_resources()
if server_caps.get("prompts"):
    prompts = await client.list_prompts()
```

ここで取得したツール定義は**セッション中ずっと使い回す**。LLM呼び出しのたびに `tools/list` を叩くわけではない。

### ステップ5：通知購読（オプション）

`listChanged: true` のサーバーは「ツール一覧が変わった」を通知できる：

```
Server → Client: notifications/tools/list_changed
    ↓
Client → Server: tools/list を再実行
```

これにより**動的にツールが追加・削除されるケース**にも対応できる。

### なぜハンドシェイクが必要か

「いきなり `tools/list` を叩けばいいのでは？」と思うかもしれない。理由は3つ：

1. **バージョン互換性**：プロトコルが進化するため、双方が話せるバージョンを決める必要がある
2. **能力の事前共有**：Server側に `sampling` を使われたくない場合、Client capabilityで宣言する必要がある
3. **状態管理**：MCPはセッション型プロトコルなので「ここから本番」という明示が必要

### 複数サーバーは並列接続

複数のMCP Serverに繋ぐ場合、**並列**にしないと起動が遅くなる：

```python
mcp_servers = config.mcp_servers
self.mcp_clients = await asyncio.gather(*[
    connect_and_initialize(server) for server in mcp_servers
])
self.all_tools = flatten([c.tools for c in self.mcp_clients])
```

Serverが5個あったら、直列だと5倍遅くなる。

---

## ⑥ システムプロンプト構築

ここまでで集めた情報を**1つのSystem Promptに組み立てる**。これがLLM呼び出しのたびに使われる土台になる。

```python
system_prompt = f"""
You are an AI assistant.

# Current Context
- Date: {datetime.now()}
- Working directory: {os.getcwd()}
- OS: {platform.system()}

# User Memory
{self.user_memory}

# Project Memory
{self.project_memory}

# Available Tools
{format_tools_for_llm(self.all_tools)}
"""
```

---

## ⑦ ユーザー入力待ち

REPLループに入る：

```python
while True:
    user_input = await prompt_user()
    if user_input == "/exit":
        break
    await self.chat(user_input)
```

---

## オプショナルな起動時タスク

実装によってはこれらも追加される。

| タスク | 目的 |
|-------|------|
| ヘルスチェック | LLM API疎通、MCP Server応答確認 |
| アップデート確認 | 新バージョン通知 |
| Telemetry初期化 | 匿名利用統計（opt-in） |
| Workspace検出 | git rootの判定、`.gitignore`読み込み |
| 権限プロンプト | macOSのファイルアクセス権限等 |
| キャッシュ初期化 | LLMレスポンスキャッシュ、Embedding cache |

---

## 起動時ログの例（Claude Code風）

```
[09:00:00] Loading config from ./CLAUDE.md and ~/.claude/config.json
[09:00:00] Logger initialized (level=INFO)
[09:00:01] Authenticated with Anthropic API (model=claude-opus-4-7)
[09:00:01] Loaded memory: user (1.2KB), project (3.4KB)
[09:00:01] Connecting to 3 MCP servers...
[09:00:02]   ✓ github-mcp (12 tools)
[09:00:02]   ✓ filesystem-mcp (8 tools)
[09:00:02]   ✗ slack-mcp (timeout)
[09:00:02] System prompt built (4520 tokens)
[09:00:02] Ready.
>
```

進捗表示が重要だ。サイレントだと「固まったのか接続中なのか」が分からない。

---

## agent-ai-cli への落とし込み

現在のフェーズ（REPL + 永続化 + LangGraph ReAct完了）を踏まえた起動シーケンス案：

```python
class AgentCLI:
    async def startup(self):
        # ① 設定
        self.config = load_config(args)

        # ② ログ
        setup_logging(self.config.log_level)

        # ③ 認証
        validate_api_keys(self.config)

        # ④ メモリ（Phase 4実装予定）
        self.memory = MemoryStore.load(self.config.memory_dir)

        # ⑤ MCP（将来実装）
        # self.mcp_clients = await asyncio.gather(*[
        #     connect_and_initialize(s) for s in self.config.mcp_servers
        # ])

        # ⑥ Graph構築（既存）
        self.graph = build_react_graph(
            llm=self.llm,
            tools=self.tools,
            system_prompt=self.build_system_prompt()
        )

        # ⑦ REPL（既存）
        await self.repl_loop()
```

---

## まとめ

| ステップ | 一般的な内容 | エラー時の影響 |
|---------|-------------|-------------|
| 1. 設定 | CLI/env/file の優先順位マージ | 起動不可 |
| 2. ログ | 早期セットアップ | 以降のデバッグ不可 |
| 3. 認証 | LLM API Keyの確認 | LLM呼び出し不可 |
| 4. メモリ復元 | External/Session Memory読み込み | コンテキスト欠落 |
| 5. ツール接続 | MCP initialize → tools/list | ツール呼び出し不可 |
| 6. プロンプト構築 | 全コンテキストの統合 | 不完全な振る舞い |
| 7. 入力受付 | REPLループ開始 | — |

ポイントは**「壊れやすいもの・遅いもの」を早めにエラーで止める**ことと、**MCP接続のような重い処理は並列化＋進捗表示**で体感を改善することだ。

「Ready.」の一行は、これだけの準備の上に成り立っている。

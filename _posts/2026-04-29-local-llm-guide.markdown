---
layout: post
title:  "ローカルでLLMを動かす方法まとめ：apfel、Ollama、LangChain連携"
date:   2026-04-29
description: macOSのapfelやOllamaを使って、無料でローカルLLMを動かす方法とLangChainでの活用法を解説
---

クラウドAPIを使わずに、自分のマシン上でLLMを動かす方法をまとめました。プライバシー重視、コスト削減、オフライン利用など、ローカルLLMには多くのメリットがあります。

## ローカルLLMの選択肢

すべて**無料・オープンソース**で利用可能です。

| 方案 | 特徴 | 向いている用途 |
|------|------|---------------|
| **apfel** | macOS標準搭載モデルを利用、設定不要 | プライバシー重視、軽量タスク |
| **Ollama** | 一行コマンドでモデル管理、最も簡単 | 日常使い、開発 |
| **LM Studio** | GUIで操作、視覚的にモデル選択 | 初心者、試行錯誤 |
| **llama.cpp** | 最軽量、C++直接実行 | 組み込み、最小依存 |
| **vLLM** | 高スループット、本番向け | 高並発環境 |

## apfel：macOS標準のAIを解放する

### apfelとは

macOS Tahoe（26）には、AppleのFoundation Model（約3Bパラメータ）が標準搭載されています。**apfel**は、このモデルをターミナルから使えるようにするオープンソースツールです。

- **ダウンロード不要** — モデルはすでにmacOSに入っている
- **API キー不要** — 完全オフライン動作
- **設定不要** — `brew install` して即使える

### インストールと基本的な使い方

```bash
brew install apfel

# 直接質問
apfel "東京の人口は？"

# JSON出力
apfel -o json "Appleを日本語に翻訳" | jq .content
```

### 3つのモード

**1. CLIツール** — パイプ処理対応のUNIXツール

```bash
cat document.txt | apfel "要約して"
```

**2. OpenAI互換サーバー** — 既存のSDKがそのまま使える

```bash
apfel --serve
# http://127.0.0.1:11434 で起動
```

**3. インタラクティブチャット** — マルチターン会話

```bash
apfel --chat -s "あなたはコーディングアシスタントです"
```

### モデルスペック

| 項目 | 値 |
|------|-----|
| パラメータ | 約 3B |
| コンテキスト | 4,096 トークン |
| 量子化 | Mixed 2/4-bit |
| 対応言語 | 英、独、西、仏、伊、日、韓、葡、中 |

### プライバシー

- 完全オンデバイス実行、ネットワーク通信なし
- Apple もプロンプト/レスポンスを収集しない
- テレメトリなし（`--update` 以外はネットワーク接続ゼロ）

### 制限事項

- コンテキストが4,096トークンのみ（長い会話には不向き）
- 3Bモデルなので複雑な推論は苦手
- 得意：テキスト変換、分類、短い要約、翻訳
- 苦手：数学、複雑なコード生成

## Ollama：本地LLMのDocker

### Ollamaとは

**Ollama**は、ローカルでLLMを動かすための管理ツールです。「本地LLMのDocker」と呼ばれるように、モデルのダウンロード・管理・実行をワンストップで提供します。

### インストールと使い方

```bash
# macOS
brew install ollama

# モデルを取得して実行
ollama pull llama3.1
ollama run llama3.1
# そのまま対話開始
```

### 主な機能

| 機能 | 説明 |
|------|------|
| モデルライブラリ | Llama、Qwen、Gemma、Mistralなど主要モデルを `ollama pull` で取得 |
| 自動量子化 | ハードウェアに応じた最適な量子化を自動選択 |
| OpenAI互換API | `localhost:11434` で OpenAI SDK 互換サービスを提供 |
| Modelfile | Dockerfileのように、カスタムモデル設定を記述可能 |

### なぜ人気か

1. **ゼロ設定** — GGUFの手動ダウンロードやGPUパラメータ設定が不要
2. **クロスプラットフォーム** — macOS / Linux / Windows 対応
3. **豊富なエコシステム** — LangChain、Open WebUI、Continueなどが公式対応

## Googleの最新オープンソースLLM：Gemma 4

2026年4月にGoogleが公開した**Gemma 4**は、Apache 2.0ライセンスのオープンモデルです。

### モデルサイズ

| サイズ | 用途 |
|--------|------|
| E2B, E4B | エッジ/モバイル（Raspberry Pi、Jetson Nano対応） |
| 26B MoE | 汎用（単一80GB GPUで動作） |
| 31B Dense | 高性能（Arenaランキング3位相当） |

### 特徴

- Gemini 3技術ベース
- Agentic AIワークフロー向けに最適化
- LangChainで Ollama 経由で利用可能

```bash
ollama pull gemma4:26b
```

## 主要オープンソースLLM比較

| モデル | 提供元 | 強み |
|--------|--------|------|
| **Gemma 4** | Google | Apache 2.0、agentic最適化、全サイズ展開 |
| **Qwen 3.5** | Alibaba | 超長コンテキスト、マルチモーダル |
| **DeepSeek-V3.2** | DeepSeek | GPT-5級の推論能力 |
| **Llama 3.1** | Meta | 成熟したエコシステム |

## ローカルLLMをAPIとして公開する

ローカルで動かしているLLMを、OpenAI互換のAPIサーバーとして公開できます。これにより、既存のOpenAI SDKやLangChainからそのまま利用可能になります。

### apfelの場合

```bash
# サーバー起動
apfel --serve
# http://127.0.0.1:11434 で OpenAI互換APIが起動

# 認証トークン付きで起動（セキュリティ強化）
apfel --serve --token "your-secret-token"
```

対応エンドポイント：
- `POST /v1/chat/completions` — チャット補完
- `GET /v1/models` — モデル一覧
- ストリーミング（SSE）、Tool Calling、`response_format: json_object` 対応

### Ollamaの場合

```bash
# Ollamaは起動時に自動でAPIサーバーが立ち上がる
ollama serve
# http://localhost:11434 で起動
```

Ollamaも OpenAI互換エンドポイントを提供：

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### vLLMの場合（高スループット向け）

```bash
# vLLMサーバー起動
vllm serve meta-llama/Llama-3.1-8B-Instruct

# http://localhost:8000/v1 で OpenAI互換API
```

### LM Studioの場合

1. LM Studioを起動
2. モデルをロード
3. 左メニューから「Local Server」を選択
4. 「Start Server」をクリック
5. `http://localhost:1234/v1` でAPI利用可能

### クライアントからの接続例

どのサーバーも OpenAI SDK でそのまま使えます：

```python
from openai import OpenAI

# apfel / Ollama
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="unused",  # ローカルサーバーは認証不要の場合が多い
)

# vLLM
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="unused",
)

# LM Studio
client = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio",
)

response = client.chat.completions.create(
    model="llama3.1",  # または apple-foundationmodel など
    messages=[{"role": "user", "content": "こんにちは"}],
)
print(response.choices[0].message.content)
```

### 外部公開する場合の注意

ローカルネットワーク外に公開する場合は、セキュリティ対策が必要です：

| 対策 | 方法 |
|------|------|
| 認証 | `--token` オプションや reverse proxy での Basic認証 |
| HTTPS | nginx や Caddy で SSL終端 |
| ファイアウォール | 特定IPのみ許可 |
| レート制限 | nginx の `limit_req` など |

```bash
# 例：apfelで認証トークンを設定
apfel --serve --token "my-secret-api-key"

# クライアント側
client = OpenAI(
    base_url="http://your-server:11434/v1",
    api_key="my-secret-api-key",
)
```

## LangChainでの活用

### apfelをLangChainで使う

```python
from langchain_openai import ChatOpenAI

# apfelサーバーを起動後
# $ apfel --serve

llm = ChatOpenAI(
    base_url="http://localhost:11434/v1",
    api_key="unused",  # apfelは認証不要だが引数は必須
    model="apple-foundationmodel",
)

response = llm.invoke("こんにちは")
print(response.content)
```

### OllamaをLangChainで使う

```python
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.1")
response = llm.invoke("Hello")
```

### Tool Calling

apfel、Ollamaともに Tool Calling に対応しています。

```python
from langchain_core.tools import tool

@tool
def multiply(a: int, b: int) -> int:
    """2つの数を掛け算する"""
    return a * b

llm_with_tools = llm.bind_tools([multiply])
response = llm_with_tools.invoke("15 × 27 は？")
```

### LangGraphでのAgent構築

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(llm, tools=[multiply])
result = agent.invoke({"messages": [("user", "15と27を掛けて")]})
```

## コストについて

**すべて無料です。**

| 項目 | コスト |
|------|--------|
| ソフトウェア | すべてオープンソース、無料 |
| モデル | オープンウェイト、無料 |
| API料金 | なし（ローカル実行） |

唯一のコストは：

- **ストレージ** — モデルファイル（数GB〜数十GB）
- **ハードウェア** — メモリ/VRAMが多いほど大きなモデルが動く

Apple Silicon Mac であれば、7B〜8Bモデルは快適に動作します。

## 用途別おすすめ

| シナリオ | おすすめ |
|----------|----------|
| 手軽に始めたい | **Ollama** |
| GUIで試したい | **LM Studio** |
| macOSネイティブ・プライバシー重視 | **apfel** |
| 本番環境・高並発 | **vLLM** |
| 最小依存・組み込み | **llama.cpp** |

Agent開発には **Ollama + Llama 3.1 8B** がバランス良くおすすめです。

## まとめ

ローカルLLMは、クラウドAPIに比べて以下のメリットがあります：

- **プライバシー** — データが外部に出ない
- **コスト** — API料金ゼロ
- **オフライン** — ネット接続不要で動作
- **カスタマイズ** — ファインチューニング可能

apfelやOllamaを使えば、数分でローカルLLM環境が整います。LangChainとの連携も簡単なので、ぜひ試してみてください。

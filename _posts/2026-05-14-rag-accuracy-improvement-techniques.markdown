---
layout: post
title:  "RAGの精度を上げる実践テクニック完全ガイド — Re-Rankから検索・チャンク・クエリ改善まで"
date:   2026-05-14
description: RAG（Retrieval-Augmented Generation）の回答精度を向上させるための主要テクニックを体系的に解説。Re-Rank、Hybrid Search、HyDE、Semantic Chunking、Self-RAGなど、実践で使える手法を優先順位付きで紹介します。
---

RAG（Retrieval-Augmented Generation）を構築してみたものの、回答の精度がいまいち安定しない——そんな経験はないだろうか。

実は RAG の精度改善には多くのアプローチがあり、大きく **「検索の質」「チャンクの質」「LLMへの渡し方」「クエリ自体」「Embedding」** の5つの方向に分類できる。本記事では、まず多くの人がつまずく「なぜ検索結果がズレるのか」という問題から出発し、各テクニックを実装コード付きで解説する。

---

## そもそもなぜ精度が出ないのか

RAG の検索フェーズ（Retriever）は、ベクトル類似度やキーワードマッチで候補を「広く」取ってくる。この初期検索は**速度優先**で設計されているため、返ってくる上位チャンクが必ずしも質問に対して最も関連性が高いとは限らない。

例えば「LangChainのメモリ機能」と聞いたとき、Retriever の top-10 に「LangChainのインストール手順」や「メモリ管理（OS）」が混ざることがある。

この「検索のズレ」を様々な角度から補正するのが、以下のテクニック群だ。

---

## 1. Re-Rank（リランク）— 検索結果の並べ替え

### 考え方

Retriever が返した候補（例: top-20〜50件）を、より高精度なモデルで**並べ替えて**、本当に関連性の高い上位 k 件だけを LLM に渡す。

```
Query
  ↓
Retriever（高速・粗い） → 候補 20件
  ↓
Re-Ranker（低速・高精度） → 上位 5件に絞る
  ↓
LLM（回答生成）
```

### なぜ効くのか

Retriever（Bi-Encoder）はクエリとドキュメントを**別々に**ベクトル化して比較する。一方 Re-Ranker（Cross-Encoder）は**クエリとドキュメントのペアを同時に**入力して関連度スコアを出す。ペアごとに処理するので遅いが、文脈の噛み合いを深く評価できる。

|  | Retriever (Bi-Encoder) | Re-Ranker (Cross-Encoder) |
|---|---|---|
| 入力 | query と doc を別々にエンコード | (query, doc) ペアを同時にエンコード |
| 速度 | 高速（数百万件OK） | 低速（数十〜数百件が限界） |
| 精度 | そこそこ | 高い |

### 代表的なサービス／モデル

- **Cohere Rerank** — API一発で使える手軽さが魅力
- **bge-reranker**, **ms-marco-MiniLM** — オープンソースの Cross-Encoder モデル
- **ColBERT** — 遅延インタラクション（Late Interaction）で速度と精度のバランスを取る

### LangChain での実装

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

# 1. ベースの Retriever（広めに取る）
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# 2. Re-Ranker でラップ
reranker = CohereRerank(model="rerank-v3.5", top_n=5)
retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever,
)

# 3. 使う
docs = retriever.invoke("LangChainのメモリ機能について")
```

ポイントは `k=20` で広めに取って、`top_n=5` で絞るという**二段構え**。

---

## 2. 検索（Retrieval）の改善

### Multi-Query Retriever

ユーザーの質問を LLM で複数の言い換えに展開し、それぞれで検索して結果を統合する。

「メモリ機能」→「会話履歴の保持方法」「チャット状態の管理」など、**表現のミスマッチ**を補える。

### HyDE（Hypothetical Document Embeddings）

クエリから LLM に「仮の回答文」を生成させ、その仮回答をベクトル検索に使う。質問文よりも回答文のほうがドキュメントと語彙・文体が近いため、検索精度が上がる。

```
"LangChainのメモリ機能とは？"
  ↓ LLMが仮回答を生成
"LangChainのメモリ機能は、ConversationBufferMemoryや
 ConversationSummaryMemoryなどを通じて会話履歴を保持し..."
  ↓ この仮回答でベクトル検索
```

欠点は LLM 呼び出しが1回増えるのでレイテンシとコストが上がること。

### Hybrid Search + Reciprocal Rank Fusion (RRF)

BM25（キーワード）とベクトル検索の結果を RRF で統合する。各検索結果の順位（rank）の逆数を足し合わせてスコア化するシンプルな方式だが、異なる検索戦略の長所を活かせる。

```python
# RRF の考え方（k=60が一般的）
score(doc) = Σ 1 / (k + rank_i)
```

Re-Rank の前段として使うと特に効果的。

### Self-Query Retriever

「2024年以降のPythonに関する記事」のようなクエリから、LLM がメタデータフィルタ（`year >= 2024`, `topic == "Python"`）を自動生成し、ベクトル検索と組み合わせる。メタデータが整備されたドキュメントで特に有効。

---

## 3. チャンク戦略の改善

### Parent Document Retriever

小さいチャンク（例: 200トークン）で検索の精度を上げつつ、LLM には親チャンク（例: 2000トークン）を渡す。**検索精度と文脈の豊富さを両立する**。

### Semantic Chunking

固定長ではなく、文の意味的な境界で分割する。隣接する文の Embedding 類似度を計算し、類似度が大きく下がる箇所を分割点にする。

```
文1 ── 類似度0.92 ── 文2 ── 類似度0.88 ── 文3 ── 類似度0.31 ── 文4
                                              ↑ ここで分割
```

LangChain の `SemanticChunker` で実装可能。

### Document Summary Index

各ドキュメント全体の要約を事前に作成し、要約に対して検索 → 該当ドキュメントの詳細チャンクを取得、という二段階検索。長い文書が多い場合に効果的。

---

## 4. LLM への渡し方の改善

### Lost in the Middle 対策

LLM は入力の**先頭と末尾に注意が偏り、中間部分を見落としがち**という研究がある。対策として、関連度の高いチャンクをプロンプトの先頭と末尾に配置し、低いものを中間に置く。または単純にチャンク数を絞る（Re-Rank が効く理由でもある）。

### Contextual Compression

取得したチャンクから、クエリに関連する部分だけを LLM で抽出・圧縮してから回答生成に渡す。ノイズが減り、コンテキストウィンドウも節約できる。

```python
from langchain.retrievers.document_compressors import LLMChainExtractor

compressor = LLMChainExtractor.from_llm(llm)
retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever,
)
```

### Self-RAG

取得したドキュメントを LLM 自身が評価し、関連性が低いと判断したら無視する、または追加検索を行うという自律的なアプローチ。「検索するかどうか」「取得結果が有用か」「回答が取得結果に裏付けられているか」を段階的に判定する。

---

## 5. クエリ自体の改善

### Step-Back Prompting

具体的すぎる質問を一段抽象化してから検索する。

「Python 3.12 の match 文でガード句はどう書く？」→「Python の構造的パターンマッチングの構文」で検索し、より広い文脈を取得してから回答。

### Query Decomposition

複合的な質問を複数のサブクエリに分解し、それぞれ検索して統合する。

「LangChainとLlamaIndexのRAG実装の違いは？」→ サブ1「LangChainのRAG実装方法」+ サブ2「LlamaIndexのRAG実装方法」→ 統合して比較回答。

---

## 6. Embedding の改善

### Fine-tuned Embeddings

ドメイン固有のデータで Embedding モデルを fine-tune する。専門用語が多い領域（例: ITSM、医療、法律）では、汎用 Embedding モデルだと語彙のミスマッチが起きやすいため効果が大きい。

### ColBERT（Late Interaction）

トークンレベルで query と document の類似度を計算する。Bi-Encoder の速度と Cross-Encoder の精度の中間に位置する。RAGatouille ライブラリで手軽に試せる。

---

## 実践的な優先順位

すべてを一度に導入するのは非現実的なので、**コスト対効果の高い順**に並べるとこうなる：

| 優先度 | 手法 | 導入コスト | 効果 |
|---|---|---|---|
| ★★★ | Hybrid Search (BM25 + Vector) | 低 | 高 |
| ★★★ | Re-Rank | 低 | 高 |
| ★★☆ | Parent Document Retriever | 低 | 中〜高 |
| ★★☆ | Multi-Query Retriever | 低 | 中 |
| ★★☆ | Semantic Chunking | 中 | 中〜高 |
| ★☆☆ | HyDE | 中 | 中 |
| ★☆☆ | Contextual Compression / Self-RAG | 高 | 高 |
| ★☆☆ | Fine-tuned Embeddings | 高 | 高（ドメイン特化時） |

まずは **Hybrid Search → Re-Rank → チャンク戦略の見直し**という順で進めるのが、最も効率的な改善パスだろう。

---

## まとめ

RAG の精度改善は「どれか一つの銀の弾丸」ではなく、**複数のテクニックを組み合わせるパイプライン設計**が鍵になる。特に Hybrid Search + Re-Rank の組み合わせは、実装が簡単なわりに効果が大きく、最初に試す価値がある。

その上で、チャンク戦略（Semantic Chunking、Parent Document）やクエリ改善（Multi-Query、HyDE）を重ねていくことで、段階的に精度を引き上げていける。

重要なのは、闇雲にテクニックを追加するのではなく、**自分のデータと質問パターンに対してどこがボトルネックになっているかを見極めること**。検索結果のノイズが問題なら Re-Rank、語彙のミスマッチなら Multi-Query や HyDE、チャンクの粒度が問題なら Semantic Chunking——というように、課題に合った手法を選ぶのが最短ルートだ。

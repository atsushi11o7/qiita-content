---
title: RAG の基礎技術を実装して整理する - ハイブリッド検索から Agentic RAG まで
tags:
  - RAG
  - LLM
  - Python
  - 自然言語処理
  - 生成AI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

<!-- メモ:
- 位置づけ：RAG の発展的内容を学ぶ前に、これまで rag-sandbox で使ってきた基礎技術を一通り整理する
- 各手法は「概観 + 最小コード例（Python）」。深掘りや未使用の発展（Contextual Retrieval、ナレッジベース/GraphRAG、セマンティックキャッシュ等）は別記事
- 対象読者：LLM / 生成AI は知っているが RAG の各手法を整理したい人
- 環境：Python（rag-sandbox の構成。FAISS / sentence-transformers / Ollama など）
-->

## RAG の基本パイプライン（Indexing / Retrieval / Generation）

<!-- メモ:
- RAG とは / なぜ必要か（LLM の知識の限界・hallucination・最新情報の反映）
- 3 段：Indexing（文書を分割→埋め込み→ベクトルストア）/ Retrieval（クエリで類似検索）/ Generation（取得文脈をプロンプトに入れて生成）
- 以降の各節がこのどこを担うかを「地図」として示す（簡単な図か箇条書き）
-->

## チャンク分割

<!-- メモ:
- なぜ分割するか（埋め込み・コンテキスト長の制約、検索粒度）
- 戦略：固定長（chunk size / overlap）、再帰分割、構造ベース（Markdown 見出し等）、（軽く）セマンティック
- トレードオフ：小さすぎ=文脈不足 / 大きすぎ=ノイズ
- 最小コード例：テキストを再帰分割する
-->

## 埋め込みとベクトル検索（dense・FAISS）

<!-- メモ:
- 埋め込み = テキストを意味ベクトル化（dense retrieval の基礎）
- 使用モデル：ruri-v3 / bge-m3（sentence-transformers / FlagEmbedding）。日本語対応
- ベクトル検索：cosine 類似度、FAISS で近似最近傍（ANN）、top-k
- 最小コード例：sentence-transformers で埋め込み → FAISS で検索
-->

## キーワード検索（BM25 + 形態素解析）

<!-- メモ:
- dense の弱点（固有名詞・専門用語・表記ゆれ）→ 語彙一致で補完
- BM25 の概念（TF-IDF の発展、語の一致スコア）
- 日本語は分かち書きが必要 → 形態素解析（fugashi + unidic-lite）でトークン化 → rank-bm25
- 最小コード例：fugashi で分かち書き → rank-bm25 で検索
-->

## ハイブリッド検索（RRF）

<!-- メモ:
- dense（意味）+ sparse（語彙）を組み合わせて再現率を上げる
- スコア統合：RRF（Reciprocal Rank Fusion）= 各検索の順位の逆数和。スケールの違う 2 手法を順位ベースで統合
- 最小コード例：dense と BM25 の結果を RRF で統合
-->

## リランキング（日本語 Reranker）

<!-- メモ:
- 一次検索（速いが粗い）で多めに取得 → reranker で並べ替え（精度重視）
- cross-encoder：クエリと文書を同時入力してスコアリング（bi-encoder より高精度・低速）
- 日本語 Reranker（bge-reranker 系 / FlagEmbedding）
- 最小コード例：上位 N 件を reranker でスコアリングして並べ替え
-->

## メタデータフィルタリング

<!-- メモ:
- チャンクに付けたメタデータ（出典・日付・カテゴリ等）で検索対象を絞る
- 用途：最新だけ / 特定ソースだけ / 権限制御
- FAISS 単体はフィルタが弱い → id とメタデータを別管理、または事前/事後フィルタ
- 最小コード例：メタデータで候補を絞る or 検索後にフィルタ
-->

## クエリ変換（HyDE・クエリ書き換え・分解）

<!-- メモ:
- 素のクエリは検索に最適とは限らない → クエリ側を変換して再現率を上げる
- HyDE：LLM に「仮の回答」を生成させ、その回答文を埋め込んで検索（回答同士の方が近い）
- クエリ書き換え / マルチクエリ：言い換えを複数生成
- クエリ分解：複雑な質問をサブクエリに分ける
- 最小コード例：Ollama で仮回答生成 → 埋め込み検索（HyDE）
-->

## 親子チャンク（Parent-Child）

<!-- メモ:
- 検索は小さいチャンク（精度）、文脈として返すのは親の大きいチャンク（情報量）= small-to-big
- 実装：子チャンクを検索 → 対応する親チャンク/文書を取得してプロンプトへ
- 最小コード例：子→親のマッピングを持ち、検索後に親を展開
-->

## 検索結果の調整（MMR・コンテキスト並び順）

<!-- メモ:
- MMR（Maximal Marginal Relevance）：関連度と多様性のバランス。似た文書ばかりになるのを防ぐ
- コンテキスト並び順：lost-in-the-middle（長文脈の中間は無視されやすい）→ 重要文書を先頭/末尾へ
- 最小コード例：MMR で多様性を持たせて選ぶ / 取得文書の並べ替え
-->

## 能動的な検索（Agentic RAG・Self-RAG）

<!-- メモ:
- naive RAG（1 回検索で固定）の限界 → LLM が検索を能動制御
- Agentic RAG：検索要否の判断、クエリ再構成、再検索ループ、複数ソース。langgraph で状態遷移
- Self-RAG：reflection（検索すべきか / 文書は関連するか / 生成は根拠があるか）を自己評価
- 最小コード例：langgraph で「検索 → 評価 →（不十分なら）再検索」ループの骨格
-->

## 評価の基礎

<!-- メモ:
- なぜ評価が要るか（手法を入れて良くなったか測れないと意味がない）
- 検索の評価：Recall@k / Precision@k / MRR / nDCG
- 生成の評価：faithfulness（根拠との整合）、answer relevance。RAGAS 等のフレームワーク（軽く）
- 指標の説明中心（必要なら最小コード例）
-->

## おわりに

<!-- メモ:
- 一通りの基礎技術を整理した。これらを土台に発展（Contextual Retrieval、ナレッジベース/GraphRAG、Self-RAG の深掘り、セマンティックキャッシュ等）は別記事で扱う予定
- rag-sandbox で実装して試した旨を一言（環境記事へのリンクも可）
-->

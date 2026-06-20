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
- 位置づけ：RAG の発展系を試す前に、基礎手法を rag-sandbox で実装して手元で動かし、各手法が「パイプラインのどこを担うか」を整理したまとめ
- 各手法は「概観 + 実装コード」で紹介
- 深掘り/発展（Contextual Retrieval、GraphRAG、Self-RAG の精緻な実装、セマンティックキャッシュ等）は別記事
- 対象読者：LLM / 生成AI は知っているが RAG の各手法を整理したい人
-->

## RAG とは

<!-- メモ:
- RAG（Retrieval-Augmented Generation）= 外部の知識を検索し、その結果を文脈として LLM の生成に与える手法
- なぜ必要か：LLM 単体は学習時点の知識しか持たず、最新情報・社内/個人の文書を知らない／もっともらしい誤り（hallucination）を出す
- 検索した根拠を与えることで、正確性・最新性を上げ、出典も示せる
- 大枠は「検索（Retrieval）＋ 生成（Generation）」。具体的な 3 段構成は次節以降の「基本パイプライン」で
-->

## 実験環境（devcontainer + uv / Ollama）

<!-- メモ:
- CUDA 12.6 + Python 3.12 の devcontainer で統一。パッケージ管理は uv（uv add / uv sync、uv.lock で再現性）。torch は CUDA wheel を別 index から取得（[tool.uv.sources]）
- 生成・HyDE・CRAG の LLM は Ollama（qwen2.5:7b）。devcontainer の外で起動し docker-compose でサービス化、コンテナ内から OLLAMA_HOST 経由でアクセス
- コード例：docker-compose.yml 抜粋 / ChatOllama(model, base_url=host)
-->

## LangChain エコシステムの整理

<!-- メモ:
- 2025 年時点で LangChain はパッケージ分割（langchain-core / -community / -ollama / -text-splitters / -classic / langgraph）。0.0.x 時代の import パスと違う点に注意（例：ParentDocumentRetriever は langchain-classic に残る）
- 表1：パッケージ → 役割。表2：ライブラリ → 役割（sentence-transformers / FlagEmbedding / faiss-cpu / fugashi+unidic-lite / rank-bm25 / langchain-ollama）
-->

## コーパスの準備（Qiita 公開 API）

<!-- メモ:
- 題材＝自分の Qiita 記事。公開 API（/users/{user}/items）で全件取得し data/corpus/<id>.md ＋ <id>.json（title/created_at/tags/likes_count/url のサイドカー）
- コード：scripts/fetch_qiita.py 抜粋 / src/rag/corpus.py の load_md_corpus（md 本文 + json メタを Document に）
- ポイント：メタを Document.metadata に載せることで後段のタグフィルタが効く
-->

## RAG の基本パイプライン

<!-- メモ:
- 3 段：Indexing（分割→埋め込み→FAISS）/ Retrieval（クエリ→埋め込み→類似検索）/ Generation（クエリ+文脈→プロンプト→LLM）
- テキスト図で示し、「以降の各節がこのどこを担うか」の地図にする
-->

## チャンク分割

<!-- メモ:
- なぜ分割するか（コンテキスト長・検索粒度）、小さすぎ=文脈不足 / 大きすぎ=ノイズ
- 実装：RecursiveCharacterTextSplitter。日本語向け separators ["\n\n","\n","。","、"," ",""] を明示（英語デフォルトだと日本語が切れない）。chunk_id="doc_id#i" を付与（評価で追跡）
- コード：src/rag/chunking.py split_documents
- 正直に：戦略は再帰分割 1 種（固定長/意味分割の比較はしていない）
-->

## 埋め込みとベクトル検索（Dense・FAISS）

<!-- メモ:
- Dense = 意味ベクトル化して類似度ランキング。モデル cl-nagoya/ruri-v3-310m（日本語特化、クエリ/文書でプレフィックス切替）
- HuggingFaceEmbeddings はプレフィックス切替非対応 → Embeddings 継承の薄いラッパー PrefixedEmbeddings。normalize_embeddings=True で FAISS の L2 ≒ cosine
- FAISS：build_faiss / load_faiss（allow_dangerous_deserialization）。build_dense_retriever
- コード：embeddings.py（embed_query/embed_documents のプレフィックス）/ store.py / dense.py
-->

## キーワード検索（BM25 + 形態素解析）

<!-- メモ:
- Dense の弱点（固有名詞・専門用語・略語＝uv / FAISS / ruri-v3 など）を語彙一致で補完
- BM25（TF-IDF の発展）。日本語は分かち書きが必要 → fugashi（MeCab）+ unidic-lite でトークン化 → rank-bm25 の BM25Okapi
- 実装：JapaneseBM25Retriever。重いオブジェクト（Tagger / BM25Okapi）は PrivateAttr + model_validator(mode="after") で初期化
- コード：bm25.py の _tokenize / _get_relevant_documents
-->

## ハイブリッド検索（RRF）

<!-- メモ:
- Dense（意味）+ BM25（語彙）を統合。RRF（Reciprocal Rank Fusion）= Σ 1/(k+rank)、k=60（標準値）。スケールの違う 2 手法を順位ベースで統合
- 実装：HybridRetriever（candidate_k=50 で多めに取得 → RRF → top_k）。chunk_id/doc_id で名寄せ
- コード：hybrid.py _get_relevant_documents（RRF 本体）
-->

## リランキング（Cross-Encoder / 日本語 Reranker）

<!-- メモ:
- 一次検索で多めに取得 → Cross-Encoder で精密に再採点。Bi-Encoder（embed 同士の内積、高速）vs Cross-Encoder（クエリ+文書を同時入力しスコア、高精度・低速）
- モデル：cl-nagoya/ruri-reranker-large。BaseDocumentCompressor 継承 → ContextualCompressionRetriever でラップ（RerankedRetriever）
- コード：rerank.py CrossEncoderReranker.compress_documents / reranked.py。hybrid → rerank の組み合わせ例
-->

## メタデータフィルタリング

<!-- メモ:
- サイドカー JSON のタグ/日付を Document.metadata に載せ、検索後にフィルタ（FAISS 単体はフィルタが弱いので事後フィルタ）
- 実装：TagFilteredRetriever（タグ OR 条件）。順番が重要：Hybrid(50) → Tag filter → Rerank(5)。Rerank の後に置くと上位 5 件にタグ一致が無いと空になる
- 注：サイドカー無し（タグ無し）文書は除外される副作用。事後フィルタなので候補を多めに取るのが前提
- コード：filtered.py
-->

## クエリ変換（HyDE・マルチクエリ・分解）

<!-- メモ:
- 素のクエリが検索に最適とは限らない → クエリ側を変換して再現率を上げる
- HyDE：LLM に仮回答を生成させ、その文で検索（仮回答と実文書が近いベクトルになる）。include_original=True で元クエリ結果と RRF 統合（固有名詞が薄まる対策）
- マルチクエリ：ParaphraseRetriever（言い換え N 個）/ DecomposeRetriever（サブクエリ分解）→ RRF 統合
- コード：hyde.py _get_relevant_documents / multi_query.py
-->

## 親子チャンク（Parent-Child）

<!-- メモ:
- 検索は小さい子チャンク（精度）、文脈として返すのは親の大きいチャンク（情報量）= small-to-big
- 実装：langchain-classic の ParentDocumentRetriever + InMemoryStore。空の faiss.IndexFlatL2 を直接作る（FAISS.from_texts のダミー文書がノイズになるのを回避）。child_k で子の取得件数
- BaseRetriever なので Reranker などのラッパーと組み合わせ可
- コード：parent_child.py build_parent_child_retriever
-->

## 検索結果の調整（MMR・コンテキスト並び順）

<!-- メモ:
- MMR（Maximal Marginal Relevance）：関連度と多様性のバランス（似た文書ばかりを防ぐ）。FAISS as_retriever(search_type="mmr", lambda_mult, fetch_k)。0.0=多様性 / 1.0=関連性。dense.py の mmr オプションとして実装
- Lost-in-the-Middle：長文脈の中間は無視されやすい → 重要文書を先頭/末尾へ。LangChain LongContextReorder を生成前に適用（generation.py の reorder フラグ）。LCEL チェーンの外で並べ替える点に注意
- コード：dense.py（mmr）/ generation.py（LongContextReorder ＋ 出典番号付与 _format_doc）
-->

## 能動的な検索（Agentic RAG / Corrective RAG）

<!-- メモ:
- ここまでは「1 回検索 → 生成」の静的パイプライン。能動的（Agentic）な RAG は LLM が検索プロセスを制御する。その代表として Corrective RAG（CRAG）を実装した（CRAG は Agentic / Adaptive RAG の一種）
- CRAG の流れ：retrieve → grade（文書ごとに LLM-as-Judge で relevant か判定）→ relevant なら generate / not_relevant なら rewrite_query して再検索（max_retries 超過で打ち切り）。LangGraph の StateGraph で実装
- ポイント：original_query と search_query を分離（クエリを書き換えても回答は元質問に対して生成）。文書は 1 件ずつ判定（全件まとめだと無関係が混ざる）。グレーディングにドメイン（IT/プログラミング）を明示しないと誤判定（"uv" を紫外線と解釈など）
- Self-RAG は「違いを一言だけ」：検索要否の判断（Adaptive Retrieval）と生成の忠実性チェックが追加される点。深掘りは別記事
- コード：graph/corrective_rag.py build_corrective_rag（retrieve / grade_documents / generate / rewrite_query ノード）
-->

## 評価の基礎

<!-- メモ:
- ※今回はラベル（qrels）を作っていないので実測結果は載せず、「こういう指標・方法がある」の紹介に留める
- 検索の指標：Recall@k / MRR@k / nDCG@k（metrics.py に実装あり、binary relevance）。正解ラベル qrels.jsonl（query と positive_doc_ids）が要る、と説明
- 生成の評価：Faithfulness（根拠との整合）/ Answer Relevance。RAGAS 等で LLM-as-Judge できる（今回は未実施、次のステップ）
- コード：metrics.py の 3 関数を「こう書ける」程度に短く
-->

## おわりに

<!-- メモ:
- Dense → BM25 → ハイブリッド → リランク → クエリ変換 → 親子チャンク → 能動的検索（CRAG）と積み上げ、各手法がパイプラインのどこを担い何を改善するかが整理できた
- 発展（Contextual Retrieval・GraphRAG・Self-RAG の精緻化・セマンティックキャッシュ・評価ラベル作成による実測）は別記事で
- 実装リポジトリ（rag-sandbox）へのリンクを追記
-->

---
title: LangChain で RAG を組むなら知っておきたい Pydantic の要素
tags:
  - Python
  - Pydantic
  - LangChain
  - RAG
  - LLM
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

<!-- メモ: LangChain で RAG を組み、自作 Retriever/Reranker を書くと Pydantic のルールに必ずぶつかる(self._model = ... が素通りしない等)。Pydantic 自体は RAG 専用ではなく FastAPI などでも使う汎用ライブラリ。本記事は rag-sandbox の実装から、LangChain で必須になる Pydantic の要素を「作る側」「受ける側」の二軸で整理する。前作(RAG 基礎 / GraphRAG)の続きの実装ノート。問題→解決の煽りは避けフラットに。 -->

## Pydantic とは

<!-- メモ: 型ヒントを使ったデータ検証・パースの汎用ライブラリ。RAG/LangChain は一利用先にすぎない。用途(FastAPI のリクエスト/レスポンス検証、pydantic-settings の設定管理、LLM の構造化出力スキーマ、SQLModel など)。v2 はコア(pydantic-core)が Rust 実装で高速。BaseModel にフィールドを型付きで宣言 → インスタンス化時に検証、という基本だけ先に示す。深掘りは後続セクション。 -->

## なぜ LangChain で Pydantic が出てくるのか

<!-- メモ: LangChain の中核クラス(BaseRetriever・BaseDocumentCompressor・Runnable 系)が Pydantic モデルとして作られている。だから自作サブクラスでも Pydantic のルール(フィールド宣言・検証)に従う必要があり、__init__ で self._model = ... と書くと素通りしない/怒られる。対比として Embeddings は素の ABC なので PrefixedEmbeddings は普通の __init__ + self._model でよい(embeddings.py 実例)。「基底クラスが Pydantic モデルか素の ABC か」で書き方が変わる、が背骨。 -->

## 押さえておきたい Pydantic の要素

<!-- メモ: この親直下は導入のみ(自作コンポーネントを Pydantic モデルとして書くときに必要な要素を 4 つ、### で)。 -->

### フィールドと BaseModel の基礎

<!-- メモ: BaseModel を継承し型付きフィールドを宣言 → インスタンス化時に検証。model_name: str = "..."、top_n: int = 5 のようにデフォルト付き。Field(default_factory=list) でミュータブル既定値、Field(description=...) で説明(構造化出力で LLM への指示にもなる)、制約(gt/min_length 等)。v2 前提。最小限に。 -->

### arbitrary_types_allowed

<!-- メモ: FAISS・CrossEncoder・numpy など Pydantic が検証方法を知らない型をフィールドに持ちたいとき、model_config = {"arbitrary_types_allowed": True} で許可する。RerankedRetriever が base_retriever: BaseRetriever / reranker: CrossEncoderReranker を持つ例。ConfigDict の書き方にも軽く触れる。 -->

### PrivateAttr

<!-- メモ: 検証・シリアライズの対象にしたくない内部状態(重いモデルなど)を持つ。_model: CrossEncoder = PrivateAttr()。フィールド(公開・検証対象)と PrivateAttr(内部・アンダースコア始まり)の違い。Pydantic モデルでは __init__ で勝手に self._x = ... できないので PrivateAttr で宣言する、という理由。 -->

### model_validator(mode="after")

<!-- メモ: __init__ を自前で書かずに、検証後の初期化フックでセットアップする。@model_validator(mode="after") def _init_model(self): self._model = CrossEncoder(self.model_name); return self。フィールド(model_name)が確定した後に走るので、それを使って重いモデルを遅延ロードできる。mode="before"/"after" の違いにも一言。 -->

## 実例: 自作 Reranker と Retriever

<!-- メモ: CrossEncoderReranker(rerank.py)全体を提示 = BaseDocumentCompressor 継承 + model_name/top_n フィールド + _model PrivateAttr + arbitrary_types_allowed + model_validator(mode=after) + compress_documents。続けて RerankedRetriever(reranked.py)= BaseRetriever 継承 + arbitrary_types_allowed + _get_relevant_documents。最後に「どの要素がどの問題を解くか」の対応表(フィールド=検証したい設定値 / arbitrary_types_allowed=未知の型を持つ / PrivateAttr=検証したくない内部状態 / model_validator=初期化後セットアップ)。 -->

## LLM の構造化出力を Pydantic で受け取る

<!-- メモ: 受ける側の軸。rag-sandbox は実際には StrOutputParser + 正規表現/startswith("relevant") で受けていた(multi_query.py の _BULLET_RE、corrective_rag.py の grade)。動くが壊れやすい(表記ゆれ・前置きで判定崩れ)。改善例として Pydantic スキーマ + llm.with_structured_output(Schema) を提示(例: class GradeDocuments(BaseModel): relevant: bool = Field(description=...))。これは実コードではなく「こう書けば堅い」改善例だと明示する。description が LLM への指示になる点。作る側(前半)と受ける側(ここ)が Pydantic の二大用途、とまとめる。 -->

## Pydantic v1 / v2 と LangChain の注意点

<!-- メモ: Pydantic は v1→v2 で破壊的変更(@validator→field_validator、@root_validator→model_validator、.dict()→model_dump()、class Config→model_config/ConfigDict)。LangChain は昔 v1 を内部使用し langchain_core.pydantic_v1 シムを公開していたが、langchain-core 0.3(2024/9)で v2 ネイティブへ移行。今は素の pydantic(v2)でよい。古い記事の langchain_core.pydantic_v1 や @validator は v1 時代の名残なので注意。短め。 -->

## おわりに

<!-- メモ: 要素の対応まとめ(作る側の 4 要素 + 受ける側の構造化出力)。基底クラスが Pydantic モデルか素の ABC かを見分けるのが第一歩。Pydantic は汎用なので LangChain 以外(FastAPI 等)でも同じ知識が効く。 -->

## 参考

<!-- メモ: Pydantic 公式ドキュメント(models / PrivateAttr / validators / ConfigDict)、LangChain の custom retriever ドキュメント、with_structured_output のドキュメント。実在確認してから貼る。 -->


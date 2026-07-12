---
title: LangChain で RAG を組むなら知っておきたい Pydantic の要素
tags:
  - Python
  - rag
  - pydantic
  - LangChain
  - LLM
private: false
updated_at: '2026-07-12T21:21:55+09:00'
id: 8105909db8af82c1d52c
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

LangChain で RAG を組んでいると、Pydantic を使うことになります。
自作の Retriever や Reranker、あるいは LLM に構造化データを返させるスキーマは、Pydantic の `BaseModel`(を継承したクラス)として書きます。

正直なところ、これまでは Pydantic を「なんとなく型の安全を守ってくれるもの」として、利点をしっかり理解しないまま使っていた面がありました。

そこで今回は、LangChain で必須になる Pydantic の要素を改めて整理してみます。
自作コンポーネントを Pydantic モデルとして書く「作る側」と、LLM の出力を Pydantic で受け取る「受ける側」の二軸で見ていきます。

## Pydantic とは

Pydantic は、型ヒントを使ってデータを検証・変換する Python のライブラリです。
`BaseModel` を継承したクラスにフィールドを型付きで宣言すると、インスタンス化のときに値が型どおりかを検証してくれます。

```python
from pydantic import BaseModel

class Config(BaseModel):
    model_name: str
    top_n: int = 5

Config(model_name="ruri", top_n=3)      # OK
Config(model_name="ruri", top_n="abc")  # ValidationError（top_n が int でない）
```

RAG や LangChain に限らず、幅広い場面で使われています。

- **FastAPI** — リクエスト/レスポンスの検証・シリアライズが Pydantic ベース
- **pydantic-settings** — 環境変数や設定ファイルを型付きで読み込む
- **LLM 周り** — OpenAI SDK の構造化出力や、LangChain のスキーマ定義
- **SQLModel** — DB のモデル定義(SQLAlchemy + Pydantic)

現在の主流は v2 で、コア部分(`pydantic-core`)が Rust で実装されていて高速です。
本記事のコードもすべて v2 を前提にしています。

## なぜ LangChain で Pydantic が出てくるのか

理由はシンプルで、一部の中核的な抽象クラスが Pydantic モデルだからです。
たとえば `BaseRetriever` や `BaseDocumentCompressor` は、Pydantic の `BaseModel` を継承しています(`BaseRetriever` は `RunnableSerializable` 経由)。
これらを継承して自作コンポーネントを書くと、自動的に Pydantic のルール(フィールドは型付きで宣言する、インスタンス化時に検証される)に従うことになります。

Pydantic が使われる理由は一つではありません。
LLM 出力の構造化だけでなく、コンポーネントの設定値の検証・シリアライズ・スキーマ化や、`Runnable` との統合にも Pydantic の機能が活きています。

ここで引っかかりやすいのが、`__init__` の中で `self._model = SomeModel()` のように属性を代入する書き方です。
素の Python クラスなら普通ですが、Pydantic モデルでは通常、宣言していない属性をそのまま持たせられないため、この書き方は通りません。
だから後述の `PrivateAttr` や `model_validator` が必要になります。

一方で、LangChain のすべてが Pydantic モデルというわけではありません。
たとえば埋め込みの基底クラス `Embeddings` は素の抽象基底クラス(ABC)なので、普通の `__init__` で属性を持たせられます。

以下は `Embeddings` を継承した自作クラスの例です。`self._model` を普通に代入できています。

```python
from langchain_core.embeddings import Embeddings
from sentence_transformers import SentenceTransformer


class PrefixedEmbeddings(Embeddings):
    def __init__(
        self,
        model_name: str = "cl-nagoya/ruri-v3-310m",
        query_prefix: str = "検索クエリ: ",
        doc_prefix: str = "検索文書: ",
        batch_size: int = 64,
    ) -> None:
        self._model = SentenceTransformer(model_name)  # 普通に代入できる
        self._query_prefix = query_prefix
        self._doc_prefix = doc_prefix
        self._batch_size = batch_size
```

つまり、LangChain で自作コンポーネントを書くときは、まず継承元が Pydantic モデルかどうかを確認する必要があります。
`BaseRetriever` や `BaseDocumentCompressor` のような Pydantic モデルを継承するときだけ、次に紹介する Pydantic の作法が必要になります。

## 自作コンポーネントで押さえておきたい Pydantic の要素

Pydantic には多くの機能がありますが、ここでは LangChain の自作コンポーネントを Pydantic モデルとして実装するときによく使う要素を 4 つ見ていきます。
順に「フィールド」「`arbitrary_types_allowed`」「`PrivateAttr`」「`model_validator`」です。

### フィールドと BaseModel の基礎

`BaseModel` を継承したクラスでは、クラス変数のように型付きでフィールドを宣言します。
宣言したフィールドはインスタンス化のときに型が検証され、デフォルト値も設定できます。

```python
from pydantic import BaseModel

class RerankConfig(BaseModel):
    model_name: str      # 必須。型は str
    top_n: int = 5       # デフォルト付き
```

もう少し細かく指定したいときは `Field` を使います。

```python
from pydantic import BaseModel, Field

class RerankConfig(BaseModel):
    model_name: str = Field(description="使用する Cross-Encoder のモデル名")
    top_n: int = Field(default=5, ge=1)                    # 1 以上に制約
    extra_models: list[str] = Field(default_factory=list)  # 可変な既定値
```

ポイントは 3 つです。

- `Field(description=...)` — フィールドの説明。後半の構造化出力では、この説明が JSON Schema 経由で LLM への指示にもなります
- `Field(ge=1)` などの制約 — 値の範囲や長さを検証できます(`ge` / `gt` / `min_length` など)
- `Field(default_factory=list)` — リストや辞書のような値を既定値にするときに使います(呼び出しごとに新しいインスタンスが作られます)

自作コンポーネントの「設定値」(モデル名や件数など)は、こうしたフィールドとして宣言していきます。

### arbitrary_types_allowed

Pydantic は、宣言したフィールドの型を検証するためのスキーマを内部で生成します。
ところが `CrossEncoder` や `FAISS`、`numpy` 配列のような外部ライブラリの型は、Pydantic が検証方法を知らないため、フィールドに宣言するとスキーマ生成に失敗します。

```python
from pydantic import BaseModel
from sentence_transformers import CrossEncoder

class Reranker(BaseModel):
    encoder: CrossEncoder   # これだけだとエラー

# pydantic.errors.PydanticSchemaGenerationError:
# Unable to generate pydantic-core schema for <class 'CrossEncoder'>
```

こういう「Pydantic が知らない型」を扱いたいときは、`model_config` で `arbitrary_types_allowed` を有効にします。
すると Pydantic はその型について独自のスキーマ生成を行わず、`isinstance` による基本的な型チェックだけで扱います。

```python
class Reranker(BaseModel):
    model_config = {"arbitrary_types_allowed": True}

    encoder: CrossEncoder   # これで宣言できる
```

たとえば別の Retriever と Reranker をフィールドに持つ `RerankedRetriever` では、この設定を付けます。

```python
class RerankedRetriever(BaseRetriever):
    base_retriever: BaseRetriever
    reranker: CrossEncoderReranker

    model_config = {"arbitrary_types_allowed": True}
```

`model_config` は辞書のほか、補完の効く `ConfigDict` でも書けます(中身は同じです)。

```python
from pydantic import ConfigDict

class Reranker(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True)
```

### PrivateAttr

フィールドは「外から渡す・検証する・シリアライズする」値のためのものです。
一方で、`CrossEncoder` のような重いモデルは、検証もシリアライズもしたくない「内部で持っておくだけ」の状態です。
こういう内部状態は、フィールドではなく `PrivateAttr` で宣言します。

```python
from pydantic import BaseModel, PrivateAttr
from sentence_transformers import CrossEncoder

class Reranker(BaseModel):
    model_name: str = "cl-nagoya/ruri-reranker-large"   # フィールド（検証・シリアライズ対象）
    _model: CrossEncoder = PrivateAttr()                # プライベート属性（対象外）
```

`PrivateAttr` で宣言した属性は、アンダースコア始まりの名前を持ち、Pydantic の検証やシリアライズの対象外です。
`model_dump()` の出力にも出てきません。

なぜフィールドではなくこれを使うのかというと、Pydantic モデルでは宣言していない属性に `self._model = ...` と代入できないからです。
`PrivateAttr` で「あとで値を入れる内部属性」を宣言しておけば、初期化のなかで代入できるようになります。
実際に値を入れるタイミングは、次の `model_validator` で扱います。

### model_validator(mode="after")

`PrivateAttr` で宣言した `_model` に、いつ値を入れるのかという問題が残ります。
Pydantic モデルでは自前の `__init__` を書く代わりに、`model_validator(mode="after")` や `model_post_init` で初期化後の処理を書きます。

```python
from pydantic import BaseModel, PrivateAttr, model_validator
from sentence_transformers import CrossEncoder

class Reranker(BaseModel):
    model_name: str = "cl-nagoya/ruri-reranker-large"
    _model: CrossEncoder = PrivateAttr()

    model_config = {"arbitrary_types_allowed": True}

    @model_validator(mode="after")
    def _init_model(self) -> "Reranker":
        self._model = CrossEncoder(self.model_name)
        return self
```

`mode="after"` を付けると、このメソッドはフィールドの検証が終わったあとに呼ばれます。
そのため `self.model_name` はすでに確定していて、その値を使って重いモデルを遅延ロードできます。
`mode="after"` のバリデータは検証後のモデルを受け取るので、最後に `self` を返すのが約束事です。

ちなみに、検証の前に生の入力(辞書)を受け取る `mode="before"` もありますが、今回のようにフィールド確定後のセットアップには `mode="after"` を使います。

なお、単に初期化後の処理をしたいだけなら、`model_post_init` を使う手もあります。

```python
from typing import Any

class Reranker(BaseModel):
    model_name: str = "cl-nagoya/ruri-reranker-large"
    _model: CrossEncoder = PrivateAttr()
    model_config = {"arbitrary_types_allowed": True}

    def model_post_init(self, __context: Any) -> None:
        self._model = CrossEncoder(self.model_name)
```

`model_validator(mode="after")` は、初期化に加えて値同士の整合性チェックまでやりたいときに向いています。

## 実例: 自作 Reranker と Retriever

ここまでの要素を組み合わせると、自作コンポーネントが書けます。
まず、`CrossEncoder` でドキュメントを再ランク付けする Reranker です。
`BaseDocumentCompressor`(Pydantic モデル)を継承し、フィールド・`arbitrary_types_allowed`・`PrivateAttr` を使っています。
初期化はモデルをロードするだけなので、`model_validator` ではなく `model_post_init` にしました。

```python
from copy import copy
from collections.abc import Sequence
from typing import Any

from langchain_core.callbacks.manager import Callbacks
from langchain_core.documents import Document
from langchain_core.documents.compressor import BaseDocumentCompressor
from pydantic import PrivateAttr
from sentence_transformers import CrossEncoder


class CrossEncoderReranker(BaseDocumentCompressor):
    model_name: str = "cl-nagoya/ruri-reranker-large"   # フィールド
    top_n: int = 5                                      # フィールド
    _model: CrossEncoder = PrivateAttr()                # 内部状態

    model_config = {"arbitrary_types_allowed": True}    # CrossEncoder を許可

    def model_post_init(self, __context: Any) -> None:
        self._model = CrossEncoder(self.model_name)     # 遅延ロード

    def compress_documents(
        self,
        documents: Sequence[Document],
        query: str,
        callbacks: Callbacks | None = None,
    ) -> Sequence[Document]:
        if not documents:
            return []
        pairs = [(query, doc.page_content) for doc in documents]
        scores = self._model.predict(pairs)
        ranked = sorted(zip(documents, scores, strict=True), key=lambda x: x[1], reverse=True)
        results = []
        for doc, score in ranked[: self.top_n]:
            new_doc = copy(doc)
            new_doc.metadata = {**doc.metadata, "rerank_score": float(score)}
            results.append(new_doc)
        return results
```

そして、この Reranker を任意の Retriever と組み合わせるラッパーです。
`BaseRetriever`(Pydantic モデル)を継承し、フィールドに別の Retriever と Reranker を持ちます。

```python
from langchain_core.callbacks.manager import CallbackManagerForRetrieverRun
from langchain_core.documents import Document
from langchain_core.retrievers import BaseRetriever


class RerankedRetriever(BaseRetriever):
    base_retriever: BaseRetriever
    reranker: CrossEncoderReranker

    model_config = {"arbitrary_types_allowed": True}

    def _get_relevant_documents(
        self,
        query: str,
        *,
        run_manager: CallbackManagerForRetrieverRun,
    ) -> list[Document]:
        candidates = self.base_retriever.invoke(query)
        return self.reranker.compress_documents(candidates, query)
```

要素と役割の対応は次のとおりです。

| 要素 | 役割 |
| --- | --- |
| フィールド(`model_name` / `top_n`) | 外から渡す設定値。型が検証される |
| `arbitrary_types_allowed` | `CrossEncoder` など Pydantic が知らない型を扱えるようにする |
| `PrivateAttr`(`_model`) | 検証・シリアライズしたくない内部状態(重いモデル)を持つ |
| `model_post_init`(または `model_validator`) | フィールド確定後に `_model` をロードする |

## LLM の構造化出力を Pydantic で受け取る

ここまでは「作る側」、つまり自作コンポーネントを Pydantic モデルとして書く話でした。
もう一つの大きな用途が「受ける側」、LLM の出力を Pydantic で受け取ることです。

たとえば「この文書は質問に関連しているか」を LLM に判定させる場面を考えます。
手軽に済ませるなら、文字列で受け取って判定する書き方です。

```python
from langchain_core.output_parsers import StrOutputParser

grade_chain = prompt | llm | StrOutputParser()
raw = grade_chain.invoke({"question": question, "document": doc}).strip().lower()

if raw.startswith("relevant"):
    # 関連あり
    ...
```

これは動きますが、壊れやすい書き方です。
LLM が `relevant` とだけ返す保証はなく、「関連しています」「Relevant: yes」などと返された瞬間に判定が崩れます。

ここで Pydantic のスキーマと `with_structured_output` を使うと、出力を型で受け取れます。

```python
from pydantic import BaseModel, Field

class GradeDocuments(BaseModel):
    relevant: bool = Field(description="文書が質問に関連していれば true、そうでなければ false")

structured_llm = llm.with_structured_output(GradeDocuments)
result = structured_llm.invoke(prompt)   # 返り値は GradeDocuments

if result.relevant:
    # 関連あり
    ...
```

`with_structured_output(GradeDocuments)` は、スキーマを LLM に渡して「この形で返して」と指示し、返ってきた出力を `GradeDocuments` として検証してくれます。
`result.relevant` は必ず `bool` なので、文字列判定のような崩れ方をしません。
このとき `Field(description=...)` の説明は、生成される JSON Schema に含まれて LLM への指示として使われるので、前半で触れた `description` がここで効いてきます。

ちなみにこの「受ける側」では、前半で出てきた `arbitrary_types_allowed` や `PrivateAttr` は出番がありません。
スキーマは、検証・シリアライズされるフィールドだけで完結するからです。

なお `with_structured_output` は、function calling / structured output に対応したモデルで使えます。

## おわりに

LangChain で RAG を組むときに出てくる Pydantic の要素を整理しました。
「作る側」では、自作コンポーネントを Pydantic モデルとして書くために、フィールド・`arbitrary_types_allowed`・`PrivateAttr`・`model_validator` を使いました。
「受ける側」では、LLM の出力を Pydantic スキーマで受け取り、文字列判定より堅く扱えることを見ました。

調べていて思ったことですが、Pydantic は本格的に Python を扱うなら知っておくべき基礎ライブラリかもしれないですね。
自分は LangChain で初めて Pydantic に触れましたが、調べれば調べるほど、LangChain に限らず広く押さえておきたいライブラリだと感じます。

## 参考

- [Pydantic - Models](https://docs.pydantic.dev/latest/concepts/models/)
- [Pydantic - Fields](https://docs.pydantic.dev/latest/concepts/fields/)
- [Pydantic - Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [LangChain - How to return structured data from a model](https://python.langchain.com/docs/how_to/structured_output/)


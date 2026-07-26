---
title: GraphRAG を LightRAG で試してみた - Qiita 記事からナレッジグラフを作って検索・可視化する
tags:
  - GraphRAG
  - LightRAG
  - Python
  - rag
  - ナレッジグラフ
private: false
updated_at: '2026-07-26T23:46:17+09:00'
id: a7269731cdef46c87434
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: de8f4a0a54ab4d37707c
agreed_posting_campaign_term: true
---

## はじめに

前回は RAG の基礎技術を実装して整理しました。

https://github.com/atsushi11o7/rag-sandbox

今回はその発展編として GraphRAG を試します。

GraphRAG は、文書から抽出したエンティティや関係をグラフとして保持し、その構造(ノード・関係・サブグラフなど)を検索や回答生成に使う手法です。
通常のベクトル検索 RAG がチャンク単位で「そこに書いてある内容」を取り出すのに対して、ナレッジグラフは文書をまたいだ関係や概念どうしのつながりを表現できます。

今回は GraphRAG を軽量に実装した [LightRAG](https://github.com/HKUDS/LightRAG)(2024 年 arXiv 公開、Findings of EMNLP 2025)を使い、自分の Qiita 記事をコーパスにナレッジグラフを構築して、検索から可視化まで一通り試してみました。

## ナレッジグラフとは

GraphRAG に入る前に、土台となるナレッジグラフを整理します。

ナレッジグラフは、エンティティ(実体)と関係(エッジ)をグラフ構造で表現したデータベースです。
たとえば「Python」「Google」「Guido van Rossum」をノードとし、「が開発した」「に勤務している」といった有向エッジで繋ぐイメージです。

「ナレッジグラフ」という語自体は、Google が 2012 年に検索基盤として発表したもの([Introducing the Knowledge Graph: things, not strings](https://blog.google/products-and-platforms/products/search/introducing-knowledge-graph-things-not/))が広く知られています。
源流をたどると RDF やセマンティックウェブといった知識表現の研究にあるらしく、RAG のために登場した技術ではありません。
AI と結びつく前から、さまざまな分野で使われています。

- **Google 知識パネル** — 検索結果の右側に出る人物・企業情報はナレッジグラフから引いています。エンティティとその属性をグラフで管理することで、横断的な情報提供ができます
- **[Wikidata](https://www.wikidata.org/)** — Wikipedia の機械可読バックエンドです。RDF 形式で大規模なトリプルを管理しています
- **医療・創薬** — 疾患・遺伝子・薬剤・副作用の関係を大規模グラフで管理し、新薬候補の探索や副作用予測に使います
- **推薦システム** — ユーザー・商品・カテゴリ・行動履歴をグラフで表現し、GNN で協調フィルタリングします
- **サイバーセキュリティ** — 攻撃者・脆弱性・マルウェア・被害組織の関係をグラフで可視化します(Threat Intelligence Graph)

医療・推薦・セキュリティの 3 つは広く知られた応用例として挙げています(特定の一次資料に基づく数値ではありません)。

### 代表的な構築・表現手法

| 手法 | 概要 |
| --- | --- |
| RDF / OWL | W3C 標準。主語・述語・目的語のトリプルでセマンティックウェブを実現し、SPARQL で検索します |
| Property Graph | ノードとエッジに任意の属性を持てる柔軟な形式。Neo4j が代表格で、Gremlin や Cypher で操作します |
| OpenKE / KGE | 既存の大規模ナレッジグラフ(Freebase・DBpedia・Wikidata)をもとに埋め込みを学習するフレームワーク |
| LLM 自動抽出 | 非構造化テキストから LLM でエンティティと関係を抽出して構築する。今回のアプローチ |

最後の「LLM でテキストからグラフを自動構築する」アプローチが、最近の AI 文脈での主流です。
手作業でトリプルを定義する必要がなく、自然言語コーパスさえあればグラフを作れるのが大きな利点で、今回もこれを使います。

## GraphRAG とは

通常の RAG は、チャンクをベクトル化して類似検索します。
クエリに近いテキスト断片を取得して生成モデルに渡す仕組みなので、「そのチャンクに書いてある内容」は取れますが、文書をまたいだ関係や概念レベルのつながりは見えません。

GraphRAG は、ナレッジグラフを検索に使う手法の総称です。
グラフは既存のナレッジグラフ(Wikidata など)を使う場合もあれば、手元のドキュメントから作る場合もあります。
ドキュメントから作る方法にも、従来の固有表現抽出・関係抽出やルールベースなどの選択肢がありますが、近年は LLM にエンティティと関係を抽出させるのが主流です。

検索の面では、「A と B はどう違うか」「この技術はどんな場面で使われているか」「A と B に共通するテーマは何か」といった、複数ドキュメントにまたがる概念的な問いに強いのが特徴です。

サーベイ論文 [Graph Retrieval-Augmented Generation: A Survey](https://arxiv.org/abs/2408.08921) では、GraphRAG の処理を次の 3 段階に整理しています。

- **G-Indexing(グラフの構築)** — 外部知識をナレッジグラフとして構築し、検索に使える形でインデックス化する
- **G-Retrieval(グラフを辿った検索)** — クエリに関連するノード・エッジ・サブグラフといったグラフ要素を取り出す
- **G-Generation(グラフを使った生成)** — 取り出したグラフ要素をコンテキストとして回答を生成する

LightRAG もこの 3 段階に沿っていて、後述の Indexing フェーズが G-Indexing、クエリフェーズが G-Retrieval と G-Generation に当たります。

### 実装の選択肢

最初は、LangChain で GraphRAG を実装しようと思っていました。
ただ、GraphRAG 系の機能(`LLMGraphTransformer`)がある `langchain-experimental` は[サンセットが表明されている](https://github.com/langchain-ai/langchain-experimental/issues/87)ので、だったら別のライブラリを使ってみようかなと考えました。

GraphRAG 専用のものでは、Microsoft の [GraphRAG](https://github.com/microsoft/graphrag) は前処理が重くローカルで気軽に動かす用途には向かなかったので、より軽量な LightRAG を選びました。

## LightRAG とは

[LightRAG](https://github.com/HKUDS/LightRAG) は、香港大学 HKUDS のチームによる軽量な Graph-based RAG フレームワークです。
論文は 2024 年 10 月に arXiv で公開され、その後 Findings of EMNLP 2025 に採択されています。

LightRAG の処理は、大きく「Indexing」と「Retrieval」の 2 つに分けられます。

### Indexing フェーズ

ドキュメントを渡すと、LightRAG はおおよそ次の流れでナレッジグラフと検索用インデックスを構築します。

1. ドキュメントをチャンクに分割する
2. 各チャンクに対して LLM がエンティティと関係を抽出する
3. Gleaning（追加抽出）により、初回抽出で見落としたエンティティや関係を補完する
4. 抽出したエンティティをノード、関係をエッジとしてナレッジグラフに格納する
5. エンティティ・関係に対応する検索用表現をベクトルストアに格納し、元チャンクは KV ストレージ側にも保持する

このように、LightRAG は単にチャンクをベクトル検索するだけではなく、
エンティティと関係から構成されるナレッジグラフを作り、その構造を検索に利用します。
また、Gleaning によって抽出を複数回行えるため、1 回目の抽出で取りこぼした情報を補いやすい点も特徴です。

### クエリフェーズ

クエリを受け取ると、次の 5 つのモードで検索と回答生成を行います。

| モード | 概要 | 向いている問い |
| --- | --- | --- |
| `naive` | チャンクのベクトル検索のみ(通常の RAG に相当) | 特定の記述を探す |
| `local` | クエリに関連するエンティティの近傍グラフを探索 | 特定概念の詳細・関連を知りたい |
| `global` | グラフ全体のコミュニティサマリーを使って俯瞰的に回答 | 全体的なテーマ・傾向を知りたい |
| `hybrid` | `local` + `global` の組み合わせ | バランス重視 |
| `mix` | `hybrid` + `naive`(グラフとベクトルの両方を使用) | 最も広いコンテキストを使いたい |

`local` と `global` がナレッジグラフ検索ならではです。
`local` は「この概念の周辺を深掘りする」、`global` は「グラフ全体から俯瞰して答える」というイメージです。

### 永続化

グラフは GraphML 形式(`graph_chunk_entity_relation.graphml`)に、ベクトル DB は nano-vectordb の JSON 形式(`vdb_entities.json` など)に保存されます。
Neo4j や専用のベクトル DB を立てなくても完結するので、追加サービスなしで動かせます。

## 実装

ここからは実際に動かしたコードを示します。
コード全体は [こちら](https://github.com/atsushi11o7/rag-sandbox) にあります。

### 環境とモデル

実験環境は次のとおりです。

- 実行環境: devcontainer(CUDA 12.6 + Python 3.12)
- パッケージ管理: uv
- LLM(エンティティ抽出・回答生成): Ollama の `qwen2.5:7b`
- 埋め込み: Ollama の `mxbai-embed-large`(1024 次元)
- グラフストレージ: `NetworkXStorage`(GraphML に永続化)

埋め込みモデルは次元に注意が必要です。
LightRAG の `ollama_embed` はデフォルトで 1024 次元を前提としているため、同じ 1024 次元の `mxbai-embed-large` を使います。
(最初は `nomic-embed-text`(768 次元)を使おうとして、次元不一致でクラッシュしました)

LightRAG とグラフ関連のライブラリは以下で入れます。

```bash
uv add lightrag-hku networkx pyvis
```

### モジュール（`lightrag_store.py`）

LightRAG をラップして、「構築・挿入・クエリ」の 3 つの関数にまとめたモジュールです。

```python
import os
from pathlib import Path

from langchain_core.documents import Document
from lightrag import LightRAG, QueryParam
from lightrag.llm.ollama import ollama_embed, ollama_model_complete
from lightrag.utils import EmbeddingFunc

_WORKING_DIR = "data/lightrag"
_EMBED_MODEL = "mxbai-embed-large"
_EMBED_DIM = 1024


async def build_lightrag(
    working_dir: str = _WORKING_DIR,
    llm_model: str = "qwen2.5:7b",
    embed_model: str = _EMBED_MODEL,
    ollama_host: str | None = None,
) -> LightRAG:
    Path(working_dir).mkdir(parents=True, exist_ok=True)
    host = ollama_host or os.environ.get("OLLAMA_HOST")

    llm_kwargs: dict = {"host": host} if host else {}
    embed_kwargs: dict = {"host": host} if host else {}

    rag = LightRAG(
        working_dir=working_dir,
        llm_model_func=ollama_model_complete,
        llm_model_name=llm_model,
        llm_model_kwargs=llm_kwargs,
        llm_model_max_async=1,
        embedding_func=EmbeddingFunc(
            embedding_dim=_EMBED_DIM,
            max_token_size=512,  # mxbai-embed-large の最大入力長
            func=lambda texts: ollama_embed(texts, embed_model=embed_model, **embed_kwargs),
        ),
        graph_storage="NetworkXStorage",
    )
    await rag.initialize_storages()
    return rag


async def insert_documents(rag: LightRAG, docs: list[Document]) -> None:
    texts = []
    for doc in docs:
        title = doc.metadata.get("title", "")
        body = doc.page_content
        texts.append(f"# {title}\n\n{body}" if title else body)

    for i, text in enumerate(texts, 1):
        print(f"  [{i}/{len(texts)}] inserting...")
        await rag.ainsert(text)  # 1 件ずつ逐次処理（後述）


async def query(rag: LightRAG, question: str, mode: str = "hybrid") -> str:
    return await rag.aquery(question, param=QueryParam(mode=mode))
```

ポイントは 2 つあります。

**`llm_model_max_async=1`** — LLM 呼び出しの最大同時実行数を制御する設定です。
デフォルトのまま複数リクエストを並列に走らせると、ローカルの Ollama 環境ではタイムアウトや処理詰まりが起きることがあったため、安定性を優先して 1 に制限しました。

**逐次 `ainsert()`** — `ainsert()` は複数ドキュメントをまとめて渡すこともできますが、まとめて投入すると複数の処理が一度に進みます。
ここでは 1 件ずつ呼び出し、どのドキュメントで失敗したかを追いやすくしています(失敗したときの挙動は後述します)。

### 実験スクリプト

挿入とクエリをまとめて実行する CLI スクリプトです。
`--mode` を省くと 5 モードを順番に実行します。

```python
import argparse
import asyncio

from src.rag.corpus import load_md_corpus
from src.rag.kg.lightrag_store import build_lightrag, insert_documents, query

CORPUS_DIR = "data/corpus"
WORKING_DIR = "data/lightrag"
MODES = ["naive", "local", "global", "hybrid", "mix"]


async def main(args: argparse.Namespace) -> None:
    print("Initializing LightRAG...")
    rag = await build_lightrag(working_dir=WORKING_DIR, llm_model=args.model)

    if not args.skip_insert:
        docs = load_md_corpus(CORPUS_DIR)
        if args.limit:
            docs = docs[: args.limit]
        print(f"{len(docs)} documents loaded. Inserting into knowledge graph...")
        await insert_documents(rag, docs)
        print("Insertion complete.\n")

    modes = [args.mode] if args.mode else MODES
    print(f"Query: {args.query!r}\n" + "=" * 60)

    for mode in modes:
        print(f"\n[{mode.upper()}]")
        result = await query(rag, args.query, mode=mode)
        print(result)


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="LightRAG GraphRAG demo")
    parser.add_argument("--query", default="uvの利点と使い方を教えてください")
    parser.add_argument("--mode", choices=MODES, default=None)
    parser.add_argument("--skip-insert", action="store_true")
    parser.add_argument("--model", default="qwen2.5:7b")
    parser.add_argument("--limit", type=int, default=None)
    return parser.parse_args()


if __name__ == "__main__":
    asyncio.run(main(parse_args()))
```

主な引数は次のとおりです。

- `--skip-insert` — 挿入を飛ばして保存済みのグラフを再利用する(2 回目以降はこれを使います)
- `--limit N` — コーパスの先頭 N 件だけ挿入する(動作確認用)
- `--mode` — 単一モードだけ実行する(省略時は 5 モードすべて)
- `--model` — 使う LLM を切り替える

### 挿入フェーズで分かったこと

実際に動かして分かったことをまとめます。

**エンティティ抽出は重い**

LightRAG の挿入は、1 チャンクにつき LLM を 2 回呼びます(一次抽出 + Gleaning)。
`qwen2.5:7b` + Ollama の環境だと 1 チャンクあたり 30〜120 秒かかり、10 チャンクの記事で 5〜20 分ほどです。
コーパス全量(数十記事)を毎回挿入するのは現実的ではないので、初回だけ挿入し、以降は `--skip-insert` で保存済みのグラフを再利用しました。

**Gleaning でタイムアウトする**

Gleaning(二次抽出)は、長いチャンクで特に時間がかかります。
技術的に複雑なチャンクでは、480 秒を超えてワーカータイムアウトが発生しました。
1 ドキュメントが失敗すると、LightRAG は次回の `ainsert()` 呼び出し時にそのドキュメントを再試行キューに積みます。
これが積み重なると複数ドキュメントが同時にキューへ入り、さらにタイムアウトしやすくなります。

動作確認の段階では、`--limit 2〜3` で小規模に試すのが現実的でした。

## 結果

コーパスから 2 記事(Next.js の App Router 記事と DPO の解説記事)を挿入し、そのうち Next.js の記事を狙った「Next.js の App Router で microCMS を使う方法は?」というクエリを 5 モードで投げました。

### 5 モードの比較

今回の実験では、コーパスが 2 記事と小さいため、モード間の差はあまり大きくありませんでした。
それでも、なんとなく次のような傾向は見えました。

| モード | 回答の特徴 | 性格 |
| --- | --- | --- |
| `naive` | チャンクの手順をほぼそのまま使う | 具体寄り(チャンクの写経に近い) |
| `local` | エンティティと関係を辿った構造化回答 | 概念寄り(今回はたまたま中国語化) |
| `global` | グラフ全体のサマリーで整理 | 俯瞰寄り(`naive` と構造は似る) |
| `hybrid` | `local` + `global` を統合 | バランス(実装パターンも反映) |
| `mix` | グラフ + ベクトルの両方 | 最も広い(具体コード例も追加) |

モード差がはっきり出るのは、本来 GraphRAG が得意とする「複数記事をまたぐ概念的な問い」のほうだと考えられます。
なお、以下は LightRAG の回答をそのまま載せています。
パッケージ名や App Router の実装として不正確な記述も一部含まれますが、ここでは「各モードがどんな回答傾向を示すか」を見る目的で掲載しています。

<details><summary>naive モードの出力</summary>

```text
Next.js 13以降のApp Routerを用いてmicroCMSと連携する手順について説明します。

1. microCMSのAPIキー設定
まず、microCMSにログインし、APIキーを確認してください。

2. Next.jsプロジェクト作成と初期設定
npx create-next-app@latest my-project --example with-app-router

3. microCMS APIクライアントのインストール
npm install microcms-js-sdk

4. サーバーサイドファイルの設定
import { MicroCMSClient } from 'microcms-js';

const client = new MicroCMSClient({
  serviceDomain: process.env.MICROCMS_SERVICE_DOMAIN,
  apiKey: process.env.MICROCMS_API_KEY,
});

export default async function Page() {
  const data = await client.get({ endpoint: 'your-endpoint' });
  return ( ... );
}
...
```

</details>

<details><summary>local モードの出力（中国語で応答したケース）</summary>

```text
使用Next.js 13及其以上版本时，你可以通过以下步骤来结合使用Next.js的App Router和microCMS API：

1. 安装依赖项
npm install next @next/font micro-cms

2. 配置微CMS客户端
// lib/microClient.js
import { MicroCmsClient } from '@microcms/client';
const client = new MicroCmsClient({
  serviceDomain: process.env.NEXT_PUBLIC_MICROCMS_SERVICE_DOMAIN,
  apiKey: process.env.NEXT_PUBLIC_MICROCMS_API_KEY,
});

3. 在页面中使用微CMS API
// pages/index.js
const fetchArticles = async (client) => {
  const response = await client.getEntries('articles');
  return response.contents.map((article) => ({
    title: article.title,
    content: article.content,
    createdAt: article.createdAt,
  }));
};
...
```

</details>

<details><summary>global モードの出力</summary>

```text
Next.jsのApp Router（エッジレンダリングに対応した新しいルーティングシステム）と
microCMSを組み合わせて利用する方法について説明します。

1. microCMS SDKのインストール
npm install @microCMS/microcms-js-sdk

2. microCMSクライアントの設定
.env.localファイルを作成し、microCMSのAPIキーを設定します：
NEXT_PUBLIC_MICROCMS_API_KEY=your_micro_cms_api_key_here

3. APIエンドポイントの作成
...
```

</details>

<details><summary>hybrid モードの出力</summary>

```text
Next.js 13以降で導入されたApp RouterとmicroCMSを組み合わせて使用する方法について解説します。

1. microCMSのセットアップ
   microcms.ioで新しいプロジェクトを作成し、APIキーを取得します。

2. データ取得用のAPIコンポーネント
   カスタムHookやコンテキストなどを用いて、microCMSからデータを取得する関数を作成します。

export default function useMicroCMSData(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    async function fetchMicroCMSData() {
      const response = await fetch(`${API_URL}${url}`, {
        headers: { 'X-MICRO-CMS': API_KEY },
      });
      ...
    }
    fetchMicroCMSData();
  }, [url]);
  return { data, error };
}
...
```

</details>

<details><summary>mix モードの出力</summary>

```text
Next.js 13以降で導入されたApp Routerを使用してmicroCMSを利用するための一般的な手順を説明します。

1. microCMS APIキーとURLの取得
まず、microCMS dashboardからAPIキーと公開URLを取得します。

2. データ取得用のAPI関数を作成
// pages/api/microcms.js
export default async function handler(req, res) {
  const { apikey, endpointUrl } = process.env;
  const response = await fetch(`${endpointUrl}/posts`, {
    headers: { 'X-API-KEY': apikey, 'Content-Type': 'application/json' },
  });
  const data = await response.json();
  res.status(200).json(data);
}

3. データを取得し表示するページを作成
// app/page.tsx
export default function Home() {
  const [posts, setPosts] = useState([]);
  useEffect(() => {
    async function fetchPosts() {
      const res = await fetch('/api/microcms');
      const data = await res.json();
      setPosts(data);
    }
    fetchPosts();
  }, []);
  return ( <div><h1>記事一覧</h1>{posts.map((post) => ( ... ))}</div> );
}
...
```

</details>

### グラフの可視化

LightRAG が構築したグラフを、実際に見てみます。

**ナレッジグラフ全体**

2 記事(DPO の解説記事 + Next.js の App Router 記事)から構築したナレッジグラフ全体です。

![LightRAG ナレッジグラフ全体](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/4f019027-d3c9-432b-8687-41f9255c14b9.png)

- ノード数: 512
- エッジ数: 433
- ノードの色は次数に応じた濃淡で、黄色いほど多くのエンティティと関係を持つハブノードです
- 2 記事でこの規模なので、コーパス全量を入れれば数万ノード規模になります

**クエリに対応したサブグラフ**

LightRAG がクエリに対してどのエンティティを参照したかを見るために、次の処理を自前で実装しました。

1. クエリを `mxbai-embed-large` で埋め込む
2. エンティティ VDB(`vdb_entities.json`)にコサイン類似度で検索し、上位 12 エンティティを取得する
3. GraphML から「それらのエンティティ + 1 ホップ近傍」のサブグラフを切り出す
4. pyvis(インタラクティブ HTML)と matplotlib(静的 PNG)で可視化する

「Next.js の App Router で microCMS を使う方法は?」をクエリした結果がこちらです。

![クエリに対応したサブグラフ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/f174e508-23c3-4b81-9464-f60fc4b35f10.png)

赤いノードがクエリ関連エンティティ、見難いですが青が近傍エンティティ(`Vercel`・`FumiBlog`・`Atomic Design` など)です。
コサイン類似度の上位は次のとおりで、`microCMS` も 0.623 でヒットしています。

| スコア | エンティティ |
| --- | --- |
| 0.858 | Next.js App Router |
| 0.779 | Next.js |
| 0.778 | Next.js 15 with App Router |
| 0.761 | App Router |
| 0.715 | AppRouter |
| 0.662 | generateStaticParams Function |
| 0.652 | RSS Feed Generation |
| 0.630 | Static Generation |
| 0.623 | microCMS |

サブグラフを切り出す可視化コードはこういう構成です。

```python
import base64, json
import numpy as np
import networkx as nx
import requests

def embed_query(text: str) -> np.ndarray:
    resp = requests.post(
        "http://localhost:11434/api/embeddings",
        json={"model": "mxbai-embed-large", "prompt": text}
    )
    return np.array(resp.json()["embedding"], dtype=np.float32)

def load_vdb(path: str) -> tuple[list[dict], np.ndarray]:
    with open(path) as f:
        vdb = json.load(f)
    # matrix は base64 エンコードされた float32 バイナリ
    matrix_bytes = base64.b64decode(vdb["matrix"])
    matrix = np.frombuffer(matrix_bytes, dtype=np.float32)
    matrix = matrix.reshape(len(vdb["data"]), vdb["embedding_dim"])
    return vdb["data"], matrix

def cosine_search(q: np.ndarray, M: np.ndarray, top_k: int = 12) -> np.ndarray:
    q = q / (np.linalg.norm(q) + 1e-9)
    M = M / (np.linalg.norm(M, axis=1, keepdims=True) + 1e-9)
    return np.argsort(M @ q)[::-1][:top_k]

def extract_subgraph(G: nx.DiGraph, seeds: list[str], hop: int = 1) -> nx.DiGraph:
    nodes = set(seeds) & set(G.nodes())
    for _ in range(hop):
        neighbors = set()
        for n in nodes:
            neighbors.update(G.predecessors(n))
            neighbors.update(G.successors(n))
        nodes |= neighbors
    return G.subgraph(nodes).copy()
```

## 気づいたこと・使い分け

### よかった点

- **実装が手軽**: `lightrag-hku` を入れて数十行書くだけで GraphRAG が動きます。Neo4j も Elasticsearch も要りません
- **5 モードの使い分け**: 同じクエリでもモードで回答の視点が変わります。`global` は「全体的な傾向」、`local` は「概念の詳細」と使い分けられます
- **グラフの永続化**: GraphML と JSON に保存されるので、一度作ったグラフは何度でも再利用できます
- **コーパス内コンテンツへの精度**: グラフに入っているドキュメントに関するクエリは、通常の RAG と遜色ない回答が返りました

### 課題・注意点

- **挿入コスト**: 1 チャンクに LLM を 2 回呼ぶため、大きなコーパスへの適用はコストと時間がかかります。`--limit N` で小規模から試すのが現実的です
- **エンティティが英語になる**: `qwen2.5:7b` で抽出すると、エンティティが英語に正規化される傾向がありました(他のモデルは未確認)。LightRAG のエンティティ抽出プロンプト(指示文と few-shot 例)がデフォルトで英語向けなためだと思われるので、日本語に揃えるには、抽出プロンプト自体のカスタマイズが必要です
- **モデル依存**: 抽出も生成もモデル頼りです。使用した `qwen2.5:7b` は、ローカルで動かせる規模としては優秀ですが、今回のタスクの要求に対しては精度が足りない場面があり、モデルの性質からか出力がたまに中国語になることもありました。RAG システム全般に言えることですが、モデルの性能によって結果がかなり左右されます

### 通常の RAG との使い分け

ベクトル検索 RAG が「チャンクに書いてある情報を正確に取り出す」のが得意なのに対して、GraphRAG は「概念間の関係を辿る」のが得意です。
どちらが優れているということではなく、問いの種類によって使い分けるか、`mix` モードのように両方を組み合わせるのが現実的だと思いました。

## おわりに

GraphRAG を LightRAG で簡単に試してみました。
ナレッジグラフが RAG のための技術ではなく、検索エンジンやセマンティックウェブから続く一般的なデータ表現だと整理できたのは収穫でした。

検索の面では、`local` と `global` という 2 つの視点が GraphRAG ならではで面白かったです。
「グラフに積み上げた知識を、文書の境界を超えて辿る」という体験は、通常のベクトル検索とは少し違う手触りでした。

一方で、挿入コストなどの課題も見えました。
実用するならば、コーパスを増やしてモードの違いをもっと際立たせたり、日本語エンティティ抽出のプロンプトを調整したりするのが、次のステップになりそうです。

## 参考

- [LightRAG（GitHub）](https://github.com/HKUDS/LightRAG)
- Zirui Guo et al. [LightRAG: Simple and Fast Retrieval-Augmented Generation](https://arxiv.org/abs/2410.05779)（Findings of EMNLP 2025）
- Boci Peng et al. [Graph Retrieval-Augmented Generation: A Survey](https://arxiv.org/abs/2408.08921)
- [Microsoft GraphRAG（GitHub）](https://github.com/microsoft/graphrag)
- [Introducing the Knowledge Graph: things, not strings](https://blog.google/products-and-platforms/products/search/introducing-knowledge-graph-things-not/)（Google, 2012）

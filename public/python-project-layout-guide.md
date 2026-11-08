---
title: Pythonプロジェクト、どこに何を置く？ - 構成の考え方と実例
tags:
  - Python
  - uv
  - DevContainer
  - 機械学習
  - プロジェクト管理
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

これまで Python プロジェクトのディレクトリ構成は、なんとなく作っていました。`src/` の下に実装の本体を置き、トップレベルには設定ファイルや環境構築用のファイル、実験するなら `notebooks/` を置く。そんな形にはしていたものの、明確な指針があったわけではありません。

よく見かける構成には、どんな理由があるのか。プロジェクトによって変わる部分は、どう判断すればよいのか。この記事では、公式資料やテンプレートで説明されている考え方を調べ、私のプロジェクトでどう取り入れたかを紹介します。

普段、依存管理や環境構築には uv を使っています。今回、uv で自分のコードをパッケージとしてインストールする設定を調べたことが、プロジェクトの構成を見直すきっかけになりました。コードをどこに置くかだけでなく、どうインストールして使うかも、合わせて考える必要があると感じたためです。

本文のインストール手順や設定例も uv を前提としています。uv の挙動については、2026年11月時点の公式ドキュメントを参照しています。

## よく見かける骨格

プロジェクトの種類が違っても、次の形はよく見かけます。

```text
project/
├── pyproject.toml   # プロジェクトの情報、依存、ツールの設定
├── uv.lock          # 解決した依存のバージョンの固定
├── README.md
├── src/
│   └── my_package/  # import するコード
├── tests/
└── docs/
```

この中で判断が必要になりやすいのは、コードの置き方です。

## flat レイアウトと src レイアウト

Python Packaging User Guide では、2つの構成を比較しています。

https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/

```text
# flat レイアウト          # src レイアウト
project/                   project/
├── pyproject.toml         ├── pyproject.toml
└── my_package/            └── src/
    └── __init__.py            └── my_package/
                                   └── __init__.py
```

- **flat レイアウト**: 設定ファイルと import するパッケージを、どちらもプロジェクトのトップレベルに置く
- **src レイアウト**: import できるようにしたいコードを、`src/` のようなサブディレクトリに移す

src レイアウトでは、通常はパッケージをインストールしてから import して使います。リポジトリのルートから Python を起動しても、`src/` 内のパッケージは自動では見つからないためです。

src レイアウトには、リポジトリのルートにいるだけでパッケージを import できてしまうことを避け、インストール設定の不備に気付きやすくする利点があります。

## 開発中に使う editable インストール

**editable インストール**は、作業中のソースコードを参照する形でパッケージをインストールする方式です。uv は、プロジェクト自体をインストールする場合、既定でこの方式を使います。

通常のインストールでは、パッケージのコードなどをまとめた wheel（`.whl` 形式の配布用ファイル）の中身が、Python 環境の `site-packages/` に展開されます。元のソースコードを編集しても、インストール済みのコードは変わらないため、変更を反映するには再インストールが必要です。

一方、editable インストールでは、`.pth` ファイルなどの仕組みで作業中のソースを読み込めるようにします。そのため、Python コードを編集すると、再インストールしなくても次の実行に変更が反映されます。コードを編集するたびにコンテナをリビルドする必要もありません。

たとえば、次の構成で `my_package` を editable インストールしたとします。

```text
project/
├── src/
│   └── my_package/
│       ├── __init__.py
│       └── model.py
├── tests/
└── notebooks/
```

同じ Python 環境を使っていれば、テストやノートブックからも、次のように import できます。

```python
from my_package import model
```

実行する場所に合わせて `sys.path.append("../src")` などを書く必要はありません。

依存関係や `[project.scripts]` のコマンド定義など、`pyproject.toml` の設定を変更した場合は `uv sync` を再実行します。

ただし、editable インストールで動いても、配布物に必要なファイルが揃っているとは限りません。wheel なら別の環境に通常インストールし、Docker イメージならその中で動かすなど、実際に届ける形で確認する必要があります。

## インストールするプロジェクト、しないプロジェクト

uv がプロジェクト自体をインストールする条件は、公式ドキュメントの「Configuring projects」に説明されています。

https://docs.astral.sh/uv/concepts/projects/config/

既定では、**build system が定義されているかどうか**によって、プロジェクト自体を仮想環境にインストールするかが決まります。

| `pyproject.toml` | 仮想環境に入るもの |
|---|---|
| `[build-system]` がない | 依存だけ |
| `[build-system]` がある | 依存とプロジェクト自体（既定は editable） |

`tool.uv.package` を `true` / `false` にすると、この判断を明示的に上書きできます。

同じドキュメントでは、パッケージが必要な例としてコマンドの追加、配布、src レイアウトとテストの利用、ライブラリの作成を挙げています。一方、スクリプトや単純なアプリケーションなど、プロジェクト自体をインストールせずに使える場合は、build system を定義しない選択もできます。

`uv init` で生成される構成は、公式ドキュメントの「Creating projects」に説明されています。

https://docs.astral.sh/uv/concepts/projects/init/

主なオプションによる違いは次のとおりです。

| オプション | 生成されるコードの場所 | build system |
|---|---|---|
| `--app`（既定） | `src/<project_name>/` | あり |
| `--lib` | `src/<project_name>/`（`py.typed` も作られる） | あり |
| `--no-package` | `main.py` | なし |

なお、uv v0.12 より前は、アプリケーションでは既定で build system が定義されませんでした。以前のバージョンを使っている場合は、生成される構成が異なります。

## プロジェクトの種類によって足すもの

骨格に何を足すかは、扱うものと、最終的に何を届けるかによって変わります。次の表は、私が整理した一例です。

| プロジェクトの種類 | 構成に加わるものの例 | 成果物の例 |
|---|---|---|
| ライブラリ | CI、公開の設定 | wheel |
| Web サービス・CLI | `Dockerfile`、環境ごとの設定 | Docker イメージ、実行ファイル |
| 機械学習・データ分析 | `data/`、`notebooks/`、`models/`、`reports/` | モデル、レポート |

機械学習・データ分析のプロジェクト構成は、プロジェクトテンプレートの Cookiecutter Data Science が参考になります。

https://cookiecutter-data-science.drivendata.org/

たとえば、`data/` を次のように分けています。

```text
data/
├── external/   # サードパーティ由来のデータ
├── raw/        # 元の、変更しないデータ
├── interim/    # 変換の途中のデータ
└── processed/  # モデリングに使う最終的なデータ
```

Cookiecutter Data Science の設計方針は、公式サイトの「Opinions」に説明されています。

https://cookiecutter-data-science.drivendata.org/opinions/

この構成では、生データを変更せずに残し、コードと生データから最終成果物を再現できることを重視しています。フォルダの名前だけでなく、データがどの段階を経て加工されるかを整理する考え方です。

## 私のプロジェクトの場合

ここからは実例として、機械学習のプロジェクトで使っている構成を紹介します。依存管理に uv、開発環境に Dev Container を使っています。

```text
.
├── .devcontainer/      # 開発環境の定義
├── data/
│   ├── raw/            # 元データそのまま
│   ├── interim/        # ログなどの途中の生成物
│   └── processed/      # 学習用に整形したデータ
├── docs/               # 調べたことのまとめ
├── src/
│   └── my_project/     # 実装一式
├── README.md
├── pyproject.toml
└── uv.lock
```

### 実装は `src/my_project/` に集約する

※ パッケージ名は記事用に置き換えています。

最初は依存を管理するだけだったので、プロジェクト自体はインストールしない設定（`tool.uv.package = false`）にしていました。その後、モデルや学習、評価のコードを、どのディレクトリからでも `import my_project` で使いたくなりました。そこで `tool.uv.package = false` の設定を削除し、build system を定義して、src レイアウトの自作パッケージをインストールする構成に変えました。`uv init` は使わず、既存の `pyproject.toml` に `[build-system]` を追記しています。

```toml
[build-system]
requires = ["setuptools>=77"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]
```

### Dev Container でのインストール

私は普段、Dev Container を使って開発しています。依存ライブラリはイメージのビルド時にインストールし、そのレイヤーをキャッシュして、コードだけを変更したときの再ビルドで再利用できるようにしています。

一方、自作パッケージはイメージのビルド時点ではまだ存在しません。ホスト側のソースは、コンテナ起動時に `/workspace` へマウントされて初めて中身が揃うためです。そのため editable インストールは、マウント後に動く `postCreateCommand` で行う必要があり、依存ライブラリのインストールとは別の段階に分けています。

仮想環境は、マウントに隠れない `/opt/venv` に置き、両方の段階で同じ環境を使います。

イメージのビルド時には、依存の定義をコピーして、依存だけをインストールします。以下は該当部分の抜粋です。ローカルの wheel に依存する場合は、そのファイルも先にコピーします。

```dockerfile
ENV UV_PROJECT_ENVIRONMENT=/opt/venv
ENV PATH="/opt/venv/bin:${PATH}"

WORKDIR /workspace
COPY pyproject.toml ./
COPY uv.lock ./

RUN uv sync --frozen --no-install-project
```

`--no-install-project` は、uv の公式 Docker ガイドでも、変更の多いプロジェクト本体と変更の少ない依存を別のレイヤーに分ける方法として紹介されています。

https://docs.astral.sh/uv/guides/integration/docker/

ただ、このままだとプロジェクト自体はインストールされません。そこで、ソースがマウントされた後に動く `postCreateCommand` でインストールします。

```json
"postCreateCommand": "uv sync --frozen"
```

これで、依存のレイヤーをキャッシュしつつ、`src/my_project/` を editable でインストールできます。

### `data/` はデータの流れで分ける

`data/` は Cookiecutter Data Science の分け方に倣いました。`data/external/` や、リポジトリ直下の `models/`、`notebooks/`、`reports/` などは、必要に応じて追加する方針です。

`data/raw/` は元データそのままで、手を加えません。実験の結果は `data/interim/` に出力します。

### なぜ src レイアウトを選んだか

Cookiecutter Data Science v2 では、パッケージをリポジトリ直下の `my_project/` に置きます。私の構成では `src/my_project/` に置いています。どちらも `import my_project` で使いますが、パッケージの置き場所が異なります。

「flat レイアウトと src レイアウト」で触れたとおり、リポジトリ直下にパッケージを置くと、ルートにいるだけで import パスにそのパッケージが入ってしまい、インストールしていなくても動いてしまうことがあります。インストールを通して初めてパッケージを使える状態にしたかったため、src レイアウトを選びました。

## おわりに

これまではディレクトリの名前や配置を中心に考えていましたが、今回は、コードをどうインストールして使うかまで含めて構成を整理できました。今後も、よく使われる構成の理由を参考にしながら、自分のプロジェクトに必要なものを選んでいこうと思います。

## 参考

- [src layout vs flat layout - Python Packaging User Guide](https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/)

- [Creating projects - uv](https://docs.astral.sh/uv/concepts/projects/init/)

- [Configuring projects - uv](https://docs.astral.sh/uv/concepts/projects/config/)

- [Using uv in Docker - uv](https://docs.astral.sh/uv/guides/integration/docker/)

- [Cookiecutter Data Science](https://cookiecutter-data-science.drivendata.org/)


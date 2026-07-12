---
title: Dev Container + Dockerfile で作る GPU 対応 Python 機械学習環境
tags:
  - Python
  - Docker
  - DevContainer
  - PyTorch
  - CUDA
private: false
updated_at: '2025-08-03T17:57:37+09:00'
id: 694e878558c7a63912bf
organization_url_name: null
slide: false
ignorePublish: false
---

## 概要

仕事やコンペなどで機械学習モデル(特に PyTorch)を扱う際、簡易的に GPU を活用できる開発環境を構築したいと思うことがあります。

今回は、Dev Container と Dockerfile を組み合わせて、CUDA 環境下で汎用的に使用できる Python 機械学習環境を構築するテンプレートを紹介します。
まず基本の構成(Dockerfile と devcontainer.json)を一式示し、そのあとで各設定の理由や、必要に応じて足す設定を解説します。

## なぜ Dev Container + Dockerfile を使うのか

VS Code の Dev Container(Remote Containers)機能を使うと、次のようなメリットがあります。

- 開発環境の再現性が高い
- ホスト環境を汚さない
- 複数のプロジェクト間で環境を分離できる
- GPU 対応の Docker イメージを使えば、CUDA や cuDNN などの依存関係の面倒なインストールを避けつつ、GPU を活用した開発ができる

なお、コンテナから GPU を使うには、イメージだけでなく ホスト側の NVIDIA ドライバ と、Linux では NVIDIA Container Toolkit などの GPU 実行環境が必要です。

### 設定ファイル

```bash
├── .devcontainer
│   ├── devcontainer.json
│   └── Dockerfile
```

## 使用する Dockerfile

以下が、今回使用する Dockerfile です。ベースは `nvidia/cuda:12.6.2-cudnn-devel-ubuntu24.04` を使用しています。

```dockerfile
FROM nvidia/cuda:12.6.2-cudnn-devel-ubuntu24.04

# ビルド中に apt を対話させない（tzdata などのプロンプト回避）
ARG DEBIAN_FRONTEND=noninteractive

# ロケール設定（日本語表示で文字化けしないように）
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8
# Python の標準出力をバッファリングしない（ログが即座に出る）
ENV PYTHONUNBUFFERED=1

# 作業ディレクトリの作成
WORKDIR /workspace

# 必要なパッケージのインストール
RUN apt-get update && apt-get install -y \
    python3 python3-pip python3-venv python3-dev \
    locales \
    git curl vim unzip \
 && locale-gen en_US.UTF-8 \
 && update-locale LANG=en_US.UTF-8 \
 && rm -rf /var/lib/apt/lists/*

# 仮想環境を作り、PATH を通す（以降 pip / python は venv のものになる）
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Python パッケージのインストール（PyTorch 含む）
RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir torch==2.7.1 --index-url https://download.pytorch.org/whl/cu126 \
 && pip install --no-cache-dir \
        numpy \
        pandas \
        matplotlib \
        scikit-learn \
        jupyterlab \
        seaborn \
        tqdm \
        ipywidgets \
        ipykernel

# Jupyter 用カーネルの登録
RUN python -m ipykernel install --name xxx-dev --display-name "Python (xxx-dev)" --sys-prefix

# コンテナ起動時のデフォルトコマンド
CMD ["bash"]
```

## devcontainer.json の設定

VS Code でこの Docker 環境を使うには、`.devcontainer/devcontainer.json` を以下のように記述します。
なお、`name` の `xxx-dev` や `/mnt/c/Users/xxx/` は環境依存なので、各自の環境に合わせて置き換えてください。

```json
{
  "name": "xxx-dev",
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",
  "workspaceFolder": "/workspace",
  "mounts": [
    "source=/mnt/c/Users/xxx/,target=/data,type=bind"
  ],
  "runArgs": ["--gpus", "all"],
  "customizations": {
    "vscode": {
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash",
        "python.defaultInterpreterPath": "/opt/venv/bin/python"
      }
    }
  },
  "remoteUser": "root"
}
```

| 項目名 | 説明 |
|---|---|
| `build` | Dockerfile を指定します。`context` は親ディレクトリ |
| `workspaceMount` | プロジェクトフォルダを `/workspace` にマウント。Dev Container が標準で行うソースのマウント先を変える設定 |
| `workspaceFolder` | `/workspace` をデフォルト作業フォルダに設定 |
| `mounts` | プロジェクト以外のフォルダを追加でマウント。データの永続化に便利 |
| `runArgs` | `--gpus all` で GPU を利用 |
| `customizations.vscode.settings` | Python の仮想環境パスを指定。VS Code の補完や Linter が仮想環境を認識します |

Dev Container は、プロジェクトフォルダを標準で自動 bind mount します。
そのマウント先を `/workspace` に変えたいときは、`mounts` ではなく `workspaceMount` を使うと意図が明確です。
`/mnt/c/...` は WSL 上で VS Code を開いている場合のパスなので、Docker Desktop を Windows 側から直接使う場合などは異なります。

## 使い方

プロジェクトルートを VS Code で開き、`Dev Containers: Reopen in Container` を選択すると、環境のビルドが行われます。

## Dockerfile の解説

### ベースイメージの指定

```dockerfile
FROM nvidia/cuda:12.6.2-cudnn-devel-ubuntu24.04
```

NVIDIA が提供している CUDA 対応の公式イメージです。
`devel` タグには、CUDA ランタイムに加えて、コンパイラ(`nvcc`)やヘッダーなどの開発用ツールが含まれます。

PyTorch を pip の公式 wheel で「動かすだけ」なら、必要な CUDA ランタイム系は wheel 側に含まれるため、より軽量な `runtime` イメージでも足りる場合があります。
一方、CUDA 拡張のビルドや `nvcc` を使う可能性があるなら `devel` が便利です。今回は汎用的な開発環境として `devel` を選んでいます。

CUDA バージョンは GPU やドライバに依存するため、環境に合わせてベースイメージを選びます。
たとえば RTX 50 シリーズのような新しい GPU では、その GPU アーキテクチャをサポートした PyTorch の CUDA ビルドが必要で、環境によっては CUDA 12.8 対応版以降の PyTorch を選ぶ必要があります。

### ロケールとビルド時の設定

```dockerfile
ARG DEBIAN_FRONTEND=noninteractive
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8
ENV PYTHONUNBUFFERED=1
```

- `DEBIAN_FRONTEND=noninteractive`：ビルド中に apt が対話プロンプト(tzdata など)を出さないようにします。
- ロケール(UTF-8)：日本語ファイル名や Jupyter Notebook での文字化けを防ぎます。ローカルのディレクトリをマウントしたときのトラブル対策でもあります。
- `PYTHONUNBUFFERED=1`：Python の標準出力をバッファリングせず、ログが即座に出るようにします。

### 作業ディレクトリの設定

```dockerfile
WORKDIR /workspace
```

コンテナ内での作業場所です。VS Code からもここが開かれるようになります。

### 必要なパッケージのインストール

```dockerfile
RUN apt-get update && apt-get install -y \
    python3 python3-pip python3-venv python3-dev \
    locales \
    git curl vim unzip \
 && locale-gen en_US.UTF-8 \
 && update-locale LANG=en_US.UTF-8 \
 && rm -rf /var/lib/apt/lists/*
```

- `python3`, `pip`, `venv`, `dev`：Python 関連ツール
- `locales`：ロケール変更のため
- `git`, `curl`, `vim`, `unzip`：開発に便利なツール

最後の `rm -rf /var/lib/apt/lists/*` で、不要な apt のキャッシュを消してイメージサイズを削減します。

### 仮想環境の作成と PATH（PEP 668）

```dockerfile
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
```

最近の Debian / Ubuntu 系ディストリビューションが提供する Python では、システムの Python に直接 `pip install` しようとすると、PEP 668(externally-managed-environment)によりエラーで止められます。
`--break-system-packages` で無理に入れることもできますが、素直に仮想環境(venv)を作り、その中にインストールします。

なお、Docker コンテナはホスト OS と分離されているので venv は不要では、という意見もあり、このあたりは諸説あります。
私は Dockerfile の中でも venv を使うことが多いです。

`ENV PATH` で venv の `bin` を先頭に通しておくと、以降の `pip` / `python` は venv のものが使われます。
毎回 `/opt/venv/bin/pip` と書く必要がなく、コンテナ内で開くシェルでも venv が有効になるので、`activate` を仕込む必要もありません。

### ライブラリのインストール

```dockerfile
RUN pip install --no-cache-dir --upgrade pip \
 && pip install --no-cache-dir torch==2.7.1 --index-url https://download.pytorch.org/whl/cu126 \
 && pip install --no-cache-dir \
        numpy pandas matplotlib scikit-learn jupyterlab seaborn tqdm ipywidgets ipykernel
```

- PyTorch は、使いたい CUDA ビルドを `--index-url`(ここでは `cu126`)で選びます。`--no-cache-dir` は pip のキャッシュを残さず、イメージサイズを抑えるためです。
- 機械学習・データ分析に必要な主要ライブラリをまとめて入れます(Jupyter 対応も含む)。

本記事では、検証時点の組み合わせとして PyTorch 2.7.1 と CUDA 12.6 を使っています。
新しく環境を作る場合は、[PyTorch 公式のインストールページ](https://pytorch.org/get-started/locally/) で利用可能な組み合わせを確認してください。

### Jupyter カーネルの登録

```dockerfile
RUN python -m ipykernel install --name xxx-dev --display-name "Python (xxx-dev)" --sys-prefix
```

明示的にカーネルを登録しておくと、Jupyter Lab や VS Code のカーネル選択画面で「Python (xxx-dev)」として識別しやすくなります(VS Code が仮想環境を自動検出する場合は、登録しなくても選べることがあります)。
`--sys-prefix` を付けると、システム全体ではなく venv の prefix 配下に登録されます。

### 起動コマンド

```dockerfile
CMD ["bash"]
```

通常の `docker run` で起動したときのデフォルトコマンドとして bash を指定しています。
Dev Container から起動する場合は、拡張機能がコンテナを維持するためにコマンドを上書きすることがあります。

## その他、必要に応じて行う設定

### uv で管理する場合

pip の代わりに [uv](https://github.com/astral-sh/uv) を使うと、依存解決とインストールがかなり速くなります。
Dockerfile では、uv を入れてから venv への導入を `uv pip` に置き換えます。

```dockerfile
# uv を入れる（公式インストーラ）
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.local/bin:$PATH"

# uv で venv を作り、そこに入れる
RUN uv venv /opt/venv
ENV VIRTUAL_ENV=/opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN uv pip install --no-cache torch==2.7.1 --index-url https://download.pytorch.org/whl/cu126 \
 && uv pip install --no-cache \
        numpy pandas matplotlib scikit-learn jupyterlab seaborn tqdm ipywidgets ipykernel

# カーネル登録は pip 版と同じ
RUN python -m ipykernel install --name xxx-dev --display-name "Python (xxx-dev)" --sys-prefix
```

`uv pip install` は、`VIRTUAL_ENV` と PATH で指定した仮想環境(`/opt/venv`)に対して実行されます(pip の `--no-cache-dir` に相当するのが uv の `--no-cache` です)。
Jupyter カーネルの登録は pip 版と同じなので、忘れないようにします。

再現性を重視するなら、`curl | sh`(常にその時点のインストーラを実行)よりも、公式イメージからバイナリをコピーしてバージョンを固定する方法もあります。

```dockerfile
# <version> は公式の最新タグに置き換え、再現性のために固定します
COPY --from=ghcr.io/astral-sh/uv:<version> /uv /uvx /usr/local/bin/
```

### GitHub CLI（gh）を入れる

PR やイシューの操作に GitHub CLI(`gh`)をよく使うなら、ビルド時に入れておくと便利です。
`gh` は Ubuntu の標準リポジトリには無いので、公式の apt リポジトリを追加してからインストールします。

```dockerfile
RUN mkdir -p -m 755 /etc/apt/keyrings \
 && curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
      -o /etc/apt/keyrings/githubcli-archive-keyring.gpg \
 && chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
      > /etc/apt/sources.list.d/github-cli.list \
 && apt-get update && apt-get install -y gh \
 && rm -rf /var/lib/apt/lists/*
```

この節は独立した RUN レイヤーなので、Dockerfile の好きな位置(パッケージインストールの後など)に足せます。
初回はコンテナ内で `gh auth login` を実行して認証します。

### 非 root ユーザーで動かす（任意）

ここまでのテンプレートは root で動かしています(`remoteUser: root`)。
手早く試すには十分ですが、ホストのファイルを root 所有で書き換えてしまうことや、セキュリティを気にするなら、非 root ユーザーを作る手もあります。

```dockerfile
# 非 root ユーザーを作成
RUN useradd -m -s /bin/bash devuser
USER devuser
```

`USER devuser` は、ライブラリのインストールなどを済ませた後(Dockerfile の終盤)に置きます。devuser で追加インストールする場合は、`/opt/venv` の所有権を渡す(`chown`)か、インストールは root 側で済ませておきます。

Linux では、bind mount したファイルの権限のため、ホストとコンテナの UID / GID を合わせる必要があります。
Dev Container の `remoteUser` には、この UID / GID を調整する仕組みがあります。
なお `USER devuser` がコンテナ全体のデフォルトユーザーを変えるのに対し、`remoteUser` は主に VS Code Server やターミナルの実行ユーザーを指定する、という違いもあります。まずは root で動かし、必要になったら移行するのが無難です。

### プロキシ環境での設定（社内・学内ネットワークなど）

企業や大学などのネットワークでは、HTTP / HTTPS プロキシの設定が必要な場合があります。
以下のような環境変数を Dockerfile の最上部に追加することで対応できます。

```dockerfile
# プロキシ環境変数
ENV http_proxy=http://address:port/
ENV https_proxy=http://address:port/
ENV no_proxy=localhost,127.0.0.1
```

認証情報を含むプロキシ設定は Dockerfile に直接書かず、Dev Container の `containerEnv` や、認証情報には BuildKit の secret を使うのが安全です(ビルド引数もイメージ履歴に残りうるため、機密情報の保存先としては十分安全とは限りません)。
プロキシは環境ごとの差が大きいので、ここでは簡易例に留めます。

### 依存ライブラリを別ファイルにまとめる

上のテンプレートは、Dockerfile に直接ライブラリを並べた最低限の状態です。
開発を進めると依存は増えていくので、他の人と共有する段階では、依存の定義を別ファイルに切り出すと管理が楽になります。

pip なら `requirements.txt` にまとめて、一括でインストールします。

```dockerfile
COPY requirements.txt /workspace/requirements.txt
RUN pip install --no-cache-dir -r /workspace/requirements.txt
```

uv なら `pyproject.toml`(と、生成される `uv.lock`)にまとめ、`uv sync` で同期します。
`uv sync` は既定でプロジェクト直下に `.venv` を作るので、ここまでの `/opt/venv` に合わせるには `UV_PROJECT_ENVIRONMENT` を指定します。
また、Dev Container ではプロジェクトルートを bind mount で上書きするため、Dockerfile 内でソースを COPY せず、依存だけを入れる `--no-install-project` が自然です。

```dockerfile
ENV UV_PROJECT_ENVIRONMENT=/opt/venv
COPY pyproject.toml uv.lock ./
RUN uv sync --locked --no-install-project
```

`--locked` は、`uv.lock` が `pyproject.toml` と整合しているかを検証し、ずれていれば失敗します(再現性重視)。
`requirements.txt` は手軽ですが、`pyproject.toml` + `uv.lock` はバージョンをロックできるので、再現性を重視するならこちらが向いています。

## おわりに

自分の環境をなるべくクリーンに保ちながら、複数のプロジェクトへ対応するために、Dev Container + Dockerfile を使用した環境構築を行いました。
特に研究や開発、コンペを並行して進めたい方におすすめです。

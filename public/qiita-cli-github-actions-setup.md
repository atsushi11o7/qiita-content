---
title: Qiita CLI と GitHub Actions で執筆環境を整えて、リポジトリ連携で Qiita の記事を投稿する
tags:
  - Qiita
  - Docker
  - GitHubActions
  - devcontainer
  - QiitaCLI
private: false
updated_at: '2026-07-12T21:21:55+09:00'
id: 78bbea57bccbb43920c3
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
## はじめに

以前、Dev Container で Zenn の執筆環境を整え、GitHub 連携で記事を公開する仕組みを作りました。

https://zenn.dev/atsushi11o7/articles/zenn-publish-via-github-with-devcontainer

Zenn は GitHub リポジトリと連携でき、リポジトリへ push するだけで記事を公開できます。
一方で Qiita には Zenn のような GitHub 連携機能は無いと思い込んでいて、記事は Web エディタで書くものだと考えていました。

ところが、Qiita にも公式の Qiita CLI があり、記事を Markdown ファイルとしてローカルで執筆し、GitHub リポジトリで管理できることを知りました。
さらに GitHub Actions と組み合わせれば、main への push をトリガーに自動公開もできます。

そこで、Zenn のときと同じように「リポジトリで記事を管理し、push で公開する」執筆環境を、Qiita CLI と GitHub Actions で整えてみました。
この記事では、その構築手順をまとめます。

## この記事で作るもの

完成するのは、次のような Qiita 記事の執筆環境です。

- 記事を Markdown ファイルとして GitHub リポジトリ（`public/` 配下）で管理する
- Dev Container で執筆環境をコンテナ化し、どのマシンでも同じ状態をすぐ再現できる
- Qiita CLI で記事の新規作成・ローカルプレビュー・既存記事の取得を行う
- main への push をトリガーに、GitHub Actions が自動で Qiita へ公開する

公開までの流れは次のとおりです。

1. Dev Container 内で記事を書く（`npx qiita preview` で見た目を確認）
2. main へ push する
3. GitHub Actions が `qiita publish` を実行する
4. Qiita に記事が公開される

以降では、この環境をゼロから構築する手順を順に見ていきます。

## 前提

- GitHub アカウント（記事リポジトリの管理と GitHub Actions に使用）
- Qiita アカウント（記事の公開先。アクセストークンを発行する）
- Docker（Dev Container の実行に使用）
- VS Code（Dev Containers 拡張機能で開発コンテナを開く）

Node.js などの実行環境は Dev Container 内に用意するため、ホスト側へインストールする必要はありません。

## GitHub リポジトリを用意する

記事を管理する GitHub リポジトリを作成します。
中身は空で構いません（このあと Dev Container や Qiita CLI の設定を加えていきます）。
リポジトリ名は任意です（筆者は `qiita-content` としました）。

以降の手順は、このリポジトリをクローンし、そのルートで作業する前提で進めます。

## Dev Container を構築する

リポジトリのルートに `.devcontainer/` ディレクトリを作り、`Dockerfile` と `devcontainer.json` を置きます。

`.devcontainer/Dockerfile`:

```dockerfile
FROM node:24-trixie-slim

# ビルド中の対話プロンプト抑止 & npm のアップデート通知抑止
ENV DEBIAN_FRONTEND=noninteractive \
    npm_config_update_notifier=false

WORKDIR /workspace

# Claude Code のインストールスクリプト取得用 & GitHub CLI apt リポジトリ取得用
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# GitHub CLI
RUN curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
      | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
      | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
 && apt-get update \
 && apt-get install -y --no-install-recommends gh \
 && rm -rf /var/lib/apt/lists/*

USER node

# Claude Code
RUN curl -fsSL https://claude.ai/install.sh | bash

# Qiita CLI のプレビューサーバ用
EXPOSE 8888

CMD ["bash"]
```

`.devcontainer/devcontainer.json`:

```json
{
  "name": "Qiita Writing",
  "build": {
    "dockerfile": "Dockerfile"
  },
  "features": {
    "ghcr.io/atsushi11o7/devcontainer-features/base-utils:2": {
      "locale": "C.UTF-8",
      "timezone": "Asia/Tokyo"
    }
  },
  "workspaceFolder": "/workspace",
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",
  "runArgs": ["--env-file", "${localWorkspaceFolder}/.env"],
  "mounts": [
    "source=qiita-content-claude-config,target=/home/node/.claude,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/node/.claude",
    "LANG": "C.UTF-8",
    "LC_ALL": "C.UTF-8"
  },
  "postCreateCommand": "test -f package.json && npm install || true",
  "customizations": {
    "vscode": {
      "extensions": [
        "anthropic.claude-code",
        "yzhang.markdown-all-in-one",
        "DavidAnson.markdownlint"
      ]
    }
  },
  "remoteUser": "node",
  "updateRemoteUserUID": true,
  "forwardPorts": [8888]
}
```

この設定例には「Qiita CLI に必須なもの」と「筆者の好みで足しているもの」が混在しています。
必須なのは Node.js 入りのベースイメージ・`git`・ポート 8888 の公開の 3 点だけです。
それ以外（自作 Feature の `base-utils`、Claude Code とその拡張機能、Claude 設定用の named volume）は筆者の環境都合なので、不要なら削って構いません。
とくに Claude Code は本記事の手順には不要で、Qiita の執筆環境だけを作るなら入れなくても動きます。

ベースイメージは `node:24-trixie-slim` です。
Qiita CLI は Node.js で動くため、Node.js 入りのイメージを使います。

`features` の `ghcr.io/atsushi11o7/devcontainer-features/base-utils:2` は、`git` や `tzdata`、`locales` といった基本的なユーティリティをまとめてセットアップしてくれる Feature です（ここでは locale を `C.UTF-8`、timezone を `Asia/Tokyo` に指定しています）。
`git` はこの記事の手順で使うので、Feature を使わない場合は Dockerfile 側で `apt-get install -y git` を追加してください。

GitHub CLI（`gh`）を入れているのは、Secret 登録や GitHub の操作をコンテナ内から行うためです。
`gh` の認証は `.env` の `GH_TOKEN`（`repo` スコープの Personal Access Token）を使い、`runArgs` の `--env-file` でコンテナに読み込むことで、起動時に認証済みの状態になります。

`mounts` は named volume を `/home/node/.claude` に当て、`containerEnv` の `CLAUDE_CONFIG_DIR` と合わせて、Claude Code の認証・設定を再ビルドをまたいで保持します。

`EXPOSE 8888` と `forwardPorts: [8888]` は、Qiita CLI のプレビューサーバ（ http://localhost:8888 ）をホストのブラウザから開くためのものです。

## Qiita CLI を導入する

リポジトリのルートで Qiita CLI を導入します。

まず `package.json` を用意し、Qiita CLI をインストールします。

```bash
npm init -y
npm install @qiita/qiita-cli --save-dev
```

次に初期化します。

```bash
npx qiita init
```

これで次のファイルが生成されます。

- `qiita.config.json`（Qiita CLI の設定）
- `.github/workflows/publish.yml`（main への push で自動公開する GitHub Actions ワークフロー）
- `.gitignore`

続いて、Qiita のアクセストークンを発行します。
[Qiita のトークン発行ページ](https://qiita.com/settings/tokens/new) を開き、`read_qiita`（記事の取得）と `write_qiita`（記事の投稿・更新）の両方にチェックを入れて発行します。
このトークンは、ローカルでの `npx qiita login` と、後述の GitHub Actions の Secret 登録の両方で使います。

![qiita_access_tokens.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/52d4784e-8094-43e7-a1b2-8654406a9099.jpeg)

発行したトークンでログインします。

```bash
npx qiita login
```

プロンプトにトークンを貼り付けるとログインが完了し、トークンは `~/.config/qiita-cli/credentials.json` に保存されます。

![qiita_login.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/72a05461-ad8e-484a-acb5-bfb8b731d641.jpeg)

すでに Qiita に投稿済みの記事があれば、ローカルに取得しておきます。

```bash
npx qiita pull
```

取得した記事は `public/` 配下に Markdown ファイルとして保存されます。

## GitHub Actions で自動公開する

`qiita init` が `.github/workflows/publish.yml` を生成しています。
中身は次のとおりです。

```yaml
# Please set 'QIITA_TOKEN' secret to your repository
name: Publish articles

on:
  push:
    branches:
      - main
      - master
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  publish_articles:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: increments/qiita-cli/actions/publish@v1
        with:
          qiita-token: ${{ secrets.QIITA_TOKEN }}
          root: "."
```

`on.push.branches` のとおり、main（または master）への push で起動します。
`increments/qiita-cli/actions/publish@v1` が `qiita publish --all` を実行し、変更のある記事だけを Qiita に反映します。
`permissions: contents: write` は、publish 後に Qiita が採番した記事 ID をリポジトリへ書き戻すために必要です（詳しくは後述の「公開フロー」で触れます）。

このワークフローは `secrets.QIITA_TOKEN` でトークンを参照するので、リポジトリに同名の Secret を登録します。
登録するトークンは、先ほど発行した `read_qiita` + `write_qiita` のトークンと同じもので構いません。

```bash
gh secret set QIITA_TOKEN
```

プロンプトにトークンを貼り付けると登録されます（GitHub の Settings → Secrets and variables → Actions からでも登録できます）。

![qiita_token.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/4f1612e1-b8b9-4c2a-89f6-92a58a41d5fb.jpeg)

これで、main へ push するたびにワークフローが走り、Qiita に自動公開される状態になりました。

## 記事を書いてプレビューする

新しい記事は `npx qiita new` で作成します。

```bash
npx qiita new <ファイル名>
```

`public/<ファイル名>.md` が生成されるので、frontmatter（`title` や `tags`）を整えて本文を書きます。
`tags` は 1〜5 個必須で、0 個だと publish に失敗するので注意してください。

書いた記事は、プレビューサーバで見た目を確認できます。

```bash
npx qiita preview
```

ブラウザで http://localhost:8888 を開くと、ローカルの記事一覧とプレビューが表示されます。

![qiita_preview.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/058ec978-5f23-4a96-a813-db44f0756ddd.jpeg)

プレビュー画面の「画像をアップロードする」を開くと、Qiita の画像アップロードページに移動します。
そこへ画像をドラッグ & ドロップしてアップロードし、表示された URL をコピーします。
コピーした URL を `![alt](URL)` の形で Markdown に貼り付けます。
ローカルの相対パスや絶対パスは Qiita 上では表示されないため、画像はこの方法で取り込みます。

:::note warn
**Dev Container でプレビューが表示されないとき**
`qiita preview` は `qiita.config.json` の `host` 既定が `"localhost"` で、Node.js 17 以降は `localhost` が IPv6（`::1`）に解決されるため、IPv6 だけにバインドします。
VS Code は IPv4（`127.0.0.1`）でポートを転送するので、これだとブラウザに何も表示されません。
`qiita.config.json` の `host` を `"127.0.0.1"` に変えると IPv4 でバインドされ、表示されるようになります。
:::

## 公開フロー

記事を公開するときは、`public/` 配下の Markdown を main へ push します。
main への push をトリガーに GitHub Actions が走り、`qiita publish --all` で Qiita に反映されます。
手元で `npx qiita publish` を実行する必要はありません。

新規記事の場合、公開時に Qiita 側で記事の ID が採番されます。
この ID は Actions が frontmatter に書き戻し、`Updated by qiita-cli` というコミットで main に push し返します。
そのため、公開後はローカルの main を `git pull` して、採番された ID を取り込んでおきます。

main への push は即公開につながるため、書きかけの記事を main に置いておくことはできません。
公開したくない記事は、記事ごとにブランチを切って執筆し、完成したら PR 経由で main にマージすると、main を常に「公開済みの状態」に保てます。

## おわりに

「Qiita は Web エディタで書くもの」と思い込んでいましたが、Qiita CLI と GitHub Actions を使えば、リポジトリで記事を管理し、push で公開する環境を作れました。
途中、Dev Container 特有の IPv4 / IPv6 の問題や、画像はアップロードページ経由で URL を貼る必要があるなど、Qiita ならではのつまずきもありましたが、一度整えてしまえば快適に書けます。

なお、この環境には Claude Code も組み込んでいます。
記事の執筆を Claude Code に任せる仕組み（CLAUDE.md・スキル・エージェント）は、別記事で紹介する予定です。

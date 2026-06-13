---
title: Qiita CLI と GitHub Actions で執筆環境を整えて、リポジトリ連携で Qiita の記事を投稿する
tags:
  - Qiita
  - QiitaCLI
  - GitHubActions
  - DevContainer
  - Docker
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに

以前、Dev Container で Zenn の執筆環境を整え、GitHub 連携で記事を公開する仕組みを作りました。

https://zenn.dev/atsushi11o7/articles/zenn-publish-via-github-with-devcontainer

Zenn は GitHub リポジトリと連携でき、リポジトリへ push するだけで記事を公開できます。一方で Qiita には Zenn のような GitHub 連携機能は無いと思い込んでいて、記事は Web エディタで書くものだと考えていました。

ところが、Qiita にも公式の Qiita CLI があり、記事を Markdown ファイルとしてローカルで執筆し、GitHub リポジトリで管理できることを知りました。さらに GitHub Actions と組み合わせれば、main への push をトリガーに自動公開もできます。

そこで、Zenn のときと同じように「リポジトリで記事を管理し、push で公開する」執筆環境を、Qiita CLI と GitHub Actions で整えてみました。この記事では、その構築手順をまとめます。

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

記事を管理する GitHub リポジトリを作成します。中身は空で構いません（このあと Dev Container や Qiita CLI の設定を加えていきます）。リポジトリ名は任意です（筆者は `qiita-content` としました）。

以降の手順は、このリポジトリをクローンし、そのルートで作業する前提で進めます。

## Dev Container を構築する

<!-- メモ: Dockerfile（Node.js ベース + gh CLI など）と devcontainer.json。実ファイル .devcontainer/ を参照して書く。Claude Code による執筆サポートもこの環境に含まれるが、詳細は別記事に回す旨をここで一言触れる -->

## Qiita CLI を導入する

<!-- メモ:
- npm install @qiita/qiita-cli
- npx qiita init（生成物: qiita.config.json / .github/workflows/publish.yml / .gitignore）
- アクセストークン発行（read_qiita + write_qiita の両方）【画像アップロード: _work/837884E2...（トークン発行画面）】
- npx qiita login【画像アップロード: _work/15F1101D...（login のターミナル出力）】
- npx qiita pull（既存記事をローカルに取得）
-->

## GitHub Actions で自動公開する

<!-- メモ:
- publish.yml の中身（main/master への push がトリガー、qiita publish --all、permissions: contents: write、publish 後に id を main へ書き戻す）
- QIITA_TOKEN をリポジトリ Secret に登録（gh secret set QIITA_TOKEN）【画像アップロード: _work/3E8AAF48...（gh secret set 成功）】
-->

## 記事を書いてプレビューする

<!-- メモ: npx qiita new <name> で雛形作成、npx qiita preview（http://localhost:8888）でプレビュー、画像はプレビュー画面にドラッグ & ドロップして Qiita にアップロード（ローカルパスは使えない） -->

## 公開フロー

<!-- メモ: ブランチ → PR → squash merge → main への push → Actions が qiita publish。新規記事は publish 時に id が採番され、Actions が main に書き戻す → ローカルは git pull で取り込む -->

## おわりに

<!-- メモ:
- この記事自体、この環境で書いた
- Claude Code による執筆サポート（CLAUDE.md + スキル + エージェント）は別記事で紹介予定（予告リンク）
-->

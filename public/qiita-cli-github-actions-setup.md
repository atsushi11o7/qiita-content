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

<!-- メモ:
- 以前 Dev Container で Zenn の執筆環境を整えた（前記事リンク: https://zenn.dev/atsushi11o7/articles/zenn-publish-via-github-with-devcontainer）
- Qiita は Zenn のような GitHub 連携機能が無いと思っていた
- 実は Qiita CLI でリポジトリ連携・ローカル執筆ができると知った → 執筆環境のリポジトリを整えてみた
- この記事で扱う範囲（執筆環境の構築。Claude Code による執筆サポートは別記事）
-->

## この記事で作るもの

<!-- メモ: 完成形の概要を先に提示。public/ に記事 Markdown を置き、Dev Container で執筆環境を再現、Qiita CLI でローカル執筆・プレビュー、main への push で GitHub Actions が自動公開、という全体像を箇条書きor図で -->

## 前提

<!-- メモ: 必要なもの（GitHub アカウント / Qiita アカウント / Docker / VS Code）。Node.js はコンテナ内で用意するので不要 -->

## GitHub リポジトリを用意する

<!-- メモ: 記事管理用のリポジトリを作成。Zenn は「アプリ連携」で GitHub と繋ぐが、Qiita にはその機能が無く、Qiita CLI + GitHub Actions で同等のことを実現する、という違いを説明 -->

## Dev Container を構築する

<!-- メモ: Dockerfile（Node.js ベース + gh CLI など）と devcontainer.json。実ファイル .devcontainer/ を参照して書く。※ AI 執筆補助（Claude Code）は別記事なのでここでは触れない -->

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

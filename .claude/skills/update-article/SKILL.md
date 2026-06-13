---
name: update-article
description: 既に公開済みの Qiita 記事を修正するために `update/<name>` ブランチを切る。main から呼ばれ、対象記事を特定して update ブランチを作成し、ユーザーに編集を促すところまでを担う。main への反映（commit → PR → squash merge）は `/publish-article` が担う。「公開済み記事を更新したい」「修正したい」「リライトしたい」「update」と言われたら使う。新規記事の初公開には使わない（その場合は `/new-article`）。
---

# Qiita 記事の更新開始

## 役割

既に Qiita に公開済み（`id` が採番済み）の記事を修正するための **`update/<name>` ブランチを切る** スキル。編集後の main への反映は `/publish-article` が `update/*` ブランチを検出して処理するため、本スキルは反映を行わない（`/publish-article` と役割を分担する）。

ブランチ運用は CLAUDE.md の「Git フロー」を参照。

## Step 1: 現在のブランチを判定

`git rev-parse --abbrev-ref HEAD` で現在のブランチを取得する。

- `main` → そのまま続行
- `update/<name>` → 既に update ブランチにいる。編集が済んでいるなら `/publish-article` を案内して中断
- `new/<name>` → 新規記事の作業中。`/publish-article`（新規公開）を案内して中断
- それ以外 → ユーザーに状況を確認する

## Step 2: 対象記事の特定

更新したい記事の name をユーザーから受け取る。明示されていなければ、`public/` 内の記事のうち **`id` が採番済み（= 公開済み）** のものを一覧提示して選んでもらう。

`id` が空の記事は未公開なので、更新ではなく新規公開（`/new-article` → `/publish-article`）の対象であることを案内する。

## Step 3: 作業ツリーの確認

`git status --porcelain` を確認する。未コミット変更がある場合は、編集対象の記事ファイルかどうかをユーザーに確認する（記事ファイルなら次の `git switch -c` で update ブランチに持ち越せる。関係ないファイルが混じっていれば対応を相談する。勝手に stash したり捨てたりしない）。

## Step 4: main を最新化してブランチを切る

```bash
git switch main
git pull --ff-only
git switch -c update/<name>
```

- `git switch -c` でブランチを切ると未コミット変更は新しいブランチへ持ち越される
- 同名のローカルブランチが残っていると `git switch -c` がエラーになる。その場合は中断してユーザーに状況を報告し、name の確認か旧ブランチの整理（`git branch -D update/<name>`）を相談する

## Step 5: 最新の本文との同期を確認

Qiita 上で記事を直接編集した可能性がある場合（ローカルが古いと publish が「ローカルが古い」エラーで止まる）、編集前に最新を取得するか確認する。

```bash
npx qiita pull <name>
```

ローカルでしか編集していないことが明らかなら省略してよい。

## Step 6: 案内して終了

> `update/<name>` ブランチを作成しました。`public/<name>.md` の修正を進めてください。
> 編集が完了したら `/publish-article` を呼ぶと、main へ反映（commit → PR → squash merge → Actions が再公開）まで進みます。

本スキルはここで終了する。**反映は行わない**（編集を挟むため、`/publish-article` が担う）。

## 重要な制約

- ユーザーの**明示的な承認**なしに状態を変える操作を行わない
- `git push --force` などの破壊的操作は絶対に使わない
- `id` / `updated_at` を手で書き換えない（Qiita CLI / Actions の管理対象）
- 反映フロー（commit / push / PR / merge）はここでは行わない（`/publish-article` の責務）

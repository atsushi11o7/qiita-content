---
name: publish-article
description: 現在の作業ブランチ（`new/<name>` 新規 or `update/<name>` 更新）の内容を main へ反映し、GitHub Actions の `qiita publish --all` を発火させる。commit → push → PR 作成 → squash merge → main 復帰までを担う。「公開して」「publish」「Qiita に出す」「初公開」「修正を反映して」「update を反映」と言われたら使う。ブランチを切る作業は担わない（新規は `/new-article`、更新は `/update-article`）。
---

# Qiita 記事の main 反映（新規公開・更新反映の共通）

## 役割

作業ブランチの内容を PR 経由で main にマージし、main への push で Actions（`.github/workflows/publish.yml`）に `qiita publish --all` を実行させて Qiita に反映する。**新規公開（`new/<name>`）と更新反映（`update/<name>`）の両方**をこのスキルで扱う（反映処理が共通のため）。

- ブランチ作成は担わない：新規は `/new-article`、更新は `/update-article` が事前に作る
- 本文の校正は `/pre-publish-check`、内容レビューは `content-reviewer` で済ませておく

CLAUDE.md の「Git フロー」に従う。**`main` 上での直接公開は行わない。**

## Step 1: 現在のブランチからモードと対象記事を判定

`git rev-parse --abbrev-ref HEAD` で現在のブランチを取得し、モードを決める。

- `new/<name>` → **新規公開モード**
- `update/<name>` → **更新反映モード**
- `main` → 中断。「作業ブランチに切り替えてからやり直してください（新規は `/new-article`、更新は `/update-article`）」と案内
- それ以外のブランチ名 → ユーザーに状況を確認

対象記事は `public/<name>.md`。`<name>` はブランチ名から導出する。以降、モード差がある箇所だけ **［新規］／［更新］** と明記する。

## Step 2: pre-publish-check の確認

ユーザーに以下を確認する。

> 「`/pre-publish-check` を実行済みですか？ まだなら先に実行することを推奨します。」

未実行なら強く推奨する（ただしユーザーが「不要」と判断すれば尊重する）。

## Step 3: 反映意思の最終確認

影響を伝えた上で、明示的な承認を得る。

［新規］

> 以下の記事を新規公開します。よろしいですか？
>
> - タイトル：<title>
> - ファイル名：<name>（public/<name>.md）
> - 公開範囲：<private が false なら「公開」/ true なら「限定共有」>
> - ブランチ：new/<name>（squash merge 後に削除されます）
>
> ※ 記事 URL（id）は publish 時に採番され、Actions が main に書き戻します。

［更新］

> 以下の記事の更新を main に反映します（Actions が再公開します）。よろしいですか？
>
> - タイトル：<title>
> - ファイル名：<name>（public/<name>.md）
> - 公開済み URL：https://qiita.com/<username>/items/<id>
> - ブランチ：update/<name>（squash merge 後に削除されます）

## Step 4: 変更ファイルの事前確認

`git status` を実行し、コミット予定のファイルを把握する。

- 対象記事 Markdown：`public/<name>.md`
- 想定外のファイル変更（CLAUDE.md / 他の記事など）があれば、ユーザーに確認する
- 画像はローカルに持たない運用のため、`public/` 配下に画像ファイルが紛れていないか確認する

## Step 5: frontmatter の確認

- ［新規］`private` を確定する（公開なら `false`、限定共有なら `true`）。`id` は空が期待値（publish 時に採番される）。既に `id` が入っていれば矛盾を報告し、`/update-article` 経由の更新では？と確認する
- ［更新］`id` が採番済みの値で入っていることを確認する。空なら新規の可能性があるので新規公開モードを案内して中断。`private` が意図せず変わっていないか確認する（意図的な変更なら尊重する）
- `id` / `updated_at` は手で書き換えない

## Step 6: コミットとプッシュ

```bash
git add public/<name>.md
git commit -m "<commit message>"
git push -u origin HEAD
```

- ブランチ上の作業コミットのメッセージは squash merge で消えるので、任意の作業メッセージでよい（公開・更新の本タイトルは Step 7 の PR タイトルで付ける）
- 既に upstream が設定されているなら `-u origin HEAD` は省略可
- 既に複数の commit が積まれていてもそのまま push する（squash merge で 1 つにまとまる）

## Step 7: PR を作成して squash merge

PR タイトルが squash merge で main に残るコミット件名になる。`--body` には要点を 1〜2 行で書く（［新規］記事の概要 /［更新］変更点）。

```bash
gh pr create --title "<PR タイトル>" --body "<要点>"
gh pr merge --squash --delete-branch
```

- PR タイトルは ［新規］`Publish: <記事タイトル>` /［更新］`Update: <記事タイトル>`（記事タイトルは frontmatter の `title`）
- `--delete-branch` で GitHub 上のブランチが自動削除される
- merge が CI 失敗や保護ルールで止まる場合は中断してユーザーに状況を報告する

## Step 8: ローカルを main に戻す

```bash
git switch main
git pull --ff-only
git branch -D <作業ブランチ>
```

- ローカルブランチは squash merge 後も「未マージ」扱いになるため `-D`（強制削除）を使う
- ［新規］Actions が publish 後に Qiita 採番の `id` を main に commit（`Updated by qiita-cli`）で書き戻す。Actions 完了まで数十秒かかるため、`git pull` でまだ反映されていなければ少し待って再度 `git pull` するよう案内する
- ［更新］`updated_at` が書き戻される場合があるので、必要に応じて `git pull` で取り込む
- 削除が失敗するなど想定外があればユーザーに報告

## Step 9: 反映確認の案内

> GitHub Actions（Publish articles）が `qiita publish` を実行します。
> 実行状況は `gh run watch` または GitHub の Actions タブで確認できます。

- ［新規］「公開後、frontmatter に `id` が書き戻されるのでローカル main を `git pull` してください」
- ［更新］「https://qiita.com/<username>/items/<id> で更新を確認してください」

反映されない場合：

- GitHub の Actions タブでワークフローのログを確認
- tags が 1 個以上あるか（0 個だと publish が失敗する）を確認
- ［更新］「ローカルが古い」エラーが出ていれば、`npx qiita pull <name>` で最新を取得してから再編集

## 重要な制約

- ユーザーの**明示的な承認**なしに状態を変える操作（`private` 書き換え、commit、push、PR 作成、merge）を行わない
- `git push --force` などの破壊的操作は絶対に使わない
- `id` / `updated_at` を手で書き換えない（Qiita CLI / Actions の管理対象）
- ブランチ作成は担わない（新規は `/new-article`、更新は `/update-article`）

---
name: new-article
description: Qiita の新規記事の雛形を作成する。記事の概要をヒアリングして name（ファイル名）/title/tags を確定し、new/<name> ブランチを切ってから npx qiita new で生成し、frontmatter を揃える。「新しい記事を書きたい」「記事を作って」「Qiita の記事を始める」「記事の雛形を用意して」のように Qiita の新規執筆を始めたい意図が見えたら必ず使う。
---

# Qiita 新規記事作成

各値の制約は CLAUDE.md の「記事執筆のルール」、ブランチ運用は CLAUDE.md の「Git フロー」を参照。本スキルは **決め方の指針** と **実行手順** に集中する。

## Step 1: 作業ツリーの事前確認

ブランチ操作の前に以下を確認する。問題があればユーザーに状況を伝えてから対応を相談する（勝手に stash したり捨てたりしない）。

- `git status --porcelain` が空か（未コミット変更がないか）
- 現在のブランチが `main` か（違う場合はなぜそうなっているか確認）

## Step 2: 記事の概要を確認

ユーザーから以下を聞き取る（分かるものから順に）。

- 何について書くか（技術トピック、解決したい問題）
- 想定読者（初心者 / 中級者 / 同じ技術を使う開発者など）

## Step 3: name / title / tags を提案

ユーザーの確認を取りながら値を確定する。各フィールドの制約は CLAUDE.md 参照。

### name（ファイル名）の決め方の指針

- 内容を表す英小文字・数字・ハイフンの組み合わせを推奨（例：`nextjs-app-router-error-handling`）
- **Qiita の URL には影響しない**（URL は publish 後に採番される `id`）。そのため後からリネームしても URL は変わらない
- ただし `new/<name>` ブランチ名にも使うので、分かりやすく一意な名前にする

### title の決め方の指針

- 検索キーワードを含む
- 煽りすぎず内容を正確に表す

### tags の決め方の指針

個数・空タグなどの制約は CLAUDE.md の frontmatter 表を参照（**1〜5 個必須**）。本スキルでは選び方に集中する。

- Qiita で実在する人気タグを優先する。例：`React` `TypeScript` `AWS` `Docker` `Python` `Go` `Node.js` `Vue.js` `Rails` `Next.js`
- 大文字小文字は表示どおりに書く（Qiita のタグ表記に合わせる）

## Step 4: ブランチを切る

name が確定したら main を最新化して `new/<name>` ブランチを切る。

```bash
git switch main
git pull --ff-only
git switch -c new/<name>
```

同名のローカルブランチが残っていると最後の `git switch -c` がエラーになる。その場合は中断してユーザーに状況を報告し、name の再考か旧ブランチの整理（`git branch -D new/<name>`）を相談する。

## Step 5: 記事ファイルを生成

事前確認：

- `public/<name>.md` が既に存在しないか
- リポジトリルート（`package.json` がある場所）にいるか

問題なければ実行：

```bash
npx qiita new <name>
```

`public/<name>.md` が生成される。生成直後のデフォルトは `title` に `<name>`、`tags` に空タグ 1 個（`- ''`）が入り、本文は `# new article body`（H1）になっている（`id` は空、publish 時に採番される）。次の Step で実値に置き換える。

## Step 6: frontmatter と本文の初期化

生成されたファイルを開き、Step 3 で確定した値を反映する（CLAUDE.md の制約に従う）。

- `title`：デフォルトの `<name>` を確定したタイトルに置き換える
- `tags`：デフォルトの空タグ `- ''` を確定したタグに置き換える
- `private`：原則 `false`（公開）。Qiita 上で限定共有しながら仕上げたい場合のみ `true`
- `id` / `updated_at`：**触らない**（Qiita CLI が管理）
- 本文：デフォルトの `# new article body`（H1）を削除する。本文の見出しは `##` から始める（CLAUDE.md のルールで `#` は使わない）

## Step 7: 完了報告

- 生成された記事ファイルパス（`public/<name>.md`）を伝える
- 現在 `new/<name>` ブランチにいることを明示する
- 画像を入れる場合は `npx qiita preview`（http://localhost:8888）の画面にドラッグ & ドロップしてアップロードする旨を案内する（ローカルパスは使わない）
- 次は `/outline-article` で構成（骨組み）を立て、`/draft-section` で各節を書くよう案内する
- 公開時は `/publish-article` を使うよう案内する（commit → PR → squash merge → Actions が `qiita publish` まで進む）

## 重要な制約

- name はユーザー承認を取ってから確定する（ブランチ名にも波及するため）
- 骨組み（アウトライン）作成は `/outline-article`、節単位の執筆は `/draft-section` の責務（本スキルは雛形と frontmatter の初期化まで）
- `public/` 直下にしか記事を置かない（サブディレクトリは Qiita CLI が認識しない）

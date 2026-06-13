# このリポジトリについて

Qiita の記事を Qiita CLI + GitHub Actions で管理・公開するためのリポジトリです。Dev Container で執筆環境を構築しています。

## ディレクトリ構成

- `public/` — 記事の Markdown ファイル（1 記事 = 1 ファイル）。**Qiita CLI はこの直下フラットのみ**を対象にする（サブディレクトリは不可）
- `_work/` — 執筆用の作業置き場（参考資料・画像の一時置き場・執筆指示書など）。**Git 管理外**（`.gitignore` 済み）で Qiita にも公開されない
- `.devcontainer/` — Dev Container 設定
- `.github/workflows/publish.yml` — main への push で `qiita publish --all` を実行する自動公開ワークフロー
- `.claude/skills/` — このリポジトリ専用のスキル定義
- `.claude/agents/` — このリポジトリ専用のエージェント定義
- `qiita.config.json` — Qiita CLI 設定

## Qiita CLI の基本

- 認証：ローカルは `npx qiita login`（トークンは `~/.config/qiita-cli/credentials.json` に保存）。CI は Secret `QIITA_TOKEN` を使う
- 新規作成：`npx qiita new <name>` → `public/<name>.md` を生成
- プレビュー：`npx qiita preview`（http://localhost:8888）で表示確認する
- 取得：`npx qiita pull`（Qiita 上の記事を `public/` に同期）
- 公開・更新：`npx qiita publish <name>` / `--all`。ただし本リポジトリでは**原則 main への push 経由で Actions に任せる**（下記「Git フロー」参照）

## 記事執筆のルール

このリポジトリの全スキル・全エージェントはこのルールに従う。

### 文体・表記

- **「です・ます調」で統一**
- 半角英数字の前後には半角スペースを入れる
  - ✅ 「Qiita CLI を使う」
  - ❌ 「Qiita CLIを使う」
- カタカナ語・固有名詞の表記を統一する
  - **Qiita**（qiita ではなく。ツール名 `qiita-cli` や設定キーは小文字のままで OK）
  - **GitHub**（Github / github ではなく）
  - **Dev Container**（DevContainer / devcontainer ではなく。ファイル名や設定キーは除く）
  - **VS Code**（VSCode / vscode ではなく）

### Markdown 記法

- 本文の見出しは `##` から始める（`#` は frontmatter の `title` が担うので使わない）
- コードブロックには必ず言語を指定する（` ```bash`、` ```json`、` ```dockerfile` など）
- 画像は **Qiita にアップロードしたホスト URL** で参照する
  - 画像は Qiita の画像アップロードページ（https://qiita.com/settings/uploaded_images）でアップロードし、得られた URL を `![alt](URL)` で貼る（プレビュー画面の「画像をアップロードする」からこのページへ移動できる。プレビュー自体に画像アップロード機能はない）
  - **ローカルの相対パス・絶対パス（`./images/...` や `/images/...`）は Qiita では表示されない**ので使わない
  - 一時的な画像素材は `_work/` に置く（リポジトリにも Qiita にも載らない）
- Qiita 独自記法（`:::note info` などの Note ブロック）を使ってもよい

### ファイル名（記事キー）

- 記事ファイル名 `public/<name>.md` の `<name>` は **ローカルでの識別子**であり、**Qiita の記事 URL には影響しない**
  - Qiita の記事 URL は `https://qiita.com/<user>/items/<id>` の `<id>` 部分。`<id>` は `qiita publish` 時に Qiita 側が採番し、frontmatter の `id` に書き戻される
  - そのため `<name>` は後からリネームしても URL は変わらない（pull した既存記事はファイル名が `<id>` のハッシュだが、分かりやすい名前にリネームしてもよい。`id` で紐付くため Qiita 側には影響しない）
- `<name>` は記事内容を表す英小文字・数字・ハイフンを推奨（例：`qiita-cli-setup`）。ブランチ名にも使う（下記「Git フロー」）

### frontmatter

| フィールド | 制約 |
| --- | --- |
| `title` | 記事タイトル（理想は 30〜50 字、長くしすぎない） |
| `tags` | **1〜5 個必須**（0 個だと publish できない）。Qiita の実在タグを優先。大文字小文字は表示どおり（例：`Docker`、`ClaudeCode`） |
| `private` | 真偽値。`false` = 公開、`true` = 限定共有（シークレット URL のみ） |
| `id` | Qiita CLI が管理。**手で触らない**（publish 後に採番される。消すと別記事として二重投稿される） |
| `updated_at` | Qiita CLI が管理。手で触らない |
| `organization_url_name` | 通常 `null`。Organization 投稿時のみ |
| `slide` | スライドモード。通常 `false` |
| `ignorePublish` | `true` にすると `qiita publish` の対象から外れる（公開したくない下書きを `public/` に置いたままにできる）。通常 `false` |

`private` の値は `/publish-article` 経由で扱い、frontmatter を手で書き換えない。

`npx qiita new` の生成直後は `title` に `<name>`、`tags` に空タグ 1 個（`- ''`）が入り、本文は `# new article body`（H1）になっている。執筆前に `title` / `tags` を実値に置き換え、本文の `#` 行を削除する（`#` は使わないルールのため）。

## ワークフローと対応スキル

| フェーズ | スキル / エージェント | 補足 |
| --- | --- | --- |
| 新規作成 | `/new-article` | name/title/tags を確定して `new/<name>` ブランチで雛形生成 |
| 構成（アウトライン） | `/outline-article` | 見出し構造と各節の方針メモ（骨組み）を立てる |
| 執筆 | `/draft-section` | 骨組みの見出し + メモから節の下書きを生成（任意。手書きでも可） |
| プレビュー | （なし） | `npx qiita preview`（http://localhost:8888）で表示確認 |
| 機械チェック | `/pre-publish-check` | CLAUDE.md ルールへの**適合**を機械的に判定（frontmatter、tags 数、表記揺れ、画像 URL など） |
| 編集レビュー | Agent: `content-reviewer` | **内容の質**を編集者視点で評価（論理展開、構成、読者適合、文体の読みやすさなど） |
| 更新の開始 | `/update-article` | 公開済み記事を修正するため `update/<name>` ブランチを切る（反映は `/publish-article`） |
| main へ反映（新規公開・更新反映の共通） | `/publish-article` | `new/<name>`（新規）でも `update/<name>`（更新）でも、commit → push → PR → squash merge（Actions が `qiita publish`） |

ユーザーから「新しい記事を書きたい」「公開前にチェックして」「Qiita に公開して」「修正を反映して」のような依頼があれば、コマンドを自分で組み立てるのではなく対応するスキル / エージェントを呼び出す。

## Git フロー

main への push をトリガーに GitHub Actions（`.github/workflows/publish.yml`）が `qiita publish --all` を実行し、Qiita に反映される。そのため **main は常に「Qiita に公開済みの状態」と一致**させ、書きかけの記事は main に載せない（ブランチ上で執筆する）。

| シナリオ | ブランチ | フロー |
| --- | --- | --- |
| 新規記事 | `new/<name>` | `/new-article` がブランチを切る → 執筆 → `/publish-article` で PR → squash merge → Actions が公開 |
| 公開後の大きい修正（追記、節追加、構成変更、コードサンプルの大幅変更など） | `update/<name>` | `/update-article` で `update/<name>` 作成 → 修正 → `/publish-article` で PR → squash merge → Actions が再公開 |
| 微小な修正（typo、1〜2 行の文言調整など） | `main` 直 | commit → push（Actions が再公開） |
| リポジトリ運用（Dev Container、CLAUDE.md、スキル定義など記事以外） | `chore/<slug>` | `chore/<slug>` でブランチ → PR → squash merge（微小な修正のみ `main` 直を許容） |

ルール：

- ブランチ名は `<type>/<slug>` で統一する：記事は `new/<name>` / `update/<name>`（`<name>` は記事ファイル名と一致）、リポジトリ運用は `chore/<slug>`。微小な修正のみ `main` 直を許容
- merge は squash merge を基本とする（履歴を「作業 1 つ = コミット 1 つ」に保つ）
- merge 後はブランチを削除する（GitHub 側・ローカル両方）
- **コミットメッセージはタイトル 1 行のみ・英語の命令形・〜72 文字・本文なし・フッターなし**（例：`Add outline-article skill`）。記事の公開・更新だけは例外で、squash merge で main に残る **PR タイトル**を `Publish: <記事タイトル>` / `Update: <記事タイトル>` とする（ブランチ上の作業コミットは squash で消えるので任意のメッセージでよい）
- 新規記事を main にマージして Actions が publish すると、Qiita が採番した `id` を Actions が main に commit（`Updated by qiita-cli`）で書き戻す。**マージ後にローカル main を `git pull` して取り込む**
- 「大きい / 微小」の境界が曖昧なときはブランチ側に倒す（事故が起きにくい）

### 複数記事の並行執筆

- 記事 1 本につき 1 ブランチ（新規は `new/<name>`、更新は `update/<name>`）。`git switch` で行き来して 1 本ずつ編集する
- 別の記事を始めるときは、作業中ブランチで変更を commit し、`git switch main` で main に戻ってから `/new-article` または `/update-article` を呼ぶ（両スキルは main からの呼び出しを前提とするため）
- 公開は独立。`/publish-article` で 1 本ずつ main にマージできる

## 環境

Dev Container 内で作業するのが前提。VS Code で `Reopen in Container` で起動する。

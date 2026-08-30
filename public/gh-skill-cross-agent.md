---
title: Claude Code と Codex で使い回せる Skill 環境を整えた話
tags:
  - ClaudeCode
  - codex
  - SKILLS
  - GitHub
  - GH
private: false
updated_at: '2026-08-30T23:20:45+09:00'
id: 85e409cbc4e6ea22f6c9
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

最近、プライベートで Claude Code に加えて Codex も併用する機会が増えてきました。

これまではプロジェクトごとに Skill を作っていましたが、同じような Skill を毎回イチから作り直すのは正直面倒です。そこで一度、汎用的な Skill を管理するリポジトリを作り、`gh skill install` でプロジェクトに取り込めるようにしました。

## Skill の置き場所が意外とややこしい

Claude Code は `.claude/skills/` を、Codex は `.agents/skills/` を参照します。

| Location | Codex | Claude Code |
| --- | --- | --- |
| Project | `.agents/skills/` | `.claude/skills/` |
| User | `~/.agents/skills/` | `~/.claude/skills/` |

https://learn.chatgpt.com/docs/build-skills

https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/

`.agents/skills/` は Codex だけでなく GitHub Copilot・Cursor・Gemini CLI なども project scope で共有しています。Claude Code だけがそこに参加せず、`.claude/skills/` という独自の場所を使う例外的な存在です。

そもそも Skill の置き場所には、業界で統一された「標準仕様」と呼べるものはまだ無いようです。SKILL.md という緩いファイル形式が、複数のツールでたまたま互換になっている段階のようです。

## gh skill について

`gh skill` は 2026 年 4 月に GitHub CLI に追加された機能です。公式のリポジトリに限らず、個人の public リポジトリでも同じように `gh skill install` の対象にできます。取り込み先はエージェントごとに `--agent`、project / user は `--scope` で指定します。

対象にするために必要な構成はシンプルで、リポジトリのどこかに `skills/<name>/SKILL.md` というフォルダを用意するだけです。

```text
<repo>/
└── skills/
    └── <skill-name>/
        └── SKILL.md
```

## Skill リポジトリを作った

そこで、この構成に沿って Skill を管理するリポジトリとして [agent-skills](https://github.com/atsushi11o7/agent-skills) を作りました。

```text
agent-skills/
├── skills/
│   ├── git-commit/
│   │   └── SKILL.md
│   └── ...
└── README.md
```

試しに `git-commit` という Skill を作ってみました。diff の内容から Chris Beams スタイルのコミットメッセージを生成し、確認したうえでコミットまで行う Skill です。Claude Code や Codex には Skill を作るための Skill（skill-creator）が用意されているので、それを使って書きました。

<details><summary>git-commit/SKILL.md の中身</summary>

````markdown
---
name: git-commit
description: 実際のdiffを確認したうえで、Chris Beamsの標準的なルール（命令形・50文字以内のsubject、必要な時だけbody）に沿った英語のコミットメッセージを作成し、確認を取ってからコミットする。pushは行わない（コミットまでがこのスキルの範囲）。ユーザーがコミットメッセージの作成を求めた時、「コミットして」と言った時、または変更がコミット可能な状態にある時に使う。会話の内容だけで推測せず、必ず実際のdiffを見て書く。プロジェクト固有のルール（AGENTS.mdなど）がある場合はそちらを優先する。
---

# git-commit

ベースはChris Beamsの「7つのルール」（出典: cbea.ms/git-commit）。ただしfooterは使わない。ルールの中身は手順5〜7にすべて書き下してあるので、このページを検索・取得しに行く必要はない。

このスキルはcommitまでを担当する。pushは行わない — pushはcommitより取り消しにくく、リモートや他の開発者・CIに影響する別種の操作なので、意図的に範囲外にしている。pushが必要な場合はユーザーに確認してから別途行うこと。

## 手順

1. **プロジェクト固有のルールを確認する**

   コミットメッセージの規約（`type(scope):`のようなConventional Commits形式にするか、bodyを常に付けるか付けないか、footerでIssueを閉じるかなど）はプロジェクトによって違う。まずAGENTS.md・CLAUDE.mdなど、リポジトリ内の規約ファイルを確認する。書かれていればそちらに従い、以降のステップはこのファイルのルールで上書きする。何も書かれていなければ、以下をデフォルトのルールとして使う。

2. **実際の変更を確認する**

   ```bash
   git status
   git diff
   ```

3. **明示的にstageする**

   ```bash
   git add <file1> <file2> ...
   ```

   `git add -A` は使わない。この変更に関係するファイルだけを選んで、無関係な変更が紛れ込まないようにする。

4. **1コミット=1論理変更。** diffに無関係な変更が複数混ざっている場合は、まとめて1つのメッセージにせず、ファイルを分けてそれぞれコミットする。

5. **subjectを書く（デフォルトルール）**

   - 命令形（英語）: "Add", "Fix", "Update" など（"Added", "Adds" ではない）
   - 大文字始まり
   - 末尾にピリオドを付けない
   - 50文字以内
   - typeプレフィックス（`feat:`など）は、プロジェクトの規約で指定されていない限り付けない

6. **bodyは必要な時だけ（デフォルトルール）**

   変更が自明でない場合（なぜその変更が必要だったか、背景を知らないと分からない場合）は、subjectと空行を挟んでbodyを書く。72文字で折り返し、「何を」「なぜ」を書く（「どうやって」はdiffが語るので書かない）。typoの修正や1行だけの自明な変更ならbodyは省略する。

7. **footerは付けない（デフォルトルール）**

   `Co-Authored-By`、`Closes #123`、`BREAKING CHANGE`などは、プロジェクトの規約で指定されていない限り書かない。

8. **確認してからコミット**

   メッセージをユーザーに見せる。OKが出たら:

   ```bash
   git commit -m "Subject here"
   ```

   bodyがある場合:

   ```bash
   git commit -m "Subject here" -m "Body here"
   ```

9. **pushはしない**

   コミットが終わったらそこで完了。リモートへの反映が必要な場合は、その旨をユーザーに伝え、別途明示的な指示を待つ。

## 例（デフォルトルールに従う場合）

**body不要な例:**
- `Fix BM25 tokenization`
- `Remove unused embedding cache logic`

**bodyが必要な例:**
```
Switch retriever default to dense embeddings

BM25 was missing semantically similar but lexically
different queries. Dense retrieval handles this better
for the current dataset.
```
````

</details>

## gh skill install でプロジェクトに入れる

実際のインストールコマンドはこうです。

```bash
# Claude Code に取り込む
gh skill install atsushi11o7/agent-skills git-commit --agent claude-code

# Codex に取り込む
gh skill install atsushi11o7/agent-skills git-commit --agent codex

# リポジトリ内の Skill をすべて取り込む
gh skill install atsushi11o7/agent-skills --all --agent claude-code
```

`--agent` は1コマンドにつき1エージェントのみの指定なので、両方に取り込みたい場合は2回実行します。実行後、プロジェクト側はこのような構成になります。

```text
my-project/
├── .claude/
│   └── skills/
│       └── git-commit/
├── .agents/
│   └── skills/
│       └── git-commit/
├── AGENTS.md
└── CLAUDE.md
```

同名の Skill が project scope と user scope の両方に存在すると衝突することがあるので、Skill 名はユニークにしておくと安全です。

## CLAUDE.md と AGENTS.md は共通化

Claude Code と Codex の両方で使いたい場合は、それぞれ `--agent` を指定して `gh skill install` を実行し、`.claude/skills/` と `.agents/skills/` にコピーします。今回は各ツールが想定する配置をそのまま使い、gh skill install だけでセットアップできる構成を選びました。

一方、CLAUDE.md と AGENTS.md はそうしなくて済みます。CLAUDE.md の先頭に `@AGENTS.md` と書くだけです。

```markdown
@AGENTS.md
```

## おわりに

agent-skills リポジトリを起点にして `gh skill install` で配布することで、新しくプロジェクトを作ったときでも、汎用的な Skill をエージェントの種類に関係なく簡単に導入できるようになりました。

今は数個の Skill しかありませんが、これから少しずつ追加していこうと思います。

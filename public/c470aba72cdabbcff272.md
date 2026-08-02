---
title: ローカルLLMはもう「とりあえずOllama」でよくない？導入から使い方までまとめて解説
tags:
  - Python
  - Docker
  - AI
  - LLM
  - ollama
private: false
updated_at: '2026-04-11T10:04:17+09:00'
id: c470aba72cdabbcff272
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに

仕事でOllamaを活用する機会がありました。用途としてはアプリにLLM/VLMを組み込む目的です。

ローカルLLMを使おうとなると、これまでは「とりあえずHuggingFace」という選択肢しか頭になかったのですが、今回Ollamaを試してみて、用途によってはこちらのほうが圧倒的に手軽だと感じました。

HuggingFaceを使用する場合、「まずとにかく動かしたい」「システムの骨格だけ先に組みたい」という場面では、セットアップのコストがそれなりにかかります。

それと比較するなら導入が圧倒的に楽です。

**LLMをプロトタイプや導入の初期フェーズ、ドメイン知識が必要でない場面で使うなら、今のところ一番の選択肢だと感じています。**

---

# Ollamaとは

OllamaはローカルマシンでLLMを手軽に動かすためのツールです。

最大の特徴はセットアップの手軽さで、以下のコマンド一発でモデルのダウンロードから推論サーバーの起動まで完了します。

```bash
ollama run gemma3:4b
```

また、
- 推論に必要なランタイムが同梱されている
- GPU/CPUの切り替えを自動で行ってくれる

などの利点もあるため、「とりあえず動かす」という目的において非常に強力です。

---

# 環境構築
私は、コンテナ内でOllamaを動かしています。
今回は VS Code の Devcontainer にGPU環境を用意して Ollama を使用する例を紹介します。

### Dockerfile

```dockerfile
FROM nvidia/cuda:12.8.1-cudnn-devel-ubuntu22.04

# プロキシ環境変数
ARG http_proxy
ARG https_proxy
ARG no_proxy
ENV http_proxy=${http_proxy} \
    https_proxy=${https_proxy} \
    no_proxy=${no_proxy}

# 環境変数設定
ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8 \
    DEBIAN_FRONTEND=noninteractive \
    TZ=Asia/Tokyo

WORKDIR /workspace

# システムパッケージのインストール
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    git \
    # ...（省略）
    && rm -rf /var/lib/apt/lists/*

# Ollama
RUN curl -fsSL https://ollama.com/install.sh | bash

CMD ["/bin/bash"]
```

ポイントは以下の行です。

```dockerfile
# Ollama
RUN curl -fsSL https://ollama.com/install.sh | bash
```

公式のインストールスクリプトをDockerfileに書いておくだけで、コンテナ起動後すぐに `ollama` コマンドが使えるようになります。

### devcontainer.json（抜粋）

```json
"runArgs": [
  "--gpus", "all",
  "--env-file", "${localWorkspaceFolder}/.env"
]
```

`--gpus all` でGPUをコンテナに渡しているのがポイントです。OllamaはこれをそのままGPU推論に使ってくれます。
Devcontainer を使わないなら、`docker run` や `docker compose` で同じ指定をしてください。

---

# サーバー起動 & モデル管理

以下のコマンド一発でモデルのダウンロードからサーバーの起動まで完了します。
```bash
ollama pull gemma3:4b  # モデルの種類
```
起動すると `http://localhost:11434` にOpenAI互換のREST APIが自動で生えるため、そのままアプリに組み込めます。

しかし、（これすらも）毎回手動でサーバーを立ち上げるのは面倒なので、以下の様なスクリプトにまとめています。

### ollama_serve.sh

```bash
#!/bin/bash
set -e

if [ -f /opt/venv/bin/activate ]; then
    source /opt/venv/bin/activate
fi

# モデルを引数で指定 (デフォルト: gemma3:4b)
MODEL="${1:-gemma3:4b}"

# すでにollamaが起動しているか確認
if curl -s http://localhost:11434 > /dev/null 2>&1; then
    echo "Ollama is already running."
else
    echo "Starting Ollama server..."
    ollama serve > /workspace/ollama.log 2>&1 &

    echo "Waiting for Ollama server to be ready..."
    until curl -s http://localhost:11434 > /dev/null 2>&1; do
        sleep 1
    done
    echo "Ollama is ready."
fi

echo "Pulling model: ${MODEL}"
ollama pull "${MODEL}"

echo "Done. Model '${MODEL}' is ready to use."


# ※補足

# このスクリプトでは `source /opt/venv/bin/activate` を書いていますが、
# Ollama自体はPython環境に依存しないため、不要であれば削除して問題ありません。

# 元々Pythonプロジェクト用のテンプレートから流用したため残っている部分です。
```

引数でモデルを指定でき、省略するとデフォルトの `gemma3:4b` が起動します。

```bash
bash ollama_serve.sh              # gemma3:4b（デフォルト）
bash ollama_serve.sh mistral:7b
bash ollama_serve.sh qwen2.5:7b
```

その他の便利コマンドも整理しておきます。

```bash
pkill ollama          # サーバー停止
tail -f ollama.log    # ログ確認
ollama list           # インストール済みモデル一覧
ollama rm gemma3:4b   # モデル削除
```

### モデルの例

| モデル名 | サイズ | 特徴 |
|---|---|---|
| gemma3:4b | 3.3GB | デフォルト。日本語対応。汎用。 |
| gemma3:1b | 800MB | 超軽量版。動作確認用に便利。 |
| mistral:7b | 4GB | 汎用性が高い。英語タスクに強い。 |
| qwen2.5:7b | 4.7GB | 多言語対応。日本語に強い。 |
| qwen2.5-coder:7b | 4.7GB | コーディング特化。 |
| deepseek-r1:7b | 4.7GB | 推論・思考に特化。 |
| llama3.2:3b | 2GB | Meta製。バランスが良い。 |
| phi4-mini | 2.5GB | Microsoft製。コード生成が得意。 |
| qwen2.5vl:7b | ~16GB | VLM。GUI操作・bbox検出が得意。 |

---

# APIの使用方法

Ollamaを起動すると、http://localhost:11434 にOpenAI互換のAPIが立ち上がります。
そのため、curlやPythonなどから簡単に利用できます。

### curlでの実行例
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "gemma3:4b",
  "prompt": "日本語で自己紹介してください"
}'
```

レスポンスはストリーミング形式（生成途中のテキストが逐次返る形式）で返ってきます。

### Pythonでの実行例（requests）

```python
import requests

url = "http://localhost:11434/api/generate"

data = {
    "model": "gemma3:4b",
    "prompt": "日本語で自己紹介してください",
    "stream": False
}

response = requests.post(url, json=data)
print(response.json()["response"])
```

### OpenAI互換APIとして使う

OllamaはOpenAI互換APIも提供しているため、既存のクライアントをそのまま利用できます。

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # ダミーでOK
)

response = client.chat.completions.create(
    model="gemma3:4b",
    messages=[
        {"role": "user", "content": "こんにちは"}
    ]
)

print(response.choices[0].message.content)
```
既存のOpenAIコードをそのまま使える
APIキー不要（ダミーでOK、というか仕様上何かしら書かないといけないっぽい）

### ストリーミングを有効にする場合
```python
response = requests.post(url, json={
    "model": "gemma3:4b",
    "prompt": "こんにちは",
    "stream": True
}, stream=True)

for line in response.iter_lines():
    if line:
        print(line)
```

# 使ってみた感想

### よかった点

**セットアップがとにかく速い**
HuggingFaceでモデルを試すには、依存ライブラリのインストール・量子化形式の選定・`device_map`の設定など、動かすまでの準備がそれなりにあります。Ollamaは `ollama run モデル名` で即座に動きます。

**REST APIが最初から使える**
`http://localhost:11434` にOpenAI互換のエンドポイントが自動で生えるため、既存のコードやツールにそのまま組み込めます。システムの骨格を作る際に非常に助かりました。

**最新モデルにも対応**
OllamaのモデルライブラリはGemma、Qwen、Mistral、DeepSeekなど主要な最新モデルをカバーしており、精度面でも実用レベルで使えることが多いです。

### 注意点

**ファインチューニングはできない**
Ollamaは推論に特化したツールであり、追加学習には対応していません。プロジェクト固有のデータでモデルを調整したい場合は、プロンプト設計で対応するか、HuggingFaceでファインチューニングしたモデルをGGUF形式に変換してOllamaに読み込む方法を取る必要があります。

**使えるモデルはOllamaのライブラリに限られる**
HuggingFaceのような網羅的なモデルハブではないため、ニッチなモデルや研究用モデルは見つからないことがあります。

---

# HuggingFaceとの使い分け

実際に使ってみて、以下のような使い分けが自分の中で定着してきました。

| フェーズ | 推奨 | 理由 |
|---|---|---|
| アイデア検証・動作確認 | **Ollama** | 即座に動かせる。試行錯誤のコストが低い。 |
| システムの骨格・プロトタイプ構築 | **Ollama** | REST APIがそのままアプリに組み込める。 |
| 本格運用・ファインチューニング | **HuggingFace** | 細かい制御や追加学習が必要な場面。 |

「まずOllamaで骨格を作り、本格運用フェーズでHuggingFaceに移行する」という流れが、今のところ一番スムーズに感じています。

---

# まとめ

Ollamaはシンプルさと導入の手軽さが最大の強みです。

ファインチューニングができないという制約はありますが、プロンプト設計で対応できる範囲であれば、LLMを業務もしくは開発で使用する際の最初の一手として非常に優秀だと感じています。

---

# 補足
Ollama自体は無料で利用できるツールですが、実際に使用するモデルのライセンスはそれぞれ異なります。
そのため、商用利用や再配布の可否などはモデルごとに確認が必要です。
Ollamaはあくまで実行環境であり、最終的な利用条件はモデル側のライセンスに依存します。

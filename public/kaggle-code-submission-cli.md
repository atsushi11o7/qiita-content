---
title: Kaggleのコード提出入門 - ローカルで学習したモデルをCLIから提出するまで
tags:
  - Kaggle
  - CLI
  - 機械学習
  - Python
  - PyTorch
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

これまで Kaggle では、手元で予測した CSV をアップロードするコンペや、エージェントのファイルを提出するシミュレーション系のコンペに参加してきました。先日、初めて Notebook で隠しテストを推論する形式のコード提出(Notebook 提出)のコンペに参加し、学習はローカルの GPU、推論は Kaggle Notebook で行う構成で提出しました。

https://www.kaggle.com/competitions/rsna-knee-abnormality-detection

学習済みの重みを Dataset として持ち込み、見えないテストデータに対して Notebook を再実行してもらう、という流れはこれまでの提出とかなり違い、何度か失敗もしました。この記事では、特定のコンペやデータには依存しない形で、次の流れをまとめます。

- 学習済みの重みを Kaggle にアップロードする
- 推論用の Notebook を Kaggle 上で実行する
- Notebook のバージョンを指定して提出する
- 採点結果を確認する

:::note warn
Kaggle CLI のコマンドやメタデータの項目は、バージョンによって変わることがあります。本稿の編集時に確認したローカル環境は Kaggle CLI 2.2.4 です(2026-10-25)。以下のコード例全体を、この日に Kaggle 上で再実行したものではありません。実際に使う前に、公式資料と `kaggle <コマンド> --help` で確認してください。
:::

## 1. 提出形式とコード提出の仕組み

この記事では、次の提出形式を区別します。

| 形式 | 中身 |
|---|---|
| CSV 提出 | ローカルで予測したファイルをアップロードする |
| コード提出(ファイル出力型) | 提出した Notebook が隠しテストで再実行され、予測ファイルを書き出す |
| コード提出(評価 API 型) | 評価用の API を通じてデータを受け取り、予測を返す |

この記事では、ファイル出力型を扱います。

コード提出では、評価用のデータ(隠しテスト)が参加者に配られません。提出すると、Kaggle が Notebook を隠しテストに差し替えて再実行し、その出力で採点します。評価データを公開せずに済み、実行環境や時間の条件も参加者間でそろえられます。

以下では、インターネットが無効な環境で推論し、`/kaggle/working/submission.csv` を書き出すコンペを想定します。

## 2. 提出までに用意するもの

### 2.1 前提条件

- Kaggle CLI のインストールと認証の設定(公式の手順に従ってください)
- 使う重み、コード、外部データが、コンペのルールとライセンスに合っていること

:::note alert
API トークンなどの認証情報を、Notebook やアップロードするファイル、ログに含めないように注意してください。
:::

### 2.2 最小構成

```text
weights/                       # private な Dataset としてアップロード
├── dataset-metadata.json
├── model.pt                   # 学習済みの重み
└── model_config.json          # モデル構成・前処理・ラベル順など

notebook/
├── inference.ipynb            # モデル定義と推論処理を含む
└── kernel-metadata.json
```

- モデル定義と推論処理は Notebook の中に書き、自作のコードを別の Dataset にするのは後回しにします。
- 重みだけでなく、**入力サイズ、正規化の方法、出力ラベルの順番など、推論を再現するための設定**も一緒に保存します。重みはあっても前処理が学習時と違う、という失敗を防げます。

`model_config.json` の例です。

```json
{
  "arch": "resnet50",
  "img_size": 384,
  "mean": [0.485, 0.456, 0.406],
  "std": [0.229, 0.224, 0.225],
  "labels": ["label_a", "label_b", "label_c"]
}
```

## 3. 最小構成で提出する

全体の流れは次のとおりです。

```text
ローカルで学習
  → 重みと設定を private な Dataset にアップロード
  → Notebook にコンペのデータと Dataset を添付
  → push して、通常の実行が成功することを確認
  → 成功した Notebook のバージョンを指定して提出
  → 隠しテストでの採点結果を確認
```

### 3.1 重みと設定の Dataset を作成する

学習済みの `model.pt` と `model_config.json` を `weights/` に配置します。メタデータのひな形を作ります。

```bash
kaggle datasets init -p weights
```

`weights/dataset-metadata.json` を編集します。

```json
{
  "title": "my-model-weights",
  "id": "<owner>/my-model-weights",
  "licenses": [{"name": "CC0-1.0"}]
}
```

上の `CC0-1.0` は例です。使用する重みやコードのライセンスに合わせて設定してください。既存の重みの利用条件を、この設定だけで変更できるわけではありません。

Dataset を作成し、処理が終わったかを確認します。CLI で作成した Dataset は、既定では private になります(念のため、作成後に Web の画面で確認してください)。

```bash
kaggle datasets create -p weights
kaggle datasets status <owner>/my-model-weights
```

重みを更新するときは、新しいバージョンを作ります。

```bash
kaggle datasets version -p weights -m "retrain with more epochs"
```

最小構成では、`weights/` の直下にファイルを置くだけにします。サブディレクトリを含める方法は、5 章で扱います。アップロード後、Web の画面か次のコマンドで、ファイル構成が想定どおりかを確認してください。

```bash
kaggle datasets files <owner>/my-model-weights
```

### 3.2 推論 Notebook を実装する

Notebook では、次の点を守ります。

- 添付した入力は `/kaggle/input` の下から読み、生成物は `/kaggle/working` に書く。
- 学習時と同じモデル構成、前処理、ラベル順で重みを読み込む。
- インターネットが使えないので、事前学習の重みを自動でダウンロードする処理は無効にする。必要なものはすべて添付する。
- 推論では `model.eval()` と `torch.no_grad()` を使う。
- テストの ID を保持し、指定された形式の `submission.csv` を書き出す。
- 書き出す前に、行数、ID、列名と列の順番、欠損値、予測値の範囲を確かめる。
- 実行環境に必要なライブラリが入っているかを確認する。追加が必要なら、wheel ファイルを Dataset に入れてオフラインでインストールする。

以下は、各ラベルの陽性確率を出力するマルチラベル分類の骨格です。回帰や多クラス分類では、出力の変換と検証条件を変更してください。タスクに依存する部分(データの読み込み、モデル定義)は、コメントの箇所に書きます。

この例の `make_loader()` は、画像テンソル `images` と、その画像に対応する文字列 ID のリスト `ids` をバッチごとに返すものとします。ID は予測と一緒に保存し、入力順が変わっても対応がずれないようにします。

```python
import json
from pathlib import Path

import numpy as np
import pandas as pd
import torch

COMP_DIR = Path("/kaggle/input/<competition>")   # マウント先は 4.1 も参照
WEIGHT_DIR = Path("/kaggle/input/my-model-weights")
OUT = Path("/kaggle/working/submission.csv")

cfg = json.loads((WEIGHT_DIR / "model_config.json").read_text())
device = "cuda" if torch.cuda.is_available() else "cpu"

# --- タスク依存: モデル定義(学習時と同じ構成。事前学習の重みはダウンロードしない) ---
model = build_model(cfg["arch"], pretrained=False, num_classes=len(cfg["labels"]))
model.load_state_dict(torch.load(WEIGHT_DIR / "model.pt", map_location="cpu"))
model.to(device).eval()

test = pd.read_csv(COMP_DIR / "test.csv", dtype={"id": str})
preds, ids = [], []
with torch.no_grad():
    # --- タスク依存: 前処理とデータの読み込み ---
    for batch in make_loader(test, cfg):
        images = batch["images"].to(device)
        probs = torch.sigmoid(model(images)).cpu()
        assert len(batch["ids"]) == len(probs)
        preds.append(probs)
        ids.extend(batch["ids"])
pred = torch.cat(preds).numpy()

sub = pd.DataFrame(pred, columns=cfg["labels"])
sub.insert(0, "id", ids)

# この例では、1 ID につき1行の予測を出す
assert test["id"].notna().all() and test["id"].is_unique
assert sub["id"].notna().all() and sub["id"].is_unique
assert len(sub) == len(test) and set(sub["id"]) == set(test["id"])
# 予測時の順番に依存せず、入力の ID 順に並べる
sub = sub.set_index("id").loc[test["id"]].reset_index()

# 書き出す前の検証(サンプルと同じ1行/ID形式を想定)
sample = pd.read_csv(COMP_DIR / "sample_submission.csv", dtype={"id": str})
assert list(sub.columns) == list(sample.columns)
values = sub[cfg["labels"]].to_numpy()
assert np.isfinite(values).all()
assert ((0 <= values) & (values <= 1)).all()
sub.to_csv(OUT, index=False)
```

### 3.3 Notebook を push して実行を確認する

`notebook/kernel-metadata.json` の最小例です。

```json
{
  "id": "<owner>/my-inference",
  "title": "my-inference",
  "code_file": "inference.ipynb",
  "language": "python",
  "kernel_type": "notebook",
  "is_private": true,
  "enable_gpu": true,
  "enable_internet": false,
  "machine_shape": "NvidiaTeslaT4",
  "dataset_sources": ["<owner>/my-model-weights"],
  "competition_sources": ["<competition>"],
  "kernel_sources": []
}
```

push すると、Kaggle 上で Notebook が実行されます。

```bash
kaggle kernels push -p notebook --accelerator NvidiaTeslaT4
```

状態を確認し、終わったら出力を取得します。

```bash
kaggle kernels status <owner>/my-inference
kaggle kernels output <owner>/my-inference -p out/
```

取得した `submission.csv` を確認し、成功した Notebook のバージョン番号を控えておきます。

### 3.4 バージョンを指定して提出し、採点結果を確認する

```bash
kaggle competitions submit -c <competition> -f submission.csv \
  -k <owner>/my-inference -v <version> -m "resnet50 384px"
```

ここで `-f` は、ローカルのファイルではなく、指定した Notebook バージョンが出力したファイルの名前です。CSV 提出のときのように、手元のファイルをアップロードするわけではありません。

- 提出できるのは、Kaggle 上で実行に成功したバージョンだけです。提出前に、そのバージョンが成功していて、出力ファイルがあることを確認します。
- 残りの提出回数は、次のコマンドで確認できます。

```bash
kaggle competitions submission-limits <competition>
```

- 採点結果は、次のコマンドで確認します。

```bash
kaggle competitions submissions <competition>
```

通常の実行が成功していても、提出後の再実行と採点が成功するとは限りません。必ず採点結果まで確認してください。また、提出 ID と、Notebook、重み、設定のバージョンの対応を記録しておくと、後で比較するときに困りません。

## 4. 実際につまずいた点

ここからは、実際に起きたことを「症状 → 確認したこと → 対処」の順に書きます。環境によって起きたり起きなかったりする可能性があるので、一般的な仕様や確定した原因としては扱わないでください。

### 4.1 アップロード後のファイル構成とパス

**症状**: Notebook で、添付した Dataset のファイルが、想定したパスに見つからなかった。

**確認したこと**:
- サブディレクトリを含めてアップロードしたところ、Dataset の上では、フォルダ名が消えて中身が直下に並んでいた。
- マウント先が `/kaggle/input/<dataset>` ではなく、`/kaggle/input/datasets/<owner>/<dataset>` だった。

**対処**: パスを固定で書かず、Notebook の中で実際のディレクトリを一覧して確認し、必要なファイルを探して場所を決めるようにした(探し方の注意は 5 章)。

### 4.2 GPU の割り当て待ちと、Notebook の起動エラー

**症状 1**: push した Notebook が、GPU の割り当て待ち(QUEUED)のまま、何時間も始まらなかった。

**確認したこと**: 最初は `machine_shape` を指定せずに push していた。`--accelerator NvidiaTeslaT4` と `machine_shape` を指定して push し直すと、すぐに実行が始まった。ただし、別の日には、指定していても 2 時間以上 QUEUED のままのことがあった。

**対処**: 指定の有無と待ち時間の因果関係までは確かめられていません。GPU の種類は明示したうえで、待ち時間は数時間かかることもある前提で予定を立てるようにした。

**症状 2**: 実行がすぐエラーで止まり、ログに `No kernel name found in notebook and no override provided.` と出た。

**確認したこと**: スクリプトで `.ipynb` を生成したため、Notebook のメタデータに kernelspec が入っていなかった。

**対処**: Notebook のメタデータに kernelspec を追加した。

```python
import nbformat
nb = nbformat.read("inference.ipynb", as_version=4)
nb.metadata["kernelspec"] = {"name": "python3", "display_name": "Python 3", "language": "python"}
nb.metadata["language_info"] = {"name": "python"}
nbformat.write(nb, "inference.ipynb")
```

### 4.3 通常の実行では成功し、隠しテストで失敗した

**症状**: 提出の状態が `COMPLETE` になったのに、**スコアの欄が空のまま**だった。完了と表示されるので、一見すると成功したように見える。

**確認したこと**:
- CLI の表示だけでは原因が分からなかった。Web の Submissions 画面で該当の提出を見ると、エラーの理由が表示されていた。
- 1 回目は、メモリ不足(Out of Memory)だった。公開テストは数件しかなく、通常の実行では問題が表に出なかった。
- 2 回目は、`Notebook Timeout` だった。Notebook の中で計った時間には余裕があるつもりだったが、時間切れになった。準備や出力など、計測範囲外の処理時間も考慮する必要があると考えた。ただし、どの処理が原因だったかまでは特定できていない。

**対処**:
- 処理を段階ごとに別プロセスで実行し、段階ごとに空きメモリをログに出すようにした(5 章)。
- 時間の余裕は、Notebook の中で計った時間よりも大きく取るようにした。
- 今回の実装では、推論全体の時間のうち、モデルの計算より、前処理やデータの読み込みが大部分を占めていた。モデルの速度だけでは、全体の時間を見積もれなかった。
- GPU の割り当て待ちの時間と、実行が始まってからの時間を分けて記録するようにした。

## 5. 大きなモデルや複数のモデルを扱う場合の工夫

ここからは、最小構成で提出できた後、必要になったときの選択肢です。

- **自作コードを Dataset にする**: 複数ファイルのコードを Dataset に含め、Notebook で `sys.path` に追加して import する。追加する場所は、対象の Dataset の中に限定する。
- **パスを探すときは範囲を絞る**: `/kaggle/input` 全体を再帰的に探すと、コンペのデータ(大量のファイル)まで探して非常に遅くなることがある。対象の Dataset と候補を限定し、必要なファイルが 1 つだけ見つかったことを確認する。名前の同じ別のモデルを拾わないようにする。
- **サブディレクトリのアップロード**: `kaggle datasets create` / `version` の `--dir-mode tar` などを使う。マウント後の構成は、必ず実際に確認する。
- **精度の選び方**: GPU が対応しているかどうかで、bf16 / fp16 / float32 を選ぶ(例: T4 は bf16 に対応していない)。精度を下げた後は、出力が有限値か、精度が落ちていないかを確認する。
- **別プロセスに分ける**: 複数のモデルを使う場合、モデルごとに別プロセスで推論して結果をファイルに書き、終了時にメモリを解放する。段階ごとの時間とメモリを記録する。
- **代替処理**: 想定内の入力の破損などには、ログ付きの代替処理を用意する。例えば、複数モデルのうち成功したモデルの予測を使う、予測できない ID には既定値を使うなど。ただし、要求された全 ID の行を維持し、失敗件数と代替処理の内容を記録する。開発中は予期しない例外を隠さず、修正する。
- **時間での打ち切り**: 時間のかかる処理に打ち切りを設ける場合は、間に合わなかったデータをどう扱うか、出力にどう影響するかも決めておく。打ち切り時間は、4.3 のとおり余裕を大きく取る。

## おわりに

CSV 提出との一番の違いは、**推論を Kaggle 上で、自分には見えないデータに対して実行する**ことでした。手元や公開テストで動いても、本番の規模で初めて問題が出ることがあります。

まずは最小構成で提出まで通し、その後に速度やメモリの対策、複数のモデルへと広げていくのがおすすめです。今回の環境では、GPU の割り当て待ちが数時間に及ぶこともあり、本番の実行にも数時間かかりました。失敗したときに直して出し直せるよう、締切の直前は避けて、時間に余裕を持って提出してください。

## 参考資料

- Kaggle CLI: https://github.com/Kaggle/kaggle-api
- Kaggle CLI のチュートリアル: https://github.com/Kaggle/kaggle-api/blob/main/docs/tutorials.md
- Kaggle のコンペのドキュメント(Notebooks-only FAQ を含む): https://www.kaggle.com/docs/competitions
- 参加するコンペの Rules、Code Requirements、Evaluation

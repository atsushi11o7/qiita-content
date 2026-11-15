---
title: NVIDIA のテーブル基盤モデル「Kumo Tabular」を TabPFN-3 と比べてみた
tags:
  - 機械学習
  - NVIDIA
  - Transformer
  - Python
  - TabPFN
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

以前、重みを更新しないテーブル基盤モデル TabPFN-3 を UCI Adult で試す記事を書きました。

https://qiita.com/atsushi11o7/items/6c7206293e1b0b39022e

その時は、前処理なし・無調整でも、公開されている LightGBM の参考スコアに近い精度を出すことができました。

今回、2026年9月に NVIDIA からテーブル基盤モデル **Kumo Tabular** が公開されたため試してみました。
重みが公開されていて商用利用もでき、TabArena（テーブルデータのベンチマーク）で総合1位と発表されています。

この記事では、カテゴリ列と欠損を含む UCI Adult（年収の二値分類）を使い、Kumo Tabular と TabPFN-3 を前処理なし・無調整のまま同じ条件で動かして、精度（ROC-AUC）と、fit と predict の合計時間を比べます。

## Kumo Tabular とは

NVIDIA が公開しているテーブルデータ向けの基盤モデルで、分類と回帰に対応しています。
ライブラリ `structured-data-models`（`sdm`）の中のモデルの1つとして提供されています。

- TabPFN と同じく **インコンテキスト学習** で予測する Transformer 系のモデル
- `fit` は重みを更新せず、学習データの前処理や、予測で使い回す情報の計算・キャッシュを行う
- `predict` で、学習データを文脈として予測対象のラベルを出す
- 回帰では値を1つではなく、999 個の分位点（`q001`〜`q999`）を出す

構造は2段になっています。

1. 列方向と行方向の Attention を交互に重ねて、1行を固定長のベクトルにまとめるエンコーダ
2. 学習データの行を文脈として、予測対象の行のラベルを出すインコンテキスト学習の Transformer

予測対象の行は学習データの行だけを参照し、予測対象どうしは参照しません（後で効いてきます）。
NVIDIA のブログでは、この列・行・インコンテキストの Attention は TabICL と TabPFN で導入されたものを使っていると説明されています。

その他、ブログに書かれている主な点です。

- 事前学習は、構造的因果モデルで生成した合成テーブルのみ
- パラメータ数は 28M〜215M の3サイズ（既定は `large`）
- 学習時に扱った規模は、文脈が最大 6 万行・100 列まで。これを大きく超えるテーブルでは精度が落ちる可能性がある
- 扱えるのは数値列とカテゴリ列（テキストや日時は前処理で特徴量にする）
- TabArena で総合1位（Elo 1950）。ただし NVIDIA 自身による評価

## 使う前に: ライセンス

比較に使う TabPFN-3 は、重みをダウンロードする前に Prior Labs でライセンスに同意し、API キー（`TABPFN_TOKEN`）を発行する必要があります。

Kumo Tabular は **登録もトークンも要りません**。重みは Hugging Face から認証なしでダウンロードされます。

- コード: Apache 2.0
- 重み: [OpenMDW-1.1](https://openmdw.ai/license/1-1/)

OpenMDW-1.1 は、重みなどのモデル一式を制限なく使うことを認めるライセンスで、NVIDIA のブログでも商用利用向けとしています。
再配布するときには、ライセンス文と著作権表示を残す必要があります。モデルの出力の利用には制限がありません。

## 試してみる

### 環境とインストール

devcontainer（CUDA 12.9 + Python 3.12、uv 管理）で実行しました。
パッケージは `structured-data-models` を入れるだけです。

```bash
uv add structured-data-models
# pip の場合
pip install structured-data-models
```

### データ

[UCI Adult](https://www.openml.org/d/1590) を使います（[TabPFN-3 の記事](https://qiita.com/atsushi11o7/items/6c7206293e1b0b39022e)と同じデータです）。
約4.9万行・14特徴の二値分類（年収が 5 万ドルを超えるか）で、カテゴリ列と欠損を含みます。
陽性率が約24%と不均衡なので、ROC-AUC を中心に見ます。

### 前処理は書かない（ただし TableTensor に変換する）

Kumo Tabular でも one-hot・スケーリング・欠損補完は書きません。
ただ、TabPFN のように DataFrame をそのまま渡すのではなく、`TableTensor` という GPU 上のテーブル型に変換してから渡します。

```python
import sdm
from sklearn.datasets import fetch_openml
from sklearn.metrics import roc_auc_score
from sklearn.model_selection import train_test_split

ds = fetch_openml("adult", version=2, as_frame=True)
X = ds.data
y = (ds.target == ">50K").astype(int).to_numpy()
X_tr, X_te, y_tr, y_te = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=0
)

# 列の型（数値 / カテゴリ）は推論させ、目的変数だけ分類であることを明示
x_stypes = sdm.infer_stypes(X)
train = sdm.TableTensor.from_pandas(
    df=X_tr.iloc[:5000].assign(target=y_tr[:5000]),  # 全量は重いので 5,000 行で
    stypes=x_stypes | {"target": "categorical"},
    device="cuda",
)
test = sdm.TableTensor.from_pandas(df=X_te, stypes=x_stypes, device="cuda")

model = sdm.models.KumoTabular(device="cuda")  # 初回は重みをダウンロード
model.fit(x=train.drop_columns("target"), y=train[:, "target"], num_estimators=8)
proba = model.predict(test).to_pandas()["1"]  # 列名で取る（後述）
print(roc_auc_score(y_te, proba))
```

型の推論では、Adult の `category` 型の列はカテゴリ、整数の列は数値と判定されました。
目的変数は 0/1 の整数なので、`categorical` と明示しないと回帰として扱われる可能性があります。

### 注意: 確率の列は名前で取る

`predict` の結果はクラスごとの確率ですが、**列名はクラスラベルの文字列で、並び順は保証されません**。
同じデータ・同じ学習行でも、実行によって `['0', '1']` だったり `['1', '0']` だったりしました。

sklearn の `predict_proba(...)[:, 1]` の感覚で位置で取ると、陰性クラスの確率を拾ってしまうことがあり、AUC が `1 - AUC` に反転します。
実際に動作確認のときは、これで AUC が 0.003 になりました。

## 結果

Kumo Tabular と TabPFN-3 のそれぞれで、学習データ量を 1,000 行から全量（約3.9万行）まで変え、固定のテストデータ（全体の20%、9,769 行）で測りました。
各学習サイズのサブサンプルは、両モデルで共通です。
単一の層化分割での結果なので、AUC の細かい差は分割によって多少動く可能性があります。

| 学習行数 | Kumo AUC | TabPFN-3 AUC | Kumo fit+predict | TabPFN-3 fit+predict |
|---:|---:|---:|---:|---:|
| 1,000 | 0.9158 | 0.9015 | 4.8s | 1.2s |
| 5,000 | 0.9267 | 0.9094 | 14.1s | 1.5s |
| 10,000 | 0.9294 | 0.9132 | 27.7s | 2.6s |
| 20,000 | **0.9322** | 0.9150 | 62.3s | 4.8s |
| 39,073（全量） | 結果を得られず | 0.9174 | - | 11.8s |

（WSL2 上の devcontainer、GPU: NVIDIA GeForce RTX 5070 Ti 16GB、WSL への割り当て RAM 15GB。数値はすべてこの環境で測り直したものです）

![kumo_vs_tabpfn.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3121510/aa1baa79-7bba-4577-9087-52f7128c5c80.png)

- **精度は Kumo Tabular が上**: どの学習サイズでも AUC が約 0.014〜0.017 高く、5,000 行の時点で TabPFN-3 の全量（0.9174）を上回りました。今回の分割では、前処理なし・無調整で 20,000 行の AUC が 0.9322 でした。
- **速さは TabPFN-3 が圧倒的**: 4〜13倍の差があり、Kumo は学習データを増やすと、fit と predict の合計時間が比例以上に伸びます。
- **Kumo の全量は結果を得られなかった**: 約3.9万行では、実行途中でプロセスが終了しました（後述）。

時間は、両モデルとも計測の前に少量のデータで `fit` と `predict` を1回空打ちし、モデルの読み込みや GPU の初期化を除いて測っています。

### 16GB の GPU では予測対象を分けて渡す

最初はテストデータ 9,769 行をまとめて `predict` に渡していましたが、学習データが 10,000 行を超えたあたりで急に終わらなくなりました。
このとき GPU メモリの使用量は上限付近に達しており、推定器を1つに減らしても 10,000 行でピークが 15GB を超えていました。

予測対象を 2,000 行ずつに分けて渡すと改善したので、メモリの負荷が影響していた可能性があります。分けた場合、10,000 行でピークメモリが 5GB 程度に収まり、30秒ほどで終わるようになりました。

```python
import pandas as pd

proba = pd.concat(
    [model.predict(test[i : i + 2000]).to_pandas() for i in range(0, len(y_te), 2000)],
    ignore_index=True,
)["1"]
```

予測対象の行は学習データの行だけを参照し、予測対象どうしは参照しないので、分けても理屈の上では予測は変わらないはずです。今回確認した 1,000 行と 5,000 行では、分ける前と後で AUC が同じでした。

全量（約3.9万行）は、この分割をしても実行途中でプロセスが終了し（終了コード 137）、結果を得られませんでした。メインメモリ（WSL への割り当て 15GB）の不足の可能性が高いです。

## Kumo Tabular の特徴

**向いていそうな場面**

- 数千〜2万行程度のデータで、無調整のまま高い精度がほしいとき
- 商用で使いたいとき（重みが OpenMDW-1.1 で、登録やトークンも不要）

**注意したい点**

- TabPFN と同じく、推論のたびに学習データ全体を文脈に入れるので、学習データが増えるほど時間とメモリが増える。今回は TabPFN-3 より伸び方がかなり大きかった
- GPU とメモリに余裕が必要。16GB の GPU では予測対象を分けて渡す必要があり、約3.9万行は手元の環境では結果を得られなかった
- 入力は `TableTensor` への変換が必要で、sklearn 互換の API ではない

## まとめ

NVIDIA のテーブル基盤モデル Kumo Tabular を、TabPFN-3 と同じ条件で UCI Adult に当ててみました。
前処理なし・無調整のまま、どの学習サイズでも TabPFN-3 より AUC が高く、今回の分割では 2万行で AUC 0.932 でした。

一方で、fit と predict の時間とメモリは TabPFN-3 よりかなり重く、16GB GPU・15GB RAM の環境では全量（約3.9万行）の結果を得られませんでした。
データの規模と計算資源に応じて選びたいモデルだと感じました。

## 参考

- [nvidia/Kumo-Tabular（Hugging Face モデルカード）](https://huggingface.co/nvidia/Kumo-Tabular)
- [NVIDIA Kumo Tabular Sets a New Accuracy-Efficiency Frontier for Tabular Prediction（Hugging Face ブログ）](https://huggingface.co/blog/nvidia/kumo-tabular)
- [NVIDIA/structured-data-models（GitHub）](https://github.com/NVIDIA/structured-data-models)
- [TabArena](https://tabarena.ai)
- [重みを更新しないテーブル基盤モデル「TabPFN-3」を試してみた（Qiita）](https://qiita.com/atsushi11o7/items/6c7206293e1b0b39022e)

## おまけ

今回使用した実験用リポジトリです。

https://github.com/atsushi11o7/tabular-sandbox

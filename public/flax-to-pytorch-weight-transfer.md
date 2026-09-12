---
title: JAX で学習した Transformer の重みを PyTorch に移植する
tags:
  - jax
  - PyTorch
  - flax
  - Transformer
  - 機械学習
private: false
updated_at: '2026-09-13T01:41:24+09:00'
id: 52c41cbfad40fee67992
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

前回、このような記事を書きました。

https://qiita.com/atsushi11o7/items/0bf94ff412c035aaa22d

JAX は、rollout のような処理をまとめてコンパイル・ベクトル化できるため、大量のシミュレーションを回す場面と相性がいいです。一方、既存の推論コードや提出環境が PyTorch で書かれている場合、学習部分だけを JAX へ移して高速化したいことがあります。

このように「学習は JAX、推論は PyTorch」と分ける場合、同じモデルの重みを両方のフレームワークで読み書きできる変換層が必要になります。

PyTorch のコードを JAX から呼び出したり変換したりするツール自体はいくつか存在するみたいです。[pytorch2jax](https://pypi.org/project/pytorch2jax/) は PyTorch モデルをラップして JAX の関数として呼び出せるようにするもので、中身は依然として PyTorch の演算がそのまま動きます。[torch2jax](https://github.com/samuela/torch2jax) のように PyTorch のコードをトレースして JAX ネイティブな計算グラフに変換するものもあります。ただ、これらは「PyTorch のコードを JAX から使う」方向の話で、今回のように独自に実装した PyTorch モデルと Flax モデルの間でパラメータ構造を対応付けて相互に読み書きする、という用途には、結局モデルごとのマッピングが必要になります。

[Flax 公式ドキュメントの "Convert PyTorch models to Flax" ガイド](https://flax.readthedocs.io/en/latest/guides/convert_pytorch_to_flax.html) も、`flax.traverse_util.flatten_dict` でパラメータをフラットな辞書にし、名前を対応付けて、Linear/Conv の重みを転置する、という手動マッピングを標準的なやり方として案内しています。Transformer の内部実装（特に Multi-Head Attention）はライブラリごとに重みのレイアウトが大きく異なるので、明示的な対応付けが現実的な解法のようです。

JAX で学習した Transformer の重みを PyTorch に移植する機会があったので、その時行ったことについてまとめます。逆方向（PyTorch→JAX）の変換も一緒に用意したので、この記事では両方とも扱います。

## Flax について

本題（重み変換）に入る前に、Flax の最低限の作法を確認しておきます。Flax は JAX の上に構築されたニューラルネットワーク用のライブラリで、レイヤーの定義・パラメータの管理・学習ループの土台を提供します。Flax には従来から使われてきた `linen` と、新しい `nnx`（よりオブジェクト指向的な API）の2系統があります。公式は新規プロジェクトに `nnx` を推奨していますが、私が使ったのが `linen` なので、この記事では `linen` を扱います（以降の `nn` は `flax.linen` を指します）。

### モデルを定義する

`nn.Module` を継承し、`@nn.compact` を付けたメソッドの中でレイヤーを呼びます。PyTorch の `__init__` でレイヤーを作って `forward` で呼ぶのと違い、レイヤーの生成と呼び出しを同じ場所に書けます。

```python
import jax
import jax.numpy as jnp
import flax.linen as nn

class TinyMLP(nn.Module):
    hidden_dim: int
    out_dim: int

    @nn.compact
    def __call__(self, x):
        x = nn.Dense(self.hidden_dim)(x)
        x = nn.relu(x)
        x = nn.Dense(self.out_dim)(x)
        return x
```

### パラメータを初期化する（`init`）

PyTorch では `Model()` を呼んだ瞬間に重みが確保されますが、Flax のモジュールは「形」の定義でしかなく、重みを持ちません。重み（パラメータ）は `init` にダミー入力を渡して初めて作られます。

```python
model = TinyMLP(hidden_dim=32, out_dim=4)
key = jax.random.key(0)
dummy_input = jnp.zeros((1, 8))  # バッチ次元込みのダミー入力
variables = model.init(key, dummy_input)
```

`variables` は `{"params": {...}}` という入れ子の辞書（PyTree）で、中身はレイヤー名ごとの `kernel`/`bias` になっています。

```python
jax.tree.map(lambda a: a.shape, variables)
# {'params': {
#     'Dense_0': {'kernel': (8, 32),  'bias': (32,)},
#     'Dense_1': {'kernel': (32, 4),  'bias': (4,)},
# }}
```

`kernel` の shape が `(in_features, out_features)` になっています。PyTorch の `nn.Linear.weight` は `(out_features, in_features)` で逆順なので、後述の変換で必ず転置が必要になります。

### 推論する（`apply`）

モデルとパラメータが分離しているので、実行時は毎回 `apply` に両方渡します。

```python
output = model.apply(variables, jnp.ones((1, 8)))
```

これが PyTorch の `model(x)`（パラメータがモデルの内部状態）と一番感覚が違う点です。Flax では「モデル定義（計算のやり方）」と「パラメータ（具体的な数値）」が完全に分離しているので、パラメータだけ差し替えて同じ計算を走らせることが自然にできます。

### 学習する（`TrainState`）

学習時は `TrainState`（`flax.training.train_state`）に `optax`（JAX 用の optimizer ライブラリ）を組み合わせるのが定番です。今回の本題（重み変換）には関係しないので、詳細は割愛します。

ここまでが最低限の作法です。本題に必要なのは「`variables["params"]` はレイヤー名をキーにした `kernel`/`bias` のネスト辞書」「`kernel` は `(in, out)`」の2点だけなので、これを踏まえて次に進みます。

## 変換の方針

変換は `state_dict`（PyTorch）と `variables["params"]`（Flax）を直接読み書きする形で書きます。PyTorch の `state_dict` はほぼ「名前→Tensor」のフラットな辞書として扱えますが、Flax の `variables["params"]` はレイヤー名ごとにネストした PyTree です。`flax.traverse_util.flatten_dict` でフラット化してから対応付けると実装しやすくなります。

双方向（JAX→PyTorch、PyTorch→JAX）を用意します。JAX で学習した重みを PyTorch 側へ持っていく用途がメインですが、逆に「PyTorch で初期化した重みを JAX 学習の初期値にする」用途もあるためです。

今回変換したのは、埋め込み（token embedding, position embedding）・LayerNorm・普通の Linear（feedforward や value head）・Multi-Head Attention の4種類です。難易度順に、簡単なものから見ていきます。

## Embedding と LayerNorm はキー名が違うだけ

一番簡単なのがこの2つです。shape も中身の並びも同じで、キー名が違うだけなので、単純に読み替えるだけで済みます。

```python
import numpy as np
import torch

# Embedding: Flax (num_embeddings, dim) == PyTorch (num_embeddings, dim)
torch_state["weight"] = torch.from_numpy(np.asarray(flax_params["embedding"]))

# LayerNorm: Flax scale/bias == PyTorch weight/bias（どちらも (dim,)）
torch_state["weight"] = torch.from_numpy(np.asarray(flax_params["scale"]))
torch_state["bias"] = torch.from_numpy(np.asarray(flax_params["bias"]))
```

転置も reshape も要りません。実際に `nn.Embed(10, 4)` と `nn.Embedding(10, 4)`、`nn.LayerNorm()` と `nn.LayerNorm(6)` のパラメータ shape を突き合わせて確認しましたが、両方とも完全に一致します。「全部のレイヤーで転置が要る」という思い込みで一律 `.T` を書いてしまうと、逆にここで壊れるので注意してください。

ただし、パラメータの shape が同じだからといって計算まで同じとは限りません。代表例が LayerNorm の epsilon で、PyTorch のデフォルトは `1e-5`、Flax linen のデフォルトは `1e-6` です。重みを完璧に移しても、両方ともデフォルト値のままだと出力は完全には一致しません。出力を厳密に揃えたい場合は、どちらかの epsilon を明示的に指定して揃える必要があります。

## Linear 層は転置が必要

Dense（全結合）層は、PyTorch と Flax で同じ情報を逆順の shape で持っています。

- PyTorch の `nn.Linear` は `y = x @ W.T + b`。`weight` の shape は `(out_features, in_features)`
- Flax の `nn.Dense` は `y = x @ W + b`。`kernel` の shape は `(in_features, out_features)`

```python
# Flax -> PyTorch
torch_weight = torch.from_numpy(np.asarray(flax_kernel).T)

# PyTorch -> Flax
flax_kernel = torch_weight.detach().cpu().numpy().T
```

## Multi-Head Attention は構造ごと違う

Embedding・LayerNorm・Linear は「同じ情報を違う形式で持っている」だけでしたが、Multi-Head Attention は PyTorch と Flax でパラメータの持ち方の構造自体が違うため、単純な転置では済みません。全体像を先に shape で示すとこうなります。

| 段階 | PyTorch | Flax |
| --- | --- | --- |
| Q/K/V（結合前） | `in_proj_weight`: `(3 * d_model, d_model)` | `query`/`key`/`value` の `kernel`: それぞれ `(d_model, d_model)` |
| Q/K/V（head 分解後） | 分解しない | `kernel`: `(d_model, heads, head_dim)` |
| out_proj | `out_proj.weight`: `(d_model, d_model)` | `out` の `kernel`: `(heads, head_dim, d_model)` |

### Q/K/V の分割・結合

PyTorch の `nn.MultiheadAttention` は、`kdim == vdim == embed_dim`（Q/K/V が同じ次元数）の通常構成では、Query/Key/Value の3つの線形射影を1枚の重み行列に連結して持っています（`kdim`/`vdim` を別々に指定した場合は `q_proj_weight`/`k_proj_weight`/`v_proj_weight` に分かれます）。

```python
# state["self_attn.in_proj_weight"].shape == (3 * d_model, d_model)
# state["self_attn.in_proj_bias"].shape   == (3 * d_model,)
```

対して Flax の `MultiHeadDotProductAttention` は、Q/K/V を `DenseGeneral`（出力を `(heads, head_dim)` のような多次元にできる Dense 層）でそれぞれ別々に持っています。JAX→PyTorch 方向では3つの `kernel` を結合する必要があります。

```python
def attention_to_torch(flax_params):
    q_kernel = np.asarray(flax_params["query"]["kernel"])
    k_kernel = np.asarray(flax_params["key"]["kernel"])
    v_kernel = np.asarray(flax_params["value"]["kernel"])
    # 各 kernel は (in, out) なので転置してから結合する
    in_proj_weight = np.concatenate([q_kernel.T, k_kernel.T, v_kernel.T], axis=0)
    return torch.from_numpy(in_proj_weight)
```

逆に PyTorch→Flax 方向では、`np.split` で3分割してから対応させます。`DenseGeneral` は bias も `(heads, head_dim)` の形を持つので、bias の reshape も忘れずに行います（kernel の head 次元への reshape は次の節で扱います）。

```python
def attention_to_flax(state, prefix, heads, head_dim):
    weight = state[f"{prefix}.in_proj_weight"].detach().cpu().numpy()
    bias = state[f"{prefix}.in_proj_bias"].detach().cpu().numpy()
    q_weight, k_weight, v_weight = np.split(weight, 3, axis=0)
    q_bias, k_bias, v_bias = np.split(bias, 3)
    q_bias = q_bias.reshape(heads, head_dim)
    k_bias = k_bias.reshape(heads, head_dim)
    v_bias = v_bias.reshape(heads, head_dim)
    return {
        "query": {"kernel": q_weight.T, "bias": q_bias},
        "key": {"kernel": k_weight.T, "bias": k_bias},
        "value": {"kernel": v_weight.T, "bias": v_bias},
    }
```

### head 次元の分解・統合

PyTorch は各 head の重みも1枚の行列にまとめて持っています（`(d_model, d_model)` のフラットな2次元）。Flax の `DenseGeneral` は、head ごとに扱いやすいよう `(d_model, heads, head_dim)` という3次元で kernel を持ちます。

```python
def to_torch_weight(flax_kernel):
    # (in, heads, head_dim) -> (in, out) -> (out, in)
    d_model, heads, head_dim = flax_kernel.shape
    return flax_kernel.reshape(d_model, heads * head_dim).T

def to_flax_kernel(torch_weight, heads, head_dim):
    # (out, in) -> (in, out) -> (in, heads, head_dim)
    d_model = torch_weight.shape[0]
    return torch_weight.T.reshape(d_model, heads, head_dim)
```

reshape と転置の順序を間違えると、shape は合うのに中身が head ごとにバラバラに混ざる、ということになります。

### out_proj は入出力方向が逆になる

Attention の最後の結合射影（out_proj）は「複数 head の出力を結合して d_model へ戻す」向きなので、分解方向が Q/K/V と逆になります。

```python
# out_proj: Flax の (heads, head_dim, d_model) から PyTorch の (d_model, d_model) へ
torch_out_weight = flax_out_kernel.reshape(-1, flax_out_kernel.shape[-1]).T
```

## 変換の正しさをどう保証するか

転置忘れ・reshape 順序ミスは例外を投げてくれないので、単体テストで「変換後に同じ入力を通したら同じ出力になる」ことを数値的に確認するのが必須になるかと思います。

```python
def test_flax_to_torch_outputs_match():
    flax_model, flax_variables = build_flax_model(seed=0)
    torch_model = flax_to_torch(flax_variables)
    torch_model.eval()

    x = sample_input()
    flax_out = np.asarray(flax_model.apply(flax_variables, x))
    with torch.no_grad():
        torch_out = torch_model(torch.from_numpy(x)).detach().numpy()

    np.testing.assert_allclose(flax_out, torch_out, atol=1e-5)
```

双方向の変換について、`flax_to_torch` で作った重みを `torch_to_flax` で戻して元のパラメータと一致するか、という往復テストも用意しておくと安心です。

出力比較は重みだけではありません。LayerNorm の epsilon に加えて、dropout を無効化しているか、モデルが `eval` モードになっているか、causal mask やパディングの扱い、activation 関数の種類なども、両モデルで揃っているか確認する必要があると思います。重み変換自体が正しくても、こうした計算条件が違うと出力は一致しません。

## おわりに

今回は、Transformer の重みを PyTorch に移植するために、Embedding・LayerNorm・Linear・Multi-Head Attention の4種類のレイヤーについて、それぞれの shape や構造の違いを確認しながら変換コードを書きました。特に Multi-Head Attention は PyTorch と Flax でパラメータの持ち方の構造自体が違うので、Q/K/V の分割・結合や head 次元の reshape など、対応付けが必要でした。

そもそも JAX と PyTorch の間でこうやって重みを変換する運用って一般的なんですかね？もっと良いやり方があるのかもしれません。JAX について理解が浅い部分が多いので、引き続き勉強していこうと思います。


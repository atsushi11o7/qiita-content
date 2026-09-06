---
title: PyTorch しか使ったことがなかった自分が、シミュレータの高速化で JAX の vmap を使用してみた話
tags:
  - jax
  - PyTorch
  - Python
  - GPU
  - 機械学習
private: false
updated_at: '2026-09-06T14:47:02+09:00'
id: 0bf94ff412c035aaa22d
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

今まで GPU を活用するには PyTorch を脳死で使用していました。ニューラルネットの学習はもちろん、GPU に乗せて並列計算したいものは全部 PyTorch の Tensor に乗せればいい、という感じです。

最近、強化学習で使うシミュレータを GPU で高速化したいという場面に出くわし、そこで初めて JAX というライブラリを知りました。PyTorch が慣れ親しんだ手続き的な書き方のまま GPU に乗せられるのに対し、JAX は `grad`・`jit`・`vmap`・`pmap` といった関数変換を自由に組み合わせられるのが特徴です。今回は「モデルの学習」ではなく「シミュレータ本体の並列化」にこの特徴を使いました。

この記事は、その過程で学んだ JAX の基礎機能の備忘録です。`jax.grad` や `jax.jit` といった基本に加えて、今回のシチュエーションで一番役に立った`jax.vmap` の使用方法までをまとめます。JAX での本格的な機械学習にはまだ手を出していないので、Flax については深くは触れていません。

## そもそも JAX とは

JAX は Google が開発している数値計算ライブラリで、ざっくり言うと

- NumPy とほぼ同じ API（`jax.numpy`。`np` の代わりに `jnp` と書くだけで大体動く）
- 自動微分ができる（PyTorch の `.backward()` にあたるもの）
- XLA という別のコンパイラで GPU/TPU 向けに最適化されたコードにコンパイルできる

という特徴を持つライブラリです。

PyTorch と決定的に違うのは、JAX では `jit`・`grad`・`vmap` の対象になる関数を「純粋関数」として書く点です。つまり、

- 配列（`jnp.array`）は書き換えられない（immutable）。`x[0] = 1` のような代入はできず、`x = x.at[0].set(1)` のように「新しい配列を作って返す」書き方になる
- 関数の中でグローバル変数を書き換えたり、乱数を勝手に進めたりしない（乱数は毎回明示的に `key` を受け渡す）

なお、ニューラルネットワークを本格的に書きたい場合は、JAX の上に構築された Flax というライブラリを使用します（この記事では触れません）。

## インストール

GPU で JAX を使うには、`jax[cuda12]` のように extras を付けてインストールする必要があります。

```bash
pip install "jax[cuda12]"
```

素の `jax` だけを指定すると CPU 専用のビルドが入ってしまうので注意してください。

PyTorch の `.to('cuda')` のような明示的な転送も不要です。GPU が使える環境であれば、配列は自動的にデフォルトデバイスに置かれます。特定のデバイスを指定したい場合だけ `jax.device_put(x, device)` を使います。

## `jax.numpy`：NumPy 互換の API

JAX の基本は `jax.numpy`（`jnp`）です。NumPy とほぼ同じ API なので、`import numpy as np` を `import jax.numpy as jnp` に変えるだけで大体動きます。

```python
import jax.numpy as jnp

x = jnp.array([1.0, 2.0, 3.0])
y = x * 2
print(y)  # [2. 4. 6.]
```

## `jax.grad`：自動微分の基本

`jax.grad` は、渡した関数を微分した新しい関数を返す機能です。PyTorch では `loss.backward()` を呼んで `.grad` を見にいきますが、JAX では「関数を微分した別の関数を作る」という発想になります。

```python
import torch

x = torch.tensor(3.0, requires_grad=True)
y = x ** 2
y.backward()
print(x.grad)  # tensor(6.)
```

```python
import jax

def square(x):
    return x ** 2

grad_square = jax.grad(square)
print(grad_square(3.0))  # 6.0 (d/dx x^2 = 2x)
```

この例はスカラー演算だけなので `jnp` は使っていませんが、配列を渡しても同じように微分できます。`jax.grad(square)` は「`square` を微分した新しい関数」を返します。これも後述する `jit`・`vmap` と同じく「関数を受け取って別の関数を返す」変換（transformation）の一つで、JAX では `jit`・`vmap`・`grad` はすべてこの「関数変換」という同じ枠組みに乗っています。シミュレータの高速化では勾配計算自体は使っていませんが、JAX の柱として押さえておきます。

## `jax.jit`：関数をコンパイルして速くする

まず基本となるのが `jax.jit` です。関数を `jax.jit(fn)` でラップするだけで、その関数の計算グラフを XLA がコンパイルし、GPU 上で効率よく実行できるようになります（`@jax.jit` のようにデコレータとしても使えます）。

```python
import jax
import jax.numpy as jnp

def add_one(x):
    return x + 1

fast_add_one = jax.jit(add_one)
```

これだけだと「PyTorch の `torch.compile` と同じでは？」と思うかもしれません。実際、役割としては近いです。

### jit の注意点：引数の形が変わると再コンパイルされる

`jax.jit` は「初めて見る引数の形（shape/dtype）」ごとにコンパイルし、その結果をキャッシュします。同じ形の引数で呼ぶ限りは2回目以降キャッシュが使われますが、形が変わると裏で再コンパイルが走ります。

```python
import jax
import jax.numpy as jnp

count = 0
def f(x):
    global count
    count += 1
    return x + 1

jit_f = jax.jit(f)
jit_f(jnp.array(1))
jit_f(jnp.array(2))
print(count)  # 1 (同じ形なのでキャッシュが効く)
jit_f(jnp.array([1, 2]))
print(count)  # 2 (形が変わったので再コンパイル)
```

「for ループの中で毎回微妙に違う形の配列を渡してしまい、ループのたびに再コンパイルされて逆に遅くなる」みたいなことがあるようです。

### 「値によって配列の形が決まる」引数は `static_argnames` で渡す

盤面のサイズのように「その値によって配列の shape が決まる」引数は、普通にトレースさせると JAX が困ります（shape は実行前に決まっていないといけないため）。こういう引数は `static_argnames` で「コンパイル時定数として扱ってね」と明示します。

```python
import jax
import jax.numpy as jnp

def make_grid(size):
    return jnp.zeros((size, size))

fast_make_grid = jax.jit(make_grid, static_argnames=["size"])
grid = fast_make_grid(size=5)
print(grid.shape)  # (5, 5)
```

static にした引数は「値が変わるたびに再コンパイルされる通常の引数」として扱われる点に注意です（shape を決めるためにはそれで正しいのですが、頻繁に変える値を static にすると再コンパイル地獄になります）。

実際には `jax.jit(jax.vmap(fn))` のように `vmap` と組み合わせて使うことが多いです。

## `jax.vmap`：for ループを書かずにバッチ化する

### やりたかったこと

たとえば「1人分のシミュレーションを1ステップ進める関数」があるとします。これを1000人分、GPU 上で並列に実行したい、という状況を考えます。

PyTorch 的な発想だと、「1000人分のデータをまとめて Tensor にして、その関数自体を最初からバッチ処理できるように書き直す」ことになると思います。次元を1つ増やして、`sum` や `where` をどの軸に対して取るか全部考え直す、みたいな感じです。

JAX の `vmap`（vectorizing map）は、これを関数を書き直さずにやってくれます。

### 具体例

「1人分、目標点に達するかターン数上限に達するまでサイコロを振り続ける」という単純な関数を考えます。`jax.lax.while_loop` の詳細は次節で説明しますが、ここでは「条件を満たす間ループする」くらいの理解で問題ありません。

```python
import jax
import jax.numpy as jnp

def play_one_game(key, target=20, max_turns=10):
    """1人分のシミュレーション: for ループも while ループも意識せず、
    「1人分」の処理だけを素直に書く。"""

    def cond(state):
        _, turn, score = state
        return (score < target) & (turn < max_turns)

    def body(state):
        key, turn, score = state
        key, subkey = jax.random.split(key)
        roll = jax.random.randint(subkey, (), 1, 7)  # 1〜6の目
        return key, turn + 1, score + roll

    init_state = (key, 0, 0)
    _, turns_used, final_score = jax.lax.while_loop(cond, body, init_state)
    return turns_used, final_score
```

ここまでは「1人分」の処理しか書いていません。これを1000人分並列に実行したい場合、`vmap` で包むだけです。

```python
keys = jax.random.split(jax.random.PRNGKey(0), 1000)  # 1000人分の乱数キー
turns_used, final_scores = jax.vmap(play_one_game)(keys)
# turns_used, final_scores はどちらも形状 (1000,) の配列
```

`play_one_game` の中身は変えていません。「先頭に長さ1000の軸が増えたら、その軸でバッチ処理してね」と `vmap` に指示しているだけです。for ループを1000回回すのではなく、1000個分をまとめたバッチ演算になり、`jit` と組み合わせれば XLA でコンパイルされます。正確な計測はしていませんが、逐次実行に比べて体感でもはっきり分かるくらい速くなりました。

PyTorch だと「バッチ対応した実装」を自分で書く必要がありますが、JAX では「1個分の実装」と「それをバッチ化する変換（`vmap`）」が分離されています。`vmap` は何重にも重ねがけできる（例：「1人分」→ `vmap` で「1ゲーム分（N人）」→さらに `vmap` で「M並列ゲーム分」）ので、部品を作る側は常に一番シンプルな粒度だけを考えればよくなります。

### `in_axes`：一部の引数だけバッチ化する

`vmap` はデフォルトで「全引数の先頭軸をバッチ軸とみなす」動きをしますが、「この引数はバッチ化せず、全インスタンスで共有したい」というケースもあります。その場合は `in_axes` で引数ごとに指定します。

```python
def step_one(state, shared_param):
    return state + shared_param

# 第1引数(state)はバッチ化(軸0)、第2引数(shared_param)はバッチ化しない(None)
batched_step = jax.vmap(step_one, in_axes=(0, None))

states = jnp.arange(5)
result = batched_step(states, 10)
print(result)  # [10 11 12 13 14]
```

「N個のインスタンスそれぞれの状態」と「全員共通の設定値（例：難易度パラメータ）」が混ざっている関数を `vmap` するときによく使う書き方です。

## `jax.lax.scan` / `jax.lax.while_loop`：JAX の中でループを書く

先ほどの例で `jax.lax.while_loop` という関数を使いました。

JAX は関数を XLA でコンパイルする都合上、Python の `for` や `while` をそのままトレースすると、ループが「展開」されてしまいます（10回ループなら10回分の計算グラフが埋め込まれる）。ターン数が可変だったり、そもそも大きい回数だったりすると、これはコンパイル時間・メモリの両方で問題になります。そこで JAX は専用のループ用関数を用意しています。

- **`jax.lax.scan`**：「決まった回数」ループを回しながら、途中経過も含めて結果を集める。Python の `for` に近い。微分可能（勾配計算の対象にできる）
- **`jax.lax.while_loop`**：条件を満たす間ループを回す。回数が可変な処理に向くが、reverse-mode の自動微分（`grad` の逆伝播）には非対応

### scan で書いていたときの問題

最初は「1ターンごとに何かを処理する」部分を `lax.scan` で固定回数分回していました。`vmap` で大量のインスタンスをバッチ処理する場合、`scan` は「全インスタンス、常に同じ回数だけループを回す」ので実装は簡単です。

ただし、処理の中身によっては「実際は数回で終わるはずの処理」を毎回上限回数まで律儀に回してしまい、無駄が大きいケースがありました。

### while_loop への切り替え

そこで、「条件を満たしたら早期終了する」処理には `lax.while_loop` を使うように書き換えました。

注意点として、（torchとかもそうですが）`vmap` で `while_loop` をバッチ化すると、各インスタンスの終了タイミングがバラバラでも、全インスタンスの条件が揃うまでループ自体は続きます。

そのため `while_loop` の恩恵（早期終了による高速化）は、「バッチ内のインスタンスの終了タイミングがある程度揃っている」場合ほど大きく、「1つだけ極端に長引くインスタンスが混ざっている」場合は伸びが小さくなります。

## JAX の注意点

そこまで深く使い込んだわけではなく、簡単に調べた範囲での話ですが、気づいた点をまとめておきます。

- `if` で分岐したい場合、値の選択だけなら `jnp.where`、計算コストが違う分岐を切り替えたいなら `jax.lax.cond` を使います（`jnp.where` は両方の分岐を計算してから選ぶため）
- 配列の一部だけ更新したいときは `x.at[i].set(v)` のように書く（`x[i] = v` はできない）
- `jit` の中で `print(x)` を書いても実行時の値は見えない。`jax.debug.print` を使う
- 乱数はグローバル状態を持たないので、`key` を明示的に受け渡す（`jax.random.split` で分岐させる）

## おわりに

JAX は「NumPy 互換 API（`jnp`）＋ 自動微分（`grad`）＋ XLA コンパイル（`jit`）」というライブラリで、`grad`・`jit`・`vmap` はいずれも「関数を受け取って別の関数を返す」という共通の枠組みに乗っています。`vmap` は1個分の処理を書くだけでバッチ化できる変換で、`lax.scan`（固定回数）と `lax.while_loop`（可変回数）を使い分けることで、シミュレータの並列実行を実装できました。

複数 GPU/TPU への分散には `pmap` が使われてきましたが、現在は `jax.shard_map` が推奨されているようです。今回は1台の GPU 内のバッチ化が目的なので `vmap` のみ扱いました。

余談ですが、PyTorch 側にも `torch.vmap` という近い機能があるようです。そのうち JAX（や Flax）を使って本格的な機械学習の実装もやってみたいと思っています。


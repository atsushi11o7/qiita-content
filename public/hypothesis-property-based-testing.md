---
title: Hypothesisでproperty-based testingを試してみた
tags:
  - Python
  - テスト
  - Hypothesis
  - JAX
  - pytest
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

Python のプロジェクトといえば、pytest を使ってテストコードを書くのが一般的だと思います。

自分はこれまで、テストコードをあまり意識しておらず、個別の入出力例を1つずつ書く example-based testing だけを書いてきました。そもそも、それが example-based testing と呼ばれる概念であることも知りませんでした。

最近「property-based testing」という考え方を知りました。具体例を並べる代わりに、常に成り立つべき性質を書き、入力の方はツールに大量生成してもらう発想です。

そのためのライブラリである Hypothesis を触ってみました。この記事はその記録です。

## property-based testing とは

example-based testing は、「この入力ならこの出力になるはず」という具体例を1つずつ書くテストです。

```python
def test_add():
    assert add(2, 3) == 5
```

一方 property-based testing は、個別の例ではなく「どんな入力でも成り立つべき性質」を書きます。

```python
from hypothesis import given, strategies as st

@given(st.integers(), st.integers())
def test_add_commutative(a, b):
    assert add(a, b) == add(b, a)
```

入力そのものは書きません。`@given`で指定した範囲から、ツールがランダムに大量の値を生成して、このテスト関数へ流し込みます。1件でも性質が崩れる入力が見つかれば失敗します。

この考え方自体は新しいものではなく、Haskell の QuickCheck(Claessen & Hughes, 2000)が元祖のようです。Hypothesis は、その考え方を Python へ持ち込んだライブラリ、という位置づけになります。

## Hypothesis ライブラリについて

Hypothesis は、Python 向けの property-based testing ライブラリです。pytest や unittest と組み合わせて使えて、`pip`や`uv`からインストールできます。

https://hypothesis.readthedocs.io/

### 記法

基本は先ほど見た`@given`と strategies の組み合わせです。strategies は「何を生成するか」を宣言的に書く部品で、`st.integers()`・`st.text()`・`st.lists(st.integers())`のように、型や範囲を指定していきます。

Hypothesis の特徴としてよく挙げられるのが、**shrinking**(縮小)という機能です。試しに、浮動小数点数の割り算と掛け算が元に戻る、という(一見成り立ちそうな)性質を書いてみました。

```python
from hypothesis import given, strategies as st

@given(
    st.floats(allow_nan=False, allow_infinity=False, min_value=-1e10, max_value=1e10),
    st.floats(min_value=1, max_value=1e10),
)
def test_division_roundtrip(x, y):
    assert x / y * y == x
```

これを実行すると、案の定、浮動小数点の誤差で失敗します。

```text
Failing test case: test_division_roundtrip(
    x=1.0,
    y=1891559671.25,
)
```

大量に試した中から失敗を見つけるだけでなく、`x`の方は`1.0`というきりのいい値まで縮めてくれています。これが shrinking です。たまたま見つかった複雑な反例をそのまま返すのではなく、「それでも失敗する、もっと単純な入力」を自動的に探してから報告してくれるので、デバッグがかなり楽になります。

もう1つ、一度見つかった失敗例は`.hypothesis/`以下のデータベースにキャッシュされ、次回のテスト実行時に真っ先に再試行されるようです。一度壊れたことのある箇所を、毎回ゼロから探し直さなくて済む仕組みのようです。

### インストール方法

`uv`を使う場合は、開発依存として次のように追加できます。

```bash
uv add --group dev hypothesis
```

`pyproject.toml`の`dependency-groups`には、こんな形で入ります。

```toml
[dependency-groups]
dev = [
    "pytest>=8",
    "hypothesis>=6",
]
```

## 試してみた

実際に手元のコードへ使ってみることにしました。対象は、**容量に上限がある入れ物へ、別の入れ物の中身を移す処理**です。移した結果が容量を超える分は捨てられる、よくあるパターンです。

手元にあったのは、コンペ用コードの中の、ある容器から別の容器へ持ち物を無条件で移す関数でした。JAX で実装していたので、この関数も JAX の配列操作(`jnp`)と`jax.lax.fori_loop`で書かれています。

```python
def dump_inventory(shed, inventory, shed_capacity):
    def body(i, carry):
        shed, inv = carry
        take = jnp.minimum(inv[i], shed_room(shed, shed_capacity))
        return shed.at[i].add(take), inv.at[i].set(0)

    return jax.lax.fori_loop(0, C.N_SHED_ITEMS, body, (shed, inventory))
```

容量を超える分は破棄される実装なので、「移す側の中身がどれだけ多くても、移した先が容量を超えることは絶対に無い」という性質を検証してみます。

```python
item_array = st.lists(
    st.integers(min_value=0, max_value=1000), min_size=C.N_SHED_ITEMS, max_size=C.N_SHED_ITEMS
)

@given(item_array, item_array, st.integers(min_value=0, max_value=200))
def test_dump_never_exceeds_capacity(shed, inventory, shed_capacity):
    shed = jnp.array(shed)
    inventory = jnp.array(inventory)
    new_shed, new_inventory = dump_inventory(shed, inventory, shed_capacity)

    assert jnp.sum(new_shed) <= shed_capacity
```

これを実行すると、いきなり失敗しました。

```text
Failing test case: test_dump_never_exceeds_capacity(
    shed=[278, 371, 148, 301, 469, 128, 304, 204, 56, 676, 281, 1000],
    inventory=[193, 259, 199, 196, 73, 413, 743, 73, 251, 434, 521, 202],
    shed_capacity=11,
)
```

よく見ると`shed`の中身の時点で既に合計が`shed_capacity=11`を超えています。`dump_inventory`は「移す先は常に容量以内である」という前提の上で、そこへ中身を足す処理なので、そもそも呼び出す前から破綻している状態を作ってしまっていました。関数のバグではなく、テストの前提条件(precondition)が足りなかっただけです。

さらにもう1つ、別の失敗も出ました。

```text
hypothesis.errors.DeadlineExceeded: Test took 1815.53ms, which exceeds the deadline of 200.00ms.
```

これは`dump_inventory`が JAX の関数で、初回呼び出し時に JIT コンパイルが走るため時間がかかり、Hypothesis のデフォルトの deadline(1回の実行にかけてよい時間の上限)を超えてしまったのが原因でした。JAX のような、初回だけ重い処理には`@settings(deadline=None)`で deadline を切っておく必要があるようです。

この2点を直しました。`shed`の各要素を個別に `0〜shed_capacity` で生成するだけでは、12個分の合計が `shed_capacity` を超えてしまうので、残り予算を減らしながら1つずつ値を引く形で、合計そのものを構造的に抑えています。deadline も無効化しました。

```python
@st.composite
def valid_shed_and_capacity(draw):
    shed_capacity = draw(st.integers(min_value=0, max_value=200))
    shed = []
    remaining = shed_capacity
    for _ in range(C.N_SHED_ITEMS):
        value = draw(st.integers(min_value=0, max_value=remaining))
        shed.append(value)
        remaining -= value
    return shed, shed_capacity

@settings(max_examples=2000, deadline=None)
@given(valid_shed_and_capacity(), item_array)
def test_dump_never_exceeds_capacity(shed_and_capacity, inventory):
    shed, shed_capacity = shed_and_capacity
    shed = jnp.array(shed)
    inventory = jnp.array(inventory)
    new_shed, new_inventory = dump_inventory(shed, inventory, shed_capacity)

    assert jnp.sum(new_shed) <= shed_capacity
    assert jnp.all(new_inventory == 0)
    assert jnp.all(new_shed <= shed + inventory)
```

今度は2000件すべて通りました。「移す側がどれだけ多くても移した先は溢れない」「移した後の移す側は必ず空になる」「移した先の中身は元の移す先+移す側の合計を超えない」という3つの性質を、具体的な数値を自分で考えることなく確認できました。

## おわりに

今回 Hypothesis を使ってみて、入力のバリエーションを自分で考えなくて済むのは便利だと感じました。「容量を超えない」という性質さえ書けば、自分では思いつかなかった値の組み合わせも確かめてもらえます。

一方で、何を前提に、どんな性質を確かめるかは自分で考える必要があります。今回も前提条件を見直したり、JAX のコードでは deadline を調整したりと、それなりに手間はありました。

これからも普段の入出力確認は pytest で書きつつ、「どんな入力でも守られてほしい条件」がある箇所では Hypothesis も使っていこうと思います。

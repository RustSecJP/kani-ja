{% raw %}
# 非決定的な変数 {#nondeterministic-variables}

Kani では、ユーザが `kani::any()` を使って非決定的な（記号的とも呼ばれる）値を作れるようにすることで、プログラムとその実行パスについて推論できます。
Kani は「ビット精度」のモデル検査器です。つまり、Kani は変数のメモリの内容に代入したときに _有効になる_ ビット値の組み合わせを、すべて考慮します。
言い換えると、`kani::any()` は、その型にとって無効な値（Rust の未定義動作につながるもの）を生成すべきではありません。

Kani には最初から、ほとんどのプリミティブ型と一部の `std` の型に対する `kani::any()` の実装が含まれています。
このチュートリアルでは、`kani::any()` を使ってほかの型の記号的な値を作る方法を示します。

## 安全な非決定的な変数 {#safe-nondeterministic-variables}

在庫管理ツールを開発していて、その API についての性質を検証し始めたいとしましょう。
簡単な例を示します（[こちら](https://github.com/model-checking/kani/blob/main/docs/src/tutorial/arbitrary-variables/src/inventory.rs)にあります）。

```rust
use std::num::NonZeroU32;
use vector_map::VecMap;

pub type ProductId = u32;

pub struct Inventory {
    /// Every product in inventory must have a non-zero quantity
    pub inner: VecMap<ProductId, NonZeroU32>,
}

impl Inventory {
    pub fn update(&mut self, id: ProductId, new_quantity: NonZeroU32) {
        self.inner.insert(id, new_quantity);
    }

    pub fn get(&self, id: &ProductId) -> Option<NonZeroU32> {
        self.inner.get(id).cloned()
    }
}
```

かなり単純な証明ハーネスを書いてみましょう。`update` で挿入した値を `get` でちゃんと取得できることだけを確かめるものです。

```rust
    #[kani::proof]
    #[kani::unwind(3)]
    pub fn safe_update() {
        // Empty to start
        let mut inventory = Inventory { inner: VecMap::new() };

        // Create non-deterministic variables for id and quantity.
        let id: ProductId = kani::any();
        let quantity: NonZeroU32 = kani::any();
        assert!(quantity.get() != 0, "NonZeroU32 is internally a u32 but it should never be 0.");

        // Update the inventory and check the result.
        inventory.update(id, quantity);
        assert!(inventory.get(&id).unwrap() == quantity);
    }
```

ここでは `kani::any()` を 2 回使っています。

1. `id` の型は `ProductId` で、これは実際にはただの `u32` なので、どんな値でもかまいません。
2. 一方、`quantity` の型は `NonZeroU32` です。
Rust では、この型が `0` という値を持つことは未定義動作になります。

ここでは、`kani::any()` が返した値が実際にゼロでないことを確かめる追加のアサーションを入れました。
これを実行すると、検証が成功することが分かります。

```bash
cargo kani --harness safe_update
```

`kani::any()` は安全な Rust なので、Kani は型の不変条件が守られる型に対してだけこれを実装しています。
`NonZeroU32` の場合、これは `0` という値を決して返さないということです。
このハーネスに書いたアサーションは、この事実を示すために追加した検査にすぎず、証明の本質的な部分ではありません。

## 独自の非決定的な型 {#custom-nondeterministic-types}

`kani::any()` は Kani が証明ハーネスに非決定性を注入するために提供する唯一の方法ですが、Kani に同梱されている実装は、安全性を保証できる少数の `std` の型に対するものだけです。
`kani::any()` の実装がない型の非決定的な変数が必要なときは、次の選択肢があります。

1. 自分の型に `kani::Arbitrary` トレイトを実装します。そうすれば、自分や下流のクレートがその型で `kani::any()` を使えます。詳しくは [Arbitrary トレイトのドキュメント](./reference/arbitrary.html)を見てください。
2. [`bolero_generator::TypeGenerator` トレイト](https://docs.rs/bolero-generator/0.8.0/bolero_generator/trait.TypeGenerator.html)を実装します。
これにより、自分や下流のクレートが [Bolero](https://camshaft.github.io/bolero/) を通じて Kani を使えるようになります。
3. 非決定的な変数からオブジェクトを組み立てる関数を書きます。

ほとんどの場合、1 つ目の方法をおすすめします。
1 つ目の方法は単純で、慣習的です。この方法なら、`Option<MyType>` や配列のようなパラメータ化された型でも使えるようになります。
Kani には derive マクロが含まれていて、構造体や列挙型のすべてのフィールドも `kani::Arbitrary` トレイトを実装していれば、その構造体や列挙型に対して `kani::Arbitrary` を自動で導出できます。
今のところのこの方法の欠点の 1 つは、`kani` クレートが Kani に同梱されてはいるものの、まだ [crates.io](https://crates.io) では公開されていないことです。
そのため、Arbitrary の実装には `#[cfg(kani)]` アトリビュートを注釈する必要があります。
derive マクロには `#[cfg_attr(kani, derive(kani::Arbitrary))]` を使ってください。

2 つ目の方法は、ファジングやプロパティテストも適用できるようにしたい場合におすすめです。
そうすることの利点は[このブログ記事](https://model-checking.github.io/kani-verifier-blog/2022/10/27/using-kani-with-the-bolero-property-testing-framework.html)で説明されています。
`kani::Arbitrary` と同じく、このトレイトも `derive` マクロで使えます。
1 つ注意すべきことは、この型ではポインタを含む任意の値をユーザが生成できることです。
そのような場合、**任意になるのはポインタが指す値だけ** で、ポインタそのものではありません。

最後に、3 つ目の方法は、データ構造の大きさの上限のようなパラメータを渡す必要があるときにおすすめです。
（これについては次の節で詳しく説明します。）
この方法は、ほかのクレートからインポートした型の非決定的な変数を生成する必要があるときにも必要です。Rust では、外部のクレートで定義されたトレイトを、自分が所有していない型に実装することはできないからです。

いずれの場合も、この関数の中では、その構成要素に対して任意の値を生成することで、単に任意の値を返します。
非決定的な構造体を生成するには、そのフィールドそれぞれに非決定的な値を生成するだけです。
ベクタやほかのコンテナのような複雑なデータ構造では、空のものから始めて、（有界な）非決定的な個数の要素を追加できます。

例えば、単純な列挙型なら、Arbitrary の derive アトリビュートを注釈するだけで済みます。

```rust
#[derive(Copy, Clone)]
#[cfg_attr(kani, derive(kani::Arbitrary))]
pub enum Rating {
    One,
    Two,
    Three,
}
```

しかし、同じ列挙型が外部のクレートで定義されている場合は、簡単な工夫が使えます。

```rust
    pub fn any_rating() -> Rating {
        match kani::any() {
            0 => Rating::One,
            1 => Rating::Two,
            _ => Rating::Three,
        }
    }
```

ここでしていることは、非決定的な整数を使って、`Rating` のどの列挙子を返すかを決めることだけです。

> **注**: このコードを乱数の値を生成するものと考えると、この関数はひどく偏っているように見えます。
> 「1 と 2 以外のすべての整数」にマッチするので、圧倒的に `Three` を生成することになるでしょう。
> しかし、Kani は意味のある 3 つの可能性を見るだけで、それぞれを互いに区別して扱うことはありません。
> 整数の「割合」は関係ありません。

## 非決定的な変数を有界にする {#bounding-nondeterministic-variables}

`[T; N]` には（`T` に実装されていれば）`kani::any()` を使えます。この配列型は正確で一定の大きさを持つからです。
しかし、大きさ `N` までのスライス（`[T]`）がほしい場合、それには `kani::any()` を使えません。
同様に、`Vec` のようなもっと複雑なデータ構造に対する `kani::any()` の実装もありません。

非決定的なベクタの難しいところは、[前の章](./tutorial-loop-unwinding.html)で調べた理由により、たいていベクタの大きさを _有界_ にする必要があることです。
`kani::any()` 関数は引数を持たないので、上限を与えることができません。

これは、非決定的なベクタを持てないという意味ではありません。
自分で組み立てる必要があるというだけです。
上の例の証明ハーネスは、空の `Inventory` から始めて非決定的な要素を 1 つ挿入するだけで、大きさ `1` の非決定的な `Inventory` を組み立てています。あるいは、自分の型に `BoundedArbitrary` を実装または導出することもできます。これは `Arbitrary` と同じように振る舞いますが、コレクションに対する固定の上限を受け取ります。詳しくは[有界な任意値](./reference/bounded_arbitrary.html)のドキュメントを見てください。

### 練習 {#exercise}

（最初の例の）（有界な）非決定的な在庫を生成する関数を書いてみてください。

```rust
fn any_inventory(bound: u32) -> Inventory {
   // ここを埋める
}
```

すぐに気づくことの 1 つは、上限をとても小さくしなければならないことです。
Kani は、ヒープ確保を伴う非決定的な大きさのデータ構造には、（まだ！）うまくスケールしません。
上の `safe_update` のような証明ハーネスでも、`any_inventory(2)` から始めると、証明に数分かかるでしょう。

この練習のヒント: 「ちょうど `bound` の大きさ」と「`bound` までの大きさ」という 2 つの異なる振る舞いのどちらかを選べます。
両方試してみてください！

解答の一例は [`exercise_solution.rs`](https://github.com/model-checking/kani/blob/main/docs/src/tutorial/arbitrary-variables/src/exercise_solution.rs) にあります。

## まとめ {#summary}

この節では、次のことを見てきました。

1. `kani::any()` が、Kani が実装しているそれぞれの型について「安全な」値を返すこと。
2. ほかの型について非決定的な値を作るために、`kani::Arbitrary` を実装するか、単に関数を書く方法。
3. 大きさの上限が必要なため、`kani::any()` を実装できない型があること。
4. 練習として、`Inventory` の有界な大きさの非決定的な値を生成しました。
{% endraw %}

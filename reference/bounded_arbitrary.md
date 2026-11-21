{% raw %}
# 有界な非決定的変数 {#bounded-non-deterministic-variables}

これは実験的な機能で、本来は非有界な型に上限（bound）を設けられるようにします。たとえば `Vec<T>` には `Arbitrary` の実装がありません。ベクタはいくらでも大きくなれるからです。このような型について証明するひとつの方法は、問題を易しくして、ある上限までに限って性質を証明することです。もちろん、その証明は上限までしか有効ではありません。それでも、コードが正しいという確信を得るのに役立つことがあります。

## 例 {#example}

簡単な例として、ベクタを 2 回反転すると元のベクタに戻ることを、ある上限まで証明してみましょう。反転する関数は次のとおりです。

```rust
fn reverse_vector<T>(mut input: Vec<T>) -> Vec<T> {
    let mut reversed = vec![];
    for _ in 0..input.len() {
        reversed.push(input.pop().unwrap());
    }
    reversed
}
```

`BoundedAny` を使って証明ハーネスを書けます。

```rust
#[kani::proof]
#[kani::unwind(17)]
fn check_reverse_is_its_own_inverse() {
    // BoundedAny を使って、長さが最大 16 のベクタを構築する
    let input: Vec<bool> = kani::bounded_any::<_, 16>();

    let double_reversed = reverse_vector(reverse_vector(input.clone()));

    // 入力の各値が、2 回反転したリストの値と同じであることをアサートする
    for i in 0..input.len() {
        assert_eq!(input[i], double_reversed[i])
    }
}
```

あとは `kani` を使えば、サイズ 16 までのベクタについて、この反転関数が実際に自分自身の逆関数であることを証明できます。

## 証明の不完全性 {#proof-incompleteness}

これはこの関数が正しいことの完全な証明では**ない**、という点に注意することがとても重要です。この点をはっきりさせるために、`reverse_vector` の次のような悪い実装を考えてみましょう。

```rust
fn bad_reverse_vector<T: Default>(mut input: Vec<T>) -> Vec<T> {
    let mut reversed = vec![];
    for i in 0..input.len() {
        if i < 16 {
            reversed.push(input.pop().unwrap());
        } else {
            reversed.push(T::default())
        }
    }
    reversed
}
```

さきほどと同じハーネスは、これでも成功してしまいます！この実装は明らかに間違っているにもかかわらずです。もう少し大きな上限を試してさえいれば……。

このように、有界な証明は役に立つこともありますが、不完全でもあることに注意してください。複数の上限を試してみる価値があるかもしれません。

## 独自の BoundedArbitrary 実装 {#custom-bounded-arbitrary-implementations}

Kani は `BoundedArbitrary` の実装をいくつか提供していますが、自分で `BoundedArbitrary` を実装することもできます。

ほとんどの場合に使える derive マクロを用意しています。

```rust
#[derive(BoundedArbitrary)]
struct MyVector<T> {
    #[bounded]
    vector: Vec<T>,
    capacity: usize
}
```

どのフィールドに上限を設けるかを `#[bounded]` アトリビュートで指定しなければなりません。それ以外のフィールドはすべて `Arbitrary` を derive していなければなりません。

### 制限事項 {#limitations}

現在は、型全体に対して上限をひとつしか指定できず、上限を設けたフィールドはすべて同じ上限を使います。異なる上限が使えると便利な場合は、[issue を作成して](https://github.com/model-checking/kani/issues/new/choose)お知らせください。おそらくこの制限を取り除けます。
{% endraw %}

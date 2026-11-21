{% raw %}
# Arbitrary トレイト {#arbitrary-trait}

`Arbitrary` トレイトは、Kani の証明ハーネスで非決定的な値を生成するための基盤です。これを使うと、ある型のあり得るすべての値を表す記号的な値を作れます。

ある型が `Arbitrary` を実装するには、Kani がその型のあり得るすべての値を表現できなければなりません。そのため、非有界な型はこれを実装できません。`Vec` などの非有界な型を非決定的に表現するには、[`BoundedArbitrary` トレイト](./bounded_arbitrary.html)を参照してください。

## 概要 {#overview}

`Arbitrary` トレイトは、任意の（非決定的な）値を生成するメソッドを定義します。

```rust
pub trait Arbitrary
where
    Self: Sized,
{
    fn any() -> Self;
    fn any_array<const MAX_ARRAY_LENGTH: usize>() -> [Self; MAX_ARRAY_LENGTH] {
        [(); MAX_ARRAY_LENGTH].map(|_| Self::any())
    }
}
```

## 基本的な使い方 {#basic-usage}

証明ハーネスで任意の値を生成するには `kani::any()` を使います。

```rust
#[kani::proof]
fn verify_function() {
    let x: u32 = kani::any();
    let y: bool = kani::any();
    let z: [char; 10] = kani::any();
    
    // x はあり得るすべての u32 の値を表す
    // y は true と false の両方を表す
    // z は長さ 10 の配列を表し、各要素はあり得るすべての char の値を取れる
    my_function(x, y, z);
}
```

Kani はプリミティブ型と一部の標準ライブラリの型に対して `Arbitrary` を実装しています。実装の完全な一覧は[クレートのトレイトのドキュメント](https://model-checking.github.io/kani/crates/doc/kani/trait.Arbitrary.html#foreign-impls)を参照してください。 

## 制約付きの値 {#constrained-values}

任意の値に制約を加えるには `any_where` または `kani::assume()` を使います。

```rust
#[kani::proof]
fn verify_with_constraints() {
    let x: u32 = kani::any_where(|t| *t < 1000); // x を 1000 未満に制約する
    kani::assume(x % 2 == 0); // x を偶数に制約する
    
    // これで x は 0 から 998 までのすべての偶数を表す
    my_function(x);
}
```

## derive による実装 {#derive-implementations}

構造体や列挙型のすべてのフィールドやバリアントが `Arbitrary` を実装していれば、Kani はその型の `Arbitrary` 実装を自動的に derive できます。

### 構造体 {#structs}

```rust
#[derive(kani::Arbitrary)]
struct Point {
    x: i32,
    y: i32,
}

#[kani::proof]
fn verify_point() {
    let point: Point = kani::any();
    // point.x と point.y は任意の i32 の値を取れる
    process_point(point);
}
```

### 列挙型 {#enums}

```rust
#[derive(kani::Arbitrary)]
enum Status {
    Ready,
    Processing(u32),
    Error { code: (char, i32) },
}

#[kani::proof]
fn verify_status() {
    let status: Status = kani::any();
    // `status` はどのバリアントにもなり得る
    match status {
        Status::Ready => { /* ... */ }
        Status::Processing(id) => { /* id は任意の u32 を取れる */ }
        Status::Error { code } => { /* code は任意の (char, i32) のタプルを取れる */ }
    }
}
```

## 手動での実装 {#manual-implementations}

制約や独自のロジックが必要なときは、`Arbitrary` を手動で実装します。たとえば Kani は、ゼロの値を除外するために [`NonZero` 型に対して `Arbitrary` を手動で実装しています](https://github.com/model-checking/kani/blob/100857e99d7506992c4589332a0d7d8dae1ee29a/library/kani_core/src/arbitrary.rs#L48-L60)。例:

```rust
impl Arbitrary for NonZeroU8 {
    fn any() -> Self {
        let val = u8::any();
        kani::assume(val != 0);
        unsafe { NonZeroU8::new_unchecked(val) }
    }
}
```

値に制約を加える別の手段として、[Invariant トレイト](https://model-checking.github.io/kani/crates/doc/kani/invariant/trait.Invariant.html)もあります。

## 関連項目 {#see-also}

- [非決定的な変数のチュートリアル](../tutorial-nondeterministic-variables.html)
- [有界な非決定的変数](./bounded_arbitrary.html)
- [はじめの一歩のチュートリアル](../tutorial-first-steps.html)
{% endraw %}

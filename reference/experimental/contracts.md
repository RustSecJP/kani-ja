{% raw %}
# 契約 {#contracts}

次の例を考えてみましょう。

```rust
fn gcd(mut max: u8, mut min: u8) -> u8 {
    if min > max {
        std::mem::swap(&mut max, &mut min);
    }

    let rest = max % min;
    if rest == 0 { min } else { gcd(min, rest) }
}
```
`gcd` を呼び出すコードを検証したいとしましょう。
[最悪の場合](https://en.wikipedia.org/wiki/Euclidean_algorithm#Worst-case)、`gcd` のステップ（再帰）の回数は、入力の数を表すのに必要なビット数の 1.5 倍に近づきます。 
そのため、2 つの大きな 64 ビットの数に対しては、`gcd` を 1 回呼ぶだけで 96 回近くの反復がかかることがあります。
Kani がこれらの反復をそれぞれ展開してから記号実行を行うのは、とてもコストがかかります。

その代わりに、`gcd` の振る舞いについての保証を記した*契約*を書けます。
`gcd` の契約が正しいことを Kani が検証すれば、`gcd` の各呼び出しをその契約で置き換えられます。これにより、`gcd` の呼び出し元の検証時間が短くなります。
たとえば、返される `result` が実際に `max` と `min` の両方を割り切ることを保証したいとしましょう。
その場合、次のような契約を書けます。

```rust
#[kani::requires(min != 0 && max != 0)]
#[kani::ensures(|result| *result != 0 && max % *result == 0 && min % *result == 0)]
#[kani::recursion]
fn gcd(mut max: u8, mut min: u8) -> u8 { ... }
```

`gcd` は `max % min` を計算する（そして場合によってはそれらの値を入れ替える）ので、引数にゼロを渡すとゼロ除算が起こるおそれがあります。
`requires` 契約は、このエラーに遭遇しないように、非決定的な入力の範囲をゼロでないものに制限するよう Kani に指示します。
`ensures` 契約は、結果が入力の正しい約数であることを実際にチェックするものです。
（再帰関数に契約を使うときは `recursion` アトリビュートが必要です。）

次に、それらの契約を*検証*するハーネスを、次のように書きます。

```rust
#[kani::proof_for_contract(gcd)]
fn check_gcd() {
    let max: u8 = kani::any();
    let min: u8 = kani::any();
    gcd(max, min);
}
```

そして `kani -Z function-contracts` を実行して検証します。

Kani が契約を検証したら、Kani の[スタブ機能](stubbing.html)を使って、`gcd` のすべての呼び出しをその契約で置き換えられます。たとえば次のようにします。

```rust
// foo() が gcd() を呼び出すと仮定する。
// stub_verified を使うことで、gcd() の呼び出しを
// 検証済みの契約で置き換えるよう Kani に指示する。
#[kani::proof]
#[kani::stub_verified(gcd)]
fn check_foo() {
    let x: u8 = kani::any();
    foo(x);
}
```
スタブ機能を活用すると、（コストの高い）`gcd` の呼び出しを、その振る舞いの*検証済みの抽象化*で置き換えられます。これにより、`foo` の検証時間を大幅に短くできます。

> **注意:** 検証済みのスタブは、戻り値を `kani::any()` で任意の値にします（havoc）。
> その戻り値の型の `kani::Arbitrary` 実装がスタブ化された関数を呼び出すと、
> スタブが自分自身に再び入り、再帰が終わらなくなります。Kani はこれを、
> 呼び出し経路を示すコンパイル時エラーとして報告します。これを直すには、
> `Arbitrary` を derive する（これはユーザの関数を呼ばずにフィールドごとに値を生成します）か、
> `Arbitrary` の実装からスタブ化された関数を呼ばないようにしてください。

契約について学ぶべきことは、ほかにもたくさんあります。
[契約についてのブログ記事](https://model-checking.github.io/kani-verifier-blog/2024/01/29/function-contracts.html)（この `gcd` の例はここから取ったものです）を読むことを強くおすすめします。また、[ドキュメント](../../crates/index.html)の `contracts` モジュールを見ることもおすすめします。
{% endraw %}

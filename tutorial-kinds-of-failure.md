{% raw %}
# Kani が見つけられる失敗 {#failures-that-kani-can-spot}

[前の節](./tutorial-first-steps.html)では、Kani が 2 つの主な種類の失敗、つまりアサーションとパニックを見つけるのを見ました。
証明ハーネスがパニックに至るプログラムの実行を許しているなら、Kani はそれを失敗として報告します。
さらに、（ごく簡単に）ほかの種類の失敗もいくつか見ました。ヌルポインタの参照外しとオーバーフローです。
この節では、これらの追加の検査を詳しく見ていき、Kani がほかにどんな問題を見つけるかを紹介します。

## 境界検査とポインタ {#bounds-checking-and-pointers}

Rust はデフォルトで安全なので、必要なところには動的な（実行時の）境界検査が含まれます。
次の Rust コードを考えてみましょう（[こちら](https://github.com/model-checking/kani/blob/main/docs/src/tutorial/kinds-of-failure/src/bounds_check.rs)にあります）。

```rust
/// Wrap "too-large" indexes back into a valid range for the array
fn get_wrapped(i: usize, a: &[u32]) -> u32 {
    if a.len() == 0 {
        return 0;
    }
    return a[i % a.len() + 1];
}
```

このコードに対しても、簡単なプロパティテストを書けます。

```rust
    proptest! {
        #[test]
        fn doesnt_crash(i: usize, a: Vec<u32>) {
            get_wrapped(i, &a);
        }
    }
```

Rust に組み込みの境界検査のおかげで、このプロパティテストはすぐに失敗するケースを見つけます。

では、この関数を `unsafe` な Rust を使うように変えたらどうなるでしょうか？

```rust
return unsafe { *a.as_ptr().add(i % a.len() + 1) };
```

すると、このテストからはエラーが見えなくなります。

```
# cargo test
[...]
test bounds_check::tests::doesnt_crash ... ok
```

プロパティテストは依然として範囲外アクセスを引き起こしていますが、この未定義動作は必ずしもすぐにクラッシュを起こすわけではありません。
（これが、未定義動作のデバッグがとても難しい理由の 1 つです。）
`unsafe` なコードを使うことで、範囲外アクセスに対する実行時の検査を取り除いてしまいました。
たまたま、ランダムに生成されたテストのどれも、実際にクラッシュする振る舞いを引き起こさなかったというだけです。
しかし、Kani の証明ハーネスを書いて、

```rust
#[cfg(kani)]
#[kani::proof]
fn bound_check() {
    let size: usize = kani::any();
    kani::assume(size < 4096);
    let index: usize = kani::any();
    let array: Vec<u32> = vec![0; size];
    get_wrapped(index, &array);
}
```

この証明を次のように実行すると、

```bash
cargo kani --harness bound_check
```

Rust の実行時の境界検査がなくても、Kani はやはり失敗を示します。

> また、検証の出力にたくさんの検査があったことにも注目してください。
> （執筆時点では 345 個です。）
> これは標準ライブラリの `Vec` の実装を使った結果です。つまり、見た目は短くても、このハーネスは実際にはかなりの量のコードを使っています。
> Kani は、コード中のアサートとして現れるよりずっと多くの検査を挿入しているので、出力が大きくなることがあります。

最後に次の要約が得られます。

```
SUMMARY: 
 ** 1 of 345 failed (8 unreachable)
Failed Checks: dereference failure: pointer outside object bounds
 File: "./src/bounds_check.rs", line 11, in bounds_check::get_wrapped

VERIFICATION:- FAILED
```

Kani にとっては、これが単純な境界検査の問題から、ポインタの検査の問題に変わったことに注目してください。
Kani はポインタに対する操作を検査し、それが不正なメモリアクセスになりえないことを確かめます。
ポインタを操作する `unsafe` なコードは、ここで見たように、その振る舞いに実際に問題があれば失敗を報告されます。

この例を使って、さらにいくつか小さな練習を試してみてください。

1. 練習: 普通の（安全な）インデックス操作に戻して、もう一度 Kani を試してください。
`unsafe` な操作と比べて、Kani の出力はどう変わるでしょうか？
（答えを予想してから、当たっていたか確かめてみてください。）
2. 練習: この例に Kani の実験的な[具体的な再生](reference/experimental/concrete-playback.html)フィーチャを試してください。
3. 練習: エラーを直して Kani を実行し、検証が成功するのを確かめてください。
4. 練習: `unsafe` なコードに（エラーを直した状態で）戻して、もう一度 Kani を実行してください。それでも検証は成功するでしょうか？

<details>
<summary>クリックすると練習 1 の説明を表示します</summary>

安全なインデックス操作に戻すと、Kani は境界検査の失敗を報告します。

```
SUMMARY:
 ** 1 of 343 failed (8 unreachable)
Failed Checks: index out of bounds: the length is less than or equal to the given index
 File: "src/bounds_check.rs", line 11, in bounds_check::get_wrapped

VERIFICATION:- FAILED
```

</details>

<details>
<summary>クリックすると練習 2 の説明を表示します</summary>

`cargo kani -Z concrete-playback --concrete-playback=inplace --harness bound_check` を実行すると、次のテストが生成されます。
```
rust
#[test]
fn kani_concrete_playback_bound_check_4752536404478138800() {
    let concrete_vals: Vec<Vec<u8>> = vec![
        // 1ul
        vec![1, 0, 0, 0, 0, 0, 0, 0],
        // 18446744073709551615ul
        vec![255, 255, 255, 255, 255, 255, 255, 255],
    ];
    kani::concrete_playback_run(concrete_vals, bound_check);
}
```
これは、証明ハーネスに具体的な値 `size = 1` と `index = 2^64` を代入すると範囲外アクセスが起こることを示しています。

</details>

## オーバーフローと算術エラー {#overflow-and-math-errors}

上の関数の別の版を考えてみましょう。

```rust
fn get_wrapped(i: usize, a: &[u32]) -> u32 {
    return a[i % a.len()];
}
```

範囲外アクセスは直しましたが、今度は「基底ケース」、つまり空のリストのときに何を返すかを省いてしまいました。
Kani はこれを境界のエラーとしてではなく、算術エラーとして見つけます。空のリストでは、剰余演算子（`%`）がゼロ除算を起こすからです。

1. 練習: この版の `get_wrapped` に対して Kani を実行し、この種の失敗がどう見えるかを確かめてください。

Rust は、境界検査と同じように、整数オーバーフローについても実行時の安全性検査を行えます。
（[Rust は `--release` モードではデフォルトでこれを無効にしますが、再び有効にできます。](https://doc.rust-lang.org/reference/expressions/operator-expr.html#overflow)）
次のコードを考えてみましょう（[こちら](https://github.com/model-checking/kani/blob/main/docs/src/tutorial/kinds-of-failure/src/overflow.rs)にあります）。

```rust
fn simple_addition(a: u32, b: u32) -> u32 {
    return a + b;
}
```

ささいな関数ですが、これに対するプロパティテストを書くと、Rust の動的な検査のおかげで、失敗する入力がすぐに見つかります。
Kani もこれらの失敗を見つけます。
Kani の出力は次のとおりです。

```
# cargo kani --harness add_overflow
[...]
SUMMARY: 
 ** 1 of 2 failed
Failed Checks: attempt to add with overflow
 File: "./src/overflow.rs", line 7, in overflow::simple_addition

VERIFICATION:- FAILED
```

この問題は、オーバーフロー時の振る舞いを明示した Rust の代わりの算術関数を使えば直せます。
例えば、ラップアラウンドする振る舞いが意図したものなら、`a + b` の代わりに `a.wrapping_add(b)` と書けます。
そうすれば、Kani は問題を報告しなくなります。

### 練習: 古典的なオーバーフローの失敗 {#exercise-classic-overflow-failure}

多くの実装で非常に長い間残っていた、気づきにくいバグの古典的な例が、クイックソートの「中点を求める」処理です。
これは素朴に書くと、よく次のようになります（コードは[こちら](https://github.com/model-checking/kani/blob/main/docs/src/tutorial/kinds-of-failure/src/overflow_quicksort.rs)にあります）。

```rust
fn find_midpoint(low: u32, high: u32) -> u32 {
    return (low + high) / 2;
}
```

```
cargo kani --harness midpoint_overflow
```

Kani は上のコードのバグをすぐに見つけます。

1. 練習: この関数を、もうオーバーフローしないように直してください。
（ヒント: どの方法をとるかによっては、証明ハーネスに `high > low` という仮定を追加する必要があるかもしれません。
すぐには追加せず、追加しないと何が起こるかを見てください。頭の片隅に置いておくだけにしましょう。）
2. 練習: テストハーネスにアサーションを追加して、新しい実装が本当に正しく中点を求めることを証明してください。

<details>
<summary>クリックするとこれらの練習の解答を表示します</summary>

オーバーフローの問題を解決するとてもよくある方法は、次のようなものです。

```rust
return low + (high - low) / 2;
```

しかし、素朴にこれを試すと（試してみてください！）、新たにアンダーフローのエラーが見つかります。`high - low` は負の数になるかもしれませんが、型は `u32` です。
そのため、それを起こりえなくするために、上で提案した仮定を追加する必要があります。
（ただし、仮定を追加するということは、「間違った使い方」をする新たな道ができるということです。それは避けたいかもしれません！仮定を避けられるでしょうか？）

その後、「新しい実装が正しいことを証明する」にはどうすればよいのか疑問に思うかもしれません。
そもそも「正しい」とはどういう意味でしょうか？
多くの場合、正しさのよい近似を使います。例えば、2 つの実装（多くの場合、一方が何らかの意味でずっと「単純」なもの）が等価であることです。
証明ハーネスに書けるアサーションの一例を示します。

```rust
assert!(result as u64 == (a as u64 + b as u64) / 2);
```

そもそもオーバーフローの問題を避ける方法として、このやり方を思いついた人もいるかもしれません！
異なるやり方を使った 2 つの異なる実装が同じ結果を返すと証明されていれば、正しい結果を計算しているという確信がより強まります。

</details>

## Kani が見つけられない失敗 {#failures-that-kani-cannot-spot}

Kani が行わない検査についての情報は、[制限事項](./limitations.html)を見てください。
特に、Kani は Rust に固有の未定義動作の概念のすべてを優先しているわけではありません。

## まとめ {#summary}

この節では、次のことを見てきました。

1. Kani が範囲外アクセスを見つけること。
2. Kani が、不正なメモリを指す生ポインタの、実際に安全でない参照外しを見つけること。
3. Kani がゼロ除算のエラーとオーバーフローする加算を見つけること。
4. 練習として、まったく自明というわけではないアサーション（中点を求める）の証明を試しました。
{% endraw %}

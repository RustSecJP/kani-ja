{% raw %}
# 最初の一歩 {#first-steps}

Kani は、すでになじみがあるかもしれないテストツールとは違います。
テストの多くは、カバーすべき新しいコーナーケースを考え出すことに関わっています。
Kani では、すべてのコーナーケースが最初からカバーされています。新たに気にすべきことは、検証器が扱える範囲にまで対象を絞り込むことです。

この最初のプログラムを考えてみましょう（[`first-steps-v1`](https://github.com/model-checking/kani/tree/main/docs/src/tutorial/first-steps-v1/) にあります）。

```rust
fn estimate_size(x: u32) -> u32 {
    if x < 256 {
        if x < 128 {
            return 1;
        } else {
            return 3;
        }
    } else if x < 1024 {
        if x > 1022 {
            panic!("Oh no, a failing corner case!");
        } else {
            return 5;
        }
    } else {
        if x < 2048 {
            return 7;
        } else {
            return 9;
        }
    }
}
```

この関数をテストするために書く必要があるテストハーネスについて考えてみてください。
それぞれの分岐を実行させるような、関数を呼び出すための引数の一式を考え出す必要があるでしょう。
また、分岐の一部が変わった場合に備えて、そのテストハーネスをコードに合わせて最新に保つ必要もあるでしょう。
そして、この関数がもっと複雑だったら（例えば、分岐の一部がグローバルな状態に依存していたら）、テストハーネスを書くのはさらに大変になるでしょう。

このような関数をプロパティテストしてみることもできますが、素朴に（あらゆる `u32` の入力を考えて）行うと、バグを見つけられる見込みはほとんどありません。

```rust
    proptest! {
        #![proptest_config(ProptestConfig::with_cases(10000))]
        #[test]
        fn doesnt_crash(x: u32) {
            estimate_size(x);
        }
    }
```

```
# cargo test
[...]
test tests::doesnt_crash ... ok
```

失敗する入力は 40 億個に 1 個しかないので、100 万個のサンプルを使っても、プロパティテストがそれを見つける可能性は限りなく低いです。

`estimate_size` に対する Kani の[_証明ハーネス_](reference/attributes.html#kaniproof)を書いてみましょう。
これはテストハーネスによく似ていますが、ここでは `kani::any()` を使って、あらゆる `u32` の値を表せます。

```rust
#[cfg(kani)]
#[kani::proof]
fn check_estimate_size() {
    let x: u32 = kani::any();
    estimate_size(x);
}
```

```
# cargo kani
[...]
Runtime decision procedure: 0.00116886s

RESULTS:
Check 3: estimate_size.assertion.1
         - Status: FAILURE
         - Description: "Oh no, a failing corner case!"
[...]
VERIFICATION:- FAILED
```

Kani はすぐに失敗を見つけました。
注目すべきなのは、証明ハーネスに明示的なアサーションを書く必要がなかったことです。デフォルトで、Kani はさまざまな誤った状態を見つけます。その中には、到達可能な `panic` の呼び出しや、失敗する `assert` が含まれます。
もし Kani がこのハーネスで成功していたら、それは `estimate_size` でパニックを起こしうる入力が存在しないことの数学的な証明に相当します。

> デフォルトでは、Kani は失敗を報告するだけで、失敗がどのように起きたかは報告しません。
> この例では、失敗を引き起こす `x` の値の具体例が得られるとうれしいでしょう。
> Kani には、この目的のための（実験的な）[具体的な再生](reference/experimental/concrete-playback.html)フィーチャがあります。
> 練習として、この例に具体的な再生を適用して、Kani が何を出力するか見てみてください。


### 練習: ほかの失敗を試す {#exercise-try-other-failures}

この関数には明示的なパニックを入れましたが、Kani が見つける失敗はそれだけではありません。
ほかの種類のエラーもいくつか試してみてください。

例えば、パニックする代わりに、ヌルポインタを明示的に参照外ししてみることもできます。

```rust
unsafe { return *(0 as *const u32) };
```

ただし、注目すべきことに、ここでは Rust コンパイラが警告を出します。

```
warning: dereferencing a null pointer
  --> src/lib.rs:10:29
   |
10 |    unsafe { return *(0 as *const u32) };
   |                    ^^^^^^^^^^^^^^^^^^ this code causes undefined behavior when executed
   |
   = note: `#[warn(deref_nullptr)]` on by default
```

それでもただの警告なので、前と同じようにテストを失敗させずにコードを実行できます。
しかし、Kani はやはりこの問題を捕まえます。

```
[...]
RESULTS:
[...]
Check 2: estimate_size.pointer_dereference.1
         - Status: FAILURE
         - Description: "dereference failure: pointer NULL"
[...]
VERIFICATION:- FAILED
```

**練習: Rust コンパイラは文句を言わないのに、Kani は文句を言う例を見つけられますか？**

<details>
<summary>クリックすると答えの一例を表示します</summary>

```
return 1 << x;
```

オーバーフロー（加算、乗算、あるいはこの場合のように[大きすぎる量のビットシフト](https://github.com/rust-lang/rust/issues/10183)）も Kani は捕まえます。

```
RESULTS:
[...]
Check 1: estimate_size.assertion.1
         - Status: FAILURE
         - Description: "attempt to shift left with overflow"

Check 3: estimate_size.undefined-shift.1
         - Status: FAILURE
         - Description: "shift distance too large"
[...]
VERIFICATION:- FAILED
```

</details>

## アサーション、仮定、ハーネス {#assertions-assumptions-and-harnesses}

この例の関数は、実際には数千個の入力しか扱うように作られていないように見えるのに、何十億もの入力に対して検査されるのは少し変に思えます。
バグを直したうえで、入力に妥当な上限をアサートして、関数についてのこの事実をコードに書き込んでみましょう。
（新しいコードは [`first-steps-v2`](https://github.com/model-checking/kani/tree/main/docs/src/tutorial/first-steps-v2/) にあります。）

```rust
fn estimate_size(x: u32) -> u32 {
    assert!(x < 4096);

    if x < 256 {
        if x < 128 {
            return 1;
        } else {
            return 3;
        }
    } else if x < 1024 {
        if x > 1022 {
            return 4;
        } else {
            return 5;
        }
    } else {
        if x < 2048 {
            return 7;
        } else {
            return 9;
        }
    }
}
```

これで、それまで暗黙だった期待を明示的に述べました。つまり、この関数は大きすぎる入力で呼び出されるべきではありません。
しかし、この変更した関数を検証しようとすると、問題にぶつかります。

```
[...]
RESULTS:
[...]
Check 3: estimate_size.assertion.1
         - Status: FAILURE
         - Description: "assertion failed: x < 4096"
[...]
VERIFICATION:- FAILED
```

私たちが求めているのは、`estimate_size` の _事前条件_ です。
つまり、関数を呼び出すたびに常に真であるべきことです。
アサーションを先頭に置くことで、その期待が満たされない場合に関数がすぐに失敗するようにしています。

しかし、証明ハーネスは依然として任意の整数で、つまり関数の事前条件を満たさない整数でもこの関数を呼び出します。
それは……役に立つ結果でも、興味深い結果でもありません。
それがうまくいかないことはもう分かっています。
どうすれば、この関数の検証を再び成功させられるでしょうか？

これこそが、証明ハーネスを書く目的です。
（このアサーションで同じく失敗する）プロパティテストと同じように、事前条件を設定し、対象の関数を呼び出し、事後条件をアサートする必要があります。
今度は成功する、証明ハーネスの修正版を示します。

```rust
#[cfg(kani)]
#[kani::proof]
fn verify_success() {
    let x: u32 = kani::any();
    kani::assume(x < 4096);
    let y = estimate_size(x);
    assert!(y < 10);
}
```

## まとめ {#summary}

この節では、次のことを見てきました。

1. Kani が、パニック、アサーションの失敗、さらにはヌルポインタの `unsafe` な参照外しのようなほかの失敗も見つけること。
2. テストでは簡単に見つけられない失敗を Kani が見つけること。
3. 証明ハーネスの書き方と `kani::any()` の使い方。
4. 証明ハーネスで `kani::assume()` を使って事前条件を設定する方法。
{% endraw %}

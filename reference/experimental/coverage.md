{% raw %}
# カバレッジ {#coverage}

[はじめの一歩](../../tutorial-first-steps.html)の `estimate_size` の例を思い出してください。
そこでは、入力の範囲を 4096 未満の整数に制約する証明ハーネスを書きました。

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

本当に関数を完全にテストできたのか、気になるところです。
関数を修正したのに、新しい入力の範囲をカバーするように証明ハーネスの仮定を更新し忘れたらどうなるでしょうか？

幸い、Kani は証明ハーネスごとにカバレッジの指標を報告できます。
`first-steps-v2` ディレクトリで、次のコマンドを実行してみてください。

```
cargo kani --coverage -Z source-coverage --harness verify_success
```

これはハーネスを検証し、その後に各行のカバレッジ情報を表示します。
この場合、`estimate_size` の各行の後に `FULL` と表示されます。これは証明ハーネスが完全なカバレッジを提供していることを示します。

証明ハーネスの仮定を `x < 2048` に変えてみてください。
これでハーネスは、あり得るすべてのケースをテストしなくなります。
コマンドを再実行してください。
次の行が表示されます。

```
src/lib.rs, 24, NONE
```

これは、`x >= 2048` の場合を扱う 24 行目を、証明がもうカバーしていないことを示しています。
{% endraw %}

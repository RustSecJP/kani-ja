{% raw %}
# 具体的な再生 (concrete playback) {#concrete-playback}

あるチェックの結果が `FAILURE` になったとき、Kani はデバッグに役立つ `concrete-playback` オプションを提供しています。この機能は、具体的な反例を使って失敗した証明ハーネスを再生する Rust の単体テストケースを生成します。

具体的な再生を有効にすると、Kani は検証中に失敗したアサーションと、到達可能な cover 文に対して単体テストを生成します。

生成したテストは、Kani の playback サブコマンドを使って実行できます。

## 使い方 {#usage}

この機能を有効にするには、`-Z concrete-playback --concrete-playback=[print|inplace]` フラグを付けて Kani を実行します。
検証が失敗すると、Kani は具体的な反例を使って失敗した証明ハーネスを再生する Rust の単体テストケースを生成します。
具体的な再生のモードの意味は次のとおりです。
* `print`: Kani は単体テストを標準出力に表示するだけです。
その後、この単体テストを証明ハーネスと同じモジュールにコピーする必要があります。
これは、`kani::any()` の呼び出しでどの値が割り当てられたかをすばやく知りたいだけの場合にも役立ちます。
* `inplace`: Kani は単体テストを自動的にソースコードにコピーします。
このモードを実行する前に、既存のコードを `git` にコミットしておくと便利かもしれません。
そうしておけば、`git revert` で単体テストを簡単に取り除けます。
なお、まったく同じテストがすでに存在することを検出した場合、Kani は単体テストをソースコードにコピーしません。 

単体テストがソースコードに入ったら、`playback` サブコマンドで実行できます。
デバッグするには、いくつかの選択肢があります。
* VSCode 向けに提供されている [Kani の実験的な拡張機能](https://github.com/model-checking/kani-vscode-extension)を試せます。
* そうでなければ、コマンドラインで単体テストをデバッグできます。

テストを手動でコンパイルして実行するには、Kani の `playback` サブコマンドを使えます。
```
cargo kani playback -Z concrete-playback -- ${unit_test_func_name}
```

このコマンドの出力は `cargo test` に似ています。
出力の最初のほうに、
`Running unittests {files} ({binary})` のような行があります。

`rust-gdb` や `lldb` などのツールで、このバイナリをさらにデバッグできます。

## 例 {#example}

次のソースファイルに対して `kani -Z concrete-playback --concrete-playback=print` を実行すると、
```rust
#[kani::proof]
fn proof_harness() {
    let a: u8 = kani::any();
    let b: u16 = kani::any();
    assert!(a / 2 * 2 == a &&
            b / 2 * 2 == b);
}
```
次のような具体的な再生の Rust 単体テストが得られます。
```rust
#[test]
fn kani_concrete_playback_proof_harness_16220658101615121791() {
    let concrete_vals: Vec<Vec<u8>> = vec![
        // 133
        vec![133],
        // 35207
        vec![135, 137],
    ];
    kani::concrete_playback_run(concrete_vals, proof_harness);
}
```
ここで、`133` と `35207` は、`a` と `b` に代入するとアサーションの失敗を引き起こす具体的な値です。
`vec![135, 137]` は `35207` のバイト配列による表現です。

## 意見の募集 {#request-for-comments}

この機能は実験的なものなので、変更される可能性があります。
この機能のユーザ体験を改善するアイデアがあれば、
[この GitHub の issue](https://github.com/model-checking/kani/issues/1536) に追加してください。
既存の機能要望は
[この GitHub のマイルストーン](https://github.com/model-checking/kani/milestone/10)で追跡しています。

## 制限事項 {#limitations}

* この機能は、パニック以外のチェック（たとえば UB チェック）の失敗に対しては単体テストを生成しません。
これは、具体的な再生の間、それらのチェックが実行時エラーを引き起こさないからです。
Kani はこれについて警告メッセージを出します。
* この機能は、同じハーネスの中で複数のアサーションが失敗した場合の単体テストの生成をサポートしていません。
この制限は将来取り除かれるかもしれません。
Kani はこれについて警告メッセージを出します。
* この機能では、テストの生成と再生に同じバージョンの Kani を使う必要があります。 
検証中に使った追加のコンパイルオプションは、再生のときにも使わなければなりません。
{% endraw %}

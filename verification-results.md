{% raw %}
# 検証結果 {#verification-results}

ハーネスに対して Kani を実行すると、次のような一連の検査を含む出力が得られます。

```
RESULTS:
Check 1: example.assertion.1
         - Status: <status>
         - Description: <description>
         - Location: <location>
[...]
```

Kani は、個々の検査（「性質」とも呼びます）の結果（つまり `<status>`）をもとに、
ハーネスの検証結果を決めます。すべての検査が成功すれば、
ハーネスの全体の検証結果は成功です。そうでなければ検証は失敗で、
検証対象のコードに問題があることを示します。

## 検査の結果 {#check-results}

Kani における検査の結果（つまり `Status`）は、次のいずれかです。

1. `SUCCESS`: 検査に合格した（つまり性質が成り立つ）ことを示します。
場合によっては、性質が _空虚に_ 成り立つことがあるので注意してください。
これは、性質に到達できない場合や、ハーネスが _制約しすぎ_ ている場合に起こります。

例:
```rust
fn success_example() {
    let mut sum = 0;
    for i in 1..4 {
        sum += i;
    }
    assert_eq!(sum, 6);
}
```
Kani の出力は、アサーションが成り立つことを示しています。
```
Check 4: success_example.assertion.4
         - Status: SUCCESS
         - Description: "assertion failed: sum == 6"
```

2. `FAILURE`: 検査に失敗した（つまり性質が成り立たない）ことを示します。
この場合は、[具体的な再生](./reference/experimental/concrete-playback.html)の節も参考にしてください。

例:
```rust
fn failure_example() {
    let arr = [1, 2, 3];
    assert_ne!(arr.len(), 3);
}
```
Kani の出力が示すとおり、アサーションは成り立ちません。
```
Check 2: failure_example.assertion.2
         - Status: FAILURE
         - Description: "assertion failed: arr.len() != 3"
```

3. `UNREACHABLE`: 検査に到達できない（つまり性質が _空虚に_ 成り立つ）ことを示します。
これは、検査のコード行に到達できる実行トレースが 1 つもない場合に起こります。
その原因としては、検査を含む関数が使われていない場合や、
検査が呼び出される条件をハーネスが引き起こさない場合が考えられます。
Kani は現在、次の種類のアサーションについて到達可能性を検査します。
    1. アサートマクロ（例: `assert`、`assert_eq` など）
    2. 算術オーバーフローの検査
    3. 符号反転のオーバーフローの検査
    4. インデックスの範囲外の検査
    5. ゼロ除算／ゼロによる剰余の検査

例:

```rust
fn unreachable_example() {
    let x = 5;
    let y = x + 2;
    if x > y {
        assert!(x < 8);
    }
}
```

Kani の出力は、アサーションに到達できないことを示しています。
```
Check 2: unreachable_example.assertion.2
         - Status: UNREACHABLE
         - Description: "assertion failed: x < 8"
```

4. `UNDETERMINED`: 性質が成り立つかどうかを Kani が結論づけられなかったことを示します。
これは、Kani が現在サポートしていない構文が Rust プログラムに含まれている場合に起こります。
Kani が現在 Rust 言語のフィーチャをどこまでサポートしているかは、
[Rust のフィーチャのサポート](./rust-feature-support.html)を見てください。

例:
```rust
fn undetermined_example() {
    let mut x = 0;
    unsupp(&mut x);
    assert!(x == 0);
}

#[feature(asm)]
fn unsupp(x: &mut u8) {
    unsafe {
        std::arch::asm!("nop");
    }
}
```
Kani の出力は、Kani がインラインアセンブリをサポートしていないために、
アサーションが未決定であることを示しています。
```
Check 2: undetermined_example.assertion.2
         - Status: UNDETERMINED
         - Description: "assertion failed: x == 0"
```

## カバー性質の結果 {#cover-property-results}

Kani には [`kani::cover`](https://model-checking.github.io/kani/crates/doc/kani/macro.cover.html) マクロがあります。これを使うと、コードのある地点である条件が起こりうるかどうかを検査できます。

カバー性質の結果は、次のいずれかです。

1. `SATISFIED`: 指定した条件を引き起こす実行を Kani が見つけたことを示します。

次の例では、`kani::cover` を使って、`while` ループを 3 回繰り返した後に `x` と `y` がそれぞれ 24 と 72 という値をとりうるかどうかを検査しています。結果として、そうなりえます。
```rust
#[kani::unwind(256)]
fn cover_satisfied_example() {
    let mut x: u8 = kani::any();
    let mut y: u8 = kani::any();
    y /= 2;
    let mut i = 0;
    while x != 0 && y != 0 {
        kani::cover!(i > 2 && x == 24 && y == 72);
        if x >= y { x -= y; }
        else { y -= x; }
        i += 1;
    }
}
```
結果:
```
Check 1: cover_satisfied_example.cover.1
         - Status: SATISFIED
         - Description: "cover condition: i > 2 && x == 24 && y == 72"
         - Location: src/main.rs:60:9 in function cover_satisfied_example
```

2. `UNSATISFIABLE`: 指定した条件が起こりえないことを Kani が _証明した_ ことを示します。

次の例では、`kani::cover` を使って、5 バイトからなる UTF-8 でエンコードされた文字列で、1 文字だけの文字列に対応するものがありうるかどうかを検査しています。
```rust
#[kani::unwind(6)]
fn cover_unsatisfiable_example() {
    let bytes: [u8; 5] = kani::any();
    let s = std::str::from_utf8(&bytes);
    if let Ok(s) = s {
        kani::cover!(s.chars().count() <= 1);
    }
}
```
そのような文字列は少なくとも 2 文字を含むので、これは起こりえません。
```
Check 46: cover_unsatisfiable_example.cover.1
         - Status: UNSATISFIABLE
         - Description: "cover condition: s.chars().count() <= 1"
         - Location: src/main.rs:75:9 in function cover_unsatisfiable_example
```

3. `UNREACHABLE`: `cover` 性質そのものに到達できない（つまり _空虚に_ 充足不能である）ことを示します。

アサーションに対する `UNREACHABLE` の結果とは違い、到達できない（または充足不能な）カバー性質は、証明が不完全であることを示しているかもしれません。

例:
この例では、`kani::cover` の呼び出しに到達できません。外側の `if` の条件が成り立つなら、空でない範囲 `r2` は空でない範囲 `r1` より真に大きく、その場合、内側の `if` の条件は起こりえないからです。
```rust
#[kani::unwind(6)]
fn cover_unreachable_example() {
    let r1: std::ops::Range<i32> = kani::any()..kani::any();
    let r2: std::ops::Range<i32> = kani::any()..kani::any();
    kani::assume(!r1.is_empty());
    kani::assume(!r2.is_empty());
    if r2.start > r1.end {
        if r2.end < r1.end {
            kani::cover!(r2.contains(&0));
        }
    }
}
```
```
Check 3: cover_unreachable_example.cover.1
         - Status: UNREACHABLE
         - Description: "cover condition: r2.contains(&0)"
         - Location: src/main.rs:90:13 in function cover_unreachable_example
```

4. `UNDETERMINED`: 通常の検査に対する `UNDETERMINED` の結果と同じです（[check_results][#check-results] を参照）。

## 検証の要約 {#verification-summary}

Kani は検証レポートの最後に要約を出力します。要約には、すべての検査の全体の結果、（パッケージにカバー性質が含まれていれば）カバー性質の全体の結果、そして全体の検証結果が含まれます。例えば次のとおりです。
```
SUMMARY:
 ** 0 of 786 failed (41 unreachable)

 ** 0 of 1 cover properties satisfied


VERIFICATION:- SUCCESSFUL
```

## SARIF 出力 {#sarif-output}

Kani は、GitHub Code Scanning などのツールと連携するために、SARIF（Static Analysis Results Interchange Format）を出力できます。

SARIF レポートを生成するには、次のようにします。
```bash
cargo kani --sarif kani.sarif
```

GitHub Actions では、SARIF ファイルを次のようにアップロードします。
```yaml
- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: kani.sarif
```

## JSON 出力 {#json-output}

Kani は実行の結果を JSON ファイルに書き出せます。
```bash
cargo kani -Z export-json --export-json kani.json
```

このファイルには、ツールのバージョン、設定、そして各ハーネスの結果が記述されます。この形式は安定しておらず、リリースの間で変わることがあります。スキーマについては [RFC 0015](https://model-checking.github.io/kani/rfc/rfcs/0015-export-json.html) を見てください。このフラグには `-Z export-json` が必要です。`-Z unstable-options` だけではもう足りません。

ビルドとハーネスの選択が成功すると、Kani は検証を始める前に、以前のファイルを `INCOMPLETE` マーカーで置き換え、最後にそのマーカーを最終的な文書で置き換えます。強制終了されたり失敗したりした実行では、マーカーが残ることがあります。マーカーが書き込まれる前に失敗した場合は、以前のファイルがそのまま残ります。その `started_at` を、実行した時刻と比べてください。
{% endraw %}

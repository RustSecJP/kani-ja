{% raw %}
# アトリビュート {#attributes}

Kani では、アトリビュートを使って関数をハーネスとして印を付け、その実行を制御します。
この節では、Kani で使えるアトリビュートと、それが検証の過程にどう影響するかを説明します。

現在、使える Kani のアトリビュートは次のとおりです。
 - [`#[kani::proof]`](#kaniproof)
 - [`#[kani::should_panic]`](#kanishould_panic)
 - [`#[kani::unwind(<number>)]`](#kaniunwindnumber)
 - [`#[kani::solver(<solver>)]`](#kanisolversolver)
 - [`#[kani::stub(<original>, <replacement>)]`](#kanistuboriginal-replacement)
 - [契約に関するアトリビュート](#contract-attributes)

## `#[kani::proof]` {#kaniproof}

**`#[kani::proof]` アトリビュートは、[関数](https://doc.rust-lang.org/reference/items/functions.html)が証明ハーネスであることを指定します。**

証明ハーネスはテストハーネス、特にプロパティベースのテストハーネスに似ています。
証明ハーネスでは Kani API の関数（たとえば `kani::any()`）を使えます。
証明ハーネスは Kani における検証の最小単位です。

Kani を `kani` と `cargo kani` のどちらで実行しても、Kani はまずすべての証明ハーネス
（つまり `#[kani::proof]` アトリビュートの付いた関数）を集め、それからそれらの検証を試みます。

### 例 {#example}

次の例で Kani を実行すると、

```rust
#[kani::proof]
fn my_harness() {
    assert!(1 + 1 == 2);
}
```

出力に `Checking harness my_harness...` という行が表示されるはずです（コード中のハーネスが `my_harness` だけだと仮定しています）。
その後に、CBMC（Kani が使う検証エンジン）からの複数のメッセージと[検証結果](../verification-results.html)が続きます。

`#[kani::proof]` なしでほかの Kani のアトリビュートを使うと、コンパイルエラーになります。

### 制限事項 {#limitations}

`#[kani::proof]` アトリビュートは、パラメータのない関数にしか付けられません。

## `#[kani::should_panic]` {#kanishould_panic}

**`#[kani::should_panic]` アトリビュートは、証明ハーネスがパニックすることが期待されていることを指定します。**

このアトリビュートを使うと、*否定的な検証*（negative verification）を行えます。
これは、Rust の単体テストで [`#[should_panic]`](https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html#testing-panics) を使って[否定的なテスト](https://en.wikipedia.org/wiki/Negative_testing)を行えるのと似ています。

このアトリビュートは**検証全体の結果にしか影響しません**。
具体的には、`#[kani::should_panic]` アトリビュートを使うと、次のいずれかの結果を返します。
  - 失敗したチェックがなかった場合は `VERIFICATION:- FAILED (encountered no panics, but at least one was expected)`。
  - 失敗したチェックはあったものの、そのすべてがパニックに関係するものではなかった場合は `VERIFICATION:- FAILED (encountered failures other than panics, which were unexpected)`。
  - それ以外の場合は `VERIFICATION:- SUCCESSFUL (encountered one or more panics as expected)`。

現時点では、チェックがパニックに関係するかどうかを判定するために、そのクラスが `assertion` かどうかを調べています。
クラスとは、プロパティ名の 2 番目の要素です。プロパティ名は `Check X: ` の後に表示される 3 つ組 `<function>.<class>.<number>` です。
たとえば、`Check 1: my_harness.assertion.1` のクラスは `assertion` なので、このチェックはパニックに関係するものとみなされます。

> **注意**: `#[kani::should_panic]` は、同じアトリビュートを使わない既存のハーネスを
> 補うハーネスを書く場合にのみおすすめします。
> 言い換えると、*否定的なハーネス*を書くのは、検証対象の関数について興味深い性質を
> うまく検証できる*肯定的な*ハーネスを書いた後にすることをおすすめします。

### 制限事項 {#limitations-1}

`#[kani::should_panic]` アトリビュートは、パニックに関係する失敗したチェックが 1 つ以上あることを検証します。
現時点では、それを特定のパニックに絞り込むことはできません。
そのため、検証対象のコードを変更した後で、**`#[kani::should_panic]` で検出されたパニックが、もともと期待していたものではない可能性があります**。

### 例 {#example-1}

次の例の `Device` を使っているとしましょう。

```rust
struct Device {
    is_init: bool,
}

impl Device {
    fn new() -> Self {
        Device { is_init: false }
    }

    fn init(&mut self) {
        assert!(!self.is_init);
        self.is_init = true;
    }
}
```

`device.init()` を 2 回以上呼ぶとパニックになることを検証したいとします。
次のハーネスでそれができます。

```rust
#[kani::proof]
#[kani::should_panic]
fn cannot_init_device_twice() {
    let mut device = Device::new();
    device.init();
    device.init();
}
```

これに対して Kani を実行すると、`VERIFICATION:- SUCCESSFUL (encountered one or more panics as expected)` という結果になります。

## `#[kani::unwind(<number>)]` {#kaniunwindnumber}

**`#[kani::unwind(<number>)]` アトリビュートは、すべてのループを `<number>` 回まで展開しなければならないことを指定します。**

デフォルトでは、Kani はすべてのループを自動的に展開しようとします。
しかし、この展開の過程は必ず終了するとは限りません。
`#[kani::unwind(<number>)]` アトリビュートは次のことを行います。
 1. 自動の展開を無効にします。
 2. すべてのループを `<number>` 回まで展開します。

展開の段階の後、Kani はハーネスの検証を試みます。
`#[kani::unwind(<number>)]` アトリビュートを指定した場合、1 つ以上のループが十分な回数だけ展開されていない可能性があります。
その場合、少なくとも 1 つの展開アサーション（unwinding assertion）が失敗し（展開アサーションはループごとに 1 つあります）、検証は失敗します。

展開について詳しくは、[*ループ、展開、上限* の節](../tutorial-loop-unwinding.html)を確認してください。

### 例 {#example-2}

ループを含む次のコードを書いたとしましょう。

```rust
fn my_sum(vec: &Vec<u32>) -> u32 {
    let mut sum = 0;
    for elem in vec {
        sum += elem;
    }
    sum
}

#[kani::proof]
fn my_harness() {
    let vec = vec![1, 2, 3];
    let sum = my_sum(&vec);
    assert!(sum == 6);
}
```

この例で Kani を実行すると、検証は成功します。
この場合、Kani は必要な展開の値（つまり、すべてのループを展開する必要のある回数）を自動的に見つけます。
これは、このあとすぐ見るように、`#[kani::unwind(<number>)]` アトリビュートが必要ないことを意味します。
一般に、必要な展開の値は、すべてのループの反復回数の最大値に 1 を足したものです。
この例で必要な展開の値は 4 です。`for elem in vec` ループの反復 3 回に 1 を足したものです。

`#[kani::unwind(3)]` でより小さな展開の値を強制するとどうなるか見てみましょう。

```rust
#[kani::proof]
#[kani::unwind(3)]
fn my_harness() {
    let vec = vec![1, 2, 3];
    let sum = my_sum(&vec);
    assert!(sum == 6);
}
```

先ほど述べたように、このハーネスを検証しようとすると展開の失敗が起こります。

```
SUMMARY:
 ** 1 of 187 failed (186 undetermined)
Failed Checks: unwinding assertion loop 0
 File: "/home/ubuntu/devices/src/main.rs", line 32, in my_sum

VERIFICATION:- FAILED
[Kani] info: Verification output shows one or more unwinding failures.
[Kani] tip: Consider increasing the unwinding value or disabling `--unwinding-assertions`.
```

展開アサーションの失敗が少なくとも 1 つあるため、Kani はこのハーネスを検証できません。
しかし、先ほど計算した正しい展開の値である `#[kani::unwind(4)]` を使えば、

```rust
#[kani::proof]
#[kani::unwind(4)]
fn my_harness() {
    let vec = vec![1, 2, 3];
    let sum = my_sum(&vec);
    assert!(sum == 6);
}
```

再び成功の結果が得られます。

```
SUMMARY:
 ** 0 of 186 failed

VERIFICATION:- SUCCESSFUL
```

## `#[kani::solver(<solver>)]` {#kanisolversolver}

**Kani の検証エンジン（CBMC）が使うソルバを変更します。**

これにより、ハーネスの検証に必要な時間が変わることがあります。

現在、`<solver>` には次のいずれかを指定できます。
 - `minisat`: [MiniSat](http://minisat.se/)。
 - `cadical`（デフォルト）: [CaDiCaL](https://github.com/arminbiere/cadical)。
 - `kissat`: [kissat](https://github.com/arminbiere/kissat)。
 - `z3`: [Z3](https://github.com/Z3Prover/z3/)。
 - `bitwuzla`: [Bitwuzla](https://github.com/bitwuzla/bitwuzla)。
 - `cvc5`: [cvc5](https://github.com/cvc5/cvc5)。
 - `bin="<SAT_SOLVER_BINARY>"`: 独自のソルバのバイナリ `"<SAT_SOLVER_BINARY>"`。パスの通った場所になければなりません。

### 例 {#example-3}

次の例では、Kani は CaDiCaL ソルバを使います。

```rust
#[kani::proof]
#[kani::solver(cadical)]
fn check() {
    let mut a = [2, 3, 1];
    a.sort();
    assert_eq!(a[0], 1);
    assert_eq!(a[1], 2);
    assert_eq!(a[2], 3);
}
```

ソルバを変えると、ハーネスによって検証時間が変わることがあります。

デフォルトのソルバは Kani のバージョンによって変わることがある点に注意してください。
ソルバの選択が性能に大きく影響する場合は、使うソルバが現在のデフォルトであっても、
ハーネスにアノテーションを付けておくことを強くおすすめします。

## `#[kani::stub(<original>, <replacement>)]` {#kanistuboriginal-replacement}

**コンパイル時に、名前が <original> の関数やメソッドを、名前が <replacement> の関数やメソッドで置き換えます。**

スタブについて詳しくは、[*スタブ* の節](../reference/experimental/stubbing.html)を確認してください。

## 契約のアトリビュート {#contract-attributes}

関数の契約とループの契約のためのアトリビュートが数多くあります。現在、それらは次のとおりです。

- 契約のための証明ハーネス: `#[kani::proof_for_contract(name-of-function)]`
- 検証済みのスタブ: `#[kani::stub_verified]`
- 関数の契約の指定: `#[kani::requires]`、`#[kani::modifies]`、`#[kani::ensures]`、`#[kani::recursion]`
- ループの契約の指定: `#[kani::loop_invariant]`、`#[kani::loop_modifies]`、`#[kani::loop_decreases]`。

詳しくは、[関数の契約](./experimental/contracts.html)と[ループの契約](./experimental/loop-contracts.html)のドキュメントを参照してください。
{% endraw %}

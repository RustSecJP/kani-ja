{% raw %}
# 遅い証明のデバッグ {#debugging-slow-proofs}

Kani は SAT/SMT ソルバを使ってコードを検証しますが、そのせいで証明が遅くなったり終了しなかったりすることがあります。この章では、遅くなる一般的な原因と、証明の性能をデバッグして改善するための戦略を概説します。

## 証明が遅くなる一般的な原因 {#common-causes-of-slow-proofs}

### 複雑な、または大きな非決定的な型 {#complexlarge-non-deterministic-types}
型によっては、記号的に表現するのがもともと高くつくものがあります。例えば、UTF-8 エンコーディングの複雑な検証規則を持つ文字列や、
大きな有界のコレクション（大きなサイズのベクタなど）です。

### 大きな値に対する演算 {#large-value-operations}
大きな値に対する算術演算は、特に大きな型（例: `u64`）では高くつくことがあります。例えば乗算、除算、剰余です。

### 有界でないループ {#unbounded-loops}
Kani がループの上限を決められない場合、いつまでも展開し続けます。[ループの展開のチュートリアル](./tutorial-loop-unwinding.html)を参照してください。

## デバッグの戦略 {#debugging-strategies}

遅い証明をデバッグするための戦略をいくつか、試すべきおおよその順に並べます。

### ループの繰り返しを制限する {#limit-loop-iterations}

まず、（有界でない）ループの展開が根本原因かどうかを見極めます。`#[kani::unwind]` アトリビュートか `--unwind` オプションを試して、[ループの展開](./tutorial-loop-unwinding.html)を制限してください。展開の値が小さすぎて証明が失敗し、かといって値を大きくすると証明が遅くなりすぎる場合は、代わりに[ループ契約](./reference/experimental/loop-contracts.html)を指定してみてください。

### 別のソルバを使う {#use-different-solvers}

Kani は複数の SAT/SMT ソルバをサポートしていて、ソルバによって特定の問題での性能が違うことがあります。`#[kani::solver]` [アトリビュート](./reference/attributes.html)か `--solver` オプションで、別のソルバを試してください。

### 非決定性の源を取り除く {#remove-sources-of-nondeterminism}

まず、`kani::any()` の呼び出しを具体的な値に置き換えて、問題を切り分けます。

```rust
#[kani::proof]
fn slow_proof() {
    // これの代わりに:
    // let x: u64 = kani::any();
    // let y: u64 = kani::any();
    
    // これを試します:
    let x: u64 = 42;
    let y: u64 = 100;
    
    let result = complex_function(x, y);
    assert!(result > 0);
}
```

具体的な値で証明が速くなるなら、問題はおそらく入力の記号的な表現にあります。その場合は、起こりうる値の異なる範囲をカバーするように[証明を分割](#partition-the-input-space)できるか確かめるか、用途として許容できるなら、証明をより狭い範囲の値に制限してください。

### コレクションの大きさを減らす {#reduce-collection-sizes}

同様に、証明でより小さな値が許容できるなら、代わりにそれを使います。

```rust
#[kani::proof]
fn test_with_small_collection() {
    // 大きな Vec の代わりに
    // let vec: Vec<u8> = kani::bounded_any::<_, 100>();
    
    // 小さな大きさから始めます
    let vec: Vec<u8> = kani::bounded_any::<_, 2>();
    
    process_collection(&vec);
}
```

### 入力空間を分割する {#partition-the-input-space}

入力空間を分割して、複雑な証明を分解します。

```rust
// 大きな入力を持つ 1 つの遅い証明の代わりに
#[kani::proof]
fn test_multiplication_slow() {
    let x: u64 = kani::any();
    let y: u64 = kani::any();
    
    // これはソルバにとって遅すぎるかもしれません
    let result = x.saturating_mul(y);
    assert!(result >= x || x == 0);
}

// 有界な入力を持つ複数の証明に分けます
#[kani::proof]
fn test_multiplication_small_values() {
    let x: u64 = kani::any_where(|x| *x <= 100);
    let y: u64 = kani::any_where(|y| *y <= 100);
    
    let result = x.saturating_mul(y);
    assert!(result >= x || x == 0);
}

// `x` と `y` のほかの範囲に対するハーネスを挿入します
```

このような分割を自動で行うサポートの追加については、この [tracking issue](https://github.com/model-checking/kani/issues/3006) を見てください。

### スタブを使う {#use-stubs}

関数の本体が複雑な場合は、[スタブ](./reference/experimental/stubbing.html)または[検証済みのスタブ](./reference/experimental/contracts.html)を使って、本体をより単純な抽象化でスタブすることを検討してください。

### 不要な検査を無効にする {#disable-unnecessary-checks}

安全性よりも機能的な正しさに焦点を当てているなら、メモリ安全性の検査を無効にできます（そのためのオプションの一覧は `kani --help` を実行して確認してください）。これらの検査を無効にすると、Kani が未定義動作を見逃すことがあるので、注意して使ってください。

あるいは、すべてのアサーションが成功すると仮定して、安全性の違反を見つけることだけに集中するには、`--prove-safety-only` オプションを使います。
{% endraw %}

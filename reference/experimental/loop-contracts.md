{% raw %}
# ループ契約 {#loop-contracts}

## はじめに {#introduction}

ループ契約は、Kani の *有界な証明* を *非有界な証明* に広げるために、ループの不変条件を指定するものです。
[ループ不変条件](https://en.wikipedia.org/wiki/Loop_invariant)とは、ループに入るときと、ループ本体を実行するたびにその後で成り立つ式のことです。
ループ契約は、1 つ以上のループ不変条件と、省略可能な `loop_modifies` アトリビュートで構成されます。
`#[kani::loop_invariant]` アトリビュートを複数持つループでは、それらの不変条件の論理積（`&&`）がそのループの不変条件になります。
ループ不変条件は、ループのどのステップでも変わらない性質を表します。

ここで、[有界な証明](../../tutorial-loop-unwinding.html#bounded-proof)と
[ループの展開](../../tutorial-loop-unwinding.html#loops-unwinding-and-bounds)についての説明を振り返っておくとよいでしょう。簡単に言うと、Kani がループを展開する回数に上限を設けると、入力のサイズにも上限がかかります。
そのため、証明は有界な証明になります。
ループ契約は、ループを展開せずに済むよう、ループをループのないブロックとして抽象化するために使います。これにより、入力にかかる上限をなくせます。

次の例を考えてみましょう。

``` Rust
fn simple_loop() {
    let mut x: u64 = kani::any_where(|i| *i >= 1);

    while x > 1 {
        x = x - 1;
    }

    assert!(x == 1);
}
```

このプログラムでは、ループが `x` を `1` になるまで減らし続けます。`x` の上限を指定していないので、この関数を検証するには、
Kani はループを `u64::MAX` 回展開する必要があります。これは現実的ではありません。ループ契約を使うと、ループの振る舞いを抽象化でき、検証のコストを大きく減らせます。

ループ契約を使うと、ループの振る舞いを不変条件で指定できます。例えば次のようにします。

``` Rust
#![feature(stmt_expr_attributes)]
#![feature(proc_macro_hygiene)]

fn simple_loop_with_loop_contracts() {
    let mut x: u64 = kani::any_where(|i| *i >= 1);

    #[kani::loop_invariant(x >= 1)]
    while x > 1 {
        x = x - 1;
    }

    assert!(x == 1);
}
```

ここでループ不変条件 `#[kani::loop_invariant(x >= 1)]` は、各反復の開始時、ループのガードを確認する前に、条件 `x >= 1` が成り立たなければならないことを指定しています。
Kani はループ不変条件が帰納的であることを検証すると、その不変条件を使ってループを抽象化し、展開を避けます。

では、ループ契約付きの証明を kani で実行してみましょう。
``` bash
kani simple_loop_with_loop_contracts.rs  -Z loop-contracts
```
この例に対して Kani が報告する出力は次のようになります。
```
...


Check 10: simple_loop_with_loop_contracts.loop_invariant_base.1
         - Status: SUCCESS
         - Description: "Check invariant before entry for loop simple_loop_with_loop_contracts.0"
         - Location: simple_while_loop.rs:15:5 in function simple_loop_with_loop_contracts

Check 11: simple_loop_with_loop_contracts.loop_assigns.1
         - Status: SUCCESS
         - Description: "Check assigns clause inclusion for loop simple_loop_with_loop_contracts.0"
         - Location: simple_while_loop.rs:15:5 in function simple_loop_with_loop_contracts

Check 13: simple_loop_with_loop_contracts.assigns.1
         - Status: SUCCESS
         - Description: "Check that x is assignable"
         - Location: simple_while_loop.rs:17:9 in function simple_loop_with_loop_contracts

Check 14: simple_loop_with_loop_contracts.loop_invariant_step.1
         - Status: SUCCESS
         - Description: "Check invariant after step for loop simple_loop_with_loop_contracts.0"
         - Location: simple_while_loop.rs:15:5 in function simple_loop_with_loop_contracts

Check 15: simple_loop_with_loop_contracts.loop_invariant_step.2
         - Status: SUCCESS
         - Description: "Check invariant after step for loop simple_loop_with_loop_contracts.0"
         - Location: simple_while_loop.rs:15:5 in function simple_loop_with_loop_contracts

...

SUMMARY:
 ** 0 of 99 failed

VERIFICATION:- SUCCESSFUL
Verification Time: 0.3897019s

Complete - 1 successfully verified harnesses, 0 failures, 1 total.
```


## 構文と意味 {#syntax-and-semantics}

### 構文 {#syntax}
> 
> \#\[kani::loop_invariant\( [_Expression_](https://doc.rust-lang.org/reference/expressions.html) \)\]
> 
>  [_LoopExpression_](https://doc.rust-lang.org/reference/expressions/loop-expr.html#grammar-LoopExpression)


不変条件の契約 `#[kani::loop_invariant(cond)]` は、ループと同じスコープから見える変数を使った、有効なブール式 `cond` を受け取ります。

### 意味 {#semantics}
ループ契約は、いくつかの仮定とアサーションに展開されます。
1. 最初の反復の直前で、不変条件をアサートします。
2. 非決定的な反復をモデル化するため、非決定的な状態の上で不変条件を仮定します。
3. 最後にもう一度不変条件をアサートし、帰納性を確かめます。

ここで働いている原理は数学的帰納法です。(1) が帰納法の基底段階を確かめ、(2) と (3) が帰納段階を確かめます。
したがって、ループを何回反復した後でも、不変条件は成り立たなければなりません。不変条件とループのガードの否定を合わせたものが、
その後のアサーションを示すのに十分でなければなりません。十分でなければ、抽象化が粗すぎるので、ユーザはより強い不変条件を与える必要があります。

要点を示すため、Kani が `simple_loop_with_loop_contracts` のループをループのないブロックとしてどう抽象化するかを示します。
``` Rust
assert!(x >= 1) // 基底段階でループ不変条件を確認する。
x = kani::any();
kani::assume(x >= 1);
if x > 1 {
    // 証明パス 1:
    //   ループのガードとループ不変条件の両方が満たされる。
    x = x - 1;
    assert!(x >= 1); // ループ不変条件が帰納的であることを確認する。
    kani::assume(false) // この証明パスを遮断する。
}
// 証明パス 2:
//   ループ不変条件は満たされ、ループのガードは満たされない。
assert!(x == 1);
```
つまり、基底段階でループ不変条件が成り立つことを確認した後、任意の反復の中にいると仮定します。帰納法の仮定（`kani::assume(x >= 1);`）のもとで、
ループに入る（証明パス 1）か、ループを抜ける（証明パス 2）かのどちらかになります。`kani::assume(false);` でパス 1 を打ち切ることで、2 つのパスを別々に証明します。
`kani::assume(false)` の後のアサーションはすべて無視されることに注意してください。どんな `p` に対しても `false => p` は `true` と導けるからです。

証明パス 1 では、ループ内部の性質を証明し、最後にループ契約が帰納的であることを確認します。

証明パス 2 では、ループを抜けた後の性質を証明します。ループを抜けるのはループのガードが満たされないときだけなので、ループの事後条件は
`!guard && inv` と表せます。この例では `x <= 1 && x >= 1` です。この事後条件から `x == 1` が導かれます。これが `simple_loop_with_loop_contracts` の最後で証明したい性質です。

### 部分正当性と完全正当性 {#partial-correctness-vs-total-correctness}

上の検証の手順で確かめられるのは、*部分正当性* と呼ばれるものです。つまり、ループが停止する**ならば**、結果は正しいということです。
これは、ループ不変条件だけで得られる標準的な保証です。

*完全正当性*、つまりループが停止し、**かつ**正しい結果を出すことを確かめるには、さらに
[decreases 節](#decreases-clauses-termination-proofs)が必要です。これは、ループが反復のたびに停止に向かって進んでいることを証明するものです。

形式的に言うと、ループの正しさを完全に論証するには、次の 4 つの手順があります。

1. **確立**（基底段階）：最初の反復の前に不変条件が成り立つ。
2. **保存**（帰納段階）：ある反復の開始時に不変条件とループのガードの両方が成り立つなら、
   その反復の終わりにも不変条件が成り立つ。
3. **事後条件**：ループを抜け（ガードが偽）、不変条件が成り立つとき、望む結果が導かれる。
4. **停止性**：*変量*（*減少関数* や *ランキング関数* とも呼ばれます）が反復のたびに真に減少し、
   かつ下に有界であるため、ループがいずれ抜けることが保証される。

Kani は、`#[kani::loop_invariant]` によって手順 1〜3 を自動的に確認します。手順 4 は
`#[kani::loop_decreases]` を追加したときに確認されます。手順 4 がなければ、Kani の証明は *部分的* です。つまり、
ループが停止すると仮定すれば健全ですが、停止しない可能性を排除することはできません。

## 過去の値と追加の変数 {#historic-values-and-extra-variables}

### 過去の値 {#historic-values}

ループ契約では、過去の値を表す 2 つの記法をサポートしています。
1. `on_entry(expr)`：ループに入る前の `expr` の値。
2. `prev(expr)`：前の反復での `expr` の値。ループ契約で `prev` を使うと、Kani はループが少なくとも 1 回反復することをアサートすることに注意してください。

例:
```Rust
#[kani::proof]
pub fn loop_with_old_and_prev() {
    let mut i = 100;
    #[kani::loop_invariant((i >= 2) && (i <= 100) && (i % 2 == 0) && (on_entry(i) == 100) && (prev(i) == i + 2))]
    while i > 2 {
        if i == 1 {
            break;
        }
        i = i - 2;
    }
    assert!(i == 2);
}
```

### `for` ループでの `kani::index` 変数 {#kaniindex-variable-in-for-loop}

Kani は、`for` ループのループ契約で使える追加の変数 `kani::index` を提供しています。
`kani::index` は、イテレータにおける現在の反復の位置（インデックス）を表します。
また、ループ契約の直後に続く `for` ループにだけ結び付けられます。

例:

```Rust
#[kani::proof]
fn forloop() {
    let mut sum: u32 = 0;
    let a: [u8; 10] = kani::any();
    kani::assume(kani::forall!(|i in (0,10)| a[i] <= 20));
    #[kani::loop_invariant(sum <= (kani::index as u32 * 20) )]
    for x in a {
        sum = sum + x as u32;
    }
    assert!(sum <= 200);
}
```

## 契約を持つ関数の中のループ契約 {#loop-contracts-inside-functions-with-contracts}
次の例に示すように、Kani はループ契約と関数の契約を組み合わせて使うことをサポートしています。
``` Rust
#![feature(proc_macro_hygiene)]
#![feature(stmt_expr_attributes)]

#[kani::requires(i>=2)]
#[kani::ensures(|ret| *ret == 2)]
pub fn has_loop(mut i: u16) -> u16 {
    #[kani::loop_invariant(i>=2)]
    while i > 2 {
        i = i - 1
    }
    i
}

#[kani::proof_for_contract(has_loop)]
fn contract_proof() {
    let i: u16 = kani::any();
    let j = has_loop(i);
}
```

ループ契約と関数の契約の両方を（フラグ `-Z loop-contracts -Z function-contracts` で）有効にすると、
Kani は、契約を証明したい関数の中のすべてのループを、展開する代わりに契約で置き換えます。

## ループの modifies 節 {#loop-modifies-clauses}
ユーザは、ループ本体の中で変更されうる、メモリが割り当てられたアドレスについて、`loop_modifies` 節を手動で指定できます。
この考え方は、CBMC の `__CPROVER_assigns` 節（https://diffblue.github.io/cbmc/contracts-assigns.html）とよく似ています。
ただし Kani では、CBMC のターゲットの代わりに、`loop_modifies` 節で使える次の 3 つの Rust の型を使います。
1. `RawPtr`：変数名をターゲットにすることはできません。代わりに、その変数へのポインタを使う必要があります。これにより、借用した参照やエイリアスを通じた変更も確認できます。
```Rust
#[kani::proof]
fn main() {
    let mut i = 0;
    #[kani::loop_invariant(i <= 20)]
    #[kani::loop_modifies(&i as *const _)]
    while i < 20 {
        i = i + 1;
    }
}
```
2. `Reference`：RawPtr と似ていますが、`__CPROVER_object_whole(ptr-expr)` の代わりとしても使えます。
例
```Rust
#[kani::proof]
fn main() {
    let mut i = 0;
    let mut a: [u8; 20] = kani::any();
    #[kani::loop_invariant(i <= 20)]
    #[kani::loop_modifies(&i, &a)]
    while i < 20 {
        a[i] = 1;
        i = i + 1;
    }
}
```
3. `FatPtr (Slice)`：`__CPROVER_object_from(ptr-expr)` と `__CPROVER_object_upto(ptr-expr, uint-expr)` の代わりに使います。
```Rust
#[kani::proof]
fn main() {
    let mut i = 3;
    let mut a: [u8; 100] = kani::any();
    #[kani::loop_invariant(i >=3 && i <= 20)]
    #[kani::loop_modifies(&i , &a[3..20])]
    while i < 20 {
        a[i] = 1;
        i = i + 1;
    }
}
```
または

```Rust
use std::ptr::slice_from_raw_parts;
#[kani::proof]
fn main() {
    let mut i = 0;
    let mut a: [u8; 100] = kani::any();
    #[kani::loop_invariant(i <= 20)]
    #[kani::loop_modifies(&i , slice_from_raw_parts(a.as_ptr(), 20))]
    while i < 20 {
        a[i] = 1;
        i = i + 1;
    }
}
```

ターゲットは、ループの直前、つまり `#[kani::loop_invariant]` がループの前に追加するコード
（例えば `on_entry` の値や、`for` ループの場合はイテレータ）の後で評価されます。
`loop_modifies` アトリビュートを複数持つループは、それらすべてのターゲットを変更してかまいません。
`#[kani::loop_invariant]` を持たないループに付いた `loop_modifies` アトリビュートは、警告を出して無視されます。
Kani はそのようなループを抽象化せず、他のループと同じように展開するからです。

`loop_modifies` 節に挙げる必要があるのは、ループの外に存在するメモリだけです。
Kani は、ループのために生成した変数（`prev` 用の変数と、`for` ループの場合はパターンの束縛とループのインデックス `kani::index`）を節に追加します。
また、ループ本体で宣言され、Kani がループの前に初期化する変数も追加します。
例えば、次のループは `sum` だけを挙げた節で検証に成功します。
```Rust
#[kani::proof]
fn main() {
    let a: [u8; 10] = kani::any();
    let mut sum: u32 = 0;
    #[kani::loop_invariant(sum <= kani::index as u32 * 255)]
    #[kani::loop_modifies(&sum)]
    for (_i, x) in a.iter().enumerate() {
        let y = *x as u32;
        sum = sum + y;
    }
}
```

## decreases 節（停止性の証明） {#decreases-clauses-termination-proofs}

### 停止性が重要な理由 {#why-termination-matters}

停止性の証明がないと、停止しないループのせいで実際には到達できない性質を、Kani が検証に成功したと報告してしまうことがあります。
例えば次のようになります。

```Rust
#[kani::proof]
fn unsound_without_termination() {
    let mut x: u64 = kani::any_where(|i| *i >= 1);

    #[kani::loop_invariant(x >= 1)]
    while true {
        x = x;  // 無限ループ — x は決して変わらない
    }

    // このアサーションは「証明」される。ループの抽象化は
    // ループを抜けると仮定するが、実際には決して抜けないからだ。
    assert!(x >= 1);
}
```

ループを抽象化すると、このアサーションは成り立つように見えます。しかし、ループは決して停止しないので、
このアサーションには到達できません。decreases 節は、ループが停止に向かって進んでいることをユーザに証明させることで、
この種の不健全な結果を防ぎます。

### 背景：Floyd の方法 {#background-floyds-method}

ここで使う手法は、Robert Floyd が 1967 年の重要な論文
[*Assigning Meaning to Programs*](https://people.eecs.berkeley.edu/~necula/Papers/FloydMeaning.pdf) で提案したものです。
その考え方は、次の性質を持つ値である *変量*（*ランキング関数* や *停止性の尺度* とも呼ばれます）を見つけることです。

1. 下に有界である（例えば、非負の整数）。
2. ループの反復のたびに真に減少する。

この値は永遠に減り続けることはできない（いずれ下限に達する）ので、ループは停止しなければなりません。

### 構文 {#syntax-1}

```
#[kani::loop_decreases(expr)]
#[kani::loop_decreases(expr1, expr2, ..., exprN)]
```

decreases 節は、ループと同じスコープから見える変数を使った、1 つ以上の算術式を受け取ります。
decreases 節は、ループ不変条件と並べて、ループ式の前に置かなければなりません。

複数の式は順序付きのタプルになり、
[辞書式順序](https://en.wikipedia.org/wiki/Lexicographic_order)で比較されます（後述）。

### 基本的な例 {#basic-example}

```Rust
#![feature(stmt_expr_attributes)]
#![feature(proc_macro_hygiene)]

#[kani::proof]
fn simple_decreases() {
    let mut x: u8 = kani::any_where(|i| *i >= 1);

    #[kani::loop_invariant(x >= 1)]
    #[kani::loop_decreases(x)]
    while x > 1 {
        x = x - 1;
    }

    assert!(x == 1);
}
```

ここで `x` は反復のたびに真に減少し、`1` で下に有界です（ループのガードが
`x > 1` を保証します）。そのため、ループは停止しなければなりません。

### 多次元の decreases（辞書式順序） {#multi-dimensional-decreases-lexicographic-ordering}

> **注:** 多次元の decreases 節はまだ完全にはサポートされていません。構文は
> 受け付けられますが、Kani の irep エンコーディングを通して渡されたタプル式を CBMC が正しく処理しません。
> これは[既知のバグ](#decreases-clause-limitations-and-comparison-with-other-tools)として追跡されています。
> 今のところは、単一の整数式を使ってください。

ループの中には、単一の式では停止性を証明できないものがあります。例えば、2 つのカウンタを持ち、
内側のカウンタがリセットされるループでは、タプル `(outer, inner)` が必要になることがあります。CBMC はタプルを
辞書式に比較します。最初の要素が減少すれば、残りの要素は増えても変わらなくてもかまいません。
最初の要素が変わらない場合に限り、2 番目の要素が減少しなければなりません。以降も同様です。

これは、[アッカーマン関数](https://en.wikipedia.org/wiki/Ackermann_function)の停止性の証明に使われるのと同じ原理です。
アッカーマン関数には 2 次元の decreases 節 `(m, n)` が必要です。

```Rust
#[kani::proof]
fn nested_counters() {
    let n: u8 = kani::any_where(|i| *i >= 1 && *i <= 5);
    let mut i: u8 = 0;
    let mut j: u8 = 0;

    #[kani::loop_invariant(i <= n && j <= n)]
    #[kani::loop_decreases(n - i, n - j)]
    while i < n {
        if j < n {
            j += 1;
        } else {
            i += 1;
            j = 0;
        }
    }

    assert!(i == n);
}
```

`j < n` のとき、最初の成分 `n - i` は変わりませんが、2 番目の成分 `n - j` は減少します。
`j >= n` のとき、最初の成分 `n - i` が減少します（`n - j` はリセットされますが、
最初の成分がすでに減少しているので問題ありません）。

### 意味 {#semantics-1}

decreases 節は、CBMC の `goto-instrument` によって、次のような計装を通じて確認されます。

1. ループ本体の始め（ループのガードが満たされた後）に、尺度の現在の値を
   一時変数（`old_measure`）に記録します。
2. ループ本体の終わり（後方辺の前）に、尺度の新しい値を記録します
   （`new_measure`）。
3. アサーションで `new_measure < old_measure`（真の減少）を確認します。

多次元の decreases `(e1, e2, ..., en)` の場合は、算術の厳密な比較が、
辞書式の厳密な比較に拡張されます。タプル `(new_e1, ..., new_en)` は、辞書式順序で
`(old_e1, ..., old_en)` より小さくなければなりません。

CBMC は、尺度が常に非負であることを要求しないことに注意してください。要求するのは、
尺度が 0 を下回るときにループがもう一度実行されないことだけです。言い換えると、尺度が
負になっても、ループのガードが偽であれば問題ありません。

### ループ不変条件との関係 {#interaction-with-loop-invariants}

decreases 節は、ループ不変条件の節と組み合わせて働きます。不変条件は、decreases の確認を行う際の
前提（帰納法の仮定）を確立します。不変条件の節を持たないループに付いた decreases 節は、
警告を出して無視されます。Kani はそのようなループを抽象化せず、他のループと同じように展開するからです。
可能な限り弱い不変条件でループの停止性を証明するには、`#[kani::loop_invariant(true)]` を使ってください。

実際には、ほとんどの場合、両方が必要になります。不変条件が状態空間を制約し、
decreases 節がその制約された空間の中で進んでいることを証明します。

### decreases 節の制限と他のツールとの比較 {#decreases-clause-limitations-and-comparison-with-other-tools}

Kani の decreases 節には、次の制限があります。比較のため、
[Dafny](https://dafny.org/latest/OnlineTutorial/Termination) と
[Verus](https://verus-lang.github.io/verus/guide/) と比べます。

1. **整数の尺度のみ。** Kani は（CBMC を通じて）、decreases 節で整数型の式しかサポートしません。
   Dafny は、停止性の尺度として、シーケンス（長さで比較）、集合（真部分集合の包含関係で比較）、
   ブール値、参照もサポートしています。Verus も同様に、より豊富な型をサポートしています。
   Kani で整数以外の尺度を使う必要がある場合は、手動で整数式に対応付けなければなりません
   （例えば、スライスそのものの代わりに `slice.len() - i` を使います）。

2. **自動推論なし。** Kani は decreases 節を自動的に推測しません。Dafny は、
   `A < B` という形のループのガードを見ると自動的に `B - A` を推論し、単純な再帰関数については
   パラメータの decreases を推論します。Kani では、ユーザが常に明示的に decreases 節を与えなければなりません。

3. **再帰関数の停止性は扱わない。** Kani の decreases 節はループにだけ適用され、
   再帰関数には適用されません。CBMC 自体が、再帰の停止性の証明をサポートしていません（CBMC の
   PR [#6236](https://github.com/diffblue/cbmc/pull/6236) で言及されています）。Dafny と Verus はどちらも、
   再帰関数や相互再帰関数への `decreases` 注釈をサポートしています。Kani では、再帰関数は
   別の手段（例えば、展開の上限）で扱わなければなりません。

4. **`decreases *` という逃げ道はない。** Dafny には、停止するかどうかが分からない、または
   意図的に停止しないループや関数（例えば、コラッツ予想やストリーム処理）について、停止性の確認を明示的に
   外す `decreases *` があります。Kani には同等のものはありません。停止性を証明できない場合は、
   単に decreases 節を省略してください。その場合もループは不変条件によって抽象化されますが、
   停止性は保証されません。

5. **副作用の確認なし。** Kani は、decreases 節の式に副作用がないことを検証しません。
   副作用（例えば、変数の変更）を持つ decreases 節は、不健全な結果につながる可能性があります。
   式が純粋であることは、ユーザが保証しなければなりません。

6. **真の減少のみ。** CBMC は、整数の厳密な比較で `new_measure < old_measure` を確認します。
   独自の整礎な順序や、ユーザ定義の比較演算子はサポートしていません。
   尺度は整数の算術に対応付けられなければなりません。

## 実例：二分探索 {#worked-example-binary-search}

二分探索は古典的なアルゴリズムですが、正しく書くのは見かけよりも難しいものです。不変条件、事後条件、
停止性の論証がどれも自明ではなく、互いに依存しているので、ループ契約の題材としてとても適しています。

```Rust
#![feature(stmt_expr_attributes)]
#![feature(proc_macro_hygiene)]

fn binary_search(arr: &[i32; 5], target: i32) -> Option<usize> {
    let mut lo: usize = 0;
    let mut hi: usize = arr.len();

    #[kani::loop_invariant(lo <= hi && hi <= arr.len())]
    #[kani::loop_decreases(hi - lo)]
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if arr[mid] == target {
            return Some(mid);
        } else if arr[mid] < target {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }

    None
}
```

正しさの 4 つの手順を順に見ていきましょう。

**1. 確立。** 最初は `lo = 0` かつ `hi = arr.len()` なので、`0 <= lo <= hi <= arr.len()` は
自明に成り立ちます。

**2. 保存。** `lo <= hi <= arr.len()` かつ `lo < hi`（ガード）であれば、ループ本体の後でも
不変条件が成り立つことを示す必要があります。鍵となるのは、
`mid = lo + (hi - lo) / 2` なので `lo <= mid < hi` だという点です。`arr[mid] < target` の分岐では、`lo` は
`mid + 1` になり、これは高々 `hi` です。`arr[mid] >= target` の分岐では、`hi` は `mid` になり、これは
少なくとも `lo` です。どちらの場合でも、`lo <= hi <= arr.len()` は保たれます。

**3. 事後条件。** ループを抜けたとき、`lo >= hi` です。これを不変条件 `lo <= hi` と組み合わせると、
`lo == hi` が得られます。早期リターンで目標が見つからなかった場合、探索空間は空です。

**4. 停止性。** decreases 節は `hi - lo` です。不変条件により、これは非負であることが保証されます。
各分岐では、`lo` が増える（`mid + 1 > lo` になる）か、`hi` が減る（`mid < hi` になる）かのどちらかです。
どちらの場合でも、`hi - lo` は真に減少します。停止性の論証が *不変条件に依存している* ことに注目してください。
`lo <= hi` が分からなければ、尺度が非負であることを保証できません。

この例は、不変条件と decreases 節が協調して働くという一般的な原則を示しています。
不変条件が状態空間を制約し、decreases 節がその制約された空間の中で進んでいることを証明します。

## 実践的な指針 {#practical-guidance}

### ループ不変条件の選び方 {#choosing-a-loop-invariant}

ループ不変条件は、ループが正しく動く本質的な理由を表します。反復の境界ごとに成り立つ、
ループの進み方の説明だと考えてください。よい進め方は次のとおりです。

- **事後条件から始める。** ループを抜けたときに何が成り立っている必要がありますか？ 不変条件は、
  それを途中の任意の反復に一般化したものであるべきです。
- **範囲を含める。** ほとんどすべてのループ不変条件には、ループ変数の範囲が必要です
  （例えば `i <= n`）。範囲がないと、Kani の havoc ステップが任意の値を代入してしまうことがあります。
- **変数間の関係を含める。** ループが変数間の関係を保っているなら
  （例えば `sum == a[0] + a[1] + ... + a[i-1]`）、その関係は不変条件に含めるべきです。

### よくある間違い {#common-mistakes}

**不変条件が弱すぎる。** これは最もよくある誤りです。症状は次のとおりです。
- *事後条件の手順* が失敗する：不変条件と `!guard` を組み合わせても、望む性質を導くのに十分な強さがないため、
  Kani はループの後のアサーションの失敗を報告します。
- *保存の手順* が失敗する：不変条件が状態を十分に制約しておらず、自身が帰納的であることを証明できません。

**不変条件が強すぎる。** 症状は次のとおりです。
- *確立の手順* が失敗する：最初の反復の前に不変条件が成り立ちません。
- *保存の手順* が失敗する：強すぎる条件をループ本体が保てません。

**decreases 節が間違っている。** 症状は次のとおりです。
- Kani が `"Check variant decreases after step for loop"` に対して `FAILURE` を報告します。これは、
  ある反復で尺度が真に減少しなかったことを意味します。よくある原因は次のとおりです。
  - 尺度が正しい量を表していない（例えば、`n - i` を使うべきところで `i` を使っている）。
  - ループ本体に、進まないパスがある（例えば、尺度を変えない `continue`）。
  - 尺度が定数式になっている。

### decreases 節のヒント {#tips-for-decreases-clauses}

- `i` を増やす `while i < n` ループでは、自然な尺度は `n - i` です。
- `x` を減らす `while x > 0` ループでは、自然な尺度は `x` です。
- 入れ子のループでは、それぞれのループに独自の decreases 節が必要です。内側のループの尺度は、
  外側のループとは独立して減少しなければなりません。
- 単一の式でうまくいかない場合は、多次元の decreases 節を試してください。どの変数が
  「ほとんどの場合に」減少するか（それを最初に置きます）、どの変数が最初の変数の「内側で」減少するか
  （それを 2 番目に置きます）を考えてください。

## 制限 {#limitations}

ループ契約には次の制限があります。

1. `while` ループと `loop` ループはサポートされています。`for` ループは、配列、スライス、Iter、Vec、Range、StepBy、Chain、Zip、Map、Enumerate に対してサポートされています。それ以外の種類のループはサポートされていません：[`while let` ループ](https://doc.rust-lang.org/reference/expressions/loop-expr.html#predicate-pattern-loops)。
2. Kani は、エイリアス解析によって *ループの modifies* を推論します。ループの modifies とは、帰納法の仮定において任意の値であると仮定する変数のことで、
   ループの実行中に書き込まれるすべてのメモリ位置を含んでいる必要があります。推論されたループの modifies が、ループ内で書き込まれるターゲットの一部を見落としていると、証明は失敗します。
   これは、ループ内で呼ばれる別の関数によって構造体の一部のフィールドが変更される場合に起こることが分かっています。
3. ループ契約（不変条件、decreases 節）に副作用がないかどうかは確認しません。副作用を持つループ契約は、不健全な証明結果につながる可能性があります。指定するループ契約に副作用がないことを確かめてください。
4. decreases 節は整数型の式しかサポートしません。他の検証ツールとの詳しい比較については、[decreases 節の制限](#decreases-clause-limitations-and-comparison-with-other-tools)の節を参照してください。
5. decreases 節には、次の既知のバグがあります（[#3168](https://github.com/model-checking/kani/issues/3168) で追跡されています）。
   - **ループの前に計算される尺度。** Kani が尺度を各反復で評価するのは、その尺度が関数の変数を直接読む場合（例えば `x`、`s.field`、`*p`）だけです。それ以外の尺度、例えばオーバーフローの確認を伴う算術（例えば `n - i`）、キャスト、インデックス、関数呼び出し、定数、クロージャがキャプチャした変数などは、ループの前に一度だけ計算されます。そのため、尺度が減少していても確認は失敗し、Kani は警告を出します（[#4585](https://github.com/model-checking/kani/issues/4585)）。
   - **多次元の decreases**（例えば `#[kani::loop_decreases(a, b)]`）は正しく処理されません。Kani の irep エンコーディングを通して渡されたタプル式に対して、CBMC が辞書式比較を行わないためです。
   - 内側と外側の両方のループに decreases を付けた**入れ子のループ**では、assigns 節の衝突が起こることがあります。
   
   これらの制限は、`tests/expected/loop-contract/` に `fixme` テストとして記録されています。
{% endraw %}

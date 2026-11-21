{% raw %}
# スタブ {#stubbing}

スタブ（またはモック）は不安定な機能で、検証の間に特定のアイテムをそのアイテムのスタブ（モック）で置き換えるよう指定できます。
現在、スタブを適用できるアイテムは関数とメソッドだけです（詳しくは[制限事項](#limitations)を参照してください）。

## スタブを検討すべきとき {#when-to-consider-stubbing}

一般に、ユーザがスタブを検討するとよい理由として、次の 3 つを見出しています。
 - **サポートされていない機能:** 検証対象のコードに、インラインアセンブリのように Kani がサポートしていない機能が含まれている。
 - **性能が悪い:** 検証対象のコードに、Kani がサポートしてはいるものの検証の性能が悪くなる機能が含まれている（たとえばデシリアライズのコード）。
 - **合成的な推論:** 検証対象のコードに、別に検証済みのコードが含まれている。
                                すでに検証済みのコードを、その振る舞いを模倣するより単純なバージョンでスタブ化すると、検証の作業量を減らせます。

ほとんどの場合、スタブを使うと、そうしなければ検証が現実的でないコードを検証できるようになります。
（通常テストで使われる）*モック*と*スタブ*の定義は人によって少し異なるかもしれませんが、私たちはよく両方の用語を同じ意味で使います。

## 構成要素 {#components}

スタブ機能は、Kani を呼び出すときに `-Z stubbing` オプションを使うと有効にできます（`-Z` は不安定な機能であることを示します）。

現在、スタブ機能の構成要素は [`#[kani::stub(<original>, <replacement>)]` アトリビュート](#the-kanistub-attribute)だけです。
これを使うと、ハーネスの中でスタブ化しなければならない関数やメソッドの組を指定できます。

<!--
the other components expected to be here in the future are: the `stub_set(...)!` macro, off-the-shelf verification-friendly implementations, and automated
stubbing suggestions)
-->

## `#[kani::stub(...)]` アトリビュート {#the-kanistub-attribute}

スタブアトリビュート `#[kani::stub(<original>, <replacement>)]` は、スタブ機能の主要な道具です。

これは、コンパイルの段階で名前が `<original>` の関数やメソッドを、名前が `<replacement>` の関数やメソッドで置き換えるよう Kani に指示します。
これらの関数やメソッドの名前は、**Rust の標準の名前解決の規則を使って解決されます**。
これには、`use foo::bar as baz` のようなインポートや、同じクレートの複数のバージョンのインポートのサポートも含まれます。

**このアトリビュートはハーネスごとに指定しなければなりません**。これによりユーザは高い柔軟性を得られます。証明ハーネスに応じて、同じアイテムを異なる置き換え先でスタブ化する（あるいはスタブをまったく使わない）ことを選べるからです。さらに、**このアトリビュートは 1 つのハーネスに複数回指定できる**ので、（衝突しない）複数のスタブの組をサポートします。

### 例: `random` をスタブ化する {#an-example-stubbing-random}

[`rand::random`](https://docs.rs/rand/latest/rand/fn.random.html) 関数を使って暗号鍵を生成する、簡単な例を見てみましょう。

```rust
#[cfg(kani)]
#[kani::proof]
fn encrypt_then_decrypt_is_identity() {
    let data: u32 = kani::any();
    let encryption_key: u32 = rand::random();
    let encrypted_data = data ^ encryption_key;
    let decrypted_data = encrypted_data ^ encryption_key;
    assert_eq!(data, decrypted_data);
}

```

現在、[issue #1781](https://github.com/model-checking/kani/issues/1781) のため、Kani はこの例の検証に失敗します。

しかし、スタブ機能のおかげで、この制限を回避できます。

```rust
use rand::random;

#[cfg(kani)]
fn mock_random<T: kani::Arbitrary>() -> T {
    kani::any()
}

#[cfg(kani)]
#[kani::proof]
#[kani::stub(rand::random, mock_random)]
fn encrypt_then_decrypt_is_identity() {
    let data: u32 = kani::any();
    let encryption_key: u32 = rand::random();
    let encrypted_data = data ^ encryption_key;
    let decrypted_data = encrypted_data ^ encryption_key;
    assert_eq!(data, decrypted_data);
}
```

ここで、`#[kani::stub(rand::random, mock_random)]` アトリビュートは、`rand::random` をスタブ `mock_random` で置き換えるよう Kani に指示します。
これは妥当な仮定であることに注意してください。`rand::random` は、`kani::any` と同じように、任意の `u32` の値を返すことが期待されています。

では、これを Kani で実行してみましょう。

```bash
cargo add rand
cargo kani -Z stubbing --harness encrypt_then_decrypt_is_identity
```

検証結果は 1 つのチェックで構成されます。`assert_eq!(data, decrypted_data)` に対応するアサーションです。

```
RESULTS:
Check 1: encrypt_then_decrypt_is_identity.assertion.1
         - Status: SUCCESS
         - Description: "assertion failed: data == decrypted_data"
         - Location: src/main.rs:18:5 in function encrypt_then_decrypt_is_identity


SUMMARY:
 ** 0 of 1 failed

VERIFICATION:- SUCCESSFUL
```

Kani はアサーションが成功することを示しており、スタブなしでコードを検証しようとしたときに現れる問題を回避できています。

## 検証出力でのスタブの表示 {#stub-visibility-in-verification-output}

ハーネスにスタブが適用されると、Kani は検証を始める前にそれらを表示します。

```
Checking harness encrypt_then_decrypt_is_identity...
  - Stub: rand::random -> mock_random
```

これにより、どのスタブが有効になっているかを確認でき、スタブによって導入された仮定が検証出力で見えるようになります。

## 外部関数のスタブ化 {#stubbing-foreign-functions}

Kani は、`extern` ブロックで宣言された外部関数のスタブ化をサポートしています。これは、システムコールや C ライブラリの関数、その他の FFI 関数を、検証しやすい実装で置き換えるのに役立ちます。

```rust
extern "C" {
    fn my_c_function(input: u32) -> u32;
}

fn my_c_function_stub(input: u32) -> u32 {
    input + 1
}

#[kani::proof]
#[kani::stub(my_c_function, my_c_function_stub)]
fn check_with_ffi_stub() {
    let result = unsafe { my_c_function(42) };
    assert_eq!(result, 43);
}
```

## スタブの互換性とライフタイムに関する注意 {#stub-compatibility-and-lifetime-considerations}

スタブの互換性を確かめるとき、Kani はライフタイムの情報を消去した後でパラメータの型とリターン型を比較します。つまり、元の関数と異なるライフタイム注釈を持つスタブも受け入れられます。たとえば、`&'static i32` を返すスタブは、`&'a i32` を返す関数の置き換えとして受け入れられます。

**警告:** Kani はそのようなスタブを受け入れますが、ライフタイムの不一致は、わかりにくい検証の失敗（たとえば "dereference failure: dead object"）を引き起こすことがあります。スタブで予期しない失敗に遭遇したら、ライフタイム注釈が一致しているかを確認してください。

## 制限事項 {#limitations}

以下では、スタブ機能の既知の制限事項を説明します。

### 機能間の相互作用 {#feature-interactions}

- **関数の契約:** スタブは関数の契約と一緒に使えます。関数をその検証済みの契約による抽象化で置き換えるには、`#[kani::stub_verified(fn)]` を使います。これには `#[kani::proof_for_contract(fn)]` ハーネスが存在している必要があります。通常の `#[kani::stub]` も、同じハーネスの中で `-Z function-contracts` と組み合わせられます。
- **ループの契約:** スタブとループの契約（`-Z loop-contracts`）は、同じハーネスの中で問題なく一緒に使えます。
- **具体的な再生:** スタブは `--concrete-playback` と互換性があります。スタブ化されたハーネスが失敗すると、Kani はスタブの振る舞いを使って失敗を再現する具体的なテストケースを生成できます。

### トレイトメソッドのスタブ化 {#trait-method-stubbing}

Kani は、完全修飾構文を使ったトレイトメソッドの実装のスタブ化をサポートしています。

```rust
trait MyTrait {
    fn method(&self) -> u32;
}

struct MyType;

impl MyTrait for MyType {
    fn method(&self) -> u32 { 0 }
}

fn stub_method(_x: &MyType) -> u32 { 42 }

#[kani::proof]
#[kani::stub(<MyType as MyTrait>::method, stub_method)]
fn check_trait_stub() {
    let x = MyType;
    assert_eq!(x.method(), 42);
}
```

これはジェネリックなトレイトでも使えます。

```rust
#[kani::stub(<MyType as Convert<u32>>::convert, stub_convert)]
```

2 つのトレイトが同じ名前のメソッドを定義している場合、完全修飾構文によって、どの実装をスタブ化するかの曖昧さがなくなります。

次のトレイトのパターンがサポートされています。

- **スーパートレイトのメソッド:** スーパートレイトで定義されたメソッド（たとえば `<MyType as Base>::method`）のスタブ化は、サブトレイトのメソッドとは独立に機能します。
- **オーバーライドされたデフォルトメソッド:** 型がトレイトのデフォルトメソッドをオーバーライドしている場合、そのオーバーライドは通常どおりスタブ化できます。
- **動的ディスパッチ:** メソッドがトレイトオブジェクト（`&dyn Trait` や `Box<dyn Trait>`）を通して呼ばれる場合でも、スタブが適用されます。

**既知の制限:** 実装する型がオーバーライド*していない*トレイトのデフォルトメソッドのスタブ化は、現在サポートされていません（[#4588](https://github.com/model-checking/kani/issues/4588)）。デフォルトメソッドの本体は `Self` をプレースホルダの型として使うので、スタブの検証中に型の不一致が起こります。これは、そのまま継承されるデフォルトメソッド（つまり `impl` ブロックが独自の定義を提供していないもの）にのみ当てはまります。

**既知の制限:** const ジェネリックパラメータを持つトレイト（たとえば `<Type as Buf<16>>::write`）や関連型の制約を持つトレイト（たとえば `<Type as Iterator<Item = u32>>::next`）は現在サポートされておらず、解決エラーになります。

### スタブセット {#stub-sets}

複数のハーネスが同じスタブの集合を共有する場合、`kani::stub_set!` マクロで再利用できるスタブセットを定義し、`#[kani::use_stub_set(...)]` で適用できます。

```rust
fn mock_read(_path: std::path::PathBuf) -> std::io::Result<Vec<u8>> {
    Ok(vec![0; 10])
}

fn mock_write(_path: std::path::PathBuf, _contents: &[u8]) -> std::io::Result<()> {
    Ok(())
}

kani::stub_set!(io_stubs,
    stub(std::fs::read, mock_read),
    stub(std::fs::write, mock_write),
);

#[kani::proof]
#[kani::use_stub_set(io_stubs)]
fn check_with_io_stubs() {
    // 両方のスタブが適用される
}
```

スタブセットは、`use_stub_set(...)` を使ってほかのスタブセットを組み合わせられます。

```rust
kani::stub_set!(all_stubs,
    use_stub_set(io_stubs),
    stub(other::func, my_func),
);
```

同じハーネスで、`#[kani::use_stub_set(...)]` と個別の `#[kani::stub(...)]` アトリビュートを組み合わせることもできます。ハーネスごとに異なるスタブセットを使えます。

スタブセットをほかのモジュールから使えるようにするには、可視性の修飾子を付けます。

```rust
mod stubs {
    kani::stub_set!(pub my_set,
        stub(real_fn, stub_fn),
    );
}
```

**注意:** スタブセットの中のパスは、スタブセットが定義されたモジュールからではなく、それを使うハーネスからの相対パスとして解決されます。

### 使用上の制約 {#usage-restrictions}

スタブの注釈（`#[kani::stub]`）はハーネスごとに指定します。クレートが異なるスタブの設定を持つ複数のハーネスを含む場合、各ハーネスはそれぞれのスタブの集合を使って独立に検証されます。

### サポート {#support}

スタブのサポートは、現在**関数とメソッドに限られています**。それ以外のアイテムはサポートされていません。

スタブの良い候補になり得るものの、サポートされていないアイテムの例を次に示します。
- 型
- マクロ
- intrinsic

メソッドのスタブのサポートが、あるべきほど使いやすくないことは認識しています。
メソッドのスタブを定義しようとするときによくある問題は、オブジェクトの非公開フィールド（つまり `self` のフィールド）にアクセスできないことです。
回避策のひとつは、次の例のように `unsafe` な関数 `std::mem::transmute` を使うことです。

  ```rust
  struct Foo {
      x: u32,
  }
  
  impl Foo {
      pub fn m(&self) -> u32 {
          0
      }
  }
  
  struct MockFoo {
      pub x: u32,
  }
  
  fn mock_m(foo: &Foo) -> u32 {
      let mock: &MockFoo = unsafe { std::mem::transmute(foo) };
      return mock.x;
  }
  
  #[cfg(kani)]
  #[kani::proof]
  #[kani::stub(Foo::m, mock_m)]
  fn my_harness() { ... }
  ```

しかし、これは `unsafe` でエラーを起こしやすいので、おすすめしません。
一般に、非公開の関数やメソッドのスタブ化はおすすめしません。
そうすると証明が壊れやすくなることがあります。非公開の関数やメソッドは、マイナーバージョンのアップグレードでも変更されたり削除されたりすることがあります（API の一部ではないからです）。
そのため、非公開の関数やメソッドのスタブ化に依存する証明は、保守の負担が大きくなるかもしれません。

## エラーになる条件 {#error-conditions}

`original` と `replacement` の組の集合が与えられたとき、次の場合に Kani はエラーで終了します。
 1. 指定した `original` 関数が存在しない。
 2. 指定した `replacement` スタブが存在しない。
 3. ユーザが同じハーネスに対して衝突するスタブを指定した（たとえば、同じ `original` 関数が複数の `replacement` 関数に対応付けられている）。
 4. `replacement` スタブのシグネチャが `original` の関数やメソッドのシグネチャと互換でない（次の節を参照）。

### スタブの互換性と検証 {#stub-compatibility-and-validation}

次の条件をすべて満たす場合に、スタブと関数やメソッドは互換であるとみなします。

- パラメータの数が同じである。
- リターン型が同じである。
- スタブの各パラメータが、元の関数やメソッドの対応するパラメータと同じ型を持つ。
- スタブは元の関数やメソッドと同じ数のジェネリックパラメータを持たなければならない。
ただし、スタブのジェネリックパラメータは、元の関数やメソッドの対応するパラメータと異なる名前でもかまわない。
たとえば、スタブ `bar<A, B>(x: A, y: B) -> B` は、関数 `foo<S, T>(x: S, y: T) -> T` と互換な型を持つとみなされる。
- 各型パラメータの境界は一致していなくてもよい。ただし、元の関数のすべての呼び出しは、スタブの境界も満たさなければならない。

最後の点がもっとも微妙です。
スタブのシグネチャの型パラメータが、元の関数やメソッドのシグネチャの対応する型パラメータと同じトレイトを実装していることは要求しません。
しかし、トレイトの不一致によって、トレイトメソッドの静的ディスパッチの呼び出しが単相化の間に解決できなくなる場合、Kani はそのスタブを拒否します。
たとえば、この制約により次のハーネスは除外されます。

```rust
fn foo<T>(_x: T) -> bool {
    false
}

trait DoIt {
    fn do_it(&self) -> bool;
}

fn bar<T: DoIt>(x: T) -> bool {
    x.do_it()
}

#[kani::proof]
#[kani::stub(foo, bar)]
fn harness() {
    assert!(foo("hello"));
}
```

型パラメータ `T` が型 `&str` でインスタンス化されると、スタブ `bar` の中のトレイトメソッド `DoIt::do_it` の呼び出しは解決できません。
一方で、この方法にはある程度の柔軟性があり、たとえば先ほどの `rand::random` をモックする例が可能になります。
`rand::random` と `my_random` はどちらも `() -> T` という型を持ちますが、前者では型 `Standard` が `Distribution<T>` を実装するように `T` が制限されているのに対し、後者では `T` が `kani::Arbitrary` を実装しなければなりません。
このトレイトの不一致が許されるのは、この呼び出し箇所で `T` が `kani::Arbitrary` を実装する `u32` でインスタンス化されているからです。
{% endraw %}

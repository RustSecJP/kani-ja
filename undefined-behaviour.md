{% raw %}
# 未定義動作 {#undefined-behaviour}

## 未定義動作がプログラムの検証に与える影響 {#the-effect-of-undefined-behaviour-on-program-verification}
Rust には、[未定義動作](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)（UB）の広い定義があります。
[Rust のドキュメントは](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)、UB が予期しない、局所的でない影響を持ちうると警告しています。


> **注**: 未定義動作はプログラム全体に影響します。例えば、C の未定義動作を示す関数を C で呼び出すと、プログラム全体に未定義動作が含まれることになり、それは Rust のコードにも影響しえます。逆に、Rust の未定義動作は、ほかの言語への FFI 呼び出しによって実行されるコードに悪影響を及ぼすことがあります。

プログラムに UB があると、プログラムの残りの部分の意味論は **未定義** になります。
その結果、検証対象のプログラムが UB を含むなら、原理的には、そのプログラム（Kani が解析する MIR での表現を含む）は **意味論を持たず**、したがって何をしてもおかしくありません。Kani が検査する保証に違反することさえありえます。
つまり、検証結果は、検証対象のプログラムが UB を含まないという条件のもとでのものです。

## Rust はどんな形の未定義動作を示しうるか {#what-forms-of-undefined-behaviour-can-rust-exhibit}

Rust の [UB の定義](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)はとても広いので、Rust には次の警告があります。

> **警告**
> 次の一覧は網羅的ではありません。`unsafe` なコードで何が許され、何が許されないかについての Rust の意味論の形式的なモデルは存在しないので、ほかにも unsafe とみなされる振る舞いがあるかもしれません。次の一覧は、未定義動作であると確実に分かっているものだけです。`unsafe` なコードを書く前に Rustonomicon（https://doc.rust-lang.org/nomicon/index.html）を読んでください。


UB の形式的な意味論がないこと、そして Kani がメモリ安全性に焦点を当てていることから、Kani が検出しない、あるいは検出を最善努力でしか試みない種類の UB があります。
[Rust のドキュメント](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)にある網羅的でない一覧をもとにした、それらの網羅的でない一覧は次のとおりです。

* データ競合。
    * Kani は逐次的なコードに焦点を当てています。
* [ポインタのエイリアシング規則](http://llvm.org/docs/LangRef.html#pointer-aliasing-rules)を破ること。
    * Kani は、ポインタの誤用がメモリ安全性やアサーションの違反を引き起こすかどうかは検出できますが、参照のライフタイムは追跡しません。
* 不変のデータを変更すること。
    * Kani は、不変のデータの変更がメモリ安全性やアサーションの違反を引き起こすかどうかは検出できますが、参照のライフタイムは追跡しません。
* コンパイラのイントリンシックを通じて未定義動作を引き起こすこと。
    * Kani はコンパイラのイントリンシックの事前条件を検査するよう最善を尽くしますが、すべての場合でそうすることは保証しません。[コンパイラのイントリンシックの現在のサポート状況](./rust-feature-support/intrinsics.html)も見てください。
* 現在のプラットフォームがサポートしていないプラットフォームのフィーチャでコンパイルされたコードを実行すること（[target_feature](https://doc.rust-lang.org/reference/attributes/codegen.html#the-target_feature-attribute) を参照）。
    * Kani はこの場合の検査を `rustc` に頼っています。
* 誤った呼び出し ABI で関数を呼び出すこと、または誤った巻き戻し ABI を持つ関数から巻き戻すこと。
    * Kani はこの場合の検査を `rustc` に頼っています。
* 無効な値を生成すること。非公開のフィールドやローカル変数の中であってもです。
    * Kani は `kani::any()` で[無効な値を作りません](./tutorial-nondeterministic-variables.html)が、無効な値を Rust の型に `transmute` しても（例えば `0` を `NonZeroU32` に）文句は言いません。
* インラインアセンブリの誤った使用。
    * Kani はインラインアセンブリをサポートしていません。
* 未初期化メモリを使うこと。
    * [Rust のフィーチャのサポート](./rust-feature-support.html#uninitialized-memory)の該当する節を見てください。
{% endraw %}

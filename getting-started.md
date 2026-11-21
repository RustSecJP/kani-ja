{% raw %}
# はじめに {#getting-started}

Kani は、[モデル検査](./tool-comparison.html)を使って Rust プログラムを解析する、オープンソースの検証ツールです。
Kani は、Rust コードの安全性と正しさの両方を確かめるのに役立ちます。
- *安全性*: Kani は多くの種類の[未定義動作](./undefined-behaviour.html)を自動で検査します。
このため、Kani は特に Rust の `unsafe` なコードブロックの検証に役立ちます。そこでは「[unsafe の強大な力](https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers)」がコンパイラによって検査されないからです。
- *正しさ*: Kani はパニック（例: `None` に対する `unwrap()`）、算術オーバーフロー、そして独自の正しさの性質を自動で検査します。独自の性質は、アサーション（`assert!(...)`）または[関数の契約](https://model-checking.github.io/kani/reference/experimental/contracts.html)の形で書きます。

Kani はモデル検査を使うので、性質を証明するか、（反例とともに）性質を反証するか、あるいはリソースが足りなくなるかのいずれかになります。

Kani は証明ハーネスを使ってプログラムを解析します。
証明ハーネスはテストハーネス、特にプロパティベーステストのハーネスに似ています。

## プロジェクトの状況 {#project-status}

Kani は現在、活発に開発されています。
リリースは[こちら](https://github.com/model-checking/kani/releases)で公開されています。
Kani の大きな変更は [RFC Book](https://model-checking.github.io/kani/rfc) に記録されています。
Kani の使用事例やフィーチャについての最新情報は、[ブログ](https://model-checking.github.io/kani-verifier-blog/)でも公開しています。

Rust 言語のフィーチャのかなり多くをサポートしていますが、すべてではありません（例: 並行性）。
サポートされているフィーチャの詳しい一覧は[制限事項](./limitations.html)を見てください。

Kani は毎月リリースされます。
リリースのたびに、Kani は Rust の最近の nightly リリースに追従します。そのため、たいていは最新の Rust 言語フィーチャに対応しています。

Kani を使っていて問題に出会ったら、ぜひ[報告してください](https://github.com/model-checking/kani/issues/new/choose)。
{% endraw %}

{% raw %}
# Kani を使う {#using-kani}

現在、Kani は 2 通りの方法で使えます。

 * `kani` コマンドで[1 つのクレートに対して](#usage-on-a-single-crate)使う。
 * `cargo kani` コマンドで [Cargo パッケージに対して](#usage-on-a-package)使う。

プロジェクトに Kani を組み込むつもりなら、`cargo kani` を使うことをおすすめします。
すでに cargo を使っているなら、依存関係を自動で扱ってくれますし、（必要なら）`Cargo.toml` で設定できます。
ただし、小さな例やテストには `kani` が便利です。

## パッケージに対して使う {#usage-on-a-package}

Kani は `cargo` と統合されていて、パッケージから次のように呼び出せます。

```bash
cargo kani [OPTIONS]
```

これは `cargo test` と同じように動きますが、すべてのテストハーネスを実行する代わりに、すべての証明ハーネスを解析します。

## よく使うコマンドラインフラグ {#common-command-line-flags}

`kani` と `cargo kani` には、共通のコマンドラインフラグがたくさんあります。

 * `--concrete-playback=[print|inplace]`: 失敗した証明ハーネスを具体的な反例を使って再生する、Rust の単体テストケースを生成する _実験的_ フィーチャです。
 `print` を指定すると、Kani は単体テストを標準出力に表示するだけです。
 `inplace` を指定すると、Kani は単体テストをユーザのソースコードの、証明ハーネスのすぐ隣に自動で追加します。詳しい手順は[具体的な再生](./reference/experimental/concrete-playback.html)の節を見てください。

 * `--tests`: 「[テストモード](https://doc.rust-lang.org/rustc/tests/index.html)」でビルドします。つまり、`cfg(test)` が設定され、（`cargo kani` を使う場合は）`dev-dependencies` が使える状態になります。

 * `--harness <name>`: デフォルトでは、Kani は見つけたすべての証明ハーネスを検査します。
   このフラグを使うと、1 つのハーネスだけを検査するように切り替えられます。

 * `--default-unwind <n>`: 証明ハーネスに対する、全体のデフォルトの[ループ展開](./tutorial-loop-unwinding.html)の上限を設定します。
   CBMC がループを際限なく展開しようとするときに、これで強制的に終了させられます。

引数の完全な一覧を見るには `cargo kani --help` を実行してください。

## 1 つのクレートに対して使う {#usage-on-a-single-crate}

小さな例や学び始めのときは、ソースファイル 1 つだけに対して Kani を実行することがとてもよくあります。
Kani を直接呼び出すコマンドラインの形式は次のとおりです。

```
kani filename.rs [OPTIONS]
```

これで `filename.rs` がビルドされ、その中で見つかったすべての証明ハーネスが実行されます。

## `Cargo.toml` での設定 {#configuration-in-cargotoml}

ユーザは、パッケージ内のハーネスを実行するためのデフォルト設定を `Cargo.toml` ファイルに追加できます。
Kani は次のセクションから引数を取り出します。

 * `[workspace.metadata.kani.flags]`
 * `[package.metadata.kani.flags]`

たとえば、（ほかで指定されていないときの）デフォルトのループ展開の上限を設定したいなら、パッケージの `Cargo.toml` に次の行を追加すればできます。

```toml
[package.metadata.kani.flags]
default-unwind = "1"
```

ここでのオプションはコマンドライン（`cargo kani --help`）と同じです。フラグ（つまり値をとらないコマンドライン引数）は `true` に設定すると有効になります。

Rust 1.80（または nightly-2024-05-05）以降、到達可能なすべての #[cfg] が、想定された設定名と値に一致するかどうかを自動で検査されるようになりました。
`cfg(kani)` についての警告を避けるには、クレートの `Cargo.toml` に次のように `check-cfg` のリント設定を追加することをおすすめします。

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(kani)'] }
```

詳しくは、この[ブログ記事](https://blog.rust-lang.org/2024/05/06/check-cfg.html)を参照してください。

## ビルドの過程 {#the-build-process}

Kani がコードをビルドするとき、次の 3 つの重要なことを行います。

1. 対象のクレートのコンパイル（依存クレートを含む）に `cfg(kani)` を設定します。
2. `kani` クレートを注入します。
3. ビルドスクリプトや手続き的マクロのクレートなど、ホスト向けのビルドターゲットに `cfg(kani_host)` を設定します。

証明ハーネス（[チュートリアルで詳しく学べます](./kani-tutorial.html)）は、テストに `#[test]` を注釈するのと同じように、`#[kani::proof]` を注釈した関数です。
ただし、Kani を使うと、`dev-dependencies` と同じような問題に出会うかもしれません。コードに直接 `#[kani::proof]` を書こうとすると、`cargo build` は `kani` クレートが何なのかを知らないので失敗します。

そのため、Rust でテストを書くときと同じ慣習をおすすめします。つまり、証明ハーネスを `cfg(kani)` の条件付きコンパイルで囲みます。

```rust
#[cfg(kani)]
mod verification {
    use super::*;

    #[kani::proof]
    pub fn check_something() {
        // ....
    }
}
```

これで、コードの普通のビルドが Kani に関係するものから一切影響を受けないことが保証されます。

この（上で見たような）`cfg(kani)` による条件付きコンパイルは、`tests/` の下に置いた Kani の証明でも必要です。
このコードが `cargo test` でビルドされるときは `kani` クレートが使えないので、そうしないとビルドが失敗してしまいます。
（一方、`tests/` の下で `dev-dependencies` を使う場合は `cfg(test)` で囲む必要はありません。そのコードはもともとテストのときにしかビルドされないからです。）

## Rust Analyzer の設定 {#rust-analyzer-setup}

（例えば VS Code で）Rust Analyzer を使っているなら、次の設定をおすすめします。これで Rust Analyzer が Kani 固有のコード（例: Kani の注釈や API など）を解析できるようになり、適切なコード補完とエラー表示が得られます。

1. パッケージの `Cargo.toml` に次を追加します。
```toml
[target.'cfg(kani_ra)'.dependencies]
kani = { git = "https://github.com/model-checking/kani" }
```

これで、`kani_ra` を条件とする依存として Kani が追加されます。

2. Rust Analyzer の設定ファイル（例: VS Code なら `settings.json`）に次を追加します。
```
    "rust-analyzer.cargo.extraEnv": {
        "RUSTFLAGS": "--cfg kani_ra --cfg kani",
        "RUSTUP_TOOLCHAIN": "nightly"
    },
```
説明:
- `kani_ra` の設定を有効にすると、Rust Analyzer は `Cargo.toml` ファイルで追加したクレート内の Kani の定義を見られるようになります。
- `kani` の設定を有効にすると、`#[cfg(kani)]` で囲まれたブロックが有効になります。
- 最後に、Rust Analyzer が Kani の依存クレート内のコードを扱えるようにするには、nightly ツールチェーンを使う必要があります。そのコードの多くが nightly のフィーチャを必要とするからです。
{% endraw %}

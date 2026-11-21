{% raw %}
# インストール {#installation}

Kani は、次の 3 つのプラットフォームで簡単にインストールできます。

* `x86_64-unknown-linux-gnu`（ほとんどの Linux ディストリビューション）
* `x86_64-apple-darwin`（Intel Mac OS）
* `aarch64-apple-darwin`（Apple Silicon Mac OS）

ほかのプラットフォームは、まだサポートされていないか、代わりに
[ソースからビルド](build-from-source.html)する必要があります。GitHub の CI ワークフローで
Kani を使うには、[GitHub CI Action](./install-github-ci.html) を見てください。

## 依存関係 {#dependencies}

次のものがあらかじめインストールされている必要があります。

* `rustup` でインストールした Rust 1.58 以降。

## 最新版のインストール {#installing-the-latest-version}

Kani の最新版のインストールは 2 段階で行います。

まず、次のコマンドで Kani のインストーラパッケージをダウンロードしてビルドします。
```bash
cargo install --locked kani-verifier
```
これで `kani` と `cargo-kani` のバイナリがビルドされ、（一般的な環境では）`~/.cargo/bin` に置かれます。

次に、インストーラを実行して、ビルド済みのバイナリと、それを支えるライブラリやデータをダウンロードしてインストールします。
```bash
cargo kani setup
```

2 つ目の段階では、Kani のコンパイラとその他の必要な依存物をダウンロードし、デフォルトでは `~/.kani/` の下に置きます。
`KANI_HOME` 環境変数を使えば、別のパスを指定できます。

## 古いバージョンのインストール {#installing-an-older-version}

```bash
cargo install --locked kani-verifier --version <VERSION>
cargo kani setup
```

## インストールの確認 {#checking-your-installation}

Kani をインストールしたら、
テスト用のファイルを作って実行してみましょう。

```rust
// ファイル: test.rs
#[kani::proof]
fn main() {
    assert!(1 == 2);
}
```

この 1 つのファイルに対して Kani を実行します。

```
kani test.rs
```

次のような結果が得られるはずです。

```
[...]
RESULTS:
Check 1: main.assertion.1
         - Status: FAILURE
         - Description: "assertion failed: 1 == 2"
[...]
VERIFICATION:- FAILED
```

テストを直すと、次のような結果になるはずです。

```
[...]
VERIFICATION:- SUCCESSFUL
```

## 次のステップ {#next-steps}

初めて Kani を学ぶなら、[チュートリアル](kani-tutorial.html)が参考になるでしょう。
{% endraw %}

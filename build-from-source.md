{% raw %}
# ソースコードからのインストール {#installing-from-source-code}

> 普通の方法で [Kani をインストール](install-guide.html)できたなら、Kani をソースからビルドする必要はありません。
> おそらく [Kani のチュートリアル](kani-tutorial.html)に進むのがよいでしょう。

## 依存関係 {#dependencies}

一般に、Kani をソースからビルドするには次の依存物が必要です。

> **注**: これらの依存物は、下に示すスクリプトを実行すればインストールされるので、
> 手でインストールする必要はありません。

1. [rustup](https://rustup.rs/) でインストールした Cargo
2. [CBMC](https://github.com/diffblue/cbmc)（最新リリース）
3. [Kissat](https://github.com/arminbiere/kissat)（リリース 4.0.1）
3. [Z3](https://github.com/Z3Prover/z3)
4. [cvc5](https://github.com/cvc5/cvc5)

Kani は [Ubuntu](#install-dependencies-on-ubuntu) と [macOS](##install-dependencies-on-macos) のプラットフォームでテストされています。

### Ubuntu での依存物のインストール {#install-dependencies-on-ubuntu}

Ubuntu 20.04、22.04、24.04 をサポートしています。
依存物をインストールするいちばん簡単な方法は（特に新しい VM を使っている場合）、
私たちの CI スクリプトに従うことです。

```
# git clone git@github.com:model-checking/kani.git
git clone https://github.com/model-checking/kani.git
cd kani
git submodule update --init
 # Ubuntu 20.04 では次を使います: `./scripts/setup/ubuntu-20.04/install_deps.sh`
./scripts/setup/ubuntu/install_deps.sh
# まだなら（または https://rustup.rs/ から）:
./scripts/setup/install_rustup.sh
source $HOME/.cargo/env
```

### macOS での依存物のインストール {#install-dependencies-on-macos}

macOS 11 をサポートしています。[Homebrew](https://brew.sh/) があらかじめインストールされている必要があります。

```
# git clone git@github.com:model-checking/kani.git
git clone https://github.com/model-checking/kani.git
cd kani
git submodule update --init
./scripts/setup/macos/install_deps.sh
# まだなら（または https://rustup.rs/ から）:
./scripts/setup/install_rustup.sh
source $HOME/.cargo/env
```

## Kani のビルドとテスト {#build-and-test-kani}

Kani パッケージをビルドするには、最適化を有効にしてコンパイルする場合は次を使います。

```
cargo build-dev -- --release
```
デバッグ／開発モードでコンパイルする場合は次を使います。
```
cargo build-dev
```

次に、必要なら回帰テストを実行します。

```
./scripts/kani-regression.sh
```

このスクリプトは出力がとても多いですが、成功すれば実行の最後に次のように表示されます。

```
All Kani regression tests completed successfully.
```

## LLBC バックエンド {#llbc-back-end}

Kani には、Low-Level Borrow Calculus（LLBC）を出力する実験的なバックエンドが含まれています。
LLBC は [Charon](https://github.com/AeneasVerif/charon) と
[Aeneas](https://github.com/AeneasVerif/aeneas) が使う形式です。このバックエンドは Rust コードを
LLBC に変換し、Aeneas フレームワーク上に作られた検証ツールとの連携を可能にします。

LLBC バックエンドを有効にして Kani をビルドするには、次を使います。

```
cargo build-dev -- --features cprover --features llbc
```

これで、デフォルトの CBMC/CPROVER バックエンドと LLBC バックエンドの両方を含む Kani がビルドされます。
LLBC 専用の回帰テストを実行するには、次を使います。

```
./scripts/kani-llbc-regression.sh
```

> **注**: LLBC バックエンドは実験的なもので、活発に開発中です。

## Kani をパスに追加する {#adding-kani-to-your-path}

ローカルでビルドした Kani をどこからでも使えるようにするには、Kani のスクリプトをパスに追加します。

```bash
export PATH=$(pwd)/scripts:$PATH
```

## 次のステップ {#next-steps}

初めて Kani を学ぶなら、[チュートリアル](kani-tutorial.html)が参考になるでしょう。
{% endraw %}

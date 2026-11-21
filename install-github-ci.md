{% raw %}
# GitHub Action {#github-action}

Kani には、CI で Kani を実行するための GitHub Action があります。
今のところ、CI で Kani を使えるのは `x86_64-unknown-linux-gnu` の Ubuntu 20.04 だけです。

## GitHub ワークフローで Kani を使う {#using-kani-in-your-github-workflow}
私たちの GitHub Action は [GitHub Marketplace](https://github.com/marketplace/actions/kani-rust-verifier) で公開されています。

次のワークフローの断片は、プッシュやプルリクエストがあるたびに、リポジトリをチェックアウトして `cargo kani` を実行します。
`<MAJOR>.<MINOR>` を、実行したい Kani のバージョンに置き換えてください。

```yaml
name: Kani CI
on:
  pull_request:
  push:
jobs:
  run-kani:
    runs-on: ubuntu-20.04
    steps:
      - name: 'Checkout your code.'
        uses: actions/checkout@v3

      - name: 'Run Kani on your code.'
        uses: model-checking/kani-github-action@v<MAJOR>.<MINOR>
```

これで、チェックアウトしたコードに対して `cargo kani` が実行されます。

### オプション {#options}

このアクションは、次の省略可能なパラメータを受け取ります。

- `command`: 実行するコマンド。
  デフォルトは `cargo kani` です。
  ほとんどの場合、変える必要はありません。
- `working-directory`: コマンドを実行するディレクトリ。
  デフォルトは `.` です。
  リポジトリに複数のクレートがあり、そのうち 1 つだけで実行したい場合に便利です。
- `args`: 指定した `${command}` に渡す引数。
  オプションの完全な一覧は `cargo kani --help` を見てください。
  便利なオプションには次のものがあります。
  - `--output-format=terse`: 簡潔な出力を生成します。
  - `--sarif <path>`: GitHub Code Scanning にアップロードできる SARIF レポートを書き出します。
  - `--tests`: `test` モジュール内の証明に対して実行します（Bolero を実行するのに必要です）。
  - `--workspace`: リポジトリ内のすべてのクレートに対して実行します。

### SARIF を GitHub Code Scanning にアップロードする {#uploading-sarif-to-github-code-scanning}

Kani を `--sarif` 付きで実行した場合、レポートをアップロードすると、結果が Code Scanning のアラートとして表示されます。

```yaml
      - name: 'Run Kani on your code.'
        uses: model-checking/kani-github-action@v<MAJOR>.<MINOR>
        with:
          args: --sarif kani.sarif

      - name: 'Upload SARIF'
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: kani.sarif
```

SARIF の結果をアップロードするには、ワークフローに `security-events: write` の権限を与える必要があります。

## FAQ {#faq}
- **Kani が CI で時間がかかりすぎる**: 望む頻度で
  [スケジュール](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule)
  実行してみてください。
- **Kani がログを残さずに黙ってクラッシュする**: いくつかの理由が考えられます。
  - Kani がメモリ（RAM）を使い果たした。GitHub が提供する RAM は最大 7GB ですが、Kani は
    それ以上使うことがあります。ローカルで実行して確かめてください。
  - GitHub は 6 時間を超えるジョブを終了させます。
  - それ以外の場合は、[こちら](https://github.com/model-checking/kani/issues)に issue を登録することを検討してください。
{% endraw %}

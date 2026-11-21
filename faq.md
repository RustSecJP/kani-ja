{% raw %}
# よくある質問 {#faqs}

この節では、Kani についてよくある質問を集めています。
ここに載せてほしい質問があれば、[issue を作成する](https://github.com/model-checking/kani/issues/new/choose)ことを検討してください。

## 質問 {#questions}

<details>
<summary><code>kani::assume(false)</code> の後で Kani が失敗しません。なぜですか？</summary>
</br>

`kani::assume(false)`（または、`cond` がプログラムの文脈で `false` になる条件である `kani::assume(cond)`）は、Kani ではエラーを引き起こしません。
その代わり、このような仮定には、その仮定から先のすべての記号実行のパスを阻止する効果があります。
したがって、仮定の後のすべての検査は [`UNREACHABLE`](#../../verification-results.md) と表示されるはずです。
これが、Kani における `kani::assume(false)` の想定どおりの振る舞いです。

ハーネス内のある検査が `UNREACHABLE` になると思っていなかった場合は、ハーネスを制約しすぎていないか確かめるために、[`kani::cover` マクロ](#../../verification-results.md#cover-property-results)を使ってどんな条件が起こりうるかを調べることをおすすめします。
</details>

<details>
<summary>自分のクレートのものではない型に <code>kani::Arbitrary</code> トレイトを実装したところ、
<code>only traits defined in the current crate can be implemented for types defined outside of the crate</code> というエラーが出ました。
これはどういう意味ですか？どうすればよいですか？</summary>
</br>

このエラーは、トレイトの実装に関する Rust の孤児ルールに違反していることが原因です。孤児ルールは[こちら](https://doc.rust-lang.org/error_codes/E0117.html)で説明されています。
その場合は、非決定的な変数からオブジェクトを組み立てる関数を書く必要があります。
この関数の中では、その構成要素に対して任意の値を生成することで、単に任意の値を返します。

例えば、扱っている型が次の列挙型だとしましょう。

```rust
#[derive(Copy, Clone)]
pub enum Rating {
    One,
    Two,
    Three,
}
```

このとき、（`kani::any` が与える）非決定的な整数に対して match して、非決定的な `Rating` の列挙子を返せます。

```rust
    pub fn any_rating() -> Rating {
        match kani::any() {
            0 => Rating::One,
            1 => Rating::Two,
            _ => Rating::Three,
        }
    }
```

この方法はほかの場合にも役立ちます。詳しくは[こちら](https://model-checking.github.io/kani/tutorial-nondeterministic-variables.html#custom-nondeterministic-types)を見てください。

その型が `std`（Rust の標準ライブラリ）のものなら、Kani のライブラリに `Arbitrary` の実装を追加するよう[リクエストを作成](https://github.com/model-checking/kani/issues/new?assignees=&labels=%5BC%5D+Feature+%2F+Enhancement&template=feature_request.md&title=)できます。
そうでなければ、もっと手間のかかる選択肢を検討することになります。
 1. その型を定義している外部のクレートのコピーをインポートし、そこで `Arbitrary` を実装する。
 2. その型を定義している外部のクレートに、`Arbitrary` の実装を貢献する。
</details>
{% endraw %}

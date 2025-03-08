---
title: "【SvelteKit】コンポーネントの冒頭文字列を取得する【記事の冒頭抜粋】"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "css"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## まとめ

- インポート時にViteの`raw`クエリを用いると、ソースコードを取得できる
- 記事の冒頭抜粋などの描画には、CSSプロパティ`line-clamp`を使うべし

<!-- begin short upcoming concert announcement -->

:::message
私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。

次回は2025年7月にシューマンの交響曲第2番。
初めての方も、そうでない方も、お気軽にお越しください！

詳しくは[チケット販売ページ](https://teket.jp/1776/44429?uid=zenn)まで。
<!-- textlint-disable -->
:::
<!-- textlint-disable -->

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、ホームページの他にブログも運営しています。

https://blog.orch-canvas.tokyo/

ブログの記事一覧ページに、記事の冒頭抜粋を表示する必要がありました。

当ブログでは、各記事をコンポーネントとして管理しています。
そのため、「コンポーネントの冒頭文字列を取得する」ことで、記事の冒頭抜粋を取得できます。

:::message
ちなみに、各記事をコンポーネントとして管理するブログの実装のポイントは過去記事にまとめていますので、ぜひご一読ください！
[SvelteKitのコンポーネントをglobでインポートする【ワイルドカードを使いたい】](https://zenn.dev/orch_canvas/articles/sveltekit-import-component-dynamically)

<!-- textlint-disable -->
:::
<!-- textlint-disable -->

## コンポーネントの描画内容の取得

SvelteKitで通常使用するコンパイルツールViteは、ソースコードをそのまま文字列としてインポートする機能があります。
インポート時に、パスにクエリ `raw` を追加する方法です。

```ts
import code = './hoge.svelte?raw'
```

https://ja.vite.dev/guide/assets.html#importing-asset-as-string

### 冒頭文字列の抽出

簡単のために正規表現で不要部分を削除する方法をとります。

具体的には、次の要素を削除します。
コンポーネントの内容に合わせて、削除する内容は調整する必要があります。

- スクリプト
- スタイル
- ヘッダー
- タグ文字列
- 空白

```ts
/**
 * 冒頭抽出文字列を生成する。
 * @param rawPost 記事のHTML文字列
 * @returns 冒頭を抽出した文字列
 */
function convertToDescription(rawPost: string): string {
  return rawPost
    .replaceAll(/<script.+?<\/script>/gs, '') // スクリプトを削除
    .replaceAll(/<style.+?<\/style>/gs, '') // スタイリングを削除
    .replaceAll(/<h\d.+?<\/h\d>/gs, '') // ヘッダーを削除
    .replaceAll(/<.+?>/gs, '') // タグ文字列を削除
    .replaceAll(/\s+/gs, '') // 空白を削除
    .substring(0, 200);  // 十分に長い部分を抽出
}
```

本当であれば、Svelteの字句解析・構文解析を行うべきでしょうか。
例えば、Svelteには`parse()`や`compile()`など、抽象構文木へ変換する組み込み関数があります。
これらを利用することで、より厳格な処理ができそうです。

https://svelte.dev/docs/svelte/svelte-compiler

## CSSプロパティ「line-clamp」

抽出した文字列を描画するとき、「一定文字列を表示＋末尾に『……』を追加」とするのがシンプルです。

しかし、よりスマートな方法があります。
巷でよく話題になるCSSプロパティ「line-clamp」を使う方法です。

行数を指定すると、その行数を越えた分は省略されます。
また、文字列の最後には「…」が表示されます。

最小再現コードを示します。

```css
/* ベンダープレフィックスが必要 */
-webkit-line-clamp: 3;

/* 以下は「おまじない」 */
display: -webkit-box;
-webkit-box-orient: vertical;
overflow: hidden;
```

コメントに付記したようにいくつかのクセはありますが、ある程度は簡潔に記述することができます。

何より、画面サイズによる表示のずれをスマートに解消することができます。

https://developer.mozilla.org/ja/docs/Web/CSS/line-clamp

---

## おわりに

CSSプロパティ「line-clamp」は、対応しているブラウザ全てでベンダープレフィックス`webkit-`が必要なようです。

["line-clamp" | Can I use... Support tables for HTML5, CSS3, etc](https://caniuse.com/?search=line-clamp)

レンダリングエンジンとしてWebKitを採用していないChromium系やFirefox系のブラウザも。
なんでなんでしょうか。

W3Cの仕様書あたりを読めばわかるのか……
あるいはリポジトリの実装履歴を読めばわかるのか……

……この沼は深そうですねぇ

---

<!-- begin long upcoming concert announcement -->

## 次回演奏会のご案内

Orchestra Canvas Tokyoは、都内を中心に活動するアマチュアオーケストラです。

日々の癒しに、新たなひらめきのきっかけに——
オーケストラの演奏会はいかがでしょうか？

初めての方も大歓迎！
ご来場をお待ちしています。

> **Orchestra Canvas Tokyo**
> **第14回定期演奏会**
>
> 2025年7月12日(土)
> 練馬区立練馬文化センター 大ホール
> シューマン / 交響曲第2番 ほか
>
> [![](/images/regular-14.png =250x)](https://www.orch-canvas.tokyo/concerts/regular-14)
>
> 詳細は[チケット販売ページ](https://teket.jp/1776/44429?uid=zenn)にて

<!-- end long upcoming concert announcement -->

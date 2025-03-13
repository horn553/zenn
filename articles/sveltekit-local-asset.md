---
title: "【SvelteKit】結局、アセット(画像)はどこに配置するのか【「ローカルなアセット」というアイデア】"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "vite"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## まとめ

- Svelteのアセット管理は、`import` 文を介した記述が望ましい
  - Viteのビルトイン・ハンドリングで良しなに処理される
- 「ローカルなアセット」になるよう、`$lib` 配下への配置は避ける
  - 各Svelteファイルと同じディレクトリに配置するとよい

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

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、ホームページやブログをSvelteKitを用いて開発しています。

小規模ながらのサイトの開発・保守を続ける中で、SvelteKitにおいて **「ローカルなアセット」** という概念が有用なのではないか？という私見が生まれました。
今回はそれについてまとめていきます。

私が調べた限りでは類似の意見を見つけきれなかったのですが、もし心あたりがある方がいらっしゃいましたら、教えていただけると嬉しいです！

---

## 基本のアセット管理

まずは、基本のアセット管理について述べます。
SvelteKitのアセット管理の基本をまとめなおしたものですので、すでに慣れている方は、本題に迫る「[ローカルなアセット](#ローカルなアセット)」まで飛ばしてください。

公式ドキュメントには、2通りの管理方法が記載されています。
すなわち、`/static` への配置と、Vite[^1]のビルトイン・ハンドリングを用いる方法です。

[^1]: SvelteKitはビルドツールとして[Vite](https://ja.vite.dev/)を使用しています。

それぞれについて述べ、その後に比較検討を行います。

### `/static`

直接配信されるべきアセット、すなわちファビコンなどは `/static` に配置することとなっています。

https://svelte.jp/docs/kit/project-structure#Project-files-static

ここに配置することで、ビルド後も `/static` をドキュメントルートに見立てたパス関係で配信されるようになります。
例えば、`/static/favicon.ico` は `example.com/favicon.ico` として配信されるようになります。

`favicon.ico` や `robots.txt` など、おまじない的ファイルの配置に向けた使用が主ではありますが、ソースコードからの参照も可能です。
例えば、次のような形です。

```html:/src/routes/+layout.svelte
<script>
  let { children } = $props();
</script>

<header>
    <!-- /static/logo.svg を配置しているものとする -->
    <a href="/"><img src="/logo.svg" alt="Orchestra Canvas Tokyo"></a>
</header>

{@render children()}
```

基本の `+layout.svelte`[^2]をベースに、リンク付きロゴ画像を貼り付けたものです。
SvelteKitやViteを介することなく、HTMLの基本的な構文として、ルートからの相対パスで画像が指定されています。

[^2]: [ルーティング • Docs • Svelte](https://svelte.jp/docs/kit/routing#layout-layout.svelte)

### Viteのビルトイン・ハンドリング

しかし、画像のベストプラクティス[^3]として、ドキュメントには「Viteのビルトイン・ハンドリング」を用いる方法が紹介されています。
これに則って先のコードを修正すると、次のような形になります。

```html:/src/routes/+layout.svelte
<script>
  // 先ほどとは異なり、/src/routes/logo.svg を配置しているものとする
  import logo from './logo.svg'; // 相対パス指定となっていることに注意
  let { children } = $props();
</script>

<header>
    <a href="/"><img src={logo} alt="Orchestra Canvas Tokyo"></a>
</header>

{@render children()}
```

これも問題なく動作しますが、Viteによる処理を介しています。

Viteのドキュメント[^4]によると、次のような処理が行われます。

- ファイル名の変更: `logo.xxxxxxx.png` とハッシュ様の文字列が挟まれる
- 一定サイズ[^5]以下である場合、インライン化される
- 指定ディレクトリ（デフォルトでは `/_assets`）へ配置される

これらの処理により、キャッシュによる更新遅延の回避、読み込みパフォーマンスの改善などが期待されます。

[^3]: [Introduction / Dynamic attributes • Svelte Tutorial](https://svelte.jp/tutorial/svelte/dynamic-attributes)。
同ページでは@sveltejs/enhanced-imgやCDNを選択肢に入れて、ベストプラクティスについて議論されている。
この議論については、本記事の要点とはずれるため割愛する。

[^4]: [静的アセットの取り扱い | Vite](https://ja.vite.dev/guide/assets)

[^5]: サイズ閾値はビルドオプションの `build.assetsInlineLimit` で指定される。
詳細は[ビルドオプション | Vite](https://ja.vite.dev/config/build-options#build-assetsinlinelimit)に記述あり。

### 使い分け

基本的にはViteのビルトイン・ハンドリング、すなわち `import` 文を介した記述が優先されると考えます。
先述したように、パフォーマンスの利点があるためです。

これから述べる、「ローカルなアセット」という考え方でも、Viteのビルトイン・ハンドリングの利用を前提としています。

しかし、次の要素のいずれかを満たす場合、`/static` フォルダへの配置が必要です。

- 決められたパス・ファイル名が必要なもの
  - 例）
    - `favicon.ico`
    - `robots.txt`
- Svelte上で参照されないもの
  - 例）
    - CSSで `@import` されるフォントファイル
    - HTMLメールなどから直接参照されるファイル
  - ただし、CSSでの `url()` インポートはViteでハンドリングされる

---

## ローカルなアセット

多くの場合、Viteのビルトイン・ハンドリングが有用ということは分かりました。
では、アセットを実際にどこに配置するのかが問題となります。

主な選択肢は次の2つです。

1. インポートするSvelteファイルと同じディレクトリへ配置する
1. `$lib` 配下[^6]に配置する

[^6]: Svelteのビルトイン・エイリアスで、`/src/lib` を指します。

私は、前者「インポートするSvelteファイルと同じディレクトリへの配置」が第一選択であると考えます。
これこそが、「ローカルなアセット」という考え方です。

これの考え方を推す理由が2つあります。
すなわち、

- `+page.svelte` の登場
- 依存関係の単純化

です。

### `+page.svelte` の登場

ご存じかとは思いますが、現行のSvelteでは[^7]ルーティングにおいて、`+page.svelte` が用いられます。

[^7]: かつては違いました

この仕様により、各エンドポイントがディレクトリ分けされています。
すなわち、各エンドポイントを構成するファイル群をまとめて管理することができます。

プレフィックス `+` も効いており、エクスプローラ上でSvelte関連のファイルが最上部に表示されるようになります。

プロジェクト構成の一例を次に示します。

```plain:プロジェクト構成（抜粋）
/src/
└ routes/
　└ about-us/
　　├ +page.svelte
　　├ logo.svg
　　├ photo.jpg
　　└ sponsor.png
```

`/src/routes/about-us/+page.svelte` で使うファイル群を `/src/routes/about-us/` に配置しています。

ちょうどディレクトリがアセットのスコープのように機能しており、アセットの依存関係が一目瞭然ではないでしょうか！

### 依存関係の単純化

配置場所のもう1つの選択肢である、「`$lib` 配下への配置」は、依存関係を複雑化させる懸念があります。
つまり、複数のSvelteファイルから参照される、いわば「グローバル・アセット」が氾濫する危険があるということです。

●悪い例

とはいえ、IDEのワークスペース横断検索を行えば、実際にどのファイルから参照されているかは分かるので、致命的な問題とはなりません。
しかし、「グローバル変数」があまり好まれないのと同様の理由で、「グローバル・アセット」も避けられてしかるべきではないでしょうか。

「`$lib` 配下への配置」に付随する問題として、`$lib` 配下のフォルダ・ファイル管理が煩雑になることも挙げられます。

## 実用上のポイント

「ローカルなアセット」の運用に際し、いくつかポイントがあります。

### 複数ファイルからインポートされるアセット

> 複数ファイルからインポートされるアセットがあるとき、ローカルなアセットでは問題が生じるのではないか！

というのは、真っ先に想定されうる懸念です。

●問題に関する説明

これに対し、`$lib/assets/` などで共通ライブラリとしてアセットを用意し、そこに配置するというのが消極的な解決方法でしょう。

しかし、コンポーネントの設計を見直すような「積極的な解決」を図ることで、多くの場合は「ローカルなアセット」を維持できます。
そのため、実際には共通ライブラリとして用意するような解決方法が必要となる場合は、非常に稀であると考えています。

例えば、ヘッダーとフッターに同じロゴ画像を用いる場合を考えます。
消極的な解決方法では、次のようなプロジェクト構成になります。

```plain:プロジェクト構成（抜粋）
/src/
├ lib/
│├ Footer.svelte
│├ Header.svelte
│└ assets/
│　└ logo.svg
└ routes/
　└ +layout.svelte
```

```html:+layout.svelte
<script>
  import Header from '$lib/Header.svelte';
  import Footer from '$lib/Footer.svelte';
  let { children } = $props();
</script>

<Header />

{@render children()}

<Footer />
```

これに対し、コンポーネントの設計を見直し、ロゴ画像をプロパティとして渡すようにすると、アセットをローカルに管理できます。
いわば、「ロゴ画像を外から注入」します。

```plain:プロジェクト構成（抜粋）
/src/
├ lib/
│├ Footer.svelte
│└ Header.svelte
└ routes/
　├ +layout.svelte
　└ logo.svg
```

```html:+layout.svelte
<script>
  import Header from '$lib/Header.svelte';
  import Footer from '$lib/Footer.svelte';
  import logo from './logo.svg';
</script>

<Header {logo} />

{@render children()}

<Footer {logo} />
```

こうすることで、アセットをローカルに保つことができました。

別案として、次のような構成も考えられます。

```plain:構成（抜粋）
/src/
├ lib/
│└ layout
│　├ Footer.svelte
│　├ Header.svelte
│　└ logo.svg
└ routes/
　└ +layout.svelte
```

前述した `+` プレフィックスによるフォルダ内の見通しの良さは失われますが、アセットのスコープが明確となる点は魅力的ではないでしょうか。

---

## おわりに

この「ローカルなアセット」という考え方、お察しかと思いますが、コンポーネントにも応用できそうです。
その詳細についてはまた来週にでも……

ここまでお読みいただきありがとうございました！

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

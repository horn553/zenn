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

- Svelteのアセット管理は主にViteのビルトイン・ハンドリングが有用
- アセットの配置は各Svelteファイルと同じディレクトリに配置する
- ●

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

小規模ながらのサイトの開発・保守を続ける中で、SvelteKitにおいて「ローカルなアセット」という概念が有用なのではないか？という私見が生まれました。
今回はそれについてまとめていきます。

私が調べた限りでは類似の意見を見つけきれなかったのですが、もし心あたりがある方がいらっしゃいましたら、教えていただけると嬉しいです！

---

## 基本のアセット管理

まずは、基本のアセット管理について述べます。
SvelteKitのアセット管理の基本をまとめなおしたものですので、すでに慣れている方は、本題に迫る「[ローカルなアセット](#ローカルなアセット)」まで飛ばしてください。

公式ドキュメントには、2通りの管理方法が記載されています。
すなわち、`/static` への配置と、Viteのビルトイン・ハンドリングを用いる方法です。

それぞれについて述べ、その後に比較検討を行います。

### `/static`

直接配信されるべきアセット、すなわちファビコンなどは `/static` に配置することとなっています。

https://svelte.jp/docs/kit/project-structure#Project-files-static

ここに配置することで、ビルド後も `/static` をドキュメントルートに見立てたパス関係で配信されるようになります。
例えば、`/static/favicon.ico` は `example.com/favicon.ico` として配信されるようになります。

この機能を使うと、次のようなソースコードを記述できます。

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

基本の `+layout.svelte`[^1]をベースに、リンク付きロゴ画像を貼り付けたものです。
SvelteKitやViteを介することなく、HTMLの基本的な構文として、絶対リンクで画像が指定されています。

[^1]: [ルーティング • Docs • Svelte](https://svelte.jp/docs/kit/routing#layout-layout.svelte)

### Viteのビルトイン・ハンドリング

しかし、画像のベストプラクティス[^2]として、ドキュメントには「Viteのビルトイン・ハンドリング」を用いる方法が紹介されています。
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

Viteのドキュメント[^3]によると、次のような処理が行われます。

- ファイル名の変更: `logo.xxxxxxx.png` とハッシュ様の文字列が挟まれる
- 一定サイズ[^4]以下である場合、インライン化される

これらの処理により、それぞれキャッシュによる更新遅延の回避、読み込みパフォーマンスの改善が期待されます。

[^2]: [Introduction / Dynamic attributes • Svelte Tutorial](https://svelte.jp/tutorial/svelte/dynamic-attributes)。
同ページでは@sveltejs/enhanced-imgやCDNを選択肢に入れて、ベストプラクティスについて議論されている。
この議論については、本記事の要点とはずれるため割愛する。

[^3]: [静的アセットの取り扱い | Vite](https://ja.vite.dev/guide/assets)

[^4]: サイズ閾値はビルドオプションの `build.assetsInlineLimit` で指定される。
詳細は[ビルドオプション | Vite](https://ja.vite.dev/config/build-options#build-assetsinlinelimit)に記述あり。

### 使い分け

基本的にはViteのビルトイン・ハンドリングが優先されると考えます。
先述したように、パフォーマンスの利点があるためです。

しかし、次の要素のいずれかを満たす場合、`/static` フォルダへの配置が必要です。

- 決められたパス・ファイル名が必要なもの
  - 例）`favicon.ico`、`robots.txt`
- Svelte上で参照されないもの
  - 例）CSSで `@import` されるフォントファイル、HTMLメールなどから直接参照されるファイル
  - ただし、CSSでの `url()` インポートはViteでハンドリングされる

---

## ローカルなアセット

前述したように多くの場合、Viteのビルトイン・ハンドリングが有用です。
では、アセットを実際にどこに配置するのかが問題となります。

主な選択肢は次の2つです。

1. インポートするSvelteファイルと同じディレクトリへ配置する
1. `$lib` 配下[^5]に配置する

[^5]: Svelteのビルトイン・エイリアスで、`/src/lib` を指します。

私は、「インポートするSvelteファイルと同じディレクトリ」への配置が第一選択であると考えます。
これこそが、「ローカルなアセット」という考え方です。

これの考え方を推す理由が2つあります。
すなわち、

- `+page.svelte` の登場
- 依存関係の単純化

です。

### `+page.svelte` の登場

ご存じかとは思いますが、現行のSvelteでは[^6]ルーティングにおいて、`+page.svelte` が用いられます。

[^6]: かつては違いました

この仕様により、各エンドポイントがディレクトリ分けされています。
すなわち、各エンドポイントを構成するファイル群をまとめて管理することができます。

プレフィックス `+` も効いており、エクスプローラ上でSvelte関連のファイルが最上部に表示されるようになります。

プロジェクト構成の一例を次に示します。

```plain:プロジェクト構成（抜粋）
/src/
└ routes/
　├ about-us/
　│├ +page.svelte
　│├ logo.svg
　│├ photo.jpg
　│└ sponsor.png
　├ +layout.svelte
　└ +page.svelte
```

`/src/routes/about-us/+page.svelte` で使うファイル群を `/src/routes/about-us/` に配置しています。

#### コンポーネントへの応用

`$lib` 配下に作成するコンポーネント[^7]でも、同様の考え方を適用できます。

[^7]: [ローカルなコンポーネントやsnippetとして記述できないか？](#ローカルなコンポーネントやsnippetとして記述できないか)で後述しますが、「ローカルなコンポーネント」やsnippetとして作成するのが適当でない場合にのみ、`$lib` 配下に作成するべきと考えています。

これはオリジナルの管理ですので、将来的に問題が生じる可能性を否定できません。
しかし、`routes/` 配下と統一感のある管理ができるため、オススメです。

```plain:プロジェクト構成（抜粋）
/src/
└ lib/
　└ Header/
　　├ +component.svelte
　　└ logo.svg
```

```html:+component.svelte
<script>
  import Header from '$lib/Header/+component.svelte';
</script>

<Header />
```

### 依存関係の単純化

配置場所のもう1つの選択肢である、「`$lib` 配下への配置」は、依存関係を複雑化させる懸念があります。
つまり、複数のファイルから参照される、いわば「グローバル・アセット」が氾濫する危険があるということです。

とはいえ、IDEのワークスペース横断検索を行えば、実際にどのファイルから参照されているかは分かるので、致命的な問題とはなりません。
しかし、「グローバル変数」があまり好まれないのと同様の理由で、「グローバル・アセット」も避けられてしかるべきではないでしょうか。

付随する問題として、`$lib` 配下のフォルダ・ファイル管理が煩雑になることも挙げられます。

## 実用上のポイント

「ローカルなアセット」の運用に際し、いくつかポイントがあります。

### 複数ファイルからインポートされるアセット

●（コンポーネント設計を）

### ローカルなコンポーネントやsnippetとして記述できないか？

●（コンポーネントもローカルに。snippetもできたしね！関数分けとファイル分けみたいな）

---

## おわりに

グローバル変数、ローカル変数といった既知の概念における「グローバル」「ローカル」の意味するところを分解し、「アセット」や「コンポーネント」に適用する――

このような応用をして遊んでみると、たま～に思いがけずキレイな知見に辿り着くことがあります。
かけがえのない、楽しい瞬間です。

情報の海の上を泳ぎ周る日々ですが、たまにはぷかぷかと漂ってみるのも悪くないものです！

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

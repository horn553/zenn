---
title: "【SvelteKit】「ローカルなコンポーネント」というアイデア"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "vite"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## まとめ

- コンポーネントの記述方法は、スコープの広さが異なるいくつかの方法が考えられる
  - snippet < ローカルなコンポーネント < 共通ライブラリとしてのコンポーネント
  - 必要最低限のスコープを有するコンポーネントを採用すべき
- ディレクトリ構造を有するコンポーネントの場合、`+page.svelte` のように、`+component.svelte` と命名するのもオススメ

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

小規模ながらのサイトの開発・保守を続ける中で、SvelteKitにおいて「ローカルなコンポーネント」という概念が有用なのではないか？という私見が生まれました。
今回はそれについてまとめていきます。

私が調べた限りでは類似の意見を見つけきれなかったのですが、もし心あたりがある方がいらっしゃいましたら、教えていただけると嬉しいです！

:::message
前回の記事[「【SvelteKit】結局、アセット(画像)はどこに配置するのか【「ローカルなアセット」というアイデア】」](https://zenn.dev/orch_canvas/articles/sveltekit-local-asset)のコンポーネント版です。

前回の記事を読んでいただくと、より見通しよく読んでいただけるかと思います！
<!-- textlint-disable -->
:::
<!-- textlint-disable -->

---

## できるだけローカルに記述する

コンポーネントはできるだけ「ローカルに記述」するのが好ましいと考えます。

具体的には、規模が小さい順に次の選択肢を採用するべきだと考えています。

1. スニペット
1. ローカルなコンポーネント
1. 共通ライブラリとしてのコンポーネント

順に述べていきます。

### スニペット

Svelte5の新機能です。
ちょうど、一般のプログラミングで **共通処理を関数として再利用** するように記述することができます。

https://svelte.jp/docs/svelte/snippet

```html:+layout.svelte
<script>
  import logoSource from './logo.svg'
  let { children } = $props();
</script>

{#snippet logo()}
  <a href="/">
    <img src={logoSource} alt="Orchestra Canvas Tokyo" width=300 height=100>
  </a>
{/snippet}

<header>
  {@render logo()}
</header>

{@render children()}

<footer>
  {@render logo()}
</footer>
```

### ローカルなコンポーネント

ちょうど、一般のプログラミングで **長めの共通処理を、別ファイルとして切り出す** ように管理することができます。

```plain:プロジェクト構成
/src/
└ routes/
　├ +layout.svelte
　├ Logo.svelte
　├ logo-large.svg
　└ logo-small.svg
```

```html:Logo.svelte
<script>
  import logoLarge from './logo-large.svg'
  import logoSmall from './logo-small.svg'
</script>

<a href="/">
  <picture>
    <srcset srcset={logoLarge} media="(min-width: 600px)">
    <img src={logoSmall} alt="Orchestra Canvas Tokyo">
  </picture>
</a>

<style>
  img {
    /* 省略 */
  }
</style>
```

```html:+layout.svelte
<script>
  import Logo from './Logo.svelte';
  let { children } = $props();
</script>

<header>
  <Logo />
</header>

{@render children()}

<footer>
  <Logo />
</footer>
```

### 共通ライブラリとしてのコンポーネント

ちょうど、一般のプログラミングで **離れた複数ファイルの処理を、ライブラリとして切り出す** ように管理することができます。

一般のプログラミングと同様に、「本当に同じことを意味する処理なのか」を検討する必要がありますが、汎用性が高く強力な一手です。
強力ではある分、依存関係が不明瞭になり、コードの見通しの悪さの原因となりえることに留意する必要があります。

```plain:プロジェクト構成
/src/
├ lib/
│└ Logo
│　├ Logo.svelte  ※ 任意のファイル名を指定可能
│　├ logo-large.svg
│　└ logo-small.svg
└ routes/
　└ +layout.svelte
```

```html:Logo.svelte
<script>
  import logoLarge from './logo-large.svg'
  import logoSmall from './logo-small.svg'
</script>

<a href="/">
  <picture>
    <srcset srcset={logoLarge} media="(min-width: 600px)">
    <img src={logoSmall} alt="Orchestra Canvas Tokyo">
  </picture>
</a>

<style>
  img {
    /* 省略 */
  }
</style>
```

```html:+layout.svelte
<script>
  import Logo from '$lib/Logo/Logo.svelte';
  let { children } = $props();
</script>

<header>
  <Logo />
</header>

{@render children()}

<footer>
  <Logo />
</footer>
```

#### ファイルの命名案：`+component.svelte`

先の例にも上げましたが、ディレクトリ構造をもつコンポーネントについて、コンポーネントに `+component.svelte` と命名するのはいかがでしょうか？
ちょうど、ルーティングにおいて `+page.svelte` や `+layout.svelte` を作成する形式をまねた者です。

[以前の記事で述べた](https://zenn.dev/orch_canvas/articles/sveltekit-local-asset)ローカルなアセットの考え方を、コンポーネントに対するアセットにも応用することができます。
アセットを用いない場合は、シンプルさを優先してディレクトリ構造を採用しないのも手です。

これはオリジナルの管理ですので、将来的に問題が生じる可能性を否定できません。
しかし、`routes/` 配下と統一感のある管理ができるため、オススメです。

先の例について、この方式を採用すると次のようになります。

```plain:プロジェクト構成
/src/
├ lib/
│└ Logo
│　├ +component.svelte
│　├ logo-large.svg
│　└ logo-small.svg
└ routes/
　└ +layout.svelte
```

```html:+layout.svelte
<script>
  import Logo from '$lib/Logo/+component.svelte';
  let { children } = $props();
</script>

<header>
  <Logo />
</header>

{@render children()}

<footer>
  <Logo />
</footer>
```

---

## おわりに

最後までお読みいただきありがとうございます！

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

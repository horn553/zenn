---
title: "【SvelteKit】「ローカルなコンポーネント・アセット」というアイデア"
emoji: "💡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## まとめ

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

小規模ながらのサイトの開発・保守を続ける中で、SvelteKitにおいて「ローカルなコンポーネント・アセット」という概念が有用なのではないか？という私見が生まれました。
今回はそれについてまとめていきます。

私が調べた限りでは類似の意見を見つけきれなかったのですが、もし心あたりがある方がいらっしゃいましたら、教えていただけると嬉しいです！

---

## コンポーネントの配置

記事タイトルの「コンポーネント・アセット」という順序には、意図があります。
コンポーネントと、それに従属するアセットという意味合いをもっています。
そのため、まずはコンポーネントの配置について論じます。

### `$lib/` こと `/src/lib/`

SvelteKitでは、`/src/lib` にライブラリのコード（ユーティリティやコンポーネント）を格納することを想定しています[^1]。
このディレクトリには、`$lib` エイリアスがデフォルトで指定されています[^2]。

[^1]: [プロジェクト構成 • Docs • Svelte](https://svelte.jp/docs/kit/project-structure#Project-files-src)
[^2]: このようなエイリアスは他に作成することもできます[^3]。
しかし、`$lib` はユーザー作成のエイリアスとは別の、ビルトイン・エイリアスとして管理されています。
[^3]: [Configuration • Docs • Svelte](https://svelte.jp/docs/kit/configuration#alias)

例えば、外部リンクを配置するコンポーネントを考えます。

```html:/src/lib/ExternalLink.svelte
<script>
  // /src/lib/external-link.svg が配置されているとする
  import icon from './external-link.svg'
  let { href, children } = $props();
</script>

<!-- noreferrer, noopener の有無については考慮しない -->
<a {href}>
  {@render children()}
  <img src={icon} alt="（外部リンク）">
</a>
```

このような記述は

## おわりに

●

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

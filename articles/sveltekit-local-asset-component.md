---
title: "【SvelteKit】「ローカルなアセット・コンポーネント」というアイデア"
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

小規模ながらのサイトの開発・保守を続ける中で、SvelteKitにおいて「ローカルなアセット・コンポーネント」という概念が有用なのではないか？という私見が生まれました。
今回はそれについてまとめていきます。

私が調べた限りでは類似の意見を見つけきれなかったのですが、もし心あたりがある方がいらっしゃいましたら、教えていただけると嬉しいです！

---

## アセット

アセットの管理、コンポーネントの管理それぞれについて、順を追って述べていきます。
まずはアセットについてです。

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

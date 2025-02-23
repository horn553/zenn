---
title: "SvelteKitでenhanced-imgからCloudflare Imagesに移行した"
emoji: "🖼️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sveltekit", "cloudflare", "cloudflareimages"]
publication_name: "orch_canvas"
published: true
---

## まとめ

- SvelteKitプロジェクトにて、enhanced-imgからCloudflare Images (Transform via URL)へ移行した
- 変換元の画像はリポジトリ上に用意した
- Cloudflare DNSのセットアップが必要だが、DNS、Imagesともに無料枠内で利用できる
- ビルドエラーの解消、ビルド時間の短縮を得られた

<!-- begin short upcoming concert announcement -->

:::message
私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。

次回は2025年7月にシューマンの交響曲第2番を演奏します。
初めての方も、そうでない方も、お気軽にお越しください！

詳しくは[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-14)まで。
<!-- textlint-disable -->
:::
<!-- textlint-disable -->

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）のWebサイトはSvelteKitで開発されており、Cloudflare Pagesにデプロイされています。

また、画像読み込みパフォーマンスの最適化として、@sveltejs/enhanced-imgを利用しています。
ビルド時にフォーマットや画像サイズの最適化を行い、pictureタグを用いた出力まで行ってくれるパッケージです。

https://svelte.dev/docs/kit/images

このパッケージは便利かつ簡単に利用できるのですが、2つの問題が生じました。

### 問題① Cloudflare Pagesビルドでの"Segmentation fault"

画像数が一定以上多くなると、Cloudflare Pagesでのビルドが失敗するようになりました。

主たるエラー内容は"Segmentation fault"と非特異的なものです。
エラー原因の特定に苦慮しましたが、enhanced-imgを利用する画像数を減らすことでエラーは出現しなくなることから、特定に至りました。

最適化する画像を絞るにしても、これからコンテンツが増えていくといずれ同じ問題に直面するであろう……
Cloudflare Pagesの継続的デリバリー（CD）は大変便利なので、何とか使い続けたい……

enhanced-imgからの乗り換え先を模索するきっかけとなりました。

### 問題② ビルド時間の長大化

何とか我慢していましたが、ビルド時間の長大化も問題でありました。

100歩譲ってCDでのビルド時間の長大化は許容するにせよ、ローカル開発環境での読み込みに時間がかかるのはストレスでした。

これらの問題を解決する移住先として、Cloudflare Imagesに白羽の矢が立ちました。

## Cloudflare Images

https://developers.cloudflare.com/images/

Cloudflare Imagesは、Cloudflareが提供する画像関連の管理・処理を簡便化するサービス群です。

このうち、今回は[Transform via URL](https://developers.cloudflare.com/images/transform-images/transform-via-url/)を利用します。

### 利用料金

5,000 unique transformations/月まで無料で利用できます。
同じ画像、同じ形式へのtransformationは月に1度しかカウントされないようです。

一般的な静的Webサイトでは、無料枠で事足りそうです。

https://developers.cloudflare.com/images/pricing/

### 利用にはCloudflare Zoneの設定が必要

Transform via URLで利用するURLは次のような形式です。

```plain
https://<ZONE>/cdn-cgi/image/<OPTIONS>/<SOURCE-IMAGE>

e.g.) https://blog.orch-canvas.tokyo/cdn-cgi/image/format=auto/https://blog.orch-canvas.tokyo/hoge.png
```

ここで`<ZONE>`とは、CloudflareでDNSセットアップされたホスト名のことです。
すなわち、**独自ドメインを所有**しており、**Cloudflare DNSを利用するように設定する必要がある**ということです。

もう一度言います。
**独自ドメインを所有**しており、**Cloudflare DNSを利用するように設定する必要がある**ということです。

でも心配ありません。
他社で取得した独自ドメインも、Cloudflareへ移管することなしにCloudflare DNSを利用するようセットアップできます。

例えば、当団が利用している`.tokyo`ドメインはそもそもCloudflareへ移管できません。
しかし、問題なくCloudflare DNSのセットアップを行うことができました。

また、Cloudflare DNSには無料プランが用意されており、Cloudflare Imagesの利用にはこれで十分です。
気兼ねなく移行していきます。

移行作業は、Cloudflareのドキュメントに従えばスムーズに完了します。
cPanel利用環境ではメールサーバー向けのDNS設定が追加で必要でしたが……

https://zenn.dev/kokkosan/scraps/0c54fd44cfbe9c

### Cloudflare Images Transformationの有効化

Cloudflare DNSのセットアップが完了したら、CloudflareダッシュボードでImagesのTransformationsを有効化する必要があります。

![Cloudflare ImagesでTransformationsを有効化する](/images/sveltekit-cloudflare-images/01.png)

## 実装

実装します。
が、その前に2つほど追加の検討が必要です。

### 検討① transformationにどのようなオプションを付与するか

今回は、次のようなオプションとしました。

- format=auto
- fit=scale-down
- height=1920 or height=3840

#### format

流行りのフォーマットはWebPやAVIFでしょうか。

AVIFはより高い圧縮率を誇ります。しかし、未対応ブラウザがあり……Can I Useで調べると……
と考えるのは楽しいですが、Cloudflare Imagesを前にそのような検討は不要です。

`format=auto`オプションでリクエストすることで、HTTPリクエストヘッダーのAcceptヘッダーをみてAVIFかWebPを適切な方を返してくれます。
とても便利なことです。

#### fit, height

事前に画像サイズが分かっていれば、それをリクエストするのみです。
もちろんwidthオプションも用意されています。

今回はある程度動的なサイズになるので、[Cloudflareの推奨](https://developers.cloudflare.com/images/transform-images/transform-via-url/#recommended-image-sizes)に従い1920pxとしました。
縦長のフライヤー画像を取り扱うので、heightを指定しています。

高ディスプレイ密度環境に対応するため、2倍のheight指定も想定しておきます。

また、小さい画像に対しわざわざ拡大されないよう、`fit=scale-down`を指定しておきます。

### 検討② どの条件でtransformationをリクエストするか

デフォルトでは、transformationはゾーンと同一のオリジンからのリクエストのみに対して行われます。

例えば、`orch-canvas.tokyo`というゾーンを指定している場合、`blog.orch-canvas.tokyo`からのリクエストは正常に処理されます。
一方で、`example.com`からのリクエストははじかれます。

この制限はダッシュボード上で簡単に無効化できますが、今回はデフォルト設定を尊重することとしました。
つまり、ホスト名が`blog.orch-canvas.tokyo`の場合のみ、transformationがかかった画像をリクエストすることとします。

### コンポーネント作成

検討事項も解消したところで、実際のコーディングに移ります。

主な仕様は次の通りです。

- ホスト名が`blog.orch-canvas.tokyo`の場合、transformationをリクエスト
- オプションはformat、fit、height
  - heightは1920と3840を指定する
  - `height=3840`は高ピクセル密度ディスプレイ向けで、imgタグのsrcset属性を使用して設定する

```html:Flyer.svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let {
    src,
    alt
  }: {
    src: string;
    alt: string;
  } = $props();

  const commonOptions = [
    ['format', 'auto'],
    ['fit', 'scale-down']
  ] satisfies [string, string][];

  let useCloudflareImages = $state(false);
  onMount(() => {
    useCloudflareImages = new URL(window.location.href).hostname === 'blog.orch-canvas.tokyo';
  });

  function getCloudflareSrc(src: string, options: [string, string][]): string {
    const optionsString = options.map(([key, value]) => `${key}=${value}`).join(',');

    // '/' 始まりの場合は除去したパスを指定する
    return `https://blog.orch-canvas.tokyo/cdn-cgi/image/${optionsString}/${src.startsWith('/') ? src.slice(1) : src}}`;
  }
</script>

{#if useCloudflareImages}
  <img
    src={getCloudflareSrc(src, [...commonOptions, ['height', '1920']])}
    srcset={`${getCloudflareSrc(src, [...commonOptions, ['height', '3840']])} 2x`}
    {alt}
  />
{:else}
  <img {src} {alt} />
{/if}
```

リファクタリングの余地はありそうですが……
本質的には問題ないかと思います。

このコンポーネントの利用方法はenhanced-imgと同様です。
例えば次のような形です。

```html:+page.svelte
<script lang="ts">
  import flyer from './flyer.png'
  import Flyer from '$lib/Flyer.svelte'
</script>

<Flyer src={flyer} alt="演奏会のフライヤー">
```

こうすることで、ビルド時に画像が静的アセットとして良しなに管理されます。

---

## おわりに

仕様の把握と設計が済んでしまえば、簡潔に実装できました。
ビルドエラーは消失し、ビルド時間も1分足らずまで短縮されています。

ちなみにダッシュボードからは、削減された帯域幅を確認できます。
とても良好です。

![ダッシュボードから確認できる帯域幅](/images/sveltekit-cloudflare-images/02.png)

きれいなダッシュボードが用意された便利なサービスを無料で利用させていただき、ありがたいものです。
これからもCloudflareを活用する日々は続きそうです。

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
> 詳細は[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-14)にて

<!-- end long upcoming concert announcement -->

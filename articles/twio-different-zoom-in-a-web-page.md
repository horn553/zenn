---
title: "Webページのズームって2種類あんねん"
emoji: "⬜"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "browser", "svelte"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## まとめ

- PC系ブラウザの拡大縮小は **page zoom** （Initial Viewportのサイズが変化する）
- スマホ系ブラウザは **the visual viewport scale factor** （Visual Viewportのscaleが変化する）
- PC系ブラウザの拡大縮小率は正確には取得できないが、スマホ系ではできる

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2026年3月にバーンスタインのシンフォニックダンス。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

---

## PC系ブラウザの拡大縮小

PC系ブラウザの拡大縮小は **page zoom** であり、Initial Viewportのサイズが変化します。

……はて？

もう少し馴染みがある言葉で表現すると、
**innerHeight/WidthとdevicePixelRatioが変化する**
とでも言えましょうか。

この線で噛み砕いていきます。

innerHeight/Widthの説明

devicePixelRatioの説明

## Layout ViewportとVisual Viewportとは？

Document（1つのWebページ全体）の中に、Layout Viewportが1つあります。
Layout Viewportの中には、Visual Viewportが1つあります。

W3Cの仕様書で、スクロールを例としたアニメーションが示されています。

https://drafts.csswg.org/cssom-view/#example-vvanimation

---

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
> **第16回定期演奏会**
>
> 2026年3月15日(日)
> 府中の森芸術劇場 どりーむホール
> バーンスタイン / 「ウェストサイドストーリー」よりシンフォニックダンス ほか
>
> [![](/images/regular-16-flyer.jpg =250x)](https://www.orch-canvas.tokyo/concerts/regular-16)
>
> 詳細は[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)にて

<!-- end long upcoming concert announcement -->

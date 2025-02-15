---
title: "音楽記号を載せたい！ → Noto Musicはいかが？"
emoji: "🎼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlefonts", "music", "svelte"]
publication_name: "orch_canvas"
published: false
published_at: 2025-02-17 06:00
---

## まとめ

- 主要な音楽記号はUnicode上にコードポイントが割り当てられている
- 音楽記号を掲載したフリーフォントとして、Noto Musicがある
  - 例にもれず、Google Fontsで配信されている

<!-- begin short upcoming concert announcement -->

:::message
私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。

次回は2025年2月にブルックナーの交響曲第8番。
初めての方も、そうでない方もお気軽にお越しください！

詳しくは[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-13)まで。
<!-- textlint-disable -->
:::
<!-- textlint-enable -->

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、SvelteKitを用いてブログを作成しています。

ブログの主要コンテンツは楽曲解説であり、音楽記号を本文中に用いることがしばしばあります。
例えば、ベートーヴェンの交響曲第8番の楽曲解説には、_**ff**_（フォルティッシモ）や _**pp**_（ピアニッシモ）の表記があります。

http://blog.orch-canvas.tokyo/post/20230708-symphony-08

## 選択肢とその検討

音楽記号をどのように表現するか、３つの選択肢を考えました。

1. 一般的な欧文フォントで対応する
1. 画像ファイルを用意する
1. 音楽記号フォント「Noto Music」を利用する

### 選択肢① 一般的な欧文フォントで対応する

楽譜の主なアルファベット表記には、一般的な欧文フォントが用いられています。

例えば老舗楽譜浄書ソフトウェアであるFinale（現在は開発終了）では、Times New Romanが用いられています。
主な楽想記号（Allegro、cresc.など）は、これに太字、斜体などのスタイルをつけることで対応できます。

これを応用し、強弱記号などアルファベットが主体のものは、同様に欧文フォントで対応できそうです。

しかし、この選択肢では音部記号（ト音記号など）や音符（八分音符など）をはじめとして、いずれ対応が難しくなります。
今後も多数の記事を作成することが想定されるため、この選択肢は見送ることとしました。

### 選択肢② 画像ファイルを用意する

シンプルな解法です。
SVGなどベクター形式の画像を用意すれば、画質の劣化なくあらゆる環境に対応できます。

一方で、この解法では音楽記号が出現するごとに新しい画像を作成する必要があります。
当団はアマチュア・オーケストラであるという性質上、定常業務のコストをできるだけ低く抑えることが望ましいです。
そのため、よりよい解決策がないか検討を続けました。

### 選択肢③ 音楽記号フォント「Noto Music」を利用する

Finaleで用いられているKousakuやFinale Maestroなどの記譜用フォントがあればなぁ……
しかし、なかなか良いフォントは見つからず。

いつものように趣味のGoogle Fontsめぐりをしていたところ、見つけてしまいました。
「Noto Music」です。

https://fonts.google.com/noto/specimen/Noto+Music

Unicodeで規定された音楽記号のコードポイントに対するフォントです。
Unicodeの音楽記号は、強弱記号や音部記号といった基本的なものから、微分音など現代音楽で用いられるものまで幅広く網羅されています。
これは使えそうです。

https://www.unicode.org/charts/nameslist/n_1D100.html

## 実装例

Svelteでの当団の実装例をお示しします。
音楽記号全般に対応するコンポーネントを用意しています。

Unicodeのコードポイントをvalueとした辞書を用意しておく形をとっています。
普段の記事追加のしやすさと、新しい音部記号への対応しやすさのバランスをとったものです。

また、前後の文章とスタイルの整合性がとれるよう、CSSで微調整も行っています。

https://github.com/orchestra-canvas-tokyo/blog/blob/8860f384a5975d8c10e27ea86f3097ae1c9689d1/src/lib/component/post/MusicalNotation.svelte

## パフォーマンスの懸念

Google FontをはじめとするWebフォントといえば、濫用によるパフォーマンスの低下が問題とされがちです。

今回については、本文中（特に、本文中盤以降）に登場するコンテンツであり、画面内に描画される（もしくはユーザーが必要とする）までは時間に余裕があります。
Noto Musicの大きさは400kB程度であり十分に許容される水準と考えました。
CJK系のフォントでなければ、お手軽なものです。

今回は詳しく触れませんが、Google Fontsで読み込むフォント内容を一部に絞り、読み込み容量をより小さくするのも手です。
`text`クエリパラメータを用いることで、対応できます。

https://developers.google.com/fonts/docs/getting_started#optimizing_your_font_requests

---

## おわりに

やはりGoogle Fontsは便利ですね！
他にも、バリアブルフォント系や非言語フォント系（バーコード、点字、ダミーテキスト）など、沼は深いです。
面白い活用方法が見つけられたらまた記事にしたいものです。

---

<!-- begin long upcoming concert announcement -->

## 次回演奏会のご案内

Orchestra Canvas Tokyoは都内を中心に活動するアマチュアオーケストラです。

日々の癒しに。新しいひらめきのきっかけに。
オーケストラの演奏会はいかがですか？

初めての方も大歓迎！
ご来場お待ちしています。

> Orchestra Canvas Tokyo
> 第13回定期演奏会
>
> 2025年2月24日(月祝)
> 横浜みなとみらいホール
>
> ブルックナー / 交響曲第8番 ほか
>
> 詳細は[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-13)にて
>
> [![](/images/regular-13.png =250x)](https://www.orch-canvas.tokyo/concerts/regular-13)

<!-- end long upcoming concert announcement -->

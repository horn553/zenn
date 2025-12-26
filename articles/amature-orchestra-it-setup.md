---
title: "【オケ奏者のための】アマチュアオーケストラのITを立ち上げるには"
emoji: "🎻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["web", "email", "domain"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## まとめ

- アマチュアオーケストラで考慮すべきITは、主に「ドメイン」「ホームページ」「メール」「クラウドストレージ」「連絡ツール」
- 特に検討がむつかしい「ドメイン」「ホームページ」「メール」について、様々な方法を比較する
- 無料で簡単に組み立てる方法から、ブランディングを重視した方法まで、多様な選択肢が検討される

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2026年3月にバーンスタインのシンフォニックダンス。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）の5年間にわたる運営で得た知見をまとめます。

一般的なアマチュアオーケストラの運営に必要であろう、ドメイン・ホームページ・メール・クラウドストレージ・連絡ツール。
このうち、特に検討がむつかしい「ドメイン」「ホームページ」「メール」をこの順に解説していき　ます！

---

## ドメイン

あまり聞きなれない用語ではないでしょうか。
しかし、まずは「ドメイン」とやらの理解と検討が必要になります。

なぜなら、今後検討していくホームページやメールに何を採用するかを検討するとき、最も大きな分岐になるのは「**独自ドメインを用意するか**」だからです。

### ドメインとは

ドメインとは、当団だと「orch-canvas.tokyo」、このWebサイトだと「zenn.dev」のような、WebページではURLのはじめの方についている部分です。

このドメインについて、自団体のものを用意することを「**独自ドメインを用意する**」と表現されます。
一方で、用意しない場合、「●●.wordpress.com」や「●●.sakura.ne.jp」（●●は好きな英数字や`-`、`_`）といったドメインを用いることになります。

### 独自ドメインのメリット

独自ドメインを用意するメリットは、**ブランディング**にあります。
ホームページのURLは「定型句＋ドメイン＋情報の場所」で表されるため、独自ドメインを用意するとフライヤーなどでの見栄えがよいです。

また、**メールアドレス**についても「info@orch-canvas.tokyo」のように独自ドメインを含められます。
独自ドメインを取得しない場合、「orchestra.canvas.tokyo@gmail.com」のようなフリーメールを用いることになります。

> 例：
> 独自ドメインあり）
> 　ホームページ: https://www.orch-canvas.tokyo/concerts/regular-1
> 　お問い合わせ: info@orch-canvas.tokyo
>
> 独自ドメインなし）
> 　ホームページ: [https://orchestra-canvas-tokyo.wordpress.com/concerts/regular-1](https://www.orch-canvas.tokyo/concerts/regular-1)
> 　お問い合わせ: orchestra.canvas.tokyo@gmail.com

### 独自ドメインのデメリット

独自ドメインを用意するデメリットは、**維持費がかかること**にあります。
ドメインの種類（`.com`にするか`.jp`にするか）にもよりますが、「.jp」だと4,000円/年程度、「.com」だと3,000円程度かかります^[[さくらのドメイン](https://domain.sakura.ad.jp/)より、2025年12月18日アクセス]。

主なドメインは次のようなものでしょうか。

| ドメイン | 説明                                                       |
| -------- | ---------------------------------------------------------- |
| .com     | もとは企業を表すが、広く使われる                           |
| .org     | 非営利団体を表す                                           |
| .jp      | 日本であることを表す                                       |
| .gr.jp   | 日本の団体であることを表す<br>申請が必要                   |
| .tokyo   | 東京であることを表す<br>他の都道府県ドメインもいくつかある |

ちなみに、ドメインの`.com`や`.jp`の部分を特に「トップレベルドメイン」「TLD」といいます。

### 独自ドメインを取得する

独自ドメインを取得するには、**ドメインレジストラ**と呼ばれるサービスと契約する必要があります。
例えば[さくらのドメイン](https://domain.sakura.ad.jp/) や[Cloudflare Registrar](https://www.cloudflare.com/ja-jp/products/registrar/)といったサービスがあります。

https://domain.sakura.ad.jp/

https://www.cloudflare.com/ja-jp/products/registrar/

次に述べる、ホームページ作成サービスやレンタルサーバーを契約する場合、抱き合わせでドメインを取得することで割引価格や永久無料となる場合も少なくありません。

---

## ホームページ

団体の顔となるWebページです。
演奏会情報をはじめとして、団の理念や奏者募集などに関する情報を公開している場合が多いでしょうか。

主な選択肢としては次の通りでしょうか。

|  #  | 手法                                                                                                                                       | 難易度 | 月額料金 | 独自ドメイン | 特徴                                                                                  |
| :-: | ------------------------------------------------------------------------------------------------------------------------------------------ | :----: | :------: | :----------: | ------------------------------------------------------------------------------------- |
|  0  | 省略                                                                                                                                       | ☆☆☆ |   無料   |      －      | [Teket](https://teket.jp/)や[i-Amabile](https://i-amabile.com/)などでの情報公開で代用 |
|  1  | ホームページ作成サービス<br>[WordPress.com](https://wordpress.com/ja/)、[Wix](https://ja.wix.com/)、[Studio](https://studio.design/ja)など | ☆☆☆ |  無料～  |  有料版で可  | Web上で編集するだけで簡単だが、制約は多い                                             |
|  2  | レンタルサーバー x ホームページ管理ツール(WordPressなど)                                                                                   | ★★☆ | 数百円～ |      可      | ある程度自由度はあるが、基本的なIT知識やデザイン能力が必要                            |
|  3  | 自作                                                                                                                                       | ★★★ |  無料～  |      可      | 圧倒的な自由度はあるが、専門的知識が必要                                              |

各選択肢を検討していきます。

### 1. ホームページ作成サービス

[WordPress.com](https://wordpress.com/ja/)、[Wix](https://ja.wix.com/)、[Studio](https://studio.design/ja)などのホームページ作成サービスを用いるものです。

ほとんど全てで無料プランが用意されており、一般のアマチュアオーケストラにとって十分な機能が網羅されています。
ただし、**独自ドメインを利用する**場合や各サービスの宣伝バナーを消したい場合、月額有料プランが必要となります。

また、**編集が簡単**であることも大きなメリットです。
「プログラミングをせずホームページを作成する」のであれば、唯一の選択肢といって過言ではありません。

質の良いテンプレートをもとに、NotionやPowerPointのような直感的な操作でホームページを構築できます。

https://wordpress.com/ja/

https://ja.wix.com/

https://studio.design/ja

### 2. レンタルサーバー x ホームページ管理ツール

レンタルサーバーを契約し、その中でWordPress(WordPress.comではない)などのホームページ管理ツールを運用するものです。
主なレンタルサーバーといえば[さくらのレンタルサーバー](https://rs.sakura.ad.jp/)や[エックスサーバー](https://www.xserver.ne.jp/)、[カラフルボックス](https://www.colorfulbox.jp/)などでしょうか。

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
::::details WordPressとWordPress.comの違い
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

WordPressは世界で最も有名な**ホームページ管理ツール**のひとつです。

WordPress.comは、そのWordPressを用いて運営されている**ホームページ作成サービス**です。
レンタルサーバーなどを用意することなく、WordPressを使ってホームページを作成すできます。

::::

主要なレンタルサーバーサービスには、WordPressを簡単にセットアップする機能があります^[[さくらのレンタルサーバーの場合](https://rs.sakura.ad.jp/column/rs/wordpress_install/) | [エックスサーバーの場合](https://www.xserver.ne.jp/manual/man_install_auto_word.php) | [カラフルボックスの場合](https://help.colorfulbox.jp/article/easy-wp-install/)]。
この機能を使うことで、Webサイトを操作するだけでWordPressを運用できるようになります。

ひとたび運用開始できれば、あとの作業は「1. ホームページ作成サービス」と同様です。
しかし、運用開始までのセットアップはマニュアルの指示通りとはいえ、「DNS」や「SSL」といったやや専門的な用語が飛び交います……
（ITパスポートの試験レベルではありますが）

また、**独自ドメインの割り引き**を受けられることが多いことも特徴です。
同一サービスでドメインを管理することで、セットアップはスムーズに済みますし、相性問題といった複雑な事項を考慮する必要がありません。
独自ドメイン付きでホームページを管理する場合、**バランスの取れた選択肢**であると言えそうです。

https://rs.sakura.ad.jp/

https://www.xserver.ne.jp/

https://www.colorfulbox.jp/

### 3. 自作

自分でコーディングし、適当なホスティングサービスにデプロイします。
特に動的なサイトにおいて、レンタルサーバーの制約に合わない場合は強力な選択肢となります。
圧倒的な自由度が魅力ですが、プログラミングやインフラに関するある程度の知識を要します。

詳細の説明は省きますが、当団ではGitHubからCloudflare WorkersにCDしています。
ドメインはカラフルボックスで取得・維持しつつ、Cloudflare DNSに紐づけて運用しています。

---

## メール

●

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

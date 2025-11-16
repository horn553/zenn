---
title: "cPanel環境下のメールサーバー認証とCloudflare DNSとの相性問題を解消する"
emoji: "📧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "cloudflaredns", "cpanel"]
publication_name: "orch_canvas"
published: true
published_at: 2025-02-03 06:00
---

## まとめ

- cPanel環境下のメールサーバー認証とCloudflare DNSとの相性問題に遭遇した
- 以下のDNSレコードを追加することで、メールサービスが正常稼働するようになった
  - `mail A <サーバーのIPアドレス>`
  - プロキシステータス：無効（DNSのみ）

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2025年11月にバレエ音楽『火の鳥』組曲。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/52506?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、レンタルサーバーとして[ColorfulBox](https://www.colorfulbox.jp/)を利用しています。

ColorfulBoxをメールサーバーとして利用しているほか、ドメインも同サービスで取得しています。
また、ColorfulBoxではサーバー管理ツールとしてcPanelが使用されています。

2025年1月ごろ、Webサイト開発の簡便化のため、Cloudflare DNSを利用するよう環境を更新する必要が生じました。
詳しい経緯はこちらの記事をご覧ください。

https://zenn.dev/orch_canvas/articles/sveltekit-cloudflare-images

## セットアップで発生した問題

前述の経緯でCloudflare DNSへの移行を決意し、セットアップを進めたところ、問題が発生しました。
順を追って説明します。

### 一見順調なCloudflare DNSのセットアップ

公式ドキュメントでオススメされている「Full setup」の手順に従い、作業を進めました。

https://developers.cloudflare.com/dns/zone-setups/

参考までに、作業ログとして利用していたスクラップのリンクも載せておきます。

https://zenn.dev/kokkosan/scraps/0c54fd44cfbe9c

Webサイトへのアクセス、メール送受信ともに明らかなダウンタイムはなく、数十分程度でセットアップが終了したように思われました。

### スマホのメーラーから送受信できない

数時間後、出先でメール送受信に問題ないことを確認しようと、スマホのメーラーを開きます。
メール送受信ともにできませんでした。

まずは公式のトラブルシューティングに従って問題の特定・解決を図りましたが、どれにもあてはまりません。

https://developers.cloudflare.com/dns/troubleshooting/email-issues/

メーラーで詳しく状況を確認すると、メールサーバーとの接続に失敗しているようでした。

セットアップ時のメール送受信機能の確認は、レンタルサーバー管理画面であるcPanel上のWebメーラーで行っていました。
もしやと改めて確認すると、Webメーラーでの送受信機能は問題なく保たれていました。

すなわち、問題は一般的なメール送受信まわりのDNSレコードでなく、**cPanel環境下でのメールサーバー認証とCloudflare DNSとの相性**ということになりました。

## 問題の解決方法

cPanelは普及したサービスですので、英語で調べると多くの情報が見つかり、同様の事例も複数報告されていました。

それらの情報をもとに次のDNSレコードを追加し、問題は数分で解消しました。

- `mail A <サーバーのIPアドレス>`
- プロキシステータス：無効（DNSのみ）

Cloudflareとしては、プロキシを有効化することを推奨しています[^1]。
IPアドレスの露出が防がれることで、攻撃可能性が下がるからです。

[^1]: [Proxy status · Cloudflare DNS docs](https://developers.cloudflare.com/dns/manage-dns-records/reference/proxied-dns-records/#dns-only-records)

しかし、元々は露出していたものでありMXレコードでは露出していること、Cloudflare DNSを利用することで得られるメリットが大きいことから、上記のレコードを追加・維持することにしました。

---

## おわりに

本文中ではサラッと判明したように述べましたが、「cPanel×Cloudflare DNSで調べるとよい」と特定するのに苦労しました……

今回のように、ログを得にくかったり、非特異的だったりするエラーが出ると難しいですね。
コツとしては、「問題となっている環境の近さ」「普及した環境への近さ」の損益分岐点を探しつつ……検索の語句を調整しつつ……

ノンテクニカルスキルとして、あえて半日ほど時間をおいてみるというのもあるでしょうか。
新たな視点、新たなアプローチを見つけやすいです。

この辺りの技術を高めていくというのも、重要なITスキルだなぁと痛感した今日この頃です。

---

<!-- begin long upcoming concert announcement -->

## 次回演奏会のご案内

Orchestra Canvas Tokyoは、都内を中心に活動するアマチュアオーケストラです。

日々の癒しに、新たなひらめきのきっかけに——
オーケストラの演奏会はいかがでしょうか？

初めての方も大歓迎！
ご来場をお待ちしています。

> **Orchestra Canvas Tokyo**
> **第15回定期演奏会**
>
> 2025年11月24日(月祝)
> ミューザ川崎シンフォニーホール
> ストラヴィンスキー / バレエ音楽『火の鳥』組曲（1945年版） ほか
>
> [![](/images/next-concert.png =250x)](https://www.orch-canvas.tokyo/concerts/regular-15)
>
> 詳細は[チケット販売ページ](https://teket.jp/1776/52506?uid=zenn)にて

<!-- end long upcoming concert announcement -->

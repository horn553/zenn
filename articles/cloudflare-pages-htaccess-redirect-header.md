---
title: "【ヘッダー】Cloudflare Pagesで.htaccessっぽいこと、できます！【リダイレクト】"
emoji: "💁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "cloudflarepages", "htaccess"]
publication_name: "orch_canvas"
published: true
published_at: 2025-02-25 06:00
---

## まとめ

- Cloudflare Pagesでは、`_redirects`、`_headers`ファイルに特別な挙動が割り当てられている
- `_redirects`では、リダイレクトルールを設定できる
- `_headers`では、ヘッダールールを設定できる
  - クローラ向けのnoindexなどを設定可能

<!-- begin short upcoming concert announcement -->

:::message
私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。

次回は2025年7月にシューマンの交響曲第2番。
初めての方も、そうでない方も、お気軽にお越しください！

詳しくは[チケット販売サービス teket](https://teket.jp/1776/47046?uid=zenn)まで。
<!-- textlint-disable -->
:::
<!-- textlint-disable -->

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、Cloudflare Pagesを用いてホームページやブログを公開しています。

かつてはレンタルサーバー（[ColorfulBox](https://www.colorfulbox.jp/)）を使用していました。
その際、`.htaccess`ファイルを用いて以下のように設定していました。

- URLが変更されたページのリダイレクト
- 開発環境がクローラにインデックスされないよう、`X-Robots-Tag: noindex`を付与

Cloudflare Pagesでは`.htaccess`を利用できません。
その代わり、リダイレクトやヘッダーをテキストファイルで管理できます。

## `_redirects`ファイル

https://developers.cloudflare.com/pages/configuration/redirects/

一般的な301（Moved Permanently）、302（Found）リダイレクトを設定できます。
ワイルドカード（splat）やプレースホルダーを利用し、柔軟なルールを作成可能です。

### 設定例

```plain:_redirects
# 古いパス、新しいパス、ステータスコード
/old-source /new-source 301

# ワイルドカード（splat）を1つだけ使用可能
# `:splat`で参照できる
/old-contents/* /new-contents/:splat

# プレースホルダーを複数使用可能
/old-posts/:date/:slug /new-posts/:date-:slug
```

ステータスコードは省略可能で、デフォルトは302です。
基本的に301、302、303、307、308のみが利用可能です。

### プレースホルダーの詳細

正規表現`:[A-Za-z]\w*`に一致する命名が有効です。
英字で始まり、英数字およびアンダースコアを含む形式です。

```plain
# 有効な例
:hoge
:hoge2
:hoge_fuga
:hoge_

# 無効な例
:_hoge
:2nd_hoge
:hoge-fuga
```

### プロキシ

実は、ステータスコードとして`200`（OK）を指定することで、リダイレクトなしに別のURLのコンテンツを提供できます。

……この機能、レンタルサーバーをいじっていた頃からたびたび出会いますが、とてつもないロマンを感じるんですよね……
ロマン、感じませんか？
……感じませんよね……

閑話休題。
この機能を利用する場合、SEOへの影響を避けるためにcanonical URLの設定が推奨されています[^1]。

[^1]: https://developers.cloudflare.com/pages/configuration/redirects/#proxying

## `_headers`ファイル

https://developers.cloudflare.com/pages/configuration/headers/

`_redirects`と同じ形式で、ヘッダーを設定できます。

### 設定例（開発環境のインデックス禁止）

```plain
https://:project.pages.dev/*
  X-Robots-Tag: noindex
```

## 各ファイルの配置

各ファイルはプロジェクト（のビルド成果物）のルートフォルダに配置する必要があります。

例えば、当団が使用しているSvelteKitの場合、`static`フォルダに配置することで、想定通りの動作を得られます。

---

## おわりに

Cloudflare Pagesは新しいサービスらしく、シンプルで分かりやすい設定が魅力です。
加えて、公式ドキュメントが非常に充実しており……ありがたい限りですね！

<!-- textlint-disable -->
今後もドキュメント巡回がはかどりそうです🤤

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
> 詳細は[チケット販売サービス teket](https://teket.jp/1776/47046?uid=zenn)にて

<!-- end long upcoming concert announcement -->

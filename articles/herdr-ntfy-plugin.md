---
title: "herdrにntfy通知をつないだら、ヘッドレスPCが増えた"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["herdr", "ntfy", "ai", "tui", "docker"]
publication_name: "orch_canvas"
published: false
# published_at: 2026-07-03 06:00
---

## まとめ

- Herdrのagentが `done` または `blocked` になったら、ntfyへ通知する小さなプラグインを作りました
- SSH先の軽快なTUIでAIエージェントを並列に走らせ、終わったら手元のノートPCやスマートフォンに知らせます
- ベンダーロックインを避けつつ、Dockerで運べる作業環境を増やしていくのが、近ごろたいへん楽しいです

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2026年9月にR. シュトラウスの《アルプス交響曲》を演奏します。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/65178?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

## 作ったもの

`herdr-ntfy` を作りました。

https://github.com/horn553/herdr-ntfy

[Herdr](https://herdr.dev/) のagentが `done` または `blocked` になったとき、[ntfy](https://ntfy.sh/) へ通知します。
中身は `sh`、`curl`、`jq` だけで動く、ちいさなプラグインです。

```sh
herdr plugin install horn553/herdr-ntfy
config_dir="$(herdr plugin config-dir horn553.herdr-ntfy)"
touch "$config_dir/.env"
chmod 600 "$config_dir/.env"
$EDITOR "$config_dir/.env"
```

`.env` には通知先を置きます。

```sh
NTFY_URL=https://ntfy.sh/your-topic
NTFY_TITLE=Herdr
NTFY_TOKEN=
NTFY_LINES=12
```

保護されたtopicを使う場合は `NTFY_TOKEN` を設定します。
以上です。勢いよく入れれば鳴ります。めでたし。

## なぜ作ったか

HerdrのTUIが好きです。

SSHで自宅のマシンへ入り、TUIでagentをいくつか立てる。
出先のノートPCでも、スマートフォンでも、やることはだいたい同じです。
画面は軽い。指は速い。余計な同期を待たなくてよい。

ただし、agentを複数走らせると、終わりを見落とします。

「ビルドしておいて」
「このPRコメントを直して」
「この実装をレビューして」

こう頼んだあと、こちらは別の作業へ移ります。
しかし、終わったかどうかを見るためにTUIを何度も覗く。
これはいけません。人間が見張り番になっておりますぞ。

そこでntfyです。
agentが終わったら `✅`、詰まったら `🚫`。
通知タイトルにはworkspaceとtabを入れ、本文にはpaneの直近出力を載せます。

```text
Title: ✅ herdr-ntfy・main (Herdr)

Implemented the ntfy plugin.
Tests not run.
```

このくらいで十分でした。
凝ったダッシュボードより、まずは「終わったぞ」と知らせてくれる方が効きます。

## 今の使い方

一番うれしいのは、自分の作業場所を手元の端末から剥がせることです。

私はいま、自宅のデスクトップやミニPCにSSHで入り、HerdrのTUIを開いています。
そこでagentをいくつか立てる。
終わったらntfyが鳴る。
これで、ノートPCのバッテリーやブラウザのタブ数に気を取られずに済みます。

通知をntfyにしたので、Herdr本体と通知先を分けられました。
作業環境はDockerで持ち運びます。
それぞれが小さな部品なので、気に入らなくなったら差し替えればよい。
この雑に組み替えられる感じがよいのです。

最近はこれが楽しくて、ヘッドレスPCを買い足しました。
言い訳を申しますと、これは浪費ではなく実験環境の増設です。
たぶん。

## dry-runもあります

設定確認用にdry-runを入れてあります。
ntfyには送らず、設定とサンプル通知を表示します。

```console
$ herdr plugin action invoke dry-run

$ herdr plugin log list --plugin horn553.herdr-ntfy --limit 1 | jq -r '.result.logs[-1].stdout'
Herdr ntfy dry-run

curl: ok
jq: ok
NTFY_URL: ok (https://ntfy.sh/your-...)
NTFY_TITLE: Herdr
NTFY_TOKEN: not set
NTFY_LINES: 12

Sample title:
✅ verification・dry-run (Herdr)

Sample body:
Herdr ntfy dry-run: no notification was sent.

Result: ok
```

通知のためのプラグインで、通知設定に失敗して詰まる。
これはなかなか味わい深い事故なので、先にdry-runしておくのがおすすめです。

## 使っていてよかったところ

私は、自宅PCへSSHして、Herdrで複数のagentを立てる使い方が合いました。
通知が鳴るので、TUIを開きっぱなしで凝視しなくてよい。
スマートフォンで通知だけ見て、必要ならSSHで戻る。
この往復が軽いです。

趣味の小物づくりでも、仕事の下ごしらえでも、私はまず小さく試します。
作って、壊して、直して、また作る。
その一巡が速くなると、作業が前へ進みます。気分もよい。

`herdr-ntfy` は、その横に置く小さな鈴です。
Herdrを使っている方は、よろしければ鳴らしてみてくだされ。

---

<!-- begin long upcoming concert announcement -->

## 次回演奏会のご案内

Orchestra Canvas Tokyoは、都内を中心に活動するアマチュアオーケストラです。

日々の癒しに、新たなひらめきのきっかけに。
オーケストラの演奏会はいかがでしょうか？

初めての方も大歓迎！
ご来場をお待ちしています。

> **Orchestra Canvas Tokyo**
> **第17回定期演奏会**
>
> 2026年9月12日(土)
> 横浜みなとみらいホール 大ホール
> R.シュトラウス / アルプス交響曲 作品64 ほか
>
> 詳細は[チケット販売ページ](https://teket.jp/1776/65178?uid=zenn)にて

<!-- end long upcoming concert announcement -->

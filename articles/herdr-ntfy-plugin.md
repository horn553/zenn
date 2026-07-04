---
title: "herdrにntfyをつなぐ！― AI agent nativeなターミナル・マルチプレクサに通知を🔔"
emoji: "🐏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["herdr", "ntfy", "ai", "tui"]
publication_name: "orch_canvas"
published: false
# published_at: 2026-07-03 06:00
---

## まとめ

- [Herdr](https://herdr.dev/) という AI agent native なターミナル・マルチプレクサにはまっている
- Herdr の agent status が `done` または `blocked` になったら、[ntfy](https://ntfy.sh/) へ通知するプラグインを作った
- オレオレ＝AIコーディングを加速セヨ！

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2026年9月にR. シュトラウスの《アルプス交響曲》を演奏します。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/65178?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

## 作ったもの

https://github.com/horn553/herdr-ntfy

[Herdr](https://herdr.dev/) の agent status は `done`、`blocked`、`idle` の3つ。
これが `done` または `blocked` になったとき、[ntfy](https://ntfy.sh/) へ通知します。

めぼしい依存は `sh`、`curl`、`jq` コマンド！
依存関係は最低限な、シェルスクリプトベースのプラグインです。

### 通知サンプル

```text
タイトル: ✅ herdr-ntfy・main (Herdr)

Implemented the ntfy plugin.
Tests not run.
```

```text
タイトル: 🚫 herdr-ntfy・main (Herdr)

Blocked because NTFY_URL is not configured.
Set it in the plugin config directory .env file.
```

## 導入方法

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

以上！

保護された topic を使う場合は `NTFY_TOKEN` を設定します。

dry-run や通知テストも備えています。
詳しくは[リポジトリ](https://github.com/horn553/herdr-ntfy)を参照してください。

## よもやま話

Herdr の TUI、いいですね！

レスポンシブな TUI で、クリックや右クリックでも操作できます。
これらをよい UX に落とし込んでいるのが、とてもありがたいです。

負けじと創作の意欲が湧くというものですね！

クライアントの柔軟性という点で Web アプリが台頭してきたと考えていますが、
ここにきて TUI が存在感を増しているのは興味深い流れです。

[Tailscale Services](https://tailscale.com/docs/features/tailscale-services)とともに、オレオレ＝Webアプリの開発に欠かせない存在となっていきそうです♡

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

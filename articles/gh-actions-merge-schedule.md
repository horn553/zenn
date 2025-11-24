---
title: "指定日時にプルリクをマージするなら：Merge Schedule【GitHub Actions】"
emoji: "⏱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "githubactions"]
publication_name: "orch_canvas"
published: true
published_at: 2025-03-03 06:00
---

## まとめ

- GitHub Actions「Merge Schedule」を利用することで達成できる
- ワークフローを作成のうえ、プルリクエスト本文末尾に`/schedule (マージ日時)`と記述する
- おすすめの設定や、詳細な挙動についても記述している

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2026年3月にバーンスタインのシンフォニックダンス。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、GitHubを用いてホームページやブログを管理しています。

https://github.com/orchestra-canvas-tokyo/blog

指定日時に情報を更新する、という需要は一定程度存在していました。
例えば、「演奏会が開演する頃に次回演奏会の案内を更新する」といった具合です。

この解決方法として、指定日時にプルリクエストをマージするGitHub Actions「Merge Schedule」に白羽の矢が立ちました。

https://github.com/marketplace/actions/merge-schedule

## セットアップ

まずは基本的な使い方を解説します。

### ワークフローの作成

公式のガイドに則り、ワークフローを作成します。

::::details ソースコード例

```yaml:/.github/workflows/merge-schedule.yaml
name: Merge Schedule

on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize
  schedule:
    - cron: '0 * * * *'

jobs:
  merge_schedule:
    runs-on: ubuntu-latest
    steps:
      - uses: gr2m/merge-schedule-action@v2
        with:
          require_statuses_success: 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

::::

### 実際の運用

指定日時にマージしたいプルリクエストを作成するとき、本文末尾に次のように記述します。

```plain
/schedule (指定日時)
```

個人的に、指定日時の記述はタイムゾーン付きISO8601が良いのではないかと考えています。
例えば、`/schedule 2025-03-03T06:00:00+09:00`といった具合です。

その他でも、JavaScriptのDateコンストラクタが解釈可能な文字列であればなんでも良いです。
詳しくはMDNのドキュメントに譲ります。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/Date

実際の動作例は次のような形です。

![スクリーンショット](/images/gh-actions-merge-schedule/01.png)

## 詳細な仕様

### マージ日時の精度

このActionsは、スケジュール時刻にcronを作成する……という仕様ではありません。
ワークフローで設定したcron頻度ごとにプルリクエストを巡回し、適宜マージするという仕様です。
すなわち、マージ日時の精度はcronの頻度や精度に依ります。

GitHub Actionsでは、cronの実行時刻制度の保証はありません。
特に、毎時の開始時点は高負荷の時間帯であり、イベントの遅延やジョブの一部の削除の可能性があるとされています。

https://docs.github.com/ja/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#schedule

私はいくつかのリポジトリで、1時間ごとのcron設定で運用していますが、概ね15分程は遅延していそうです。

### その他の引数

注意が必要な引数として`time_zone`が挙げられます。
その他、公式のコード例に十分な説明がある引数`merge_method`、`require_statuses_success`、`automerge_fail_label`は割愛します。

引数`time_zone`は、プルリクエストへの出力文字列には用いられますが、**入力文字列の解釈には用いられません**。
前述したように、入力文字列はそのままDateコンストラクタの第一引数として用いられます。

一部でタイムゾーンが考慮されるのは混乱の元となるのではないか？と考え、タイムゾーンはUTCに統一する方針としました。
入力文字列もあえてISO8601でタイムゾーンを明記することを推奨する方針としています。

---

## おわりに

指定日時のマージを行うため、手始めにGoogle検索を行いましたが、思うようなライブラリは見あたりませんでした。

コピペやAIによる実装も視野に入れましたが、保守コストがかさみそう……
そこで、GitHub Actions Marketplaceで調べていくと、手早くいいActionsを見つけることができました。

AIの台頭で「ググり力」もターニングポイントを迎えていますが、もっと基礎的なググり力を鍛えなければいけないなぁと痛感した今日この頃です。

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
> 詳細は[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)にて

<!-- end long upcoming concert announcement -->

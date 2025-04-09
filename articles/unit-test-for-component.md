---
title: "理論からコンポーネントの単体テストにアプローチしてみる"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [""]
publication_name: "orch_canvas"
published: false
# published_at: 2025-03-03 06:00
---

## 背景

●AIで単体テストの作業工数が減っており、学ぶにうってつけの環境です。
●「単体テストの考え方/使い方」とかに触れたい。

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2025年7月にシューマンの交響曲第2番。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/44429?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

---

## 関数の単体テスト

コンポーネントの単体テストを考える前に、より基本たる関数の単体テストについて考えます。

ここはオーケストラ団体らしく、次のような関数`getContemporaryComposers()`について考えます。
これは、架空のパッケージ`composer-info`を用いて、指定された作曲家と同年代に生きていた作曲家の一覧を取得する関数です。

```ts
import { getComposerLifeSpan, getComposersAliveInYear } from "composer-info";

/**
 * 指定された作曲家の同時代に生きていた作曲家を返す。
 * 
 * @param {string} targetName
 * @returns {string[]} 同時代に生きていた作曲家の配列
 */
function getContemporaryComposers(targetName) {
    const resultSet = new Set();

    const { birthYear, deathYear } = getComposerLifeSpan(targetName);
    for (let year = birthYear; year <= deathYear; year++) {
        const composers = getComposersAliveInYear(year);
        composers.forEach(composer => resultSet.add(composer));
    }

    return Array.from(resultSet);
}
```

これに対する単体テストを設計するとき、大きな分岐点があります。
パッケージ`composer-info`をモックするかどうかです。

「単体テストの考え方/使い方」に基づき、地に足をつけて考えていきます。
単体テストの定義、単体テストの評価について考え、そのあとに先の関数`getContemporaryComposers()`について具体的に考えます。

### 単体テストの定義

●

### 単体テストの評価

●

### 関数`getContemporaryComposers()`での応用

●

## コンポーネントの単体テスト

### 引数、戻り値に対応するもの

●関数の引数・返り値との対応。

### 何を評価するか

●スタイルの取扱。

### 具体例を考える

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

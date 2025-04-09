---
title: "理論からコンポーネントの単体テストにアプローチしてみる"
emoji: "🧑🏼‍🏫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unittest", "component", "typescript"]
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
これは、指定された作曲家と同年代に生きていた作曲家の一覧を取得する関数です。

```ts
import { getComposerLifeSpan, getComposersAliveInYear } from "./composers";

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
        composers.forEach(composer => {
            if (composer === targetName) return;
            resultSet.add(composer);
        });
    }

    return Array.from(resultSet);
}
```

これに対する単体テストを設計するとき、大きな分岐点があります。
依存関係にある`./composers`をモックするかどうかです。

ここでは、名著「単体テストの考え方/使い方」に基づき、地に足をつけて考えていきます。
単体テストの定義、評価について考え、そのあとに先の関数`getContemporaryComposers()`について具体的に考えます。

### 単体テストの定義

> 単体テストの考え方/使い方
> └第2章 単体テストとは何か？
> 　└第1節 単体テストの定義

単体テストは、自動化されているうえで、次の3つの性質を兼ね備えたテストと定義されています。

- 「単体 unit」と呼ばれる少量のコードを検証する
- 実行時間が短い
- 隔離された状態で実行する

### 2つの学派

単体テストの考え方について、大きく2つの学派があります。
**古典学派**と**ロンドン学派**です。
それぞれ、『テスト駆動開発』『実践テスト駆動開発』が正書として紹介されています。

これらの学派の違いをみていき、比較評価していきます。
その前にまずは、用語の定義を確認します。

#### テスト・ダブルの種類

> 単体テストの考え方/使い方
> └第5章 モックの利用とテストの壊れやすさ
> 　└第1節 モックとスタブの違い

少し脇道にそれ、今後のために用語の定義を確認します。

テスト・ダブルとは、テストのために用意された偽の依存のことです。
「モック」はこの一種ですが、その他に次の5種類が定義されています。

| 種類 | 説明 |
| --- | --- |
| **モック** | **外部に向かう**コミュニケーション（出力）の模倣と検証に用いられる |
| **スタブ** | **内部に向かう**コミュニケーション（入力）の模倣に用いられる |
| スパイ | モックと同様に出力の模倣と検証を行うが、モック・フレームワークを用いない<br>いわば「手書きのモック handwritten mock」|
| フェイク | スタブと同様に入力の模倣を行うが、まだ存在しない依存を置き換える |
| ダミー | 実行に必要だが結果生成には参加しない、適当な値 |

ダミーのみが値に関する定義であり、他と粒度が異なる点に注意が必要です。

これらの定義に基づき、次のような性質が特に重要となります。

- スタブとのコミュニケーションは検証してはいけない
- モック、スタブ双方の性質をもつテスト・ダブルは存在しうる

#### ロンドン学派

単体テストの2つの考え方――古典学派とロンドン学派――のうち、まずはロンドン学派についてみていきます。
こちらの情報の方が、巷にあふれている印象があります。

基本的な考え方は「テスト対象の依存を、全てテスト・ダブルで置き換える」というものです。
これにより、単体テストの定義の1つである「隔離された状態で実行する」を満足するこができます。

先の関数`getContemporaryComposers()`での例を示します。
簡単のために、正常系のみを実装します。

```ts
import { describe, it, expect, vi } from 'vitest';
import { getContemporaryComposers } from './getContemporaryComposers';
import * as composersModule from './composers';

describe('getContemporaryComposers', () => {
  it('正常系：同時代に生きていた作曲家を返す', () => {
    // 準備 Arrange

    // 関数`getComposerLifeSpan()`のスタブを用意する
    vi.spyOn(composersModule, 'getComposerLifeSpan').mockReturnValue({
      birthYear: 1800,
      deathYear: 1802
    });

    // 関数`getComposersAliveInYear()`のスタブを用意する
    const mockGetComposersAliveInYear = vi.spyOn(composersModule, 'getComposersAliveInYear');
    mockGetComposersAliveInYear.mockImplementation((year) => {
      const mockData: Record<number, string[]> = {
        1800: ['Beethoven', 'Haydn'],
        1801: ['Beethoven', 'Schubert'],
        1802: ['Beethoven', 'Mozart'],
      };
      return mockData[year] ?? [];
    });

    // 実行 Act
    const result = getContemporaryComposers('Beethoven');

    // 検証 Assert
    expect(result.sort()).toEqual(['Haydn', 'Mozart', 'Schubert'].sort());
  });
});
```

関数`getContemporaryComposers()`が依存している関数を、全てテスト・ダブル（今回は入力の模倣のみなのでスタブ）で置き換えています。

これにより、単体テストの定義を十分に満足したテストが記述できました。

- 「単体 unit」と呼ばれる少量のコードを検証する
  - `getContemporaryComposers()`の実装のみを検証している
- 実行時間が短い
  - そのように想定される
- 隔離された状態で実行する
  - 関数`getContemporaryComposers()`を、独立した状態で実行する

#### 古典学派

単体テストの2つの考え方――古典学派とロンドン学派――のうち、残りの古典学派についてみていきます。

この2つの考え方の最大の違いは、「隔離」に対する考え方の違いです。
この「隔離」というのは、単体テストの定義で登場した概念です。再掲します。

> - 「単体 unit」と呼ばれる少量のコードを検証する
> - 実行時間が短い
> - 隔離された状態で実行する

古典学派では「テスト・ケース同士が隔離されている」ことを隔離とします。
これに対しロンドン学派では、依存をテスト・ダブルに置き換えていたので「コードが隔離されている」ことを隔離としている、と言うことができます。

先の関数`getContemporaryComposers()`での例を示します。
簡単のために、正常系のみを実装します。

```ts
// getContemporaryComposers.classical.test.ts
import { describe, it, expect, vi } from 'vitest';
import { getContemporaryComposers } from './composers';

describe('getContemporaryComposers', () => {
  it('正常系：同時代に生きていた作曲家を返す', () => {
    // 準備 Arrange
    // 実行前のクラスのインスタンス化など：今回はない
    
    // 実行 Act
    const result = getContemporaryComposers('Beethoven');

    // 検証 Assert
    expect(result.sort()).toEqual(['Haydn', 'Mozart', 'Schubert'].sort());
  });
});
```

今回はシンプルなコード例なので記述はありませんが、実行準備としてクラスのインスタンス化などを行います。
ちょうど、プロダクション・コードで行われる準備をそのまま行うイメージです。

では、古典学派ではどのような依存のテスト・ダブルを用いるのでしょうか。
まずは「依存」の種類を整理したうえで、述べていきます。

##### 依存の種類

次の3つがあります。

| 種類 | 説明 |
| --- | --- |
| 共有依存 | テスト・ケース間で共有される依存 |
| プライベート依存 | 共有されない依存 |
| プロセス外依存 | 別プロセスで動作する依存<br>共有依存の場合が多いが、必ずしもそうではない（Dockerでコンテナ化している場合など） |

##### 古典学派でテスト・ダブルに置き換える依存

前述のように、古典学派では「テスト・ケース同士が隔離されている」ことを隔離としています。
このことから考えると、テスト・ダブルで置き換えるべき依存は「共有依存」であると考えられます。

例えば、データベースやファイルシステムへアクセスする場合、これは共有依存であり置き換える必要があります。
（コンテナ化されている場合は共有依存とならない可能性がありますが）

一方で、先述の例における関数`getComposerLifeSpan()`、`getComposersAliveInYear()`はプライベート依存であり、共有依存ではありません。
そのため、テスト・ダブルへの置き換えの対象とはなりません。

### 単体テストの評価

ここまでで、古典学派とロンドン学派の概要を述べてきました。
では、どちらを使うのがよいのでしょうか？

どちらの学派もあることから分かるように、統一的な見解はありません。
その代わりに、単体テストの評価について考えていき、各学派の特徴や傾向を理解していきます。

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

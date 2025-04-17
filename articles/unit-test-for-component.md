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

AIの台頭により、単体テストの実装工数は大きく減っています。
これにより、人間は単体テストの評価が主担当となり、むしろ単体テストに対する深い理解が必要となっています。

そこで、「単体テストの考え方/使い方」をベースに、単体テスト（特にコンポーネントの単体テスト）について、理論的な側面に注目して考えていきます。

https://books.google.co.jp/books/about/%E5%8D%98%E4%BD%93%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E8%80%83%E3%81%88%E6%96%B9_%E4%BD%BF%E3%81%84%E6%96%B9.html?id=7-mnEAAAQBAJ&source=kp_book_description&redir_esc=y

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

次のような関数`getContemporaryComposers()`に注目します。
これは、指定された作曲家と同年代に生きていた作曲家の一覧を取得する関数です。

```ts
import { getComposerLifeSpan, getComposersAliveInYear } from "./composers";

/**
 * 指定された作曲家の同時代に生きていた作曲家を返す。
 * 
 * @param {string} targetName
 * @returns {string[]} 同時代に生きていた作曲家の配列
 */
function getContemporaryComposers(targetName: string): string[] {
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
**依存関係にある`./composers`をモックするかどうか**です。

単体テストの定義、次に評価について考え、そのあとに先の関数`getContemporaryComposers()`について具体的に考えます。

### 単体テストの定義

> 単体テストの考え方/使い方
> └第2章 単体テストとは何か？
> 　└第1節 単体テストの定義

単体テストは、自動化されているうえで、次の3つの性質を兼ね備えたテストと定義されています。

- 「単体 unit」と呼ばれる少量のコードを検証する
- 実行時間が短い
- 隔離された状態で実行する

### 単体テストの評価

次に、単体テストの評価について考えていきます。
より見通しよく実際の実装例を解釈していくことが狙いです。

#### 良い単体テストを構成する4つの柱

> 単体テストの考え方/使い方
> └第4章 良い単体テストを構成する4つの柱
> 　└第1節 良い単体テストを構成する4つの柱

良い単体テストを構成する4つの柱について、次のように示されています。

- 退行（regression）に対する保護
- リファクタリングへの耐性
- 迅速なフィードバック
- 保守のしやすさ

「リファクタリングへの耐性」を最大化しつつ、二律背反である「退行に対する保護」と「迅速なフィードバック」のバランスをとっていくことが必要であると本書中で述べられています。

また、「リファクタリングへの耐性」を最大化するコツとして、「実装の詳細でなく**観察可能な振る舞い**をテストする」ことが重要であるとされています。

### 単体テストの考え方

単体テストの考え方について、大きく2つの学派があります。
**古典学派**と**ロンドン学派**です。
それぞれ、『テスト駆動開発』『実践テスト駆動開発』が正書として紹介されています。

この2つの考え方の最大の違いは、**「隔離」に対する考え方の違い**です。
この「隔離」というのは、単体テストの定義で登場した概念です。再掲します。

> - 「単体 unit」と呼ばれる少量のコードを検証する
> - 実行時間が短い
> - 隔離された状態で実行する

これらの学派について具体的にみていき、比較評価していきます。
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

基本的な考え方は、**テスト対象の依存を、全てテスト・ダブルで置き換える**というものです。
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
詳細は次の通りです。

- 「単体 unit」と呼ばれる少量のコードを検証する
  - `getContemporaryComposers()`の実装のみを検証している
- 実行時間が短い
  - そのように想定される
- 隔離された状態で実行する
  - 関数`getContemporaryComposers()`を、独立した状態で実行する

#### 古典学派

単体テストの2つの考え方――古典学派とロンドン学派――のうち、残りの古典学派についてみていきます。

この2つの考え方の最大の違いは、**「隔離」に対する考え方の違い**でした。

古典学派では**テスト・ケース同士が隔離されている**ことを隔離とします。
これに対しロンドン学派では、依存をテスト・ダブルに置き換えていたので「コードが隔離されている」ことを隔離としている、と言うことができます。

先の関数`getContemporaryComposers()`での例を示します。
簡単のために、正常系のみを実装します。

```ts
import { describe, it, expect, vi } from 'vitest';
import { getContemporaryComposers } from './composers';

describe('getContemporaryComposers', () => {
  it('正常系：同時代に生きていた作曲家を返す', () => {
    // 準備 Arrange
    // 依存するクラスのインスタンス化など：今回はない
    
    // 実行 Act
    const result = getContemporaryComposers('Beethoven');

    // 検証 Assert
    expect(result.sort()).toEqual(['Haydn', 'Mozart', 'Schubert'].sort());
  });
});
```

今回はシンプルなコード例なので記述はありませんが、実行準備としてクラスのインスタンス化などを行います。
ちょうど、プロダクション・コードで行われる準備をそのまま行うイメージです。

では、古典学派はテスト・ダブルを用いないのかというとそういう訳ではありません。
古典学派ではどのような依存のテスト・ダブルを用いるのでしょうか。
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

「副作用をもつ依存」はテスト・ダブルへの置換対象となりうる、と考えることもできそうです。

一方で、先述の例における関数`getComposerLifeSpan()`、`getComposersAliveInYear()`はプライベート依存であり、共有依存ではありません。
そのため、テスト・ダブルへの置き換えの対象とはなりません。

---

## コンポーネントの単体テスト

ここまで、一般の関数に対する単体テストについてみていきました。
これを応用して、コンポーネントの単体テストについて考えていきます。

### 戻り値に対応するもの

先で議論したような、一般の関数では戻り値は自明です。

しかし、コンポーネントでは検討が必要です。
コンポーネントはDOM構造のほか、スタイルを含むことがほとんどです。
スクリプトを含む場合も少なくありません。

戻り値は、DOM構造＋スタイル＋スクリプトであると考えられます。

### スタイルは単体テスト対象としにくい

戻り値は「観察可能な振る舞い」であり、テストの対象となりうります。

先に検討した、コンポーネントの戻り値を構成する要素（DOM構造＋スタイル＋スクリプト）のうち、スタイルは検証が難しいです。
CSSそのものを検証する方法は、実装の詳細のテストとなりうるため好ましくありません。
統合テストとしてビジュアル・リグレッション・テストを用いる方法も考えられますが、これはもはや単体テストではありません。

そこで、スタイルを除いた「DOM構造＋スクリプト」を検証対象とするのがよいのではないでしょうか。

### 具体例を考える

次のようなボタンを考えます。

```html:Nabeatsu.svelte
<!-- クリックでカウントアップし、3を含む数か3の倍数で色が変わる -->
<script lang="ts">
  let count = $state(0);
  let isAho = $derived(() => count.toString().includes('3') || count % 3 === 0);
</script>

<button on:click={() => count++} class:aho={isAho}>{count}</button>

<style>
  .aho {
    color: red;
  }
</style>
```

これに対し、例えば次のような単体テストが書けるでしょう。
DOM構造（ボタンとその表示内容、クラス）とスクリプト（クリックによるカウントアップ、スタイル更新）を検証しています。

あくまで検証しているのは付与されたクラスまでであり、スタイルについては検証されていません。

```ts
import { render, screen, fireEvent } from '@testing-library/svelte';
import { describe, it, expect } from 'vitest'; // vitest の場合

import Nabeatsu from '$lib/Nabeatsu.svelte'; // 対象コンポーネントのパス

describe('Nabeatsu.svelte', () => {
  it('初期状態でボタンに 0 が表示され、ahoクラスが付いていないこと', () => {
    // コンポーネントをレンダリング
    render(Nabeatsu);

    // ボタン要素を取得
    const button = screen.getByRole('button');

    // 初期テキストが '0' であることを確認
    expect(button).toHaveTextContent('0');
    // 初期状態で 'aho' クラスが付与されていないことを確認
    expect(button).not.toHaveClass('aho');
  });

  it('ボタンをクリックするとカウントがインクリメントされること', async () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    // 1回クリック
    await fireEvent.click(button);
    expect(button).toHaveTextContent('1');
    expect(button).not.toHaveClass('aho');

    // もう1回クリック
    await fireEvent.click(button);
    expect(button).toHaveTextContent('2');
    expect(button).not.toHaveClass('aho');
  });

  it('カウントが3の倍数になるとahoクラスが付与されること', async () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    // 2回クリック（カウントは2）
    await fireEvent.click(button);
    await fireEvent.click(button);
    expect(button).toHaveTextContent('2');
    expect(button).not.toHaveClass('aho');

    // 3回目のクリック（カウントは3）
    await fireEvent.click(button);
    expect(button).toHaveTextContent('3');
    expect(button).toHaveClass('aho'); // 3の倍数なのでクラスが付くはず
  });

  it('カウントに3が含まれる数字になるとahoクラスが付与されること', async () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    // カウントが12になるまでクリック (ahoクラスは付かない)
    for (let i = 0; i < 12; i++) {
      await fireEvent.click(button);
    }
    expect(button).toHaveTextContent('12');
    expect(button).toHaveClass('aho'); // 12は3の倍数

    // カウントが13になるまでクリック
    await fireEvent.click(button);
    expect(button).toHaveTextContent('13');
    expect(button).toHaveClass('aho'); // 13は3を含むのでクラスが付くはず

    // カウントが14になるまでクリック
    await fireEvent.click(button);
    expect(button).toHaveTextContent('14');
    expect(button).not.toHaveClass('aho'); // 14は条件に合わない
  });
});
```

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

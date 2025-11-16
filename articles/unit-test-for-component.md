---
title: "理論からコンポーネントの単体テストにアプローチしてみる"
emoji: "🧑🏼‍🏫"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["unittest", "component", "typescript", "testing"]
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
> 次回は2025年11月にバレエ音楽『火の鳥』組曲。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/52506?uid=zenn)まで。

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
    const resultSet = new Set<string>();

    const lifeSpan = getComposerLifeSpan(targetName);
    if (!lifeSpan) return [];
    const { birthYear, deathYear } = lifeSpan;

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
それぞれ『テスト駆動開発』.『実践テスト駆動開発』が正書として紹介されています。

この2つの考え方の最大の違いは、**「隔離」に対する考え方の違い**です。
この「隔離」というのは、単体テストの定義で登場した概念です。再掲します。

> - 「単体 unit」と呼ばれる少量のコードを検証する
> - 実行時間が短い
> - 隔離された状態で実行する

これらの学派の違い、特に「隔離」の捉え方をより深く理解するために、まずはテストで使われる「偽物」、すなわち**テスト・ダブル**について整理しておきましょう。特に、モックとスタブの違いは重要なポイントです。

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
    // 結果の順序は保証されないため、ソートして比較する
    expect(result.sort()).toEqual(['Haydn', 'Mozart', 'Schubert'].sort());

    // （オプション）依存関数の呼び出しを検証することも可能（モック的な使い方）
    expect(composersModule.getComposerLifeSpan).toHaveBeenCalledWith('Beethoven');
    expect(mockGetComposersAliveInYear).toHaveBeenCalledTimes(3); // 1800, 1801, 1802
  });
});
```

関数`getContemporaryComposers()`が依存している関数を、全てテスト・ダブル（今回は主にスタブ）で置き換えています。

これにより、単体テストの定義を十分に満足したテストが記述できました。
詳細は次の通りです。

- 「単体 unit」と呼ばれる少量のコードを検証する
  - `getContemporaryComposers()`の実装のみを検証している
- 実行時間が短い
  - 依存先の実際の処理を実行しないため高速
- 隔離された状態で実行する
  - 関数`getContemporaryComposers()`を、その依存から切り離して（隔離して）実行する

#### 古典学派

単体テストの2つの考え方――古典学派とロンドン学派――のうち、残りの古典学派についてみていきます。

この2つの考え方の最大の違いは、**「隔離」に対する考え方の違い**でした。

古典学派では**テスト・ケース同士が隔離されている**ことを隔離とします。
テスト対象のコードが、そのプライベートな依存関係（他のクラスや関数など）と協調して動作することを検証します。

先の関数`getContemporaryComposers()`での例を示します。
簡単のために、正常系のみを実装します。

```ts
import { describe, it, expect } from 'vitest';
// 依存関係も含めてテスト対象とするため、実際のモジュールをインポート
import { getContemporaryComposers } from './getContemporaryComposers';
// import './composers'; // 必要に応じて依存先の初期化などを行う

describe('getContemporaryComposers (Classical School)', () => {
  it('正常系：同時代に生きていた作曲家を返す', () => {
    // 準備 Arrange
    // - 依存するクラスのインスタンス化など：今回は特になし
    // - テストに必要なデータが `./composers` 内で適切に設定されている前提
    //   (例: getComposerLifeSpan('Beethoven')が{1770, 1827}を返すなど)

    // 実行 Act
    // 実際の getContemporaryComposers を、実際の依存と共に実行
    const result = getContemporaryComposers('Beethoven'); // 仮の作曲家名

    // 検証 Assert
    // 期待される結果（実際の依存関係に基づいて計算されるはずの結果）を検証
    // 例: もしベートーヴェンと同時代の作曲家としてリストアップされるべき人が分かっていればそれを書く
    // expect(result.sort()).toEqual(['Haydn', 'Mozart', 'Schubert'].sort());
  });
});
```

今回はシンプルなコード例なので記述はありませんが、実行準備としてクラスのインスタンス化などを行います。
プロダクション・コードで使われる際と同じように依存関係を準備するイメージです。

では、古典学派はテスト・ダブルを用いないのかというとそういう訳ではありません。
古典学派ではどのような依存のテスト・ダブルを用いるのでしょうか。
それについて議論する前に、「依存」の種類を整理します。

##### 依存の種類

次の3つがあります。

| 種類 | 説明 |
| --- | --- |
| 共有依存 | テスト・ケース間で共有される依存（例: 静的変数、外部DB）<br>テストの実行順序によって結果が変わる可能性がある |
| プライベート依存 | 共有されない依存（例: 通常のクラス、関数呼び出し）<br>テスト対象ごとに独立している |
| プロセス外依存 | 別プロセスで動作する依存（例: DBサーバー、外部API）<br>ネットワーク越しなど、実行が遅く不安定になる要因 |

前述のように、古典学派では「テスト・ケース同士が隔離されている」ことを重視します。
この観点から、テストの実行順序に影響を与えたり、テストの実行速度や安定性を著しく損なったりする依存、すなわち**共有依存**や**プロセス外依存**は、テスト・ダブル（スタブやフェイクなど）で置き換えることが推奨されます。

例えば、データベースやファイルシステムへのアクセス、外部API呼び出しなどは、共有依存でありプロセス外依存でもあるため、置き換え対象となることが多いです。

一方で、先述の例における関数`getComposerLifeSpan()`や`getComposersAliveInYear()`が、状態を持たず副作用もない純粋な関数や、テストごとに独立してインスタンス化されるクラス（つまりプライベート依存）であれば、古典学派ではこれらをそのまま利用し、テスト・ダブルには置き換えません。

##### 古典学派でテスト・ダブルに置き換える依存

前述のように、古典学派では「テスト・ケース同士が隔離されている」ことを隔離としています。
このことから考えると、テスト・ダブルで置き換えるべき依存は「共有依存」であると考えられます。

例えば、データベースやファイルシステムへアクセスする場合、これは共有依存であり置き換える必要があります。
（コンテナ化されている場合は共有依存とならない可能性がありますが）

一方で、先述の例における関数`getComposerLifeSpan()`、`getComposersAliveInYear()`はプライベート依存であり、共有依存ではありません。
そのため、テスト・ダブルへの置き換えの対象とはなりません。

---

## コンポーネントの単体テスト

ここまで、一般の関数に対する単体テストについてみていきました。
これを応用して、コンポーネントの単体テストについて考えていきます。

### 戻り値に対応するもの

先の議論で重要だった「観察可能な振る舞い」は、一般の関数では戻り値や（場合によっては）副作用として捉えられます。

では、コンポーネントにおける「観察可能な振る舞い」とは何でしょうか？
コンポーネントは通常、ユーザーインターフェースの一部を構成します。そのため、その振る舞いは主に**レンダリング結果**と**ユーザー操作への反応**として観察されます。

具体的には、次の要素の組み合わせと考えられます。
 * **DOM構造**: どのようなHTML要素が生成されるか。
 * **表示内容**: テキスト、属性（`class`, `id`, `disabled`など）がどうなっているか。
 * **状態変化**: ユーザー操作（クリックなど）によって、表示内容やDOM構造が期待通りに変化するか。

スタイル（CSS）もレンダリング結果の一部ですが、単体テストでの扱いは少し注意が必要です。

### スタイルは単体テスト対象としにくい

コンポーネントの「観察可能な振る舞い」のうち、スタイル（見た目）そのものを単体テストで厳密に検証するのは難しい場合があります。

CSSそのもの（例: `color: red;`）を直接検証する方法は、実装の詳細に深く依存し、リファクタリングで壊れやすいテストになりがちです。
統合テストやE2Eテストの範疇で、Visual Regression Testing (VRT) のような手法を用いて見た目の変化（デグレ）を検出する方法もありますが、これはもはや単体テストではありません。

なぜスタイル検証が単体テストで難しいかというと、見た目の確認は環境差（ブラウザ、OSなど）の影響を受けやすく、またCSSのクラス名や構造といった**実装の詳細**に強く依存しがちで、リファクタリングで壊れやすいテストになってしまうためです。

そこで、単体テストのレベルでは、**スタイルを直接検証するのではなく、スタイルを適用するための条件（例: 特定のクラスが付与されているか、特定の属性が設定されているか）**を検証するのが現実的かつ効果的と考えられます。つまり、検証対象は主に「DOM構造＋表示内容＋状態変化」となります。

### 具体例を考える

次のようなSvelteのボタンコンポーネントを考えます。クリックでカウントアップし、3の倍数または3を含む数字のときにahoクラスが付与されます。

```html
<script lang="ts">
  import { state } from 'svelte/internal'; // $state を使うために必要（Svelte 5 の場合）

  let count = $state(0);
  // count が 3 の倍数か、文字列として '3' を含むかを判定
  let isAho = $derived(count > 0 && (count % 3 === 0 || count.toString().includes('3')));
</script>

<button on:click={() => count++} class:aho={isAho}>{count}</button>

<style>
  .aho {
    /* スタイル自体はテストしないが、クラスの有無をテストする */
    color: red;
    font-weight: bold;
  }
</style>
```

これに対し、次のような単体テストが書けます。
DOM構造（ボタンの存在、表示テキスト、クラス属性）と、スクリプトによる状態変化（クリックによるカウントアップ、クラスの動的な付与/削除）を検証しています。

```ts
import { render, screen, fireEvent } from '@testing-library/svelte';
import { describe, it, expect, afterEach } from 'vitest';
import Nabeatsu from './Nabeatsu.svelte'; // 対象コンポーネント

// 各テスト後にレンダリングされたコンポーネントをクリーンアップ
afterEach(() => {
  render(Nabeatsu).unmount(); // 簡単なクリーンアップ例
});

describe('Nabeatsu.svelte', () => {
  it('初期状態でボタンに 0 が表示され、ahoクラスが付いていないこと', () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    expect(button).toHaveTextContent('0');
    // 0 の場合は aho クラスは付かない
    expect(button).not.toHaveClass('aho');
  });

  it('ボタンを1回、2回クリックしてもahoクラスが付かないこと', async () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    await fireEvent.click(button); // count = 1
    expect(button).toHaveTextContent('1');
    expect(button).not.toHaveClass('aho');

    await fireEvent.click(button); // count = 2
    expect(button).toHaveTextContent('2');
    expect(button).not.toHaveClass('aho');
  });

  it('ボタンを3回クリックするとahoクラスが付与されること', async () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    await fireEvent.click(button); // 1
    await fireEvent.click(button); // 2
    expect(button).not.toHaveClass('aho');

    await fireEvent.click(button); // 3
    expect(button).toHaveTextContent('3');
    expect(button).toHaveClass('aho'); // 3 は 3 の倍数
  });

  it('カウントが13になるとahoクラスが付与されること', async () => {
    render(Nabeatsu);
    const button = screen.getByRole('button');

    // count が 12 になるまでクリック (12回)
    for (let i = 0; i < 12; i++) {
      await fireEvent.click(button);
    }
    expect(button).toHaveTextContent('12');
    expect(button).toHaveClass('aho'); // 12 は 3 の倍数

    // count が 13 になる (13回目)
    await fireEvent.click(button);
    expect(button).toHaveTextContent('13');
    expect(button).toHaveClass('aho'); // 13 は '3' を含む
  });

  it('カウントが14になるとahoクラスが付かないこと', async () => {
     render(Nabeatsu);
    const button = screen.getByRole('button');

    // count が 13 になるまでクリック (13回)
    for (let i = 0; i < 13; i++) {
      await fireEvent.click(button);
    }
    expect(button).toHaveTextContent('13');
    expect(button).toHaveClass('aho');

    // count が 14 になる (14回目)
    await fireEvent.click(button);
    expect(button).toHaveTextContent('14');
    expect(button).not.toHaveClass('aho'); // 14 は条件に合わない
  });
});
```

このテストでは、`.aho`クラスが付与されるかどうかを検証しており、そのクラスによって具体的にどのようなスタイル（`color: red`など）が適用されるかまでは検証していません。これが、コンポーネント単体テストにおける現実的な落とし所と言えるでしょう。

#### 他のコンポーネントパターンと学派の視点

今回は内部状態を持つシンプルなコンポーネント例でしたが、実際の開発では様々なコンポーネントが登場します。

例えば、**Props**を受け取って表示内容が変わるコンポーネントでは、テスト時に異なるPropsを渡し、それぞれ期待通りにレンダリングされるか（表示テキスト、属性、子要素など）を検証します。
また、ボタンクリックなどで親コンポーネントに情報を伝えるために**カスタムイベントを発行する**コンポーネントでは、特定の操作を行った際に、期待されるイベント名とデータでイベントが発行されるかを（例えば`vitest`の`vi.fn()`などを使って）検証します。


では、古典学派とロンドン学派の考え方はコンポーネントテストにどう適用できるでしょうか？
今回のような自己完結したコンポーネントでは、依存関係がないため学派によるアプローチの違いは顕著には現れません。


しかし、例えば**外部APIを非同期に呼び出して結果を表示する**コンポーネントをテストする場合を考えてみましょう。
ロンドン学派のアプローチでは、API通信を行う依存オブジェクト（例: Workspaceや特定のAPIクライアント関数）をモックやスタブで置き換えるのが一般的です。これにより、テスト対象コンポーネントのロジックのみを隔離して検証できます。
一方、古典学派では、プロセス外依存としてAPI通信部分をテスト・ダブル（モックやFake実装）で置き換えることもありますが、場合によってはテスト用のインメモリDBやFakeサーバーを用意し、依存関係ごと（より統合テストに近い形で）テストすることもあります。

どちらのアプローチを取るかは、テストの目的（コンポーネント単体のロジック検証か、連携を含めた振る舞い検証か）、保守性、チームの方針などによって選択することになるでしょう。

## おわりに

今回は、単体テストの理論、特に古典学派とロンドン学派の考え方をベースに、コンポーネントテストへのアプローチを探ってみました。AIがテスト実装を助けてくれる今だからこそ、「良いテストとは何か？」を見極める視点が大切になりますよね。この記事が、皆さんのテスト戦略を考える上でのヒントになれば嬉しいです 🤔✨

コンポーネントのどこまでを「観察可能な振る舞い」と捉えるかなど、悩ましい点も多いですが、今回の内容が議論のきっかけや、より良いテストへの一歩となれば幸いです。ご意見や皆さんの工夫など、ぜひコメントで教えてくださいね！最後までお読みいただき、ありがとうございました！ 👋

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

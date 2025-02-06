---
title: "SvelteKitでコンポーネントをglobインポートする"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sveltekit", "svelte", "vite"]
publication_name: "orch_canvas"
published: true
published_at: 2025-02-10 06:00
---

## まとめ

- SvelteKitでコンポーネントをglobインポートしたい
- Viteの`import.meta.glob()`を使うことで実現できる

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、SvelteKitを用いてブログを作成しています。

https://blog.orch-canvas.tokyo

<!-- begin short upcoming concert announcement -->

> Orchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
> 次回は2025年2月にブルックナーの交響曲第8番。
> 初めての方も、そうでない方もお気軽にお越しください！
> 詳しくは[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-13)まで。

<!-- end short upcoming concert announcement -->

各記事をコンポーネントとして作成する形をとっています。
また、その記事コンポーネントから、メタ情報を変数としてエクスポートしています。

```html:/src/lib/posts/a-random-post/post.svelte
<script lang="ts" context="module">
  import type { Metadata } from '$lib/posts/index.ts';

  export const metadata: Metadata = {
    published: true,
    // 略
  };
</script>

<h3>はじめに</h3>

<!-- 略 -->
```

このような形で管理すると、次のようなメリットがあります。

- Svelteで本文を記述できる
  - 記事で用いる共通コンポーネントを整備しつつ
  - 各記事の高い自由度を維持できる
- メタ情報が型安全になる

しかし、この形式には大きなハードルがありました。
**コンポーネントをglobインポートする必要がある**という点です。

今回はこれに対する解決案を提示します。

## Viteのglobインポート

Svelte(Kit)はViteの上での利用を想定されています。
Viteはフロントエンドのビルドツールです。Svelteで記述されたファイルはViteを通して、一般的なWebサーバーで実行可能な形式にビルドされます。

そんなViteには、独自のglobインポート機能があります。
これを使うとうまくいきそうです。

https://ja.vite.dev/guide/features#glob-%E3%81%AE%E3%82%A4%E3%83%B3%E3%83%9B%E3%82%9A%E3%83%BC%E3%83%88

## 実装

まずは、最終的な実装を示します。

のちに解説する3つのポイントをコメントで示しています。

```ts:/src/lib/posts/index.ts
import type { Component } from 'svelte';

/** 記事のメタ情報の型 */
export type Metadata = {
  published: boolean;
  // 略
};

// ポイント(2)
/** ポストコンポーネントの型 */
export type Post = {
  metadata: Metadata;
  slug: string;
  default: Component;
  description: string;
};

// ポイント(1)
// 動的に記事のSvelteファイルを取得する
const modules = import.meta.glob('./**/post.svelte', { eager: true }) as Record<string, Post>;
const posts: { [slug: string]: Post } = {};

Object.keys(modules).forEach((path) => {
  const slug = /^.+\/(?<slug>[^/]+)\/post\.svelte$/.exec(path)?.groups?.slug;
  if (slug === undefined) return;

  posts[slug] = {
    metadata: modules[path].metadata,
    slug: slug,
    default: modules[path].default
  };
});
```

```html:+page.svelte（抜粋）
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>


<!-- ポイント(3) -->
<data.post.default />
```

3つのポイントについて、順を追って解説していきます。

### ポイント①：`import.meta.glob()`

今回の主役です。
globインポートを可能にする、Viteの独自関数です。

この関数は2つの引数で構成されています。
順を追ってみていきます。

#### 第1引数：パス条件

globの一般的な特殊文字である、`*`や`**`、`!`を用いることができます。
いくつか例を示します。

```ts
// シンプルな例
const modules1 = import.meta.glob('./dir/*.ts')

// 再帰的なパスを指定する
const modules2 = import.meta.glob('./dir/**/*.ts')

// 複数の条件を指定する
const modules3 = import.meta.glob(['./dir1/*.ts', './dir2/*.ts'])

// 否定条件を指定する
const modules4 = import.meta.glob(['./dir/**/*.ts', '!./dir/SECRET/*.ts'])
```

また、後述しますが、ここには静的な文字列（かその配列）を指定する必要があります。

#### 第2引数：オプション

オプションとして、次の3つが指定できます。
それぞれをキーとするオブジェクトを指定します。
いずれもオプションです。

- `import`
  - インポートするモジュールを明示的に指定します
- `query`
  - 各パスにつけるクエリを指定します
  - Viteのインポート時には`url`や`raw`、`inline`などが指定できます[^1]
- `eager`
  - `true`を設定することで、同期読み込みとなります
  - 未指定あるいは`false`では、Promiseが返されます

[^1]: [静的アセットの取り扱い | Vite](https://ja.vite.dev/guide/assets.html)

今回は、ビルド時に実行されるコードのためパフォーマンスの優先度は低いと考えました。
そのため、実装の簡素化のために`eager: true`を指定しました。

### ポイント②：インポートされるコンポーネントの型

```ts
import type { Component } from 'svelte';
```

これで問題なさそうです。

コンポーネントはデフォルトエクスポートされているため、Postコンポーネントの型は次のように定義しています。

```ts
export type Post = {
  metadata: Metadata;
  slug: string;
  default: Component;
  description: string;
};
```

型の問題は、TypeScriptでトリッキーなことをする際の大きなハードルの印象です。
端的な解決に至って一安心しつつ、次に進みます。

### ポイント③：コンポーネントの描画

Svelte5から、dot notationを用いたコンポーネントの指定が正しく解釈されるようになりました[^2]。
初めてJSXを見たときのようなぞわぞわを感じつつも、自信をもって記述していきます。

[^2]: [Svelte 5 migration guide • Docs • Svelte](https://svelte.dev/docs/svelte/v5-migration-guide#svelte:component-is-no-longer-necessary-Dot-notation-indicates-a-component)

```html
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

<data.post.default />
```

以上が実装における3つのポイントです。
細々としたハードルを整理すれば、ずいぶんシンプルなのではないでしょうか？

## Viteのglobインポートの特徴

Viteのglobインポートにはいくつかの制限があります。
代表的なものとして、次のようなコードは書くことができません。

```ts
// これはビルド時にエラーとなる
const images = import.meta.glob(`./${slug}/*.png`)
```

これは、第1引数にはリテラル値（静的な値）のみを指定する必要があるためです。

Viteは、ビルド時に`import.meta.glob()`は一般に実行可能なコードへ変換します。

```ts
const modules = import.meta.glob('./dir/*.ts', { eager: true })

// ↓ ビルド時に次のようなコードに変換される ↓

import * as __glob__0_0 from './dir/module1.ts'
import * as __glob__0_1 from './dir/module1.ts'
const modules = {
  './dir/module1.ts': __glob__0_0,
  './dir/module2.ts': __glob__0_1,
}
```

---

## おわりに

最終的なコードが簡潔なものとなり、大満足です！

実は、記事としてまとめる作業の中でより良い書き方をいくつか見つけることができました。
悩みぬいて「これで完璧だ！」と思ったものでも、こうやってまとめると新たな発見があるものですね。

これからも、いいネタがあれば書いていきたいなと感じた今日この頃です。

---

<!-- begin long upcoming concert announcement -->

## 次回演奏会のご案内

Orchestra Canvas Tokyoは都内を中心に活動するアマチュアオーケストラです。

日々の癒しに。新しいひらめきのきっかけに。
オーケストラの演奏会はいかがですか？

初めての方も大歓迎！
ご来場お待ちしています。

> Orchestra Canvas Tokyo
> 第13回定期演奏会
>
> 2025年2月24日(月祝)
> 横浜みなとみらいホール
>
> ブルックナー / 交響曲第8番 ほか
>
> 詳細は[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-13)にて
>
> [![](/images/regular-13.png =250x)](https://www.orch-canvas.tokyo/concerts/regular-13)

<!-- end long upcoming concert announcement -->

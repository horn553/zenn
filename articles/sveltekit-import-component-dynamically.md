---
title: "SvelteKitでコンポーネントをglobインポートする"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sveltekit", "svelte", "vite"]
publication_name: "orch_canvas"
published: false
---

## まとめ

- ●

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、SvelteKitを用いてブログを作成しています。

https://blog.orch-canvas.tokyo

各記事をコンポーネントとし、メタ情報をプロパティとして各記事がもっています。

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

このような形で管理をすると、次のようなメリットがあります。

- Svelteで本文を記述できる
  - 共通コンポーネントを整備しつつ
  - 高い自由度を維持できる
- メタ情報が型安全になる

しかし、この形式には大きなハードルがありました。
**globを用いた動的インポートが必要である**という点です。

今回はこの点の解決法を提示します。

## Viteのglobインポート

Svelte(Kit)はViteの上での利用を想定されています。
Viteはフロントエンドのビルドツールです。Svelteで記述されたファイルはViteを通して、一般的なWebサーバーで実行可能な形式にビルドされます。

そんなViteには、globインポートの機能があります。
これを使うとうまくいきそうです。

[特徴 | Vite](https://ja.vite.dev/guide/features#glob-%E3%81%AE%E3%82%A4%E3%83%B3%E3%83%9B%E3%82%9A%E3%83%BC%E3%83%88)

## 実装

最終的な実装は次のような形になりました。

```ts:/src/lib/posts/index.ts
import type { Component } from 'svelte';

/** 記事のメタ情報の型 */
export type Metadata = {
  published: boolean;
  // 略
};
/** ポストオブジェクトの型 */
export type Post = {
  metadata: Metadata;
  slug: string;
  default: Component & { render: () => { html: string } };
  description: string;
};

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

<data.post.default />
```

順を追って解説していきます。

### `import.meta.glob()`

今回の主役です。
2つの引数を指定できます。

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

#### 第2引数：オプション

オプションとして、次の3つが指定できます。

- `import` インポートするモジュールを明示的に指定
  - デフォルトインポートは`default`を指定
- `query` 各パスにつけるクエリを指定
  - Viteのインポート時には`url`や`raw`、`inline`などが指定できます[^1]
- `eager` `true`を設定することで、同期読み込みとなる
  - 未指定あるいは`false`では、Promiseが返される

[^1]: [静的アセットの取り扱い | Vite](https://ja.vite.dev/guide/assets.html)

今回は、ビルド時に実行されるコードだったため、実装の簡素化のために`eager: true`を指定しました。

### インポートされるコンポーネントの型

```ts
import type { Component } from 'svelte';
```

これで問題なさそうです。

TypeScriptでトリッキーなことをする際のハードルである、型の問題が解決しました。
後は、ウイニング・ランも同然です。

### コンポーネントの描画

Svelte5から、dot notationを用いたコンポーネントの指定が正しく解釈されるようになりました[^2]。
初めてJSXを見たときのようなぞわぞわを感じつつも、自信をもって記述していきます。

[^2]: [Svelte 5 migration guide • Docs • Svelte](https://svelte.dev/docs/svelte/v5-migration-guide#svelte:component-is-no-longer-necessary-Dot-notation-indicates-a-component)

```html
<data.post.default />
```

以上が実装の要点です。
細々としたハードルを整理すれば、ずいぶんシンプルなのではないでしょうか？

## Viteのglobインポートの特徴

●ビルド時に置換される。

---

## おわりに

●

---

<!-- begin upcoming concert announcement -->

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

<!-- end upcoming concert announcement -->

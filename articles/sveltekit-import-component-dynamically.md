---
title: "SvelteKitでコンポーネントを動的に、globでインポートする"
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
import type { ComponentType } from 'svelte';
import { convertToDescription } from '$lib/util';
import { composers, type composerSlug } from './composers';
import type { concertSlug } from './concerts';
import type { Tag } from './tags';

/** 記事のメタ情報の型 */
export type Metadata = {
  published: boolean;
  // 略
};
/** ポストオブジェクトの型 */
export type Post = {
  metadata: Metadata;
  slug: string;
  default: ComponentType & { render: () => { html: string } };
  description: string;
};

// 動的に記事のSvelteファイルを取得する
const modules = import.meta.glob('./**/post.svelte', { eager: true }) as Record<string, Post>;
const rawModules = import.meta.glob('./**/post.svelte', {
  query: '?raw',
  eager: true
}) as Record<string, { default: string }>;

const posts: { [slug: string]: Post } = {};
Object.keys(modules).forEach((path) => {
  const slug = /^.+\/(?<slug>[^/]+)\/post\.svelte$/.exec(path)?.groups?.slug;
  if (slug === undefined) return;

  posts[slug] = {
    metadata: modules[path].metadata,
    slug: slug,
    default: modules[path].default,
    description: convertToDescription(rawModules[path].default)
  };
});
```

順を追って解説していきます。

### `import.meta.glob()`

●

### `Post`型のオブジェクト

●

## Viteのglobインポートの特徴的な仕組み

●ビルド時に置換される。

---

## おわりに

●

---

<!-- begin upcoming concert announcement -->

## 次回演奏会のご案内

日々の癒しに。新しいひらめきのきっかけに。
オーケストラの演奏会はいかがですか？

初めの方も大歓迎！
ご来場お待ちしております。

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

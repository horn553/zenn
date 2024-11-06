---
title: "【②フォーム作成】SvelteKit on Cloudflareでお問い合わせフォームをつくる"
emoji: "🎺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "cloudflarepages", "zod"]
publication_name: "orch_canvas"
published: false
---

私たちは都内を中心に活動しているアマチュアオーケストラの [Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/) です。

弊団のホームページ、ブログのリファクタリングにおいてできた、お問い合わせフォーム実装に関する知見をまとめた本シリーズ。
今回は SvelteKit でフォームの概形を作成していきます！

---

おしながき

1. [① サイト作成：SvelteKit x Cloudflare Pages](https://zenn.dev/orch_canvas/articles/create-contact-form-01)
1. **② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3 ← 今回の記事**
1. [③ セッション管理：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. [④ データベース管理：SvelteKit x Cloudflare D1](https://zenn.dev/orch_canvas/articles/create-contact-form-04)
1. [⑤ メール送信：SvelteKit x Resend](https://zenn.dev/orch_canvas/articles/create-contact-form-05)

このシリーズで完成したもの：
https://github.com/horn553/zenn-contact-form

---

## 要件

今回のフォームの要素は次の通りです：

| 名称           | 種類           | 概要                                         |
| -------------- | -------------- | -------------------------------------------- |
| 氏名           | テキスト       | 送信者の氏名。任意。99 文字以下。            |
| メールアドレス | テキスト       | 送信者のメールアドレス。必須。254 文字以下。 |
| カテゴリー     | ドロップダウン | お問い合わせのカテゴリー。必須。             |
| 本文           | テキストエリア | 必須。999 文字以下。                         |

## 技術選定

フォーム作成のゴールドスタンダードは[Felte](https://felte.dev/)などのフォームライブラリと[Zod](https://zod.dev/)などのバリデーションライブラリを使用することです。
しかし、今回はフォームの規模が大きくなく、いずれの要素もシンプルなものでしたので、バリデーションライブラリとして Zod を使用するのみとしました。

また、bot による自動送信を防ぐため、CAPTCHA を用意するのも王道です。
せっかく Cloudflare を利用しているので[Cloudflare Turnstile](https://www.cloudflare.com/ja-jp/application-services/products/turnstile/)の使用を検証しましたが、相性の問題かクライアントが頻繁にクラッシュしてしまい……
ここは定番の[Google reCAPTCHA v3](https://developers.google.com/recaptcha/docs/v3?hl=ja)を使用することとしました。

## クライアントサイドのフォーム実装

### HTML マークアップ

Svelte のドキュメント通りに実装していきます。
[日本語版ドキュメント：svelte.js](https://svelte.jp/)は現在対応中とのことですので、英語版ドキュメントを参照する必要があります。
[Form actions • Docs • Svelte](https://svelte.dev/docs/kit/form-actions)

```html:/src/routes/contact/+page.svelte
<form method="POST">
  <label>
    お名前
    <input name="name" type="text" maxLength="99">
  </label>
  <label>
        メールアドレス
    <input name="email" type="email" maxLength="254" required>
  </label>
  <label>
        カテゴリー
    <select name="category" required>
            <option value="" hidden selected></option>
            <option value="concert">演奏会について</option>
            <option value="others">その他</option>
        </select>
  </label>
  <label>
    本文
    <textarea name="body" maxLength="999"></textarea>
  </label>
  <button type="submit">送信</button>
</form>
```

これで `npm run dev` から開発環境を起動し、指定された URL から`/contact`にアクセスすると、プレーンなフォームが生まれています。

![プレーンなフォームのスクリーンショット](/images/create-contact-form-02/01.png)

## サーバーサイド：Form Action で受け取る

SvelteKit のフォーム操作、form action で送信内容を受け取ってみます。

先に作成したような、素直なフォームなら簡単に受け取ることができます。

```ts:/src/routes/contact/+page.server.ts
import type { Actions } from './$types';

/**
 * FormDataからObjectに変換する
 * @param d 変換元のFormData
 * @returns 変換後のObject
 */
function convertToObject(d: FormData): Record<string, FormDataEntryValue> {
  const result: Record<string, FormDataEntryValue> = {};
  d.forEach((value, key) => {
    result[key] = value;
  });
  return result;
}

export const actions = {
  default: async ({ request }) => {
    const rawRequestBody = convertToObject(await request.formData());
    console.log({ rawRequestBody });

    return { success: true };
  }
} satisfies Actions;
```

`FormData`型から取り回しがよいように`Object`型への変換を挟んでいるため、やや行数はかかりましたが、非常に簡単にリクエストを受け取ることができます！

![サーバーサイドで受け取ったリクエストのスクリーンショット](/images/create-contact-form-02/02.png)

### サーバーサイドからの結果を受け取る

`Actions`からの戻り値を`ActionData`として受け取ることができます。

```svelte:/src/routes/contact/+page.svelte（抜粋）
<script lang="ts">
  import type { PageData, ActionData } from './$types';

  let { data, form }: { data: PageData, form: ActionData } = $props();
  console.log({form})
</script>
```

![クライアントサイドに戻ってきたレスポンスのスクリーンショット](/images/create-contact-form-02/04.png)

## Zod でバリデーション

`type="email"`、`maxLength`属性などを指定しているため、基本的にはクライアントサイドでバリデーションがかかっています。
しかし、安全性を考慮するとサーバー**サイド**で**再度**のバリデーションは必要です。

Zod を使うと、 schema を作成するだけで型やバリデート関数を用意してもらうことでできます！
早速インストールします。

```shell
npm install zod
```

`/src/lib`配下に schema を置くのも手ですが、お問い合わせフォームでしか使わないので、ここではローカルに配置しておきます。

```ts:/src/routes/contact/schema.ts
import { z } from 'zod';

export const requestBodySchema = z.object({
  name: z.string().max(99),
  email: z.string().email().max(254),
  category: z.enum(['concert', 'others']),
  body: z.string().max(999)
});

export type RequestBody = z.infer<typeof requestBodySchema>;
```

actions のコードでバリデーションをかけるようにします。

```ts:/src/routes/contact/+page.server.ts（抜粋）
export const actions = {
  default: async ({ request }) => {
    const rawRequestBody = convertToObject(await request.formData());

    // バリデーションをかける
    const validationResult = requestBodySchema.safeParse(rawRequestBody);
    if (!validationResult.success) {
      return { success: false, message: 'Invalid request body' };
    }
    const requestBody = validationResult.data;
    console.log({ requestBody });

    return { success: true };
  }
} satisfies Actions;
```

きちんとバリデーションが通っていることを確認できます！

![バリデーションをかけたリクエストのスクリーンショット](/images/create-contact-form-02/03.png)

### 共通パラメータを整理する

最大文字数やカテゴリーの key などが各所に散らばってしまっており、保守性があまりよくありません。

例えば、`schema.ts`にまとめるとすっきり書くことができます。

```ts:/src/routes/contact/schema.ts（抜粋）
export const NAME_MAX_LENGTH = 99;
export const EMAIL_MAX_LENGTH = 254;
export const BODY_MAX_LENGTH = 999;

const categoryKeys = ['concert', 'others'] as const;
export const CATEGORY_OPTIONS: Record<(typeof categoryKeys)[number], string> = {
  concert: '演奏会について',
  others: 'その他'
} as const;

export const requestBodySchema = z.object({
  name: z.string().max(NAME_MAX_LENGTH),
  email: z.string().email().max(EMAIL_MAX_LENGTH),
  category: z.enum(categoryKeys),
  body: z.string().max(BODY_MAX_LENGTH)
});
```

```ts:/src/routes/contact/+page.server.ts（抜粋）
import type { Actions, PageServerLoad } from './$types';
import {
  NAME_MAX_LENGTH,
  EMAIL_MAX_LENGTH,
  CATEGORY_OPTIONS,
  BODY_MAX_LENGTH,
  requestBodySchema
} from './schema';

export const load: PageServerLoad = async () => {
  return {
    NAME_MAX_LENGTH,
    EMAIL_MAX_LENGTH,
    CATEGORY_OPTIONS,
    BODY_MAX_LENGTH
  };
};
```

```svelte:/src/routes/contact/+page.svelte（抜粋）
<script lang="ts">
    import type { PageServerData } from "./$types";

    export let data: PageServerData
</script>

<form method="POST">
  <label>
    お名前
    <input name="name" type="text" maxLength="{data.NAME_MAX_LENGTH}">
  </label>
  <label>
        メールアドレス
    <input name="email" type="email" maxLength="{data.EMAIL_MAX_LENGTH}" required>
  </label>
  <label>
        カテゴリー
    <select name="category" required>
            <option value="" hidden selected></option>
            {#each Object.entries(data.CATEGORY_OPTIONS) as [key, description]}
                <option value={key}>{description}</option>
            {/each}
        </select>
  </label>
  <label>
    本文
    <textarea name="body" maxLength="{data.BODY_MAX_LENGTH}"></textarea>
  </label>
  <button type="submit">送信</button>
</form>
```

## reCAPTCHA v3 を導入

reCAPTCHA Admin Console にて site key と secret を発行します。

この際、ドメインを allow list 形式で指定します。
登録したドメインのサブドメインも自動で許容されます。

検証環境として、`localhost`、`127.0.0.1`も指定しておくと便利です。

![クライアントサイドに戻ってきたレスポンスのスクリーンショット](/images/create-contact-form-02/07.png)

### クライアントサイド

まず、TypeScript 向け型定義をインストールしておきます。

```shell
npm install @types/grecaptcha
```

後は、公式のガイドに従い導入します。
[reCAPTCHA v3  |  Google for Developers](https://developers.google.com/recaptcha/docs/v3?hl=ja)

毎度チェックボックスをクリックさせるのもカッコよくない（要検証）ので、動的にトークンを取得する形を採用しました。

```svelte:/src/routes/contact/+page.svelte（抜粋）
<script lang="ts">
  import { applyAction, deserialize } from "$app/forms";
  import type { ActionResult } from "@sveltejs/kit";
    import type { ActionData, PageServerData } from "./$types";
  import { invalidateAll } from "$app/navigation";

    /* 省略 */

  async function handleSubmit(event: { currentTarget: EventTarget & HTMLFormElement}) {
    const data = new FormData(event.currentTarget);

    // reCAPTCHAトークンを発行
    // eslint-disable-next-line no-undef
    const reCaptchaToken = await grecaptcha.execute(RECAPTCHA_SITE_KEY, { action: 'submit' });
    data.append('reCaptchaToken', reCaptchaToken);

    // サーバーサイドに送信
    const response = await fetch('/contact', {
      method: 'POST',
      body: data
    });
    const result: ActionResult = deserialize(await response.text());

    // リクエストが成功した場合の一連のおまじない
    if (result.type === 'success') {
      // rerun all `load` functions, following the successful update
      await invalidateAll();
    }
    applyAction(result);
  }
</script>

<svelte:head>
  <script src="https://www.google.com/recaptcha/api.js?render={RECAPTCHA_SITE_KEY}" async></script>
</svelte:head>

<form method="POST" on:submit|preventDefault={handleSubmit}>
    <!-- content -->

    <!-- ref: https://developers.google.com/recaptcha/docs/faq?hl=ja#id-like-to-hide-the-recaptcha-badge.-what-is-allowed https://developers.google.com/recaptcha/docs/faq?hl=ja#id-like-to-hide-the-recaptcha-badge.-what-is-allowed-->
    <p class="recaptcha-description">
        このサイトはreCAPTCHAによって保護されており、Googleの
        <a href="https://policies.google.com/privacy">プライバシーポリシー</a>
        と
        <a href="https://policies.google.com/terms">利用規約</a>
        が適用されます。
    </p>
</form>

<style>
  :global(.grecaptcha-badge) {
    visibility: hidden;
  }
</style>
```

schema にも反映させておきます。

```ts:/src/routes/contact/schema.ts（抜粋）
export const requestBodySchema = z.object({
  name: z.string().max(NAME_MAX_LENGTH),
  email: z.string().email().max(EMAIL_MAX_LENGTH),
  category: z.enum(categoryKeys),
  body: z.string().max(BODY_MAX_LENGTH),
  reCaptchaToken: z.string()
});
```

:::message
今回のように custom event listener（関数`handler()`のような）を利用する場合、ページのリロードが行われず、変数`form`定義直後の`console.log()`はうまく発火しません。

Svelte5 の rune システムを活用し、`form`についてリアクティブな処理とする必要があります。

```svelte:/src/routes/contact/+page.svelte
<script lang="ts">
  /** 省略 */

  $effect(() => {
    console.log({ form });
  });
</script>
```

:::

### サーバーサイド

#### 環境変数の設定

Google の API を叩き、検証を行います。

secret の管理が必要です。環境変数ファイルを作成します。
`.gitignore`に指定されていることを確認しておきましょう。

```/.local.env
RECAPTCHA_SECRET_KEY=xxxxxxxxxxxxxxxxxx
```

忘れる前に、Cloudflare Pages の管理画面にも登録しておきます。

![環境変数の追加画面のスクリーンショット](/images/create-contact-form-02/05.png)

:::message

画面上部のプルダウンから選択し、プロダクション、プレビューそれぞれで登録する必要があります！

![環境の選択画面のスクリーンショット](/images/create-contact-form-02/06.png)

:::

#### コーディング

SvelteKit における環境変数の取り扱いを鑑みつつ、実装していきます。

見通しをよくするため、別ファイルに切り出します。

```ts:/src/routes/contact/reCaptchaVerifier.ts
import { RECAPTCHA_SECRET_KEY } from '$env/static/private';

export async function verifyCaptcha(token: string): Promise<boolean> {
  const body = new FormData();
  body.append('secret', RECAPTCHA_SECRET_KEY);
  body.append('response', token);
  const response = await (
    await fetch('https://www.google.com/recaptcha/api/siteverify', {
      body: body,
      method: 'POST'
    })
  ).json();

  if (response?.success) return true;
  return false;
}
```

```ts:/src/routes/contact/+page.server.ts（抜粋）
    /** 省略 */

    // reCAPTCHAを検証
    const captchaResult = verifyCaptcha(requestBody.reCaptchaToken);
    if (!captchaResult) {
      return { success: false, message: 'Invalid CAPTCHA token' };
    }

    /** 省略 */
```

### UX の仕上げ

使い勝手を良くするため、

- フォーム送信中はフォームの各要素を`disabled`にする
- フォーム送信成功後はフォームの各要素を初期化する

といった処理を入れていきます。

#### 送信中は disabled

フラグ`isSubmitting`を Svelte5 の runes システムに乗せます。

```svelte:/src/routes/contact/+page.svelte
<script lang="ts">
  /** 省略 */

  const RECAPTCHA_SITE_KEY = '6LcrCHcqAAAAAGwoYDnJR4xmIUNSfzCdgYZowBpX';
  let isSubmitting = $state(false);
  async function handleSubmit(event: { currentTarget: EventTarget & HTMLFormElement }) {
    isSubmitting = true;

    /** 省略 */

    applyAction(result);
    isSubmitting = false;
  }
</script>

<!-- 省略 -->
    <input name="name" type="text" maxLength={data.NAME_MAX_LENGTH} disabled={isSubmitting} />
<!-- 省略 -->
```

#### フォーム送信後に初期化

前述した変数`form`の更新を察知し、処理に成功した場合に初期化処理を走らせます。

```svelte:/src/routes/contact/+page.svelte
<script lang="ts">
  $effect(() => {
    if (form?.success) {
      // フォームを初期化する
			(document.querySelector('[name=name]') as HTMLInputElement).value = '';
			(document.querySelector('[name=email]') as HTMLInputElement).value = '';
			(document.querySelector('[name=category]') as HTMLSelectElement).selectedIndex = 0;
			(document.querySelector('[name=body]') as HTMLTextAreaElement).value = '';
    }
  });
</script>

<!-- 省略 -->
```

## おわりに

長丁場、おつかれさまでした！
実装した要素の数こそ多いものの、一つひとつがフォームを輝かせる一要素になるのは魅力的ですよね。

フロントエンドはここまでとし、次回からはいよいよサーバーサイドの深みに潜っていきます！

---

1. ① サイト作成：SvelteKit x Cloudflare Pages
1. **[② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3 ← 次の記事](https://zenn.dev/orch_canvas/articles/create-contact-form-02)**
1. [③ セッション管理：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. [④ データベース管理：SvelteKit x Cloudflare D1](https://zenn.dev/orch_canvas/articles/create-contact-form-04)
1. [⑤ メール送信：SvelteKit x Resend](https://zenn.dev/orch_canvas/articles/create-contact-form-05)

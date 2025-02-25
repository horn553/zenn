---
title: "【④ データベース】SvelteKit on Cloudflareでお問い合わせフォームをつくる"
emoji: "🥁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "cloudflarepages", "d1"]
publication_name: "orch_canvas"
published: true
---

私たちは [Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/) です。

弊団のホームページ、ブログのリファクタリングにおいてできた、お問い合わせフォーム実装に関する知見をまとめた本シリーズ。
今回は SvelteKit x Cloudflare D1 で、データベースとの連携処理を実装します！

<!-- begin short upcoming concert announcement -->

:::message
私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。

次回は2025年7月にシューマンの交響曲第2番を演奏します。
初めての方も、そうでない方も、お気軽にお越しください！

詳しくは[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-14)まで。
<!-- textlint-disable -->
:::
<!-- textlint-disable -->

<!-- end short upcoming concert announcement -->

---

#### このシリーズの記事一覧

1. [① サイト作成：SvelteKit x Cloudflare Pages](https://zenn.dev/orch_canvas/articles/create-contact-form-01)
1. [② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3](https://zenn.dev/orch_canvas/articles/create-contact-form-02)
1. [③ セッション：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. **④ データベース：SvelteKit x Cloudflare D1 ← 今回の記事**
1. [⑤ メール送信：SvelteKit x Resend](https://zenn.dev/orch_canvas/articles/create-contact-form-05)

このシリーズの完成物
https://github.com/horn553/zenn-contact-form

---

## はじめに

お問い合わせフォームの送信内容を保存する方法はいくつか考えられます。

例えば、シンプルにお問い合わせ内容を関係者にメールし、周知とログを兼ねる方法が考えられます。
最近だと、Slack に投稿するという方法も考えられそうです。

古典的なレンタルサーバーであれば、PHP 経由でローカルにファイルを作成するという方法もあります。

非技術者による管理や分析のしやすさを考え、Google スプレッドシート + Google Apps Script と連携するという方法も便利そうです。

前回紹介した、Key-Value ストレージを使うことも選択肢ではあります。

### Cluodflare D1

より普遍的に考えると、多くのアプリケーションにはデータベースがつきものです。
今回のようなシンプルなアプリケーションは、データベースに慣れる絶好のチャンスではないか！

そこで、今回はデータベースを利用することとしました。

Cloudflare Pages、KV と Cloudflare 系のサービスを用いていますので、同社の SQL データベースサービスである Cloudflare D1 を利用する方針としました。

## 技術選定

データベースには [Cloudflare D1](https://www.cloudflare.com/ja-jp/developer-platform/products/d1/) を用います。

データベースをアプリケーションから操作する場合、ORM を用いることが通例です。
プログラムの見通しをよくできますし、SQL インジェクションによる脆弱性を回避しやすくなります。

今回は、SvleteKit、Cloudflare D1 いずれとも公式で連携が想定されている、[Drizzle](https://orm.drizzle.team/)を用いることとします。

### Cloudflare Pages x D1

前回述べたように、Cloudlare Workers でできることは Pages Functions を用いて Cloudflare Pages でもできます。

今回も、Pages Functions を用いて D1 を操作していきます。

### 価格

Cloudflare D1 の無料枠は次の通りです。
参考：[Pricing - Cloudflare D1 docs](https://developers.cloudflare.com/d1/platform/pricing/)

- 読み取り行上限: 5,000,000/日
- 書き込み行上限: 100,000/日
- ストレージ上限: 5GB

Cloudflare Pages Functions については前回の記事で述べた通りです。

これまた、ある程度のアプリケーションであれば実用に耐えるものではないでしょうか？

## 環境構築

### 依存関係のインストール

まずは依存関係をインストールします。
参考：[Drizzle ORM - Cloudflare D1](https://orm.drizzle.team/docs/connect-cloudflare-d1)

```shell
npm i drizzle-orm dotenv
npm i -D drizzle-kit
```

:::message alert
執筆時点では各種パッケージが Svelte5 をサポートしておらず、エラーとなります。
過渡期の対応として、`--force` オプション付きで実行します。

```shell
npm i --force drizzle-orm dotenv
npm i -D --force drizzle-kit
```

:::

### Drizzle の設定ファイルを作成

```ts:/drizzle.config.ts
import 'dotenv/config';
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  out: './drizzle',
  schema: './src/db/schema.ts',
  dialect: 'sqlite'
});
```

### Wrangler の設定

前回の記事で初期設定済みですので割愛します。

### KV 名前空間の作成

Cloudflare ダッシュボード（GUI）で作成してもいいですが、ここでは Wrangler（CLI）で作成します。
また、この規模だとプレビュー環境と本番環境は同じでもよい気がしますが、ここではあえて異なる名前空間で管理する方法をまとめます。

前回と同じく、攻めの姿勢です。

```shell
npx wrangler d1 create zenn-contact-form # プレビュー環境用
npx wrangler d1 create zenn-contact-form-production # プロダクション環境用
```

こうすると 2 つの `id` が生成されます。
それぞれを wrangler の設定ファイルに反映させます。

ローカル環境の `database_id` には適当な文字列を指定します。
ここでは、プレビュー環境と同じ ID を指定しておきます。

キー `binding` の値は、SvelteKit から呼び出す際のキーの名称です。
プロジェクト内で一意であればよいので、`DB` とします。

```diff toml:/wrangler.toml
  # Generated by Wrangler on Fri Nov 08 2024 18:04:42 GMT+0900 (日本標準時)
  name = "zenn-contact-form"
  pages_build_output_dir = ".svelte-kit/cloudflare"
  compatibility_date = "2024-11-05"

  # ローカル環境
  [[kv_namespaces]]
  binding = "KV"
  id = "6653db06adad47b39dd539b60abe5e6c"
  preview_id = "6653db06adad47b39dd539b60abe5e6c"
+
+ [[d1_databases]]
+ binding = "DB"
+ database_name = "zenn-contact-form"
+ database_id = "931be2fb-7a5c-4155-8c22-8534b80c5410"

  # プレビュー環境
  [[env.preview.kv_namespaces]]
  binding = "KV"
  id = "95ae2b54ae2f4977a23f144e2eb575ef"
+
+ [[env.preview.d1_databases]]
+ binding = "DB"
+ database_name = "zenn-contact-form"
+ database_id = "931be2fb-7a5c-4155-8c22-8534b80c5410"

  # プロダクション環境
  [[env.production.kv_namespaces]]
  binding = "KV"
  id = "353de8673ca54e078cfdc52b6735947c"
+
+ [[env.production.d1_databases]]
+ binding = "DB"
+ database_name = "zenn-contact-form-production"
+ database_id = "20e85731-7775-44dd-8f25-85340144d34b"
```

### .gitignore の更新

これも前回やったので割愛します。

### SvelteKit の設定

`DB` がバインドされていることを明示します。

```diff ts:/src/app.d.ts
  // See https://kit.svelte.dev/docs/types#app
  // for information about these interfaces
  declare global {
    namespace App {
      // interface Error {}
      // interface Locals {}
      // interface PageData {}
      // interface PageState {}
      interface Platform {
        env: {
          KV: KVNamespace;
+         DB: D1Database;
        };
      }
    }
  }

  export {};
```

## 実装

今回の実装はサーバーサイドのみです。

### スキーマの作成

DB の schema（table、row の定義）を作成します。
Drizzle のドキュメントに従い、`/src/db/schema.ts` に作成します。
参考：[Drizzle ORM - Schema](https://orm.drizzle.team/docs/sql-schema-declaration)

```ts:/src/db/schema.ts
import { sqliteTable, text } from 'drizzle-orm/sqlite-core';

export const contacts = sqliteTable('contacts', {
  id: text('id').primaryKey(),
  status: text('status').notNull(),
  receivedAt: text('receivedAt').notNull(),
  name: text('name').notNull(),
  email: text('email').notNull(),
  category: text('category').notNull(),
  body: text('body').notNull(),
  csrfToken: text('csrfToken').notNull(),
  reCaptchaToken: text('reCaptchaToken').notNull()
});
```

### マイグレーション

Drizzle Kit でマイグレーションファイルを作成し、各種環境に適用します。

```shell
npx drizzle-kit generate
npx wrangler d1 migrations apply DB --local
npx wrangler d1 migrations apply DB --remote
npx wrangler d1 migrations apply DB --remote --env production
```

### SvelteKit から呼び出す

#### schema を用意する

Zod を用いて schema を用意します。

```diff ts:schema.ts
+ export const statuses = ['invalid csrf', 'invalid captcha', 'have sent email'] as const;
+ export const statusScheme = z.enum(statuses).brand<'Status'>();
+ export type Status = z.infer<typeof statusScheme>;
+
+ export const contactLogSchema = requestBodySchema.extend({
+   id: z.string().uuid(),
+   receivedAt: z.string().datetime(),
+   status: statusScheme
+ });
+ export type ContactLog = z.infer<typeof contactLogSchema>;
```

#### ORM からデータベースに書き込む関数を作成

```ts:/src/routes/contact/logger.ts
import { drizzle, type AnyD1Database } from 'drizzle-orm/d1';
import { contacts } from '../../db/schema';
import type { ContactLog } from './schema';

export async function log(db: AnyD1Database, value: ContactLog) {
  const drizzleDb = drizzle(db);
  const result = await drizzleDb.insert(contacts).values(value);
  return result;
}
```

#### 書き込むデータの用意し、書き込む

受け取ったデータに各種データを補いつつ、schema に格納していきます。

```diff ts:+page.server.ts

  export const actions = {
-   default: async ({ locals, request }) => {
+   default: async ({ locals, platform, request }) => {
      const { session } = locals;
      const rawRequestBody = convertToObject(await request.formData());

      // バリデーションをかける
      const validationResult = requestBodySchema.safeParse(rawRequestBody);
      if (!validationResult.success) {
        return { success: false, message: 'Invalid request body' };
      }
      const requestBody = validationResult.data;

+     // id採番、受信日時取得
+     let currentStatus: (typeof statuses)[number] = 'have sent email';
+     const contactLog: ContactLog = {
+       id: crypto.randomUUID(),
+       status: statusScheme.parse(currentStatus),
+       receivedAt: new Date().toISOString(),
+       ...requestBody
+     };
+
      // CSRFトークンを検証
      const csrfResult = requestBody.csrfToken === session.data.csrfToken;
      if (!csrfResult) {
+       currentStatus = 'invalid csrf';
+       contactLog.status = statusScheme.parse(currentStatus);
+       log(platform?.env.DB, contactLog);
+
        return { success: false, message: 'Invalid CSRF token' };
      }

      // reCAPTCHAを検証
      const captchaResult = verifyCaptcha(requestBody.reCaptchaToken);
      if (!captchaResult) {
+       currentStatus = 'invalid captcha';
+       contactLog.status = statusScheme.parse(currentStatus);
+       log(platform?.env.DB, contactLog);
+
        return { success: false, message: 'Invalid CAPTCHA token' };
      }

+     // TODO: メールを送信
+
+     currentStatus = 'have sent email';
+     contactLog.status = statusScheme.parse(currentStatus);
+     log(platform?.env.DB, contactLog);
+
      return { success: true };
    }
  } satisfies Actions;
```

……イマイチ実装はパッとしないものになっていますが、処理としては問題ないかと思います。

## 検証

### ローカル環境

前回に引き続き、`npm run dev` は使うことができませんので、一度ビルドし、Wrangler 経由で実行します。

```shell
npm run build
npx wrangler pages dev ./.svelte-kit/cloudflare/
```

`/contact` にアクセスし、フォームを操作します。
処理が正常に完了し、フォームが初期化されます。

Wrangler 経由で SQL 文を実行し、きちんと送信されていることを確認します。

```shell
npx wrangler d1 execute DB --command "select id, receivedAt, status from contacts"
```

最新の日時での送信内容が確認できます！

### プレビュー環境、プロダクション環境

これをデプロイすると、それぞれの環境でそれぞれの環境の D1 に接続し動作します。
一方で、それぞれの環境の中では同一の D1 に接続するため、過去のデプロイで作成・編集したデータを引き継ぎます。

Cloudflare ダッシュボードから D1 のページに行き、各データベースの内容を直接見て確認できます。
Wrangler からの確認も可能です。

---

## おわりに

データベースとの連携……これだけで世の中の多くのアプリケーションは実装できることになります。
なんと甘美な響き……たまりませんね！

次回は Cloudflare から少し離れて、Resend を用いたメール送信機能を実装し、フォームを仕上げていきます！

---

1. [① サイト作成：SvelteKit x Cloudflare Pages](https://zenn.dev/orch_canvas/articles/create-contact-form-01)
1. [② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3](https://zenn.dev/orch_canvas/articles/create-contact-form-02)
1. [③ セッション：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. ④ データベース：SvelteKit x Cloudflare D1
1. **[⑤ メール送信：SvelteKit x Resend ← 次の記事 ← 次の記事](https://zenn.dev/orch_canvas/articles/create-contact-form-05)**

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
> 詳細は[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-14)にて

<!-- end long upcoming concert announcement -->

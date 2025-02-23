---
title: "【⑤ メール送信】SvelteKit on Cloudflareでお問い合わせフォームをつくる"
emoji: "🎼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "cloudflarepages", "resend"]
publication_name: "orch_canvas"
published: true
---

私たちは [Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/) です。

弊団のホームページ、ブログのリファクタリングにおいてできた、お問い合わせフォーム実装に関する知見をまとめた本シリーズ。

今回は最終回！
SvelteKit x Resend で、メールの送信機能を実装します！！

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
1. [④ データベース：SvelteKit x Cloudflare D1](https://zenn.dev/orch_canvas/articles/create-contact-form-04)
1. **⑤ メール送信：SvelteKit x Resend ← 今回の記事**

このシリーズの完成物
https://github.com/horn553/zenn-contact-form

---

## はじめに

お問い合わせフォームにメール送信機能を実装するのはよいアイデアですし、もはや必須といっても過言ではありません。

理由は主に 2 つあります。

1. 送信者が送信できたことやその内容を確認できる
1. 個別の回答へのスムーズな導線を用意できる

2 つ目をチャットページへの誘導で代替している場合もありますが、工数がけた違いとなってしまいます。

メール送信機能の実装は迷わず行う方針として、技術選定に移ります。

## 技術選定

Cloudflare で管理されているドメインであれば、Email Routing の機能を用いてメール送信ができるようです。
参考：[Send emails from Workers | Cloudflare Email Routing docs](https://developers.cloudflare.com/email-routing/email-workers/send-email-workers/)

Cloudflare で取得できる＝移管できるドメインは数多くありますが、日本の地名ドメインなど、対応していないものも散見されます。
弊団のドメイン `orch-canvas.tokyo` も Cloudflare では取得できません。
参考：[Cloudflare Registrar - Extensions](https://domains.cloudflare.com/extensions)

その他のメール送信の手段として、[Resend](https://resend.com/) を用いる方法が紹介されています。
今回はこの方法を採用します。
参考：[Send Emails With Resend | Cloudflare Workers docs](https://developers.cloudflare.com/workers/tutorials/send-emails-with-resend/)

### Resend

既存の独自ドメインに対し、メール送信 API を提供するサービスです。
DNS レコードを追加し、Resend サーバーをメール送信サーバーとして登録することで実現させています。

今回は使いませんが、予約送信、開封・リンク追跡など多数の機能を兼ね備えています。

### 価格

Resend の無料枠の概要は次の通りです。
参考：[Pricing · Resend](https://resend.com/pricing)

- メール送信上限: 100 通/日 かつ 3,000 通/月

多くのアプリケーションでは問題とならなさそうですね！

## 環境構築

公式が Cloudflare Workers との連携ガイドを提供しています。
今回利用する Cloudflare Pages Functions は Workers と同等ですので、このガイドに従う方針でよさそうです。
参考：[Cloudflare Workers - Resend](https://resend.com/docs/send-with-cloudflare-workers#3-send-the-email-using-react-and-the-sdk)

前述した通り、Cloudflare 側もガイドを提供しています。相思相愛！
参考：[Send Emails With Resend | Cloudflare Workers docs](https://developers.cloudflare.com/workers/tutorials/send-emails-with-resend/)

今回は、そのうち必要な部分のみをかいつまんで実装していきます。

### 依存関係のインストール

まずは依存関係をインストールします。

```shell
npm i resend
```

:::message alert
執筆時点では各種パッケージが Svelte5 をサポートしておらず、エラーとなります。
過渡期の対応として、`--force` オプション付きで実行します。

```shell
npm i --force resend
```

:::

### Resend にドメインを登録

Resend のアカウントを作成し、ドメインを登録します。
アカウントページの Domains から、Add Domain をクリックし、画面の指示に従っていきます。

DNS のセットアップ画面に移行しますので、それぞれすべてを設定する独自ドメインの DNS に設定します。
メール送信の核となる MX レコードのほかに、DMARC(SPF+DKIM) に必要な TXT レコードも指定されます。

しばらくし、ドメインのステータスが Verified になれば完了です！

![Domainsページのスクリーンショット](/images/create-contact-form-05/01.png)

### API Key を発行

Resend のアカウントページから API Keys に遷移し、Create API Key から作成します。

![API Keysページのスクリーンショット](/images/create-contact-form-05/02.png)

発行した API Key は環境変数に追加しておきます。

```diff ini:/.env.local
  RECAPTCHA_SECRET_KEY=xxxxxxxxxxxxxxxx
+ RESEND_API_KEY=xxxxxxxxxxxxxxxx
```

忘れる前に Cloudflare ダッシュボードにも追加しておきます。

![環境変数の追加画面のスクリーンショット](/images/create-contact-form-02/05.png)

:::message

画面上部のプルダウンから選択し、プロダクション、プレビューそれぞれで登録する必要があります！

![環境の選択画面のスクリーンショット](/images/create-contact-form-02/06.png)

:::

### 文面の作成

基本的なメールの形態はテキストですが、HTML 版を用意するという手があります。

Svelte でもっとも有名な HTML メール作成ライブラリは [svelte-email](https://svelte-email.vercel.app/docs/overview/svelte-email) でしょうか。
しかし、1 年以上メンテナンスされておらず、Svelte5 への対応が行われるか判断がつきません。

安全のため、また HTML メールは Resend の紹介記事において本質でないことから、テキストのみのメールを送信することにします。
守りの姿勢です。

ちなみに、メールの構成要件はシンプルであるため、弊団のホームページでは文字列処理だけで HTML メールを作成する方針としていたりします。

……閑話休題。
メール送信機能は別ファイルに切り出しました。

API Key を用いてクラス `Resend` をインスタンス化し、`Resend.emails.send()` を呼び出します。
参考：[Send Email - Resend](https://resend.com/docs/api-reference/emails/send-email)

```ts:/src/routes/contact/emailSender.ts
import { Resend } from 'resend';
import { type RequestBody } from './schema';
import { RESEND_API_KEY } from '$env/static/private';

export async function sendEmail(content: RequestBody) {
  const resend = new Resend(RESEND_API_KEY);
  await resend.emails.send({
    from: 'お問い合わせフォーム <webadmin@orch-canvas.tokyo>',
    to: [content.email],
    cc: 'contact@example.com',
    replyTo: 'contact@example.com',
    subject: 'お問い合わせを承りました',
    text: generateBody(content)
  });
}

function generateBody(content: RequestBody): string {
  let body: string = `ホームページより、お問い合わせを承りました。
必要に応じてメールにてご返答いたします。
なお、メールアドレスが正しく入力されていない場合、返答いたしかねます。ご了承ください。
* * *
分類：${content.category}
本分：
${content.body}`;

  // 名前が送信されている場合、宛名と適当な改行を挿入
  if (content.name)
    body =
      `${content.name}さま
` + body;

  return body;
}
```

……改行含みの文をテンプレートリテラルでやるのはやや可読性に欠けますが……
各行の配列にして `.join('\n')` すべきか、悩みどころです。

### 送信処理を呼び出す

特記することはありません。
粛々とコーデイングします。

```diff ts:/src/routes/contact/+page.server.ts
  import type { Actions, PageServerLoad } from './$types';
+ import { sendEmail } from './emailSender';
  import { log } from './logger';
  import { verifyCaptcha } from './reCapthaVerifier';

  /* 省略 */

  export const actions = {
    default: async ({ locals, platform, request }) => {

      /* 省略 */

      // reCAPTCHAを検証
      const captchaResult = verifyCaptcha(requestBody.reCaptchaToken);
      if (!captchaResult) {
        currentStatus = 'invalid captcha';
        contactLog.status = statusScheme.parse(currentStatus);
        log(platform?.env.DB, contactLog);

        return { success: false, message: 'Invalid CAPTCHA token' };
      }

-     // TODO: メールを送信
+     sendEmail(requestBody);

      currentStatus = 'have sent email';
      contactLog.status = statusScheme.parse(currentStatus);
      log(platform?.env.DB, contactLog);

      return { success: true };
    }
  } satisfies Actions;
```

これでメール送信機能の核は実装完了です！

---

## おわりに

メール送信機能を構成する技術の複雑さを感じさせない、とてもスマートな API ですね！
これが 100 通/日までは無料で使えるということですから、ありがたい時代になったものです。

このシリーズは以上となります。
様々なアプリケーションの核となる機能を Cloudflare ベースで実装していきました。

この記事が少しでも皆さんのお役に立てば、それほど嬉しいことはありません。

---

1. [① サイト作成：SvelteKit x Cloudflare Pages](https://zenn.dev/orch_canvas/articles/create-contact-form-01)
1. [② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3](https://zenn.dev/orch_canvas/articles/create-contact-form-02)
1. [③ セッション：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. [④ データベース：SvelteKit x Cloudflare D1](https://zenn.dev/orch_canvas/articles/create-contact-form-04)
1. ⑤ メール送信：SvelteKit x Resend

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
> 詳細は[当団ホームページ](https://www.orch-canvas.tokyo/concerts/regular-14)にて

<!-- end long upcoming concert announcement -->

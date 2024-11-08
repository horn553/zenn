---
title: "【① サイト作成】SvelteKit on Cloudflareでお問い合わせフォームをつくる"
emoji: "🎻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["svelte", "sveltekit", "cloudflare", "cloudflarepages"]
publication_name: "orch_canvas"
published: false
---

私たちは都内を中心に活動しているアマチュアオーケストラの [Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/) です。

今回、弊団のホームページ、ブログをリファクタリングし、パブリックリポジトリとして公開しました 🎉
[homepage](https://github.com/orchestra-canvas-tokyo/homepage) / [blog](https://github.com/orchestra-canvas-tokyo/blog)

その中の最大の山場であった「お問い合わせフォーム」まわりについて、数記事にわたり知見を共有し尽していきたいと考えています。

---

#### このシリーズの記事一覧

1. **① サイト作成：SvelteKit x Cloudflare Pages ← 今回の記事**
1. [② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3](https://zenn.dev/orch_canvas/articles/create-contact-form-02)
1. [③ セッション管理：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. [④ データベース管理：SvelteKit x Cloudflare D1](https://zenn.dev/orch_canvas/articles/create-contact-form-04)
1. [⑤ メール送信：SvelteKit x Resend](https://zenn.dev/orch_canvas/articles/create-contact-form-05)

このシリーズで作成したお問い合わせフォームはこちら。
https://github.com/horn553/zenn-contact-form

---

## この記事でやること

- ホームページの要件や技術構成を紹介
  - お問い合わせフォームの背景に触れます！
- Cloudflare Pages に SvelteKit プロジェクトをデプロイ
  - これらの組み合わせの便利さを語ります！

## 今回作成したサイトの要件

大部分は、団体や演奏会について紹介する静的なサイトです。

![トップページのスクリーンショット](/images/create-contact-form-01/01.png)

その中で、ひとつだけ動的なページが必要となりました。お問い合わせフォームです。

### お問い合わせフォーム

演奏会に関する質問、チラシの配布依頼、はたまた参加希望と団外からのお問い合わせはしばしば発生します。

今回は、画像のようなお問い合わせフォームを用意することとしました。

![お問い合わせフォームのスクリーンショット](/images/create-contact-form-01/02.png)

## 今回採用した技術構成

[ブログ](https://blog.orch-canvas.tokyo)などの関連 Web ページを Cloudflare Pages で構築していたため、それに準ずる形としました。

| 種類           | 仕様技術                                                                                  | 備考                                                                                           |
| -------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| フレームワーク | SvelteKit                                                                                 | Svelte4(TypeScript, HTML, CSS)                                                                 |
| リポジトリ     | GitHub                                                                                    |                                                                                                |
| ホスティング   | [Cloudflare Pages](https://www.cloudflare.com/ja-jp/developer-platform/products/pages/)   | GitHub と連携                                                                                  |
| セッション管理 | [Cloudflare KV](https://www.cloudflare.com/ja-jp/developer-platform/products/workers-kv/) | ライブラリとして[svelte-kit-sessions](https://www.npmjs.com/package/svelte-kit-sessions)を使用 |
| データベース   | [Cloudflare D1](https://www.cloudflare.com/ja-jp/developer-platform/products/d1/)         | ORM として[Drizzle](https://orm.drizzle.team/)を使用                                           |
| CAPTCHA        | [Google reCAPTCHA v3](https://developers.google.com/recaptcha?hl=ja)                      |                                                                                                |

## 環境構築

今回は、類似のアプリケーションをイチから作ることで、各技術の概要を紹介していきます。

### SvelteKit プロジェクトの作成

まずは SvelteKit のプロジェクト作成します。

今回は次のような設定としました。

- SvelteKit demo テンプレート
- TypeScript を使用
- Prettier、ESLint を使用

```shell
npx sv create my-app
cd my-app
npm install
npm run dev
```

この後、画面の指示にしたがって Git へのコミットまで済ませます。

参考：[Creating a project • Docs • Svelte](https://svelte.dev/docs/kit/creating-a-project)

#### adapter-cloudflare の導入

Cloudflare Pages 向けにビルドするための adapter が用意されています。
ありがたく使わせていただきます。

```shell
npm i @sveltejs/adapter-cloudflare
```

:::message alert
執筆時点では各種パッケージが Svelte5 をサポートしておらず、エラーとなります。
過渡期の対応として、`--force` オプション付きで実行します。

```shell
npm i --force @sveltejs/adapter-cloudflare
```

:::

SvelteKit のコンフィグファイルも更新します。
参考：[Cloudflare Pages • Docs • Svelte](https://svelte.dev/docs/kit/adapter-cloudflare)

```js:/svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';

export default {
  kit: {
    adapter: adapter({
      // See below for an explanation of these options
      routes: {
        include: ['/*'],
        exclude: ['<all>']
      },
      platformProxy: {
        configPath: 'wrangler.toml',
        environment: undefined,
        experimentalJsonConfig: false,
        persist: false
      }
    })
  }
};
```

### GitHub リポジトリとの作成

GitHub でリポジトリを作成します。
表示されたガイドに従い、ローカルの Git に origin として設定します。

```shell
git remote add origin git@github.com:horn553/zenn-contact-form.git
git branch -M main
git push -u origin main
```

### Cloudflare Pages アプリケーションの作成

[Cloudflare ダッシュボード](https://dash.cloudflare.com/)にログインし、Pages アプリケーションを作成します。

GitHub との連携を選択し、認可を進めていくと、先ほど作成したリポジトリを使用したアプリケーションを作成できます。

「ビルドとデプロイのセットアップ」でビルドの設定をする必要があります。
フレームワーク プリセットに SvelteKit が用意されているので、それを選択します。
（他の項目は自動的に埋められます）

![ビルドとデプロイのセットアップ画面のスクリーンショット](/images/create-contact-form-01/03.png)

「保存してデプロイ」を行うことで、プロジェクトの作成と最初のビルド・デプロイが行われます。

:::message alert
初回ビルドは node.js のバージョンの問題で失敗します。
例えば、ルートディレクトリに `.node-version`ファイルを作成すると解消します。

ここでは、執筆時点で最新の LTS なバージョンにします。

```plain:.node-version
22.11.0
```

参考：[Languages and runtime - Build image | Cloudflare Pages docs](https://developers.cloudflare.com/pages/configuration/build-image/#languages-and-runtime)

:::

:::message
たまに、非特異的なエラーでビルドに失敗することがあります。
「デプロイを再試行」すると通ったりします。

謎です。

![デプロイを再試行画面のスクリーンショット](/images/create-contact-form-01/08.png)

:::

デプロイ成功後にデプロイ先の URL へアクセスすると、元気に動く SvelteKit の姿を見ることができます！

#### デプロイの仕様

管理画面で確認できるように、GitHub と連携して各プッシュに対し、ビルド・デプロイが自動実行されます。
本番環境として設定したブランチに対するプッシュは本番環境に、それ以外はプレビュー環境にデプロイされます。
本番環境、プレビュー環境、ブランチエイリアスの他、各プッシュごとの URL も発行され、大変便利です。

![デプロイの詳細画面のスクリーンショット](/images/create-contact-form-01/04.png)

プレビュー環境として一部のブランチのみを（allow list / deny list で）指定したい場合など、詳細な設定も用意されています。

![デプロイの詳細画面のスクリーンショット](/images/create-contact-form-01/05.png)

#### カスタムドメインを使用

Pages プロジェクト作成時は、`*.pages.dev`ドメインが設定されています。
これにカスタムドメインをあてることが可能です。

アプリケーションのページの「カスタムドメイン」タブから管理できます。

![デプロイの詳細画面のスクリーンショット](/images/create-contact-form-01/06.png)

画面の指示に従うことで、セットアップは簡単（<10 分）に終了します。
他社のネームサーバーを使用しているドメインの場合も、CNAME レコードを設定することで簡単にセットアップができます！

![CNAMEセットアップ画面のスクリーンショット](/images/create-contact-form-01/07.png)

### おわりに

今回は、採用技術の概要と、SvelteKit プロジェクトを Cloudflare Pages にデプロイする方法についてまとめました。

次回からは本格的にフォームの作成を進めていきます！

---

1. ① サイト作成：SvelteKit x Cloudflare Pages
1. **[② フォーム作成：SvelteKit x Zod x Google reCAPTCHA v3 ← 次の記事](https://zenn.dev/orch_canvas/articles/create-contact-form-02)**
1. [③ セッション管理：SvelteKit x Cloudflare KV](https://zenn.dev/orch_canvas/articles/create-contact-form-03)
1. [④ データベース管理：SvelteKit x Cloudflare D1](https://zenn.dev/orch_canvas/articles/create-contact-form-04)
1. [⑤ メール送信：SvelteKit x Resend](https://zenn.dev/orch_canvas/articles/create-contact-form-05)

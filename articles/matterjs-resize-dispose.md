---
title: "【Matter.js】実装のTips――高DPI対応、空間のリサイズ、削除ほか"
emoji: "🎱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["matterjs", "javascript", "typescript"]
publication_name: "orch_canvas"
published: false
# published_at: 2025-04-07 06:00
---

## まとめ

- Matter.jsは、Webサイト上で動作する2D物理演算ライブラリ
- 物理エンジン空間や描画空間のサイズを動的に変更できる
- 円に画像を貼り付けたり、互いに跳ねさせたり

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2025年7月にシューマンの交響曲第2番。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/44429?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、エイプリルフールとして、「Orchestra "Nyan"vas Tokyo」への改名企画を行いました。
インタラクティブなコンテンツとして、物理演算を用いて肉球で遊べるウェブページを作成しました。

https://www.orch-canvas.tokyo/nyanvas

今回は、物理演算周りの実装について、知見をまとめてみました！

基本的な実装方法などは、色々な記事に分かりやすくまとまっていますので、あまり情報がない枝葉の部分を中心に取り上げていきます。

https://zenn.dev/chot/articles/21d6d94c314979

https://zenn.dev/sdkfz181tiger/books/5fb7ad6afa3a5e

---

## 物理演算空間の調整

### 高DPIディスプレイ対応

`window.devicePixelRatio`を用いることで、CSSピクセル解像度に対するディスプレイの物理的なピクセル解像度の比率を取得できます。
ズーム時のSafari系で変更が反映されないというバグがあるようですが、ほとんどの環境・条件で問題なく取得できます。

https://developer.mozilla.org/ja/docs/Web/API/Window/devicePixelRatio

Matter.js側では、Renderの初期化時の引数で比率を設定できます。

```ts
const parentElement = document.body;
const width = window.innerWidth;
const height = window.innerHeight;
const pixelRatio = window.devicePixelRatio;

const engine = Matter.Engine.create()
const render = Matter.Render.create({
    element: parentElement,
    engine,
    options: {
        width,
        height,
        hasBounds: true,
        pixelRatio,
        wireframes: false
    }
});
```

### リサイズ

今回作成したWebサイトでは、空間いっぱいに背景として広がる、物理演算空間を用意しました。
画面のリサイズ（ウィンドウサイズ変更、画面の回転、アドレスバーの引っ込み）で物理演算空間のサイズを変更する必要があります。

先のコードに追加実装してみます。

```ts
const parentElement = document.body;
const width = window.innerWidth;
const height = window.innerHeight;
const pixelRatio = window.devicePixelRatio;

const engine = Matter.Engine.create()
const render = Matter.Render.create({
    element: parentElement,
    engine,
    options: {
        width,
        height,
        hasBounds: true,
        pixelRatio,
        wireframes: false
    }
});

window.addEventListener('resize', () => {
    const newWidth = window.innerWidth;
    const newHeight = window.innerHeight;

    // 物理空間のサイズを変更
    render.bounds.max.x = newWidth;
    render.bounds.max.y = newHeight;

    // 描画キャンバスのサイズを変更
    render.canvas.style.width = `${newWidth}px`;
    render.canvas.style.height = `${newHeight}px`;
})
```

### 削除

canvas要素を削除することで対応できます。

```ts
render.canvas.remove();
```

## 円に関する微調整

肉球を円として描画していますが、小気味よく動作するような微調整を加えました。

### 跳ねさせる

円だけでなく、壁にも反発係数を設定する必要があります。
重さも設定する必要があります。

```ts
// engine、renderを定義済みとする

const width = window.innerWidth;
const height = window.innerHeight;
const wallThickness = 999;
const restitution = 1;

const walls = [
    // 天井
    Matter.Bodies.rectangle(width / 2, -thickness / 2, width, thickness, {
        restitution: this.restitution,
        isStatic: true
    }),

    // 床
    Matter.Bodies.rectangle(width / 2, height + thickness / 2, width, thickness, {
        restitution: this.restitution,
        isStatic: true
    }),

    // 壁
    Matter.Bodies.rectangle(-thickness / 2 - 1, height / 2, thickness, height, {
        restitution: this.restitution,
        isStatic: true
    }),
    Matter.Bodies.rectangle(width + thickness / 2, height / 2, thickness, height, {
        restitution: this.restitution,
        isStatic: true
    })
];

Matter.Composite.add(engine.world, walls)

// 円を描画する
const circleSize = 30;
const circleMass = 75;
const circle = Matter.Bodies.circle(100, 100, circleSize, {
    mass: circleMass,
    restitution
})
```

### 画像を貼る

ソースコードを覗くに、`.jpg`、`.gif`、`.gif`が対応しているようです。

https://github.com/liabru/matter-js/blob/acb99b6f8784c809b940f1d2cf745427e088e088/src/render/Render.js#L1534-L1543

```ts
// 円を描画する
const circleSize = 30;
const circleMass = 75;

const circleTexture = './assets/circle.png';
const imageSize = 1000;
const textureScale = circleSize / (imageSize / 2);

const circle = Matter.Bodies.circle(100, 100, circleSize, {
    mass: circleMass,
    restitution,
    render: {
        sprite: {
            texture: circleTexture,
            xScale: textureScale,
            yScale: textureScale
        }
    }
})
```

---

## おわりに

ここまでお読みいただきありがとうございました！

冒頭で紹介した、当団エイプリルフールのページをいじっていただくとお分かりかと思いますが、Matter.jsは軽量です。
ちょっとしたインタラクションとして、有用なライブラリではないでしょうか！

一方で、[ドキュメント](https://brm.io/matter-js/)が少し古いのか、実装と食い違っている部分があります。
ネット上の情報も古いものが散見されます。

適宜、[ソースコード](https://github.com/liabru/matter-js)を参照するとよさそうです。
GitHub Copilotと一緒にソースコードを探索できる今は比較的簡単に読めて……ありがたいものです！

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

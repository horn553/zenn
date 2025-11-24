---
title: "【Matter.js】物理演算で遊べるサイトを作る【エイプリルフール】"
emoji: "🐾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["matterjs", "typescript", "svelte", "sveltekit"]
publication_name: "orch_canvas"
published: true
published_at: 2025-04-07 06:00
---

## まとめ

- Matter.jsは、Webサイト上で動作する2D物理演算ライブラリ
- これを使って、インタラクティブな肉球を背景にするエイプリルフール企画を作成した
- Matter.jsの基本的な実装から、細かいTipsをまとめた

<!-- begin short upcoming concert announcement -->

> 私たちOrchestra Canvas Tokyoは、都内を中心に活動するアマチュア・オーケストラです。
>
> 次回は2026年3月にバーンスタインのシンフォニックダンス。
> 初めての方も、そうでない方も、お気軽にお越しください！
>
> 詳しくは[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)まで。

<!-- end short upcoming concert announcement -->

---

## 背景

当団（[Orchestra Canvas Tokyo](https://www.orch-canvas.tokyo/)）では、エイプリルフールとして、「Orchestra "Nyan"vas Tokyo」への改名企画を行いました。
インタラクティブなコンテンツとして、物理演算を用いて肉球で遊べるウェブページを作成しました。

https://www.orch-canvas.tokyo/nyanvas

今回は、物理演算周りの実装について、知見をまとめていきます。
加速度センサーを用いていますが、そこはクセのある知見となりましたのでまた来週に……

偉大なる先人方の記事も参考になりますので、ぜひ！

https://zenn.dev/chot/articles/21d6d94c314979

https://zenn.dev/sdkfz181tiger/books/5fb7ad6afa3a5e

実際に動作する、完成形のコードは、こちらのブランチにまとまっています。

https://github.com/orchestra-canvas-tokyo/homepage/tree/develop/2025-04-01

---

## 要件定義

次のような要件で実装を進めました。

- 背景に物理演算空間を用意する
- 重力の効いた肉球を配置する

これではややイメージが掴みにくいかもしれませんので、実際のページを再掲しておきます。

https://www.orch-canvas.tokyo/nyanvas

## 物理演算空間の準備

今回は、物理演算を管理したクラスを用意しました。
基本的な物理演算空間の用意まで一気に実装してしまいます。

```ts:呼び出し元
const width = window.innerWidth;
const height = window.innerHeight;

const pawEngine = new PawEngine(document.body, [width, height]);
```

```ts:PawEngine.ts
export class PawEngine {
    private engine: Matter.Engine;
    private render: Matter.Render;
    private boxBodies: Matter.Body[] = [];
    private paws: Matter.Body[] = [];

    constructor(element: HTMLElement, [width, height]: [number, number]) {
        this.engine = Matter.Engine.create();
        this.render = Matter.Render.create({
            element: element,
            engine: this.engine,
            options: {
                width,
                height,
                hasBounds: true,
                background: '',
                wireframes: false
            }
        });

        // 箱（壁の組み合わせ）を用意する
        this.updateBox(width, height);

        // おまじない
        Matter.Render.run(this.render);
        Matter.Runner.run(Matter.Runner.create(), this.engine);
        Matter.World.add(this.engine.world, mouseConstraint)
    }

    updateBox(width: number, height: number) {
        if (0 < this.boxBodies.length) Matter.Composite.remove(this.engine.world, this.boxBodies);

        const thickness = 9999;
        this.boxBodies = [
            // 天井
            Matter.Bodies.rectangle(width / 2, -thickness / 2, width, thickness, {
                isStatic: true
            }),

            // 床
            Matter.Bodies.rectangle(width / 2, height + thickness / 2, width, thickness, {
                isStatic: true
            }),

            // 壁
            Matter.Bodies.rectangle(-thickness / 2 - 1, height / 2, thickness, height, {
                isStatic: true
            }),
            Matter.Bodies.rectangle(width + thickness / 2, height / 2, thickness, height, {
                isStatic: true
            })
        ];

        Matter.Composite.add(this.engine.world, this.boxBodies);
    }
}
```

これに味付けをしていきます。
肉球に関する処理は、後ほど章だてします。

### 高DPIディスプレイ対応

`window.devicePixelRatio`を用いることで、CSSピクセル解像度に対するディスプレイの物理的なピクセル解像度の比率を取得できます。
ズーム時のSafari系で変更が反映されないというバグがあるようですが、ほとんどの環境・条件で問題なく取得できます。

https://developer.mozilla.org/ja/docs/Web/API/Window/devicePixelRatio

Matter.js側では、Renderの初期化時の引数で比率を設定できます。

```ts:呼び出し元
const width = window.innerWidth;
const height = window.innerHeight;
const pixelRatio = window.devicePixelRatio;

const pawEngine = new PawEngine(document.body, [width, height], pixelRatio);
```

```diff ts:PawEngine.ts
-constructor(element: HTMLElement, [width, height]: [number, number]) {
+constructor(element: HTMLElement, [width, height]: [number, number], pixelRatio: number) {
    this.engine = Matter.Engine.create();
    this.render = Matter.Render.create({
        element: element,
        engine: this.engine,
        options: {
            width,
            height,
            hasBounds: true,
            background: '',
+           pixelRatio,
            wireframes: false
        }
    });
```

### 画面リサイズ対応

今回作成したWebサイトでは、空間いっぱいに背景として広がる、物理演算空間を用意しました。
画面のリサイズ（ウィンドウサイズ変更、画面の回転、アドレスバーの引っ込み）で物理演算空間のサイズを変更する必要があります。

先のクラスに追加実装していきます。

```ts:呼び出し元
window.addEventListener('resize', () => {
    const newWidth = window.innerWidth;
    const newHeight = window.innerHeight;
    
    pawEngine.resize(newWidth, newHeight);
})
```

```ts:PawEngine.ts
resize(width: number, height: number) {
    // 周囲の壁を更新
    this.updateBox(width, height);

    // 物理エンジンのサイズを変更
    this.render.bounds.max.x = width;
    this.render.bounds.max.y = height;

    // 描画キャンバスのサイズを変更
    this.render.canvas.style.width = `${width.toString()}px`;
    this.render.canvas.style.height = `${height.toString()}px`;
}
```

### ページ遷移時の後片付け

今回は、Svelte＋SvelteKitで実装されたホームページに追加実装しました。
エイプリルフール期間中はあらゆるページの背景に表示するため、`+layout.svelte`上に実装していきました。

しかし、そのままではページ遷移時に前のキャンバスが残ってしまいます。
適切に後片付けを行います。

```ts:+layout.svelte
import { beforeNavigate } from '$app/navigation';

beforeNavigate(() => {
    pawEngine.destroy();
});
```

```ts:PawEngine.ts
destroy() {
    this.render.canvas.remove();
}
```

## 肉球（円）の生成

まずは、お決まりの基本的な実装コードを追加します。
今回は、面白みとして肉球のテクスチャにランダム要素を追加していきます。

```ts:PawEngine.ts
import paw from './paws/paw.png';
import pawNikukyu from './paws/paw-nikukyu.png';
import pawGold from './paws/paw-gold.png';

static getPawTexture(): string {
    const value = Math.random();

    // 5%の確率で金色を
    // 10%の確率で肉球色を
    if (value < 0.05) return pawGold;
    if (value < 0.05 + 0.1) return pawNikukyu;
    return paw;
}

async addPaw(x: number, y: number) {
    const circleSize = 30;
    const imageSize = 1000;
    const mass = 75;

    const scale = circleSize / (imageSize / 2);
    const paw = Matter.Bodies.circle(x, y, circleSize, {
        mass,
        render: {
            sprite: {
                texture: PawEngine.getPawTexture(),
                xScale: scale,
                yScale: scale
            }
        }
    });

    Matter.World.add(this.engine.world, paw);
    this.paws.push(paw);
}
```

この後、小気味よく動作するような微調整を加えていきます。

### 跳ねるようにする

反発係数`restitution`を、初期化時などに設定することで可能になります。
一例として、肉球（円）に追加するコードをお示しします。

```diff ts:PawEngine.ts
+private restitution = 1;

async addPaw(x: number, y: number) {
    const circleSize = 30;
    const imageSize = 1000;
    const mass = 75;

    const scale = circleSize / (imageSize / 2);
    const paw = Matter.Bodies.circle(x, y, circleSize, {
        mass,
+       restitution: this.restitution,
        render: {
            sprite: {
                texture: PawEngine.getPawTexture(),
                xScale: scale,
                yScale: scale
            }
        }
    });

    Matter.World.add(this.engine.world, paw);
    this.paws.push(paw);
}
```

### テクスチャ画像の対応形式

ソースコードを覗くに、`.jpg`、`.gif`、`.gif`にのみ対応しているようです。

https://github.com/liabru/matter-js/blob/acb99b6f8784c809b940f1d2cf745427e088e088/src/render/Render.js#L1534-L1543

WebPやAVIFが対応していないのはまだしも、`.jpeg`も対応していない点には注意が必要そうです。

### クリックで追加・消去する

エイプリルフール中は、ホームページ上のあらゆるページに肉球が出現します。
そのため、UXの低下を最低限とするために、肉球をクリックで消去する機能を追加することにします。

反対に、肉球以外をクリックした場合は、肉球が増殖することにしてみます。

Matter.jsにはクリックイベントがあるので、それを利用することとします。

```diff ts:PawEngine.ts
constructor(element: HTMLElement, [width, height]: [number, number], pixelRatio: number) {
    this.engine = Matter.Engine.create();
    this.render = Matter.Render.create({
        element: element,
        engine: this.engine,
        options: {
            width,
            height,
            hasBounds: true,
            background: '',
            wireframes: false
        }
    });

    // 箱（壁の組み合わせ）を用意する
    this.updateBox(width, height);

    // おまじない
    Matter.Render.run(this.render);
    Matter.Runner.run(Matter.Runner.create(), this.engine);
    Matter.World.add(this.engine.world, mouseConstraint)

+    // クリックイベントを作成する
+    const mouse = Matter.Mouse.create(this.render.canvas);
+    const mouseConstraint = Matter.MouseConstraint.create(this.engine, {
+        mouse: mouse,
+        constraint: { stiffness: 0.2, render: { visible: false } }
+    });
}

+onClick(x: number, y: number) {
+    let pawRemoveFlag = false;
+    this.paws.forEach((paw, index) => {
+        const dx = paw.position.x - x;
+        const dy = paw.position.y - y;
+        const distance = Math.sqrt(dx * dx + dy * dy);
+
+        // 型ガード的な処理
+        if (!paw?.circleRadius) return;
+
+        if (distance < paw.circleRadius) {
+            // クリック範囲が肉球内
+            Matter.World.remove(this.engine.world, paw);
+            this.paws.splice(index, 1);
+            pawRemoveFlag = true;
+        }
+    });
+
+    // 肉球外のクリックである
+    if (!pawRemoveFlag) this.addPaw(x, y);
+}
```

### 画面サイズ変更時も画面内に留まるように

画面の向き変更など、多きなサイズ変更時に肉球が画面外へ飛んでいってしまいます。
これでは寂しいので、画面内に移動するような処理を追加していきます。

```diff ts:PawEngine.ts
resize(width: number, height: number) {
+    // 肉球の位置を更新
+    this.paws.forEach((paw) => {
+        const x = paw.position.x;
+        const y = paw.position.y;
+
+        const newX = (x / this.render.bounds.max.x) * width;
+        const newY = (y / this.render.bounds.max.y) * height;
+
+        Matter.Body.setPosition(paw, { x: newX, y: newY });
+
+        // 肉球に慣性のような速度を与える
+        const velocityCoefficient = 0.01;
+        Matter.Body.setVelocity(paw, {
+            x: velocityCoefficient * (newX - x),
+            y: velocityCoefficient * (newY - y)
+        });
+    });
+
    // 周囲の壁を更新
    this.updateBox(width, height);

    // 物理エンジンのサイズを変更
    this.render.bounds.max.x = width;
    this.render.bounds.max.y = height;

    // 描画キャンバスのサイズを変更
    this.render.canvas.style.width = `${width.toString()}px`;
    this.render.canvas.style.height = `${height.toString()}px`;
}
```

---

## おわりに

ここまでお読みいただきありがとうございました！

[実際のページ](https://www.orch-canvas.tokyo/nyanvas)をいじっていただくとお分かりかと思いますが、Matter.jsは軽量です。
ちょっとしたインタラクションとして、有用なライブラリではないでしょうか！

一方で、少し古いのか[ドキュメント](https://brm.io/matter-js/)と実装が食い違っている部分があります。
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
> **第16回定期演奏会**
>
> 2026年3月15日(日)
> 府中の森芸術劇場 どりーむホール
> バーンスタイン / 「ウェストサイドストーリー」よりシンフォニックダンス ほか
>
> 詳細は[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)にて

<!-- end long upcoming concert announcement -->

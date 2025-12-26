---
title: "【Web API】加速度センサーの情報を取得する 2025年版【DeviceMotionEvent】"
emoji: "🪃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "typescript", "svelte", "sveltekit"]
publication_name: "orch_canvas"
published: true
published_at: 2025-04-14 06:00
---

## まとめ

- 加速度センサーの情報はDeviceMotionEventインターフェースを介して取得できる
- iOS系では、`requestPermission()`を呼び出す必要がある
- iOS系では、さらに各加速度の正負が反転している

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

前回の記事では、物理演算周りの実装についてまとめました。

https://zenn.dev/orch_canvas/articles/matterjs-resize-dispose

今回は、加速度センサーに関する実装について、知見をまとめていきます！

---

## 加速度センターの値を取得する

`accelerationIncludingGravity`プロパティを使うことで、取得できます。
デバイスが重力の影響を除くことに対応している場合、`acceleration`プロパティを使うことも可能です。

また、Windowインターフェースに用意されている、`deviceMotion`イベントを利用することで、変化した値を随時取得できます。

```ts
window.addEventListener('devicemotion', (e: DeviceMotionEvent) => {
    if (
        !e.accelerationIncludingGravity ||
        !e.accelerationIncludingGravity.x ||
        !e.accelerationIncludingGravity.y ||
        !e.accelerationIncludingGravity.z
    )
        return;

    // 重力加速度
    const gx = e.accelerationIncludingGravity.x;
    const gy = e.accelerationIncludingGravity.y;
    const gz = e.accelerationIncludingGravity.z;

    const acceleration = [gx, gy, gz];

    doSomething(acceleration)
})
```

### 画面の描画方向との整合性をとる

先の方法で取得される加速度は、端末の方角と対応したものです。

具体的には、次のような対応関係になっています。

- X軸：西から東へ
- Y軸：南から北へ
- Z軸：下から上へ

一般的なスマホでいえば、インカメラが北側、ホームバーやナビゲーションメニューが南側にあります。

https://developer.mozilla.org/ja/docs/Web/API/DeviceMotionEvent/accelerationIncludingGravity

この「方角」は、画面の方向では変わりません。
そのため、画面の方向を`window.screen.orientation.type`で取得して、調整する必要があります。

https://developer.mozilla.org/ja/docs/Web/API/ScreenOrientation/type

とある情報筋によると、一部のiPadなどではもう少し込み入った考慮が必要なようですが……
今回は目をつむることにしたいと思います。

https://blog.oimo.io/2023/11/28/devicemotion/

以上の情報をもとに、先のコードを修正します。

```ts
window.addEventListener('devicemotion', (e: DeviceMotionEvent) => {
    if (
        !e.accelerationIncludingGravity ||
        !e.accelerationIncludingGravity.x ||
        !e.accelerationIncludingGravity.y ||
        !e.accelerationIncludingGravity.z
    )
        return;

    // 重力加速度
    const gx = e.accelerationIncludingGravity.x;
    const gy = e.accelerationIncludingGravity.y;
    const gz = e.accelerationIncludingGravity.z;

    let acceleration: [number, number, number]

    // 傾きに応じて重力を調節
    switch (window.screen.orientation.type) {
        case 'landscape-primary':
            // 横長
            acceleration = [gy, gx, gx];
            break;
        case 'landscape-secondary':
            // 横長逆転
            acceleration = [-gy, -gx, gz];
            break;
        case 'portrait-secondary':
            // 縦長逆転
            acceleration = [gx, -gy, gz];
            break;
        default: // case 'portrait-primary'
            // 縦長
            acceleration = [-gx, gy, gz];
            break;
    }

    return acceleration;
})
```

### iOS系では加速度を反転させる

iOS系（iOSやiPadOS）では、仕様と加速度の方向が異なっており、この反転作業を行う必要があります。

次に、iOS系を検出する方法が問題となります。
ここでは、後述する`requestPermission()`メソッドを有する場合をiOS系として判定します。

```ts
window.addEventListener('devicemotion', (e: DeviceMotionEvent) => {
    if (
        !e.accelerationIncludingGravity ||
        !e.accelerationIncludingGravity.x ||
        !e.accelerationIncludingGravity.y ||
        !e.accelerationIncludingGravity.z
    )
        return;

    // 重力加速度
    const gx = e.accelerationIncludingGravity.x;
    const gy = e.accelerationIncludingGravity.y;
    const gz = e.accelerationIncludingGravity.z;

    let acceleration: [number, number, number]

    // 傾きに応じて重力を調節
    switch (window.screen.orientation.type) {
        case 'landscape-primary':
            // 横長
            acceleration = [gy, gx, gx];
            break;
        case 'landscape-secondary':
            // 横長逆転
            acceleration = [-gy, -gx, gz];
            break;
        case 'portrait-secondary':
            // 縦長逆転
            acceleration = [gx, -gy, gz];
            break;
        default: // case 'portrait-primary'
            // 縦長
            acceleration = [-gx, gy, gz];
            break;
    }

    const unknownDeviceMotionEvent = e as unknown;
    const isWithRequestPermission =
        typeof unknownDeviceMotionEvent === 'function' &&
        unknownDeviceMotionEvent !== null &&
        'requestPermission' in unknownDeviceMotionEvent &&
        typeof unknownDeviceMotionEvent.requestPermission === 'function';
    
    const isIOS = isWithRequestPermission;
    if (isIOS)
        acceleration = [-acceleration[0], -acceleration[1], -acceleration[2]];

    return acceleration
})
```

`acceleration.map(a => -a)`と実装すると、正しく`[number, number, number]`に型推論されないため、上記のような実装としています。

## iOS系でセンサー仕様の許可を得る

ユーザーのインタラクション（ボタンのクリックイベントなど）をもとに、`DeviceMotionEvent.requestPermission()`を呼び出す必要があります。

加速度センターはなかなかの情報量を有するので、こうなるのは仕方ないですね。

### 仕様の整理

`requestPermission()`メソッドの仕様を整理します。
主な仕様は次の通りです。

- iOS系で用意されている静的メソッド
  - その他のブラウザでは用意されておらず、TypeScriptでは型定義もされていない
- `Promise<'granted' | 'denied'>`を返す
  - ユーザーが許諾したら`granted`、拒否したら`denied`を返す
- ユーザのインタラクションなしに呼び出した場合
  - すでに許諾されている場合（最近許諾されているなど？）、`granted`を返す
  - 許諾されていない場合、rejectされる

### 実装

今回は、次のようなコードで実装しました。
Svelte(Kit)で実装しています。

特に、`requestPermission()`メソッドはiOS系以外には実装されておらず、TypeScriptの型定義上も用意されていません。
any型にキャストしてしまうのも手ですが、今回は丁寧に記述してみました。

```ts:DeviceMotionController.ts
export class DeviceMotionController {
    readonly isWithRequestPermission: boolean;
    readonly updatePermissionStatusCallback: (permitted: boolean) => void;

    /**
     * コンストラクタ
     * @param updatePermissionStatusCallback
     * 加速度センサーにアクセスするためのパーミッション状態が変更されたときに呼び出されるコールバック。
     * このコールバックは、`DeviceMotionController` のパーミッション状態が変更されたときに呼び出されます。
     */
    constructor(updatePermissionStatusCallback: typeof this.updatePermissionStatusCallback) {
        this.updatePermissionStatusCallback = updatePermissionStatusCallback;

        const unknownDeviceMotionEvent = window.DeviceMotionEvent as unknown;
        this.isWithRequestPermission =
            typeof unknownDeviceMotionEvent === 'function' &&
            unknownDeviceMotionEvent !== null &&
            'requestPermission' in unknownDeviceMotionEvent &&
            typeof unknownDeviceMotionEvent.requestPermission === 'function';
    }

    /**
     * 加速度センサーにアクセスするためのパーミッションを要求します。
     * iOS では、`DeviceMotionEvent.requestPermission()` を呼び出して
     * パーミッション状態を変更することができます。
     */
    requestPermission(): void {
        const deviceMotionEvent = window.DeviceMotionEvent as unknown;

        // 型定義、型ガード関数を用意する
        type deviceMotionEventWithRequestPermission = typeof window.DeviceMotionEvent & {
            requestPermission: () => Promise<'granted' | 'denied'>;
        };

        const isDeviceMotionEventWithRequestPermission = (
            unknownDeviceMotionEvent: unknown
        ): unknownDeviceMotionEvent is deviceMotionEventWithRequestPermission =>
            this.isWithRequestPermission;

        if (!isDeviceMotionEventWithRequestPermission(deviceMotionEvent)) {
            // 許可取得が不要な環境
            return;
        }

        unknownDeviceMotionEvent
            .requestPermission()
            .then((permissionState) => {
                this.updatePermissionStatusCallback(permissionState === 'granted');
            })
            .catch(() => {
                // ユーザーのインタラクションに依らない許可申請
                // かつ、許可されていない場合
                this.updatePermissionStatusCallback(false);
            });
    }
}
```

```html:+page.svelte
<script lang="ts">
    import { afterNavigate } from '$app/navigation';
    import { DeviceMotionController } from './nyanvas/DeviceMotionController';

    let deviceMotionController: DeviceMotionController | null = null;
    let showPermissionToast = $state(false);

    afterNavigate(() => {
        const updatePermissionStatusCallback = (permitted: boolean) => {
            showPermissionToast = !permitted;
        };
        deviceMotionController = new DeviceMotionController(updatePermissionStatusCallback);

        // 一回呼び出しし、ユーザーの許可状態を確認する
        deviceMotionController.requestPermission();
    });

    const onclickGrantPermission = () => {
        if (!deviceMotionController) return;
        deviceMotionController.requestPermission();
        showPermissionToast = false;
    };
</script>

<div class="toast" class:show={showPermissionToast}>
    <p>ぜひ、加速度センサー付きでご覧ください！</p>
    <button onclick={onclickGrantPermission}>進む</button>
</div>

<style>
    /* 省略 */
</style>
```

---

## おわりに

ここまでお読みいただきありがとうございました！

やはり、「目に見えて動くものを作る」というのはプログラミング欲の原点たる悦びがありますね。

環境間の実装差異をコードで吸収する作業も、IE時代を思い出してちょっとエモかったですね～
……これぐらいの実装差異で済んでいるからかもしれませんが。

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
> [![](/images/regular-16-flyer.jpg =250x)](https://www.orch-canvas.tokyo/concerts/regular-16)
>
> 詳細は[チケット販売ページ](https://teket.jp/1776/59938?uid=zenn)にて

<!-- end long upcoming concert announcement -->

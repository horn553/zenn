---
title: "asideでGAS開発を快適にする（モックでテストもあるよ）"
emoji: "⛽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GAS", "aside", "clasp"]
published: true
---

快適な GAS コーディングをサポートする aside を実際に使ってみました。
初期設定やテスト用モックの準備で何箇所か沼ってしまったので、知見を共有したいと思います。

## aside についてくるもの

- clasp (Typescript)
- EditorConfig
- ESLint
- Prettier
- Jest
- 開発環境、本番環境へのデプロイ

便利！

:::message
執筆現在、aside で新規プロジェクトを作成する場合、スプレッドシート付きのスクリプトが作成されます。

具体的には `--type sheets` オプション付きで `npx clasp create` が実行されます。
:::

:::message alert
aside は Apache-2.0 ライセンスです。
:::

## aside のセットアップ

### Script API を有効化

clasp のお作法に従い、[https://script.google.com/home/usersettings](https://script.google.com/home/usersettings) から Google Apps Script API を有効化しておく必要があります。

有効化せず aside を利用した場合、`Error: ENOENT: no such file or directory, lstat 'dist/.clasp.json'` と非特異的なエラーが出て沼ります。

### aside を実行

```shell
npx @google/aside init
```

clasp 未セットアップ環境での初回実行時には `clasp login` が実行されます。

## セットアップされるもの

### 開発環境

- EditorConfig
- ESLint
- Prettier
- Jest

必要に応じて VSCode などの拡張機能を入れるとより幸せです。

### 便利なスクリプト

`package.json` にいくつかのスクリプトが登録されています。
特に、次のコマンドは普段の開発や CI/CD で便利そうです 👀

```shell
npm run lint
npm run test
npm run deploy
npm run deploy:prod
```

## モックしてテストする

次の関数のテストを作成してみます 👀

```ts:spreadsheet-utils.ts
/**
 * シートを取得する。
 * 該当のシートがない場合、シートを作成する。
 *
 * @param name - シートの名称
 * @returns 取得または作成したシート
 */
export function getSheet(name: string): GoogleAppsScript.Spreadsheet.Sheet {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = spreadsheet.getSheetByName(name);
  if (sheet !== null) return sheet;
  return spreadsheet.insertSheet().setName(name);
}
```

### グローバルオブジェクト `SpreadsheetApp` を用意する

```json:jest.config.json (抜粋)
"globals": {
  "SpreadsheetApp": {}
}
```

### spreadsheet、sheet のモックを作成する

`GoogleAppsScript.Spreadsheet.Sheet` はインターフェースです。
インターフェースからモックを作成するのに、今回は [jest-mock-extended](https://www.npmjs.com/package/jest-mock-extended) を使用します。

```shell
npm install jest-mock-extended
```

```ts:spreadsheet-utils.test.ts (抜粋)
const spreadsheet: MockProxy<GoogleAppsScript.Spreadsheet.Spreadsheet> =
  mock<GoogleAppsScript.Spreadsheet.Spreadsheet>();
const sheet: MockProxy<GoogleAppsScript.Spreadsheet.Sheet> =
  mock<GoogleAppsScript.Spreadsheet.Sheet>();
```

呼び出す関数の返り値をモックしていきます。
次に示すのは、「シートが存在する場合」のモックです。

```ts:spreadsheet-utils.test.ts (抜粋)
spreadsheet.getSheetByName.mockReturnValue(sheet);
SpreadsheetApp.getActiveSpreadsheet = jest
  .fn()
  .mockReturnValue(spreadsheet);
```

### 呼び出し、評価

```ts:spreadsheet-utils.test.ts (抜粋)
const sheetName = '存在するシート';
expect(getSheet(sheetName)).toBe(sheet);
expect(spreadsheet.getSheetByName).toBeCalledWith(sheetName);
```

### コード全体

存在しないシート名を指定した場合を含めた、テストの記述例です。

```ts:spreadsheet-utils.test.ts
import { mock, MockProxy } from 'jest-mock-extended';

describe('example-module', () => {
  describe('getSheet', () => {
    it('存在するシートを指定する', () => {
      const spreadsheet: MockProxy<GoogleAppsScript.Spreadsheet.Spreadsheet> =
        mock<GoogleAppsScript.Spreadsheet.Spreadsheet>();
      const sheet: MockProxy<GoogleAppsScript.Spreadsheet.Sheet> =
        mock<GoogleAppsScript.Spreadsheet.Sheet>();

      spreadsheet.getSheetByName.mockReturnValue(sheet);
      SpreadsheetApp.getActiveSpreadsheet = jest
        .fn()
        .mockReturnValue(spreadsheet);

      const sheetName = '存在するシート';
      expect(getSheet(sheetName)).toBe(sheet);
      expect(spreadsheet.getSheetByName).toBeCalledWith(sheetName);
    });

    it('存在しないシートを指定する', () => {
      const spreadsheet: MockProxy<GoogleAppsScript.Spreadsheet.Spreadsheet> =
        mock<GoogleAppsScript.Spreadsheet.Spreadsheet>();
      const sheet: MockProxy<GoogleAppsScript.Spreadsheet.Sheet> =
        mock<GoogleAppsScript.Spreadsheet.Sheet>();

      spreadsheet.getSheetByName.mockReturnValue(null);
      spreadsheet.insertSheet.mockReturnValue(sheet);
      sheet.setName.mockReturnValue(sheet);
      SpreadsheetApp.getActiveSpreadsheet = jest
        .fn()
        .mockReturnValue(spreadsheet);

      const sheetName = '存在しないシート';
      expect(getSheet(sheetName)).toBe(sheet);
      expect(spreadsheet.insertSheet).toBeCalledTimes(1);
      expect(sheet.setName).toBeCalledWith(sheetName);
    });
  });
});
```

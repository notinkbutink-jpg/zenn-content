---
title: "GASのベストプラクティスを調べてみた"
emoji: "💡"
type: "tech"
topics: [GoogleAppsScript,gas,ベストプラクティス]
published: true
---
## はじめに
:::message
**※この記事について**
本記事は、Google Apps Script(GAS)のベストプラクティスについてGemini（AI）と壁打ちした内容をベースに執筆しています。
AIの回答に含まれる技術情報については、**筆者が公式ドキュメントを参照して裏付けを取り、誤りがないことを確認した上で**構成しています。参考にしたドキュメントは末尾に記載します。
:::

📝 元記事について
本記事は、2025年12月17日にQiitaに投稿した内容を、Zennへの拠点移行に伴い写したものです。

* ベストプラクティスを新しく見かけたら追記します。
* GASの構成はGoogle スプレッドシート + Google App Scriptを想定します。

## ベストプラクティス

### プロジェクト名
Googleドライブ上で見える名前です。「何のための」「どの環境の」スクリプトかが一目で分かるようにします。

#### 推奨フォーマット
`【環境】[システム名]_[機能概要]`

* **環境**: 本番以外の場合に付ける（例: `DEV`, `TEST`）
* **システム名**: どの業務/シートに紐づくか（例: `Attendance`, `Invoice`）
* **機能概要**: 何をするものか（例: `DailyReport`, `TextProcessor`）

#### 具体例
* `Kintai_ShiftManager_PROD` （勤怠管理システムのシフト管理・本番用）
* `【DEV】Text_Converter_Batch` （開発中のテキスト変換バッチ）

> ※開発環境には【DEV】などの隅付き括弧や絵文字（🚧）を先頭につけると、本番環境と誤って実行する事故を防げます。

### スクリプトファイル名（.gsファイル）
エディタ左側に並ぶファイル名です。GASはプロジェクト内に複数の `.gs` ファイルを作れますが、実行時にすべてロードされ、単一のグローバルスコープを共有します。そのため、ファイルの分け方は「人間の管理しやすさ」のためだけに行います。

GASにはフォルダがないため、アンダースコアを使って擬似的に階層構造を作ります。また、ファイルはアルファベット順に並ぶため、先頭に番号を振るのも有効です。

#### 構成例
ファイルを役割ごとに分類し、接頭辞を統一します。
`Utils` や `Helper` といった名前で、「部品（汎用関数）」を切り出す癖をつけると、コードが読みやすくなります。

| ファイル名 | 役割・中身 |
| :--- | :--- |
| `00_Config.gs` | 定数、APIキー、シートID、設定値など。 |
| `01_Main.gs` | トリガー実行されるメイン関数や、`onOpen` 関数などエントリーポイント。 |
| `10_SheetUtils.gs` | スプレッドシート操作の汎用関数群（読み込み、書き込みなど）。 |
| `11_DriveUtils.gs` | Googleドライブ操作の汎用関数群。 |
| `20_API_Slack.gs` | 外部API（Slackなど）との通信ロジック。 |
| `Logic_XXXX.gs` | 特定のビジネスロジック |
| `Test_Unit.gs` | テスト用の関数群。 |

### 実装

#### API呼び出し
APIの呼び出し回数は極限まで減らします。GASで最も処理時間を食うのは、Googleのサーバー間通信（スプレッドシートの読み書きなど）です。ループ内でAPIを呼ぶのはアンチパターンです。

**NGパターン**
```javascript
// 1000回API通信が発生
for (let i = 0; i < 1000; i++) {
  sheet.getRange(i + 1, 1).setValue("data");
}
```

**OKパターン**
```JavaScript
// API通信は「読み込み1回」「書き込み1回」だけ
const values = sheet.getRange("A1:A1000").getValues(); // 二次元配列で取得
// ... JSのメモリ上で処理 ...
sheet.getRange("A1:A1000").setValues(newValues); // 一括書き込み
```

#### 変数宣言
2025年12月現在はV8ランタイムが標準[1]ですので、レガシーな var は禁止とし、ブロックスコープを意識します。
* 基本: ```const``` （再代入しない変数に用いる）
* 例外: ```let```（ループカウンタや再代入が必要な場合のみ）
* 禁止: ```var``` （予期せぬスコープ汚染を防ぐため）

#### 関数の命名規則
一般的なJavaScriptの記法に従いますが、```.gs``` ファイル内の関数はすべてグローバルに公開され、エディタの「実行」プルダウンメニューに表示されてしまいます。これを防ぐ規約が重要です。
* **パブリック関数（トリガーやボタンから呼ぶもの）**
  * 通常の ```camelCase```
  * **動詞 + 名詞** の形にする（何を・どうするか明確にする）
  * 例: ```main(), sendDailyReport()```

* **プライベート関数（ロジック、ヘルパー）**
  * **末尾にアンダースコア ```_``` を付ける**
  * 「GASのエディタの実行メニュー」や「スプレッドシートのカスタム関数候補」に表示されなくなる
  * 他の関数から呼ばれるだけの部品につけると良い
  * 例: ```getDataFromSheet_(), calculateTotal_()```

#### ハードコーディングの禁止
ID、トークン、Webhook URLなどをコードに直書きするのは、セキュリティ的にも保守的にもNGです。
* **推奨:** PropertiesService.getScriptProperties() を使用
* **設定場所:** エディタの「プロジェクトの設定（歯車アイコン）」→「スクリプト プロパティ」でGUIから設定できます。

```JavaScript
// NG
const SHEET_ID = "1A2b3C..."; 


// OK
const SHEET_ID = PropertiesService.getScriptProperties().getProperty("TARGET_SHEET_ID");
```

#### クラス構文の使用
複雑なロジックの場合、グローバル関数が散らばると管理不能になります。ES6の ```class``` 構文を使ってカプセル化することを強く推奨します。
```JavaScript
// User.gs
class User {
  constructor(email) {
    this.email = email;
    this.sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Users');
  }


  /**
   * ユーザーが存在するか確認する
   * @return {boolean}
   */
  exists() {
    // ...ロジック...
  }
}


// main.gs
function main() {
  const user = new User('test@example.com');
  if (user.exists()) {
    // ...
  }
}
```

#### ドキュメンテーション
JSDoc（```/** ... */```）を書くと、自作関数でもオートコンプリート時に型情報や説明が表示されます。
* 特に重要なタグ
  * ```@param {Type} name:``` 引数の型
  * ```@return {Type}:``` 戻り値の型
  * ```@customfunction:``` スプレッドシートのセルで使う自作関数に必須。このタグをつけると、予測変換に関数が出現する。JSDocの最後につけることに注意。

```JavaScript
/**
 * 指定したシートからデータを取得し、オブジェクト配列に変換します。
 * * @param {string} sheetName - 対象のシート名
 * @return {Object[]} ヘッダーをキーとした連想配列のリスト
 */
function getSheetData_(sheetName) { ... }
```

### 参考記事・関連記事
[1] Google Workspace > App Script > ガイド > V8 ランタイムの概要

https://developers.google.com/apps-script/guides/v8-runtime?hl=ja

[2] GASでSpreadsheetを操作する自分的ベストプラクティス

https://qiita.com/ryan5500/items/e72eb205fbe006c2eb6f

[3] GAS(Google Apps Script)安全な日次トリガー管理のベストプラクティス

https://qiita.com/Hide331/items/438855278177efd09d28

[4] 現実世界でGASを使うためのTips

https://zenn.dev/phpmyadmin/books/49bd37e6de7e31/viewer/b041a4

[5]【GAS】GASのファイル構成ベストプラクティス

https://eileben.com/%e3%80%90gas%e3%80%91gas%e3%81%ae%e3%83%95%e3%82%a1%e3%82%a4%e3%83%ab%e6%a7%8b%e6%88%90%e3%83%99%e3%82%b9%e3%83%88%e3%83%97%e3%83%a9%e3%82%af%e3%83%86%e3%82%a3%e3%82%b9/


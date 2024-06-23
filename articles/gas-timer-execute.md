---
title: "GASからいつも決まった時間に何かを動かしたい人へ"
emoji: "⛽️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GAS"]
published: false
publication_name: "spacemarket"
---

なんか最近GASばっかり書いている気がします。
気のせいかもしれません。

## 結論
とりあえず以下のコード＋設定を用意すれば動きます！！！


```js
/**
 * トリガーを設定するまでの何らかの処理
 */
const setupTrigger = () => {
  // ここら辺にトリガーセットのための条件判定が必要であれば何かしらの処理を書く

  // 例えば当日の19時に実行するならこのように書く
  const today = new Date()
  today.setHours(19)
  today.setMinute(0)
  createTrigger('main', today)
}

/**
 * トリガーの設定を行う
 * @params targetFunctionName {string}
 * @params triggerDate {Date}
 */
const createTrigger = (targetFunctionName, triggerDate) => {
  ScriptApp.newTrigger(targetFunctionName).timeBased().at(triggerDate).create()
}

const main = () => {
  // 19時に動いたら何かしたい処理をここに書く
}

/**
 * 削除対象外のトリガー名
 */
const excludeDeleteTriggerName = [
  'setupTrigger',
  'deleteEndTrigger',
  // ここに削除対象のトリガー名を入れる
]
/**
 * 無効になったトリガーを削除するスクリプト
 */
const deleteEndTrigger = () => {
  const triggers = ScriptApp.getProjectTriggers()
  triggers.forEach(v => {
    const handlerName = v.getHandlerFunction()
    if (excludeDeleteTriggerName.includes(handleName)) {
      return
    }
    ScriptApp.deleteTrigger(v)
  })
}
```

上記のコードを書いた後にトリガーを設定します。
例えば毎週月曜日の19時ぴったりに動かしたいのであれば18時-19時より前の時刻を選択すればOKです。

![](https://storage.googleapis.com/zenn-user-upload/f54b43c94a3a-20240623.png)

## はじめに

GASって便利ですよね、下手に実行環境を用意しなくてもいろんなことができます。

弊社ではスプレッドシートで毎週の当番を決めて、その当番表をもとに毎週月曜日にSlackに通知を飛ばすようなことをしています。
他にも日や週によって異なる朝会や定例のファシリテーターを、Slack通知したりチームメンションを飛ばしたりもしています。

が、こう言うことをやるにあたり厄介なこともあります。

「朝会は毎朝11時スタートだから10分前に通知して欲しい」

みたいなことができません。
GASは正確にXX時XX分に実行して欲しいトリガーを設定することはできますが、それを繰り返し登録することができません。
例えば週ベースで毎週月曜日に実行するトリガーを設定しようとすると1時間単位でおおよその時間は決められますが、特定の「時」「分」に実行するみたいなことはできません。

![週ベースのトリガー](https://storage.googleapis.com/zenn-user-upload/a403f81df14c-20240623.png)


「時」「分」まで登録するなら時間指定のトリガーになります。
しかしこれは毎週繰り返しみたいなことができません。
そのため自分でトリガー登録をする手間が増えます。

![時間指定のトリガー](https://storage.googleapis.com/zenn-user-upload/872de94e0c4d-20240623.png)

そんなのめんどくさい！！！！！！！

ということで今回はこれらの課題を解決し、快適なトリガーライフを送っていくことをゴールとします。

## コードの解説
結論でコードは上げていますので、細かく実装を見ていきましょう。

### 定期的に動くトリガーセットアップ関数

まずは「アバウトな時間」に動く処理を書いていきます。
例えば月曜日の0時〜1時に動く処理です。

ここで行うことは以下の2点です。

- 決まった時間に動くトリガーをセットするかどうかの判定（※不要な場合もある）
- 時間指定のトリガーのセット

基本的にはGAS特有の処理はなく、一般的な処理しかないので説明は割愛します。
条件判定が必要な場合はよしなに記載していってください。
```js
/**
 * トリガーを設定するまでの何らかの処理
 */
const setupTrigger = () => {
  // ここら辺にトリガーセットのための条件判定が必要であれば何かしらの処理を書く

  // 例えば当日の19時に実行するならこのように書く
  const today = new Date()
  today.setHours(19)
  today.setMinute(0)
  createTrigger('main', today)
}
```

さて、そうしたら次はトリガーの設定を行います。
GASには「ScriptApp.newTrigger」で新たなトリガーを設定できるようになります。

めっちゃ簡単！！！

なのでここで時間指定のトリガーを設定します。
timeBasedを呼び出しそのトリガーに対してatで時間を設定し、最終的にcreateすることでトリガーが作成されます。


```js
/**
 * トリガーの設定を行う
 * @params targetFunctionName {string}
 * @params triggerDate {Date}
 */
const createTrigger = (targetFunctionName, triggerDate) => {
  ScriptApp.newTrigger(targetFunctionName).timeBased().at(triggerDate).create()
}
const main = () => {
  // 19時に動いたら何かしたい処理をここに書く
}
```

また、newTrigger関数を呼び出す場合は、トリガーによって呼び出される関数名を文字列で指定します。
GUIで操作した場合の「実行する関数を選択」と同じことですね。
今回はmain関数を指定しましたが、任意の関数を指定してください。
![](https://storage.googleapis.com/zenn-user-upload/aa95da849c33-20240623.png)

はい！！！これで完成です！！！

### トリガーを掃除する

ごめんなさい、嘘です。

これだけだと実は不十分です。
正確に言うとある日突然トリガーが動かなくなります。

どういうことかというとGASのトリガーは設定数に上限があるためです。
トリガーの場合、一般ユーザーおよびWorkspaceアカウントどちらでも20件までです。

https://developers.google.com/apps-script/guides/services/quotas?hl=ja#current_limitations

つまり、毎週の定期的なトリガーに加えて時間指定のトリガーが増えていくのでこの制限に引っ掛かります。
実行済みのトリガーが勝手に削除されるということはなく、スクリプトで設定したトリガーは残り続けることに注意してください。

そのため、定期的に実行済みのトリガーを消してあげる必要が出てきました。
当然手で消してもいいのですがそんな面倒なことをしたくないので、こちらも定期的にトリガーを掃除してもらいましょう。

```js
/**
 * 削除対象外のトリガー名
 */
const excludeDeleteTriggerName = [
  'setupTrigger',
  'deleteEndTrigger'
  // ここに削除対象のトリガー名を入れる
]
/**
 * 無効になったトリガーを削除するスクリプト
 */
const deleteEndTrigger = () => {
  const triggers = ScriptApp.getProjectTriggers()
  triggers.forEach(v => {
    const handlerName = v.getHandlerFunction()
    if (excludeDeleteTriggerName.includes(handleName)) {
      return
    }
    ScriptApp.deleteTrigger(v)
  })
}
```

excludeDeleteTriggerNameでは除外対象の関数名を入れています。
例えばsetupTriggerは毎週実行され、新たなトリガーを作る関数なのでこのトリガーは消しては困るので対象外としています。
またこの削除関数自体もこれから定期的なトリガーを設定するので除外します。

deleteEndTriggerではプロジェクト内で設定している全てのトリガーを取得し、設定されている実行関数名を基準にトリガーを削除していきます。
getProjectTriggersで全トリガー取得し、deleteTriggerで引数に渡したトリガーを削除できると言うことですね。
これでどんどん増えていくトリガーにも対応できます。

コードの実装が終わったらdeleteTriggerも定期的なトリガーとして登録しておきましょう。
業務で使うのであれば例えば毎週日曜日の適当な時間帯に動かすでもいいと思います。

## おわりに
これで快適なGASのトリガーライフが送れるようになりました。
Slack通知以外にも定期的なバッチのように動かしたりもできるのでぜひ色々試してみてください。

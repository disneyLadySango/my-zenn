---
title: "[GAS]子課題単位でバーンダウンチャートが見れない、終わったわ[JIRA]"
emoji: "🌪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GAS", "JIRA"]
published: false
---
進捗はいい（はず）のに、報告できない。

どうもスペースマーケットでエンジニアリングマネージャーをしている人です。
強風でオールバックになっているかどうかは対して気にしていません（？）

## はじめに

弊社ではスクラムのチケット管理に[JIRA](https://www.atlassian.com/ja/software/jira)を使用しています。

https://www.atlassian.com/ja/software/jira

まず用語ですが~~長くて書くのが面倒なので~~

- プロダクトバックログアイテム → PBI
- スプリントバックログアイテム → SBI

という形で省略させてもらいます。

JIRAの運用は下記のようにしています。

- ストーリー（課題）をPBIとして扱う
- PBIに対して子課題を作る形でSBIのチケットを作る
  - この時は概ね1つのチケットが30分で終わるように分解する

PBIにはストーリーポイントが入っています。
これはリファインメントでプランニングポーカーを行いPBIの複雑度に対して相対見積もりを行なっています。
反対にSBIはPBIを達成するためにどういったことが必要かという単位でチケットを作成していきます。
ここではチームで決めた30分（＝1ポモドーロ）という単位でチケットが切られ絶対時間での見積もりになります。
例えば下記のようなイメージです。

- hogeをfugaにする機能
  - hogeをfugaに変換する単体テストを実装する
  - hogeをfugaに変換する関数を実装し適用する
  - リリース

そのため普段のデイリーではプロダクトバックログアイテム単位で、スプリントバックログアイテムの消化状況を確認し上から順番に「このPBIは終われそうか」という観点を入れて進みます。
もし終われない場合は重要度の高いものから終わらせる必要があるので差し替えなどを行うことになっています。
また、スプリントの期間に対してもSBIがどれだけ消化されていてあとどのくらい残っているのか、スプリントの途中でSBIが急に増えたりしていないかなどもチームでウォッチしています。
思ったより進んでないということは「何か困っていること」や「差し込みで何かが起きている」という異変があるはずなのでそこにチームとして気づけるための仕組みとして運用しています。
この確認はバーンダウンチャートを使っています。

さて、ここで問題になってくるのがJIRAのレポート機能です。

JIRAにはレポート機能があります。
バーンダウンチャートを見ることはできますがこれはストーリーポイント単位でのグラフです。
スプリント単位で子課題の消化率を見ることはできません。
（もしかしたらできるのかもしれませんが笑）
しかし当然スプリント中で追いたいものはSBI単位のバーンダウンチャートです。

そのため何かしらの方法でバーンダウンチャートを算出できるようにしなければいけません。

もちろん手作業でもいいです、誰も止める人はいません。
ただ我々はエンジニアです。
そんな面倒なことは自動化させたくなる生き物ですよね？

よし、それでは自動化しちゃいましょう

## 実装

さて実装をしたいのですがどうするのがいいでしょうか。
まずは情報を整理してみましょう。

まずはJIRAで必要な情報を整理します。
どういった情報が必要でどういう使い方をしたいのか書き出してみました。

- 現在走っているスプリントのデータを集計したい
- 毎日のデイリーでタスクの消化状況をグラフで確認できるようにしたい
- プロダクトバックログアイテム単位でタスクのステータスごとで件数が集計されて欲しい
- プロダクトバックログアイテムが終わらなければ翌スプリントにそのまま引き継がれる、前スプリントで終わったものは集計対象から外したい

このような情報が出てきました。
ある1つの点（ここでは日付）で、データを集計する仕組みが必要です。
つまりどこかでデータを永続化する必要がありそうです。
しかしこんなことのためにわざわざデータベースを作るのも面倒です。
ということで楽に作るためにスプレッドシートをデータベースとして使ってしまいましょう。
どうせグラフも出しますし、仰々しいアプリケーションを作る必要もありません。
こっちの方がはるかに楽です。

## JIRAのAPIを利用する

さて、方針は決まりました。
問題はどのようにJIRAにあるデータを取得してくるかです。
実はJIRAではREST APIが公開されていて使用できます。

https://developer.atlassian.com/server/jira/platform/rest-apis/

それではAPIにアクセスするための設定をしていきましょう。

JIRAを開き、右上の自分のアカウントアイコンをクリック。
そこからアカウント管理をクリックします。
その後飛んだページで上部のタブから「セキュリティ」を選択します。
![セキュリティ](https://storage.googleapis.com/zenn-user-upload/e290eaf930a9-20230705.png)

セキュリティの中にAPIトークンという項目があるので下にあるリンクの「APIトークンの作成と管理」をクリックしてください。

![管理](https://storage.googleapis.com/zenn-user-upload/cb51353ef5f8-20230705.png)

トークン作成ボタンがあるのでクリックしましょう。
わかりやすい名前をつけて作成ボタンを押すとトークンが発行されます。

発行されたトークンをコピーしてどこかにメモしておいてください。
最悪無くしたら再発行できます

またこのトークンはユーザーに紐づいているので自分のアカウントのメールアドレスもご用意ください。

### スプレッドシートの用意

GASを描き始める前にまずはスプレッドシートを用意しましょう。
そしたらまずはデータベースという名のシートを作成します。
今回はわかりやすいようにデータベースとしましたが正直名前なんてものはなんでもいいです。
テーブル名っぽい名前の方がいいかもしれませんね。

そうしたらそのシートの1行目にまずはわかりやすいようにデータベースのカラム名を入れましょう。

```txt
日付 / スプリント名 / ID / タイトル / StoryPoint / 合計チケット数 / Done / Doing / ToDo
```

![カラム名](https://storage.googleapis.com/zenn-user-upload/41956ecf9b3f-20230721.png)

今回はこのようにしました。
ここに取得したデータを追加していくだけです。

では次はコードを書いていきましょう。
対象のスプレッドシートへAppScriptを追加してください。

### 認証情報の作成

GASはJavaScriptっぽい感じでかけるのでおそらくそこまで学習コストはかからないと思います。
Docコメントを書いておけばちゃんと型が認識されてサジェストもしてくれます。
優秀ですね。

それではまずはJIRAとの疎通確認を行いたいのでFetch処理を作ってみましょう。
fetch処理を作るにはその前に認証情報を用意する必要があります。

今回はBasic認証を使いますので先ほどのトークンとユーザーのメールアドレスを用意してください。
その情報を元にBase64で変換したものを使います。
秘匿情報ですがどうせ誰に共有するわけでもなく社内で利用するものです。
面倒なのでハードコードしちゃいます。

```js:fetch.gs
const USER_NAME = "hoge@hoge.co.jp";
const API_TOKEN = "hoge";
const CREDENTIAL = Utilities.base64Encode(USER_NAME + ":" + API_TOKEN);
```

今作った認証情報をリクエストする際のヘッダーに載せるとJIRAのAPIを使えるようになります。

`Authorization: Basic XXXXXXXXXX`

という形式です。
これでAPIにリクエストを飛ばす準備ができました。

### プロジェクトから特定の情報を取得する

この情報を元にまずはfetch処理を作りましょう。
取得できるデータのイメージがないと作りづらいですもんね。

JIRAはプロジェクトによってサブドメインが設定されていたりするのでURLの設定には気をつけてください。
まずリクエスト先のhostは `https://{ドメイン}.atlassian.net` となります。
今回はJQLを使って特定のチケット情報を取得したいので、リクエスト先のパスは `/rest/api/3/search` になります。

https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/#api-group-issue-search

まずはJQLを書いていきます。
一度JIRAの画面上で欲しい情報を取得するためのJQLを作って、その画面のURLに設定されているクエリパラメータから移植してくるのがいいと思います。

- 特定のプロジェクトのボードからデータがほしい
- 現在走っているスプリントのデータがほしい
- 上記二つを満たすPBIの一覧取得する

ということができればいいのでまずはそれ用のクエリを作ります。
現在走っているスプリントはJQL用の関数に `openSprints` というものがあるので何も考えずに使えますね。
あとはプロジェクトのIDが分ければOKです。
最終的には下記のようになります。

`jql=project=<プロジェクトのKEY> AND Sprint in openSprints()`

そして先ほどのURLにクエリパラメータとして付与します。

`https://hoge.atlassian.net/rest/api/3/search?jql=project=HOGE AND Sprint in openSprints()`

これでリクエストの準備は整いました。
他にもfieldsを指定することで特定のフィールだけを取得してくる設定も可能です。

- タイトル
- ストーリーポイント
- 作成日
- 担当
- 子課題
- 属しているスプリント

今回欲しいの情報はこちらです。
ここで注意してほしいのはストーリーポイントなどのパラメータはJIRAで用意しているカラムに入っている場合と、カスタムフィールドに入っている場合があります。
そのため一度画面上からほしい情報をフィルタかけるなどしてこのパラメータはこういうfiled名なんだというのをメモしていきましょう。
カスタムフィールドの場合は `customfield_XXX` のようになります。
欲しいフィールドが用意できたら先ほどのJQLのあとにfiledsというパラメータを付与します。
最終的には以下のようになりました。

`https://hoge.atlassian.net/rest/api/3/search?jql=project=HOGE AND Sprint in openSprints()&fields=summary&fields=customfield_10021&fields=customfield_10016&fields=subtasks&fields=assignee&fileds=created`

これでURLの準備はできました。
早速fetch処理を書いていきましょう。
普段JS/TSを書いているとちょっと居心地の悪さがあるのですがGASはPromiseを返さない（？）ようでasync/awaitは不要らしいです。
サクサク書いていきましょう。

```js:fetch.gs
const USER_NAME = "hoge@hoge.co.jp";
const API_TOKEN = "hoge";
const CREDENTIAL = Utilities.base64Encode(USER_NAME + ":" + API_TOKEN);

const fetchJiraOpenSprintTicket = () => {
  const fetchArgs = {
    contentType: "application/json",
    headers: { "Authorization": `Basic ${CREDENTIAL}` },
    muteHttpExceptions: true
  };
  const url = `https://hoge.atlassian.net/rest/api/3/search?jql=project=HOGE AND Sprint in openSprints()&fields=summary&fields=customfield_10021&fields=customfield_10016&fields=subtasks&fields=assignee&fileds=created`

  const httpResponse = UrlFetchApp.fetch(url, fetchArgs);
  const response = httpResponse.getResponseCode();
  switch (response) {
    case 200:
      return JSON.parse(httpResponse.getContentText());
    default:
      Logger.log("Error: " + data.errorMessages.join(","));
      return { issues: [] };
  }
}
```

このような実装になります。
めちゃくちゃ簡単！！！
その上で型情報がないと今後実装しづらいと思うのでDOCコメントをつけてあげましょう。

```js:fetch.gs
/**
 * Jiraの対象プロジェクトからオープンになっているチケットを取得してくる
 * 
 * @return {{
 *   issues: Array<{ 
 *    expand: string
 *    id: string
 *    key: string
 *    fields: {
 *      customfield_10016: number | null  // Storypoint
 *      summary: string                   // Title
 *      customfield_10021: Array<{
 *        id: number
 *        name: string
 *        state: string
 *        boardId: number
 *        goal: string
 *        startDate: string // yyyy-MM-ddThh:mm:ss.sssZ
 *        endDate: string.  // yyyy-MM-ddThh:mm:ss.sssZ
 *      }>
 *      subtasks: Array<{
 *        id: string
 *        key: string
 *        self: string
 *        fields: {
 *          summary: string
 *          status: {
 *            self: string
 *            description: string
 *            iconUrl: string
 *            name: string
 *            id: string
 *            statusCategory: unknown
 *          }
 *          priority: unknown
 *          issuetype: unknown
 *        }
 *      }>
 *    }
 *  }>
 */
const fetchJiraOpenTicket = () => {
  // 省略
}
```

これでチケットの取得処理が完成しました。
となるとあとはこれを整形してスプレッドシートにどんどん追加していくだけです。

### 子課題の完了日を取得する

とその前に・・・
今filedにsubtaskを設定しているので子課題を取得できますが、この中には完了日を持っていません。
スプリントで終わりきらなかったPBIはスプリントをまたぐので完了日が必要です。
そのため完了日を取得するために子課題の詳細情報を取得する処理を書きます。
先ほどと同じようにfetch処理を追加するだけです。

引数にはsubtasksの中にあるselfの値を取ります。

```js:fetch.gs

/**
 * @description 
 * { 
 *   expand: 'renderedFields,names,schema,operations,editmeta,changelog,versionedRepresentations',
 *   id: '36427',
 *   self: 'https://hoge.atlassian.net/rest/api/3/issue/36427',
 *   key: 'TEAMAR-699',
 *   fields: { created: '2023-06-13T15:07:08.737+0900', updated: '2023-06-13T15:07:08.737+0900' } 
 * }
 * 
 * @return {{
 *   expand: string
 *   id: string
 *   self: string
 *   key: string
 *   fields: {
 *     created: string
 *     updated: string
 *   }
 * }}
 *
 */
const findJiraDoneSubTaskTicket = (subTaskSelf) => {
  const fetchArgs = {
    contentType: "application/json",
    headers: { "Authorization": "Basic " + CREDENTIAL },
    muteHttpExceptions: true
  };
  const httpResponse = UrlFetchApp.fetch(`${subTaskSelf}?fields=created&fields=updated`, fetchArgs);
  const response = httpResponse.getResponseCode();
  switch (response) {
    case 200:
      const value = JSON.parse(httpResponse.getContentText())
      return value;
    default:
      Logger.log("Error: " + data.errorMessages.join(","));
      return { issues: [] };
  }
}
```

### データを集計する

それではようやくここからデータの集計を行なっていきます。
取得した結果を元に配列のデータを作っていきます。
GASでシートにデータを追加する都合上1件のデータも配列で管理しておくと楽だと思います。

```js:main.gs
const excute = () => {
  const results = fetchJiraOpenTicket()
  // 今日の日付を持っておく
  const currentDate = Utilities.formatDate(new Date(), 'Asia/Tokyo', 'yyyy/MM/dd');
  /**
   * テーブルは下記の順番で設定する
   * | 今日の日付 | スプリント名 | チケットID | StoryPoint | チケット数 | Done数 | 進行中数 | To Do数 | 
   */
  const table = filterIssues.map((value) => {
    const { key, fields } = value
    const { customfield_10016, summary, customfield_10021, subtasks } = fields
    // スプリントの開始日
    const sprintStartDate = new Date(customfield_10021[0].startDate)
    // スプリント名
    const sprintName = customfield_10021[0].name
    // 子課題の集計を実施する
    const resultCount = subtasks.reduce((prev, current) => {
      if (current.fields.status.name === '進行中') {
        prev.doing += 1
        return prev
      }
      if (current.fields.status.name === 'To Do') {
        prev.todo += 1
        return prev
      }
      // 完了のチケットの内、今スプリントより前のデータを判定したいのでデータを取得
      const detail = findJiraDoneSubTaskTicket(current.self, current.id)
      // 日付をチェックして過去の完了（更新日）のものは間引く
      const updatedDate = new Date(detail.fields.updated)
      if (updatedDate < sprintStartDate) {
        return prev
      }
      // 上記以外は完了として集計
      prev.done += 1
      return prev
    }, {
      done: 0,
      doing: 0,
      todo: 0,
    })

    const ticketLength = resultCount.done + resultCount.doing + resultCount.todo

    return [
      currentDate,
      sprintName,
      key,
      summary,
      customfield_10016,
      ticketLength,
      resultCount.done,
      resultCount.doing,
      resultCount.todo
    ]
  })
}
```

### データを設定する

それでは最後にデータをスプレッドシートに転記しましょう

```js:main.gs
const excute = () => {
  // 省略
  // 編集対象のシートを設定
  const targetSheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('データベース');
  // データをインサートしていく
  table.forEach((row) => {
    targetSheet.insertRowBefore(2)
    row.forEach((col, index) => {
      targetSheet.getRange(2, index + 1).setValue(col)
    })
  })
}
```

これでデータベースが出来上がりました。
あとはここからピボットなりで集計を行いグラフにするだけです。
私たちの場合はここからさらに表生成するスクリプトを入れて別シートに表を設定しています。

![表](https://storage.googleapis.com/zenn-user-upload/6b1f48daa5f8-20230721.png)

![グラフ](https://storage.googleapis.com/zenn-user-upload/e8e87d33d236-20230721.png)

この辺りはやりやすい方法で表示してもらうのがいいと思います。

## まとめ

このような形でJIRAから1日単位でデータを集計できるようになりました
データベースを作っておけば色々と汎用的に使えると思います。
私は思いつきません笑

また、他にもやり方はあると思いますのでいい方法があれば教えてください


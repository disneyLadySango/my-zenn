---
title: "[GAS]サブチケット単位でバーンダウンチャートが見れない、終わったわ[JIRA]"
emoji: "🌪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GAS", "JIRA"]
published: false
---
進捗はいい（はず）のに、報告できない。

どうもスペースマーケットでエンジニアリングマネージャーをしている人です。

# はじめに
弊社ではスクラムのチケット管理に[JIRA](https://www.atlassian.com/ja/software/jira)を使用しています。

https://www.atlassian.com/ja/software/jira

JIRAの運用は下記のようにしています

- ストーリー（課題）をプロダクトバックログアイテムとして使う
- プロダクトバックログアイテムに対して子課題を作る形でタスクのチケットを作る
  - 子課題は概ね1つのチケットが30分で終わるように分解する

プロダクトバックログアイテムは相対見積もりを行いストーリーポイントが入っています。
対してそこに作られる子課題は30分というつまりは絶対時間での見積もりになっています。

JIRAにはレポート機能があります。
バーンダウンチャートを見ることはできますがこれはストーリーポイント単位でのグラフです。
毎朝デイリーで確認しますが、予定通りに進んでいるかどうかというのは絶対時間で作られたタスクのチケット数で確認したいものです。
予定に対して進んでいるのか、遅れているのか、この状態をJIRAのレポート機能では見ることができません。

そのため何かしらの方法が必要になります。

日々の進捗状況をトラッキングしたいのでどこかにデータを蓄える必要があるのですが、データベースをわざわざ作るほどのことでもありません。
ましてやそれをグラフ表示するとなると余計に面倒な手間がかかります。

そこでデータベースのようにデータを蓄えられるものがないかと考えた結果スプレッドシートに行きつきました。
スプレッドシートをデータベースに見立ててそこのデータをフォーマットしてあげればグラフだって表示できます。
イメージはできました、実装してみましょう。

# 実装
## 情報の整理
まずはJIRAで必要な情報を整理します。
どういった情報が必要でどういう使い方をしたいのか書き出してみました。

- 現在走っているスプリントのデータを集計したい
- 毎日のデイリーでタスクの消化状況をグラフで確認できるようにしたい
- プロダクトバックログアイテム単位でタスクのステータスごとで件数が集計されて欲しい
- プロダクトバックログアイテムが終わらなければ翌スプリントにそのまま引き継がれる、前スプリントで終わったものは集計対象から外したい

このような情報が出てきました。
この情報を元にデータベースとなるシートをまず作ってみましょう

## データベースシートの作成
今回は下記のようなデータベースになりました

```
日付 / スプリント名 / ID / タイトル / StoryPoint / 合計チケット数 / Done / Doing / ToDo
```

この項目に合うデータを取得して各カラムにデータを設定できればやりたいことが実現できます。

## JIRAのAPIを利用する
さて、ここまでできましたが問題はどのようにJIRAにあるデータを取得してくるかです。
実はJIRAではAPIを公開しています。

https://developer.atlassian.com/server/jira/platform/rest-apis/

まずは初期設定をしていきます。

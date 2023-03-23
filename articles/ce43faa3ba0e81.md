---
title: "GitHub Actionsさんはリリース用プルリクエストの作成も自動化できるようです"
emoji: "🌀"
type: "tech"
topics:
  - "github"
  - "githubactions"
  - "shell"
  - "ci"
published: true
published_at: "2022-10-07 17:33"
publication_name: "spacemarket"
---

**「そんなもの余裕なんですけど」**
と言ってくださったのでGitHub Actionsでリリース用PRを作成するお話をしていきます。

はじめまして[スペースマーケット](https://www.spacemarket.com/)でフロントエンドエンジニア兼リーダーをしている和山です。

# なぜやるのか

さっそくですが、今回なぜGitHub Actionsにこのようなお仕事をお願いすることになったのかその経緯からまずはお話させていただきます。
スペースマーケットでのブランチ運用は下記のようになっています（※リポジトリのできた時期によって若干運用が異なる場合があります）。
![](https://storage.googleapis.com/zenn-user-upload/0a9e72450a33-20220930.png)

大きく分けると3種類、featureの開発ブランチとstaging環境と同期しているstagingブランチ、本番環境と同期しているmaster(main)ブランチとなります。
また環境と紐づくブランチではpushイベントを契機としてAWSのCodePiplineを通してそれぞれの環境へ資材が反映されます（※この辺りも一部運用が異なるケースがあります）。
hotfix系があれば直接本番へ載せることもあるためmasterに向くこともありますが、そうでなければ開発した資材は基本的に一度staging環境へ向けたプルリクエストが立てられます。
コードレビュー、テストが完了したらマージを行いstagingへ反映され、staging環境で最後の動作確認、そしてmasterへの本番反映プルリクエストを立ててPMまたはリーダー・マネージャーからの承認をもらって初めてリリースすることになります。

さて、本番反映する際には毎回毎回プルリクエストを立てることになりますが、これはちょっと面倒な手順です。
リリース用には普段使っているプルリクエストのテンプレートでは内容が過剰すぎで、それを毎回毎回書き換えるのも煩わしい作業です。
ちょっとだけの手間かもしれませんが、こういう煩わしい作業は人間がやらなくてもいいんじゃないかというお話です。
そこで「退屈なことはPythonにやらせよう」の如く「面倒なことはGitHub Actionsにやらせよう」ということで今回はGitHub Actionsを使ってstaging -> masterへ向ける本番反映用のプルリクストを作ります。
このGitHub Actionsが満たすべき要件は下記になります。

- stagingへpushされたらプルリクエストを作成する
- プリリクエストの向き先はmaster(またはmain）
- プルリクエストのタイトルは「【本番反映】」という文字を先頭に付与し、何をリリースするのかわかりやすいようにするためコミットメッセージの１行目を設定する
- 開発担当をassigneesに追加する
- 本番反映用のリリース用テンプレートを使ってプルリクエストを作成する
- 基本は1PR1機能だが、軽微な場合やサービスに影響のない変更（README更新など）を相乗せする場合があるので、その考慮が必要

また今回こちらを作るにあたり [K.Shidaさん](https://zenn.dev/kshida) の　[GitHub Actionsでブランチに合わせてリリース用PRを自動生成する](https://zenn.dev/kshida/articles/auto-generate-release-pr-with-github-actions) を参考にさせていただきました。

https://zenn.dev/kshida/articles/auto-generate-release-pr-with-github-actions

この場を借りてお礼をお伝えさせていただきます、ありがとうございます！


# 実装してみた
## 完成系
ということで、解説なんかいい！早くブツを出しやがれ！という方もいるかと思います。
焦らなくても大丈夫です。分かっています、分かっています。
これが欲しいんですよね、どうぞこちらをお使いください。

```yml
on:
  push:
    branches: [ staging ]

jobs:
  create-release-pr:
    runs-on: ubuntu-latest

    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    steps:
      - uses: actions/checkout@v3

      - name: Check PullRequest Exists
        id: check_pr
	env:
	  HEAD_MESSAGE: ${{ github.event.head_commit.message }}
        run: |
          COMMIT_MESSAGE=$(echo "${HEAD_MESSAGE}" | sed -n -e 1p)
          echo "::set-output name=message::${COMMIT_MESSAGE}"
          echo "::set-output name=count::$(gh pr list -S '本番反映'in:title | wc -l)"
      - name: Create Release Pull Request
        if: ${{ steps.check_pr.outputs.count == 0 }}
        run: |
          gh pr create \
            -B main \
            -t '【本番反映】${{ steps.check_pr.outputs.message }}' \
            -a ${{ github.actor }}  \
            --body-file ./.github/RELEASE_WORKFLOW_TEMPLATE.md
      - name: Edit Release Pull Request
        if: ${{ steps.check_pr.outputs.count != 0 }}
        run: |
          pr_data=$(gh pr list -S '本番反映'in:title \
            --json "title" \
            | jq -c .[])
          TITLE="$(echo $pr_data | jq -r '.title')"
          echo $TITLE
          gh pr edit  ${{ github.ref_name }} \
            -t "${TITLE} / ${{ steps.check_pr.outputs.message }}"
```

以上です。
これにてZennでの初記事投稿を終了します。
ご視聴ありがとうございました、先生の次回作にご期待ください。



はい、これではあまりに早すぎるということでこちらの中身について解説させていただきます。
とはいえ私自身普段GitHub Actionsを使って頻繁にワークフローを書くようなことをしていないので、これを見て初めてGitHub Actionsを書くという方にも分かりやすいように１つずつ説明していきます。
コマンド一つ一つ深堀している記事もなかなかなかったので誰かの助けになれば幸いです。

## トリガーとなるイベントを設定する
すごい簡単なGitHub Actionsのお作法的なところを知っていきます。
まずは[GitHub Actionsの実行トリガー](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#on)を定義していきます。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#on

ドキュメントと今回した記述を参考にしながら見ていきましょう。

```yml
on:
  push:
    branches: [ staging ]
```

今回はonの次にpushを記載しています。
これは [pushイベントをトリガーに実行する](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#onpushbranchestagsbranches-ignoretags-ignore) ということを指しています。
今回はpushですが他にはpull_requestやscheduleなどでcron指定などもできるようです。
今回はpushをトリガーに実行するのでpushで大丈夫です。

on.pushのあとには ブランチ / タグ を指定できます。
今回はstagingブランチへのマージを起因として実行しプルリクエストを立てるのでstagingを指定します。
そのため `branches: [staging]` となっています。
この他には `feature/**` のような指定もできますので、使いたい要件に合わせて指定すると良いです。

またブランチを絞り込まず除外したい場合は、`branches-ignore` や `tags-ignore`で記述できます。
またさらに複雑ですが否定パターンなどので記述もできるとのことです。

```yml
on:
  push:
    branches:
      - 'features/**'
      - '!features/**-test'
```

この場合 `features/dev-hoge` や `features/hoge/fuga` といったブランチではpushに応じて実行されますが、 `features/dev-hoge-test` や `features/hoge/fuga-test`では`!`が否定を意味するため実行されません。
色々な設定ができるので動かしながら試してみるのが一番良いかと思います。
私自身、実際にこちらをテストしていた時はonを外したり、他の条件に変えてみてテストして雰囲気が分かったので「百聞は一見にしかず」というやつです。

## ジョブを作成する
それでは次に[ジョブの作成](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#jobs)に移ります。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#jobs

```yml
jobs:
  create-release-pr:
    runs-on: ubuntu-latest

    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

ワークフロー実行は、１つ以上のjobsから構成されています。
基本的には並列実行されるのですが、直列実行する際は`jobs.<id>.needs`を指定してあげることでジョブ同士に依存関係を定義することができます。

ということで1行目は「これからジョブを定義していきます」という言葉ですね。
そこから２行目でジョブのIDを定義しジョブを作成しています。
IDは先頭が`文字または_`で始まる必要があり、また許容される文字の種類は`英数字、-および_`のみになります。
今回は `create-release-pr` としました。
まあ今回作るのは１つだけなので並列実行とか直列実行とかあんまり関係ないんですけどね・・・

さて、そしたら3行目の`runs-on`です。
ここではジョブの実行環境、つまりマシンの種類を定義する必要があります。
指定できるのは[GitHubホステッドランナーかセルフホステッドランナーのいずれかのみ](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#github-%E3%83%9B%E3%82%B9%E3%83%86%E3%83%83%E3%83%89-%E3%83%A9%E3%83%B3%E3%83%8A%E3%83%BC%E3%81%AE%E9%81%B8%E6%8A%9E)です。
今回は特に理由もないのでUbuntuを使います。
またバージョンについても希望はなくコマンドが実行できれば良いのでlatestを指定します。
他にはWindows ServerやMacOSがあるようです。
特に希望がなければubuntu-latestを指定するのが良いと思います。

最後に[envの設定](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#env)です。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#env


ここでは環境変数を定義します。
今回やろうとしていることはGitHubのリポジトリにプルリクエストを立てることです。
そのため、認証情報が必要になります。
[GitHubは枠ーフローで使用するシークレットキーを自動的に生成](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)してくれています。
それが `${{ secrets.GITHUB_TOKEN }}`となります。
これを`GH_TOKEN`という名称で設定しています。
後述しますがこのあとGitHub CLIを使っていくのでそこの認証情報として使えるようにする必要があるためこのようにしています。
設定すべき[色々な環境変数を定義できます](https://cli.github.com/manual/gh_help_environment)ので状況に応じて使い分けをしてみると良いです。

https://cli.github.com/manual/gh_help_environment

## 実行ステップ

お次はjobsの中にあるstepsの部分です。
今回一番のキモはこちらです。
各stepごとに解説させていただきます。

### Step1 - コードのチェックアウト
まずはコードのチェックアウトが必要です。
そこで下記の記述をします。

```yml
- uses: actions/checkout@v3
```

実際にコードをCheckoutをする場合はコマンドを記述する必要があるのですが、こちらの提供されているアクションを実行することでコードを書かずにやりたいことが実行できるようになります。
[actions/checkout](https://github.com/actions/checkout)についての詳細は当該のREADMEを参考にされると良いかと思います。

https://github.com/actions/checkout

今回は提供されている物を使いましたが、自身で作成したものを使いたい時はディレクトリパスを指定しても使えますし、Dockerイメージなども指定することが可能のようです。
この辺りも[ドキュメントに詳しく載っています](https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsuses)、ほんとすごい・・・

今回はとてもシンプルにプルリクエストを立てるだけで良いのでチェックアウトだけを使用します。

### Step2 - プルリクエストの存在確認
お次はこちらです。
```yml
- name: Check PullRequest Exists
  id: check_pr
  env:
    HEAD_MESSAGE: ${{ github.event.head_commit.message }}
  run: |
    COMMIT_MESSAGE=$(echo "${HEAD_MESSAGE}" | sed -n -e 1p)
    echo "::set-output name=message::${COMMIT_MESSAGE}"
    echo "::set-output name=count::$(gh pr list -S '本番反映'in:title | wc -l)"
```
nameというのは実行のタイトルです。
Actionsの実行結果をみる際にどれが実行されたかわかりやすくするだけのGUI上での名前なのでいい感じの名前をつけてあげてください。

次にidですがこちらは一意となる識別子をつけてあげることができます。
またこの識別子があるとContextを使用できます。
後述しますが今回２つの値を引き継ぎたいのでidを指定します。

そしてenvについてです。
まずは変数である`${{ github.event.head_commit.message }}`から解決していきましょう。
これはイベントに応じて設定されるContextの値を参照しています。
`github.event` で今回はpushイベントによって使える値を参照します。
さらに続く `head_commit` で最新のコミットへアクセスします。
ここには他にidやコミットのurl、ユーザの情報が含まれます。
その中から今回は `message` を取得します。
これで最新のコミットメッセージが echo の後に代入された状態となります。
[使える値については他にも色々ありましたのでドキュメントから必要なものを探していただければと思います。](https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#push)
これをstepごとで使える環境変数に代入します。
なぜ環境変数に代入するかというと[スクリプトインジェクション](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections)を防ぐためとなります。

https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections

今回はenvにし中間環境変数とすることでインジェクション対策を行っています。
直接指定してしまうと例えば `a"; ls $GITHUB_WORKSPACE"` のようなコマンドが実行できてしまうので気をつけてください。
内部的なプロジェクトで使う場合はほとんど影響がないかもしれませんが、攻撃として実行していなくても今回はコミットメッセージを使っているので意図せず失敗する可能性も考えると念の為対策はしておいた方が良いでしょう。

:::message
[hankei6kmさん](https://zenn.dev/hankei6km)より[コメント](https://zenn.dev/link/comments/175e79c12d1772)をいただきまして上記内容を修正させていただきました、ありがとうございます！
:::

お次にrunですがこれは見ての通りコマンド実行です。

それでは実行しているコマンドの内容を見ていきます

```shell
COMMIT_MESSAGE=$(echo "${HEAD_MESSAGE}" | sed -n -e 1p)
echo "::set-output name=message::${COMMIT_MESSAGE}"
```
まずは１行目です。
ここでは先頭のコミットメッセージから1行だけを切り出しています。
先ほど定義した中間環境変数を呼び出しします。
そして`sed -n -e 1p`を実行していますね、こちらの内容を見ていきます。
sedコマンドは、テキスト処理系のコマンドで、入力を行単位で読み取ることができます。
そこにオプションをつけていますね。
`-e`はスクリプト(コマンド)を追加することができます。
`1p`の`p`の部分がコマンドとなります。行の指定ができるので今回は1行目だけが必要なので`1p`です。
`-n`は出力コマンド以外の出力を行わないという指定です。
今回指定しないと１行目以降のコミットメッセージまで取得されてしまいます。
コミットメッセージを今回はタイトルで使うのですが、メッセージ自体は１行の時ばかりではなく文字数が思いっきり超過してしまうのでこの考慮が必要になります。
これでCOMMIT_MESSAGEの変数の中にstagingブランチにpushされた最新のコミットメッセージから1行が代入されました。

お次にこの値を出力します。
なぜかというとこの後のPR作成の時も編集の時もタイトルとしてこの値を使うためです。
`set-output`が必要になります、これは[ワークフローコマンド](https://docs.github.com/ja/actions/using-workflows/workflow-commands-for-github-actions)です。


`set-output`することで出力した値を他のstepに渡すことができるようになります。
`name=XXXX`でoutputした値に名前をつけてあげます、今回はmessageという名前にしました。
これを実際に取り出す際は下記のようになります。
```shell
# echo ${{ steps.<stepのID>.outputs.<指定した名前> }}
echo ${{ steps.check_pr.outputs.message }} # ex. > feat: hoge
```

そしたら最後にこちらです。

```shell
echo "::set-output name=count::$(gh pr list -S '本番反映'in:title | wc -l)"
```
まずset-outputは先ほど記載した通りなので割愛します。
nameにはcountを指定しています。
ここからGitHub CLIのコマンドの出力結果を代入しています。

[gh pr list](https://cli.github.com/manual/gh_pr_list)でリポジトリ内のプルリクエストの一覧を取得できます。
デフォルトの設定ではopenのものだけが取得されます。
そこに-Sで検索条件を指定します。
今回は `'本番反映'in:title` としています。
これで「本番反映という文字を含むタイトル」から結果が返されるようになります。
リリースするプルリクエストには習慣として「本番反映」という文字を含むようにしているのでこのようにしています。
他にもプロジェクトごとに最適解があると思うので、プロジェクトに合わせて使っていただけると良いですね。
[-Sで指定できる検索クエリについては公式を確認すると良いです。](https://docs.github.com/en/search-github/searching-on-github/searching-issues-and-pull-requests)

https://docs.github.com/en/search-github/searching-on-github/searching-issues-and-pull-requests

そしてこのコマンドの後ろに `wc -l` があります。
wcコマンドは文字数を数えるコマンドです。
-lで改行の数を表示できるので、gh pr listの結果何件のデータがあったかを取得できるようになります。
これは「すでにPRがあったら編集する、なければPRを作成する」という条件分岐で使う必要があるのでこのようにしています。

### Step3 - プルリクエストの作成
それでは本題のプルリクエストの作成になります。
```yml
- name: Create Release Pull Request
  if: ${{ steps.check_pr.outputs.count == 0 }}
  run: |
    gh pr create \
      -B main \
      -t '【本番反映】${{ steps.check_pr.outputs.message }}' \
      -a ${{ github.actor }}  \
      --body-file ./.github/RELEASE_WORKFLOW_TEMPLATE.md
```

nameの部分は割愛します。
まずはifのところを見ていきましょう。
こちらは先ほど説明した話の通りです。
前のステップで実行したカウントの結果から0件の場合のみ実行するという話です。
もちろんif文なので&&などでつなぐこともできます。
また実行が失敗したらといった`if: ${{failure()}}`といった記述もできるようです。

https://docs.github.com/ja/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepsif


そのあとはようやくGithub CLIでプルリクエストの作成に移ります。
使うコマンドは[gh pr create](https://cli.github.com/manual/gh_pr_create)です。

それでは一つ一つオプションを確認していきます。
まず `-B` で向き先を確定させます。
今回はstaging -> master(main)になるので `-B main`となります。

次は `-t` です。こちらではタイトルを指定します。
前回のステップで生成したmessageの値と本番反映という文字を繋げてあります。
`【本番反映】hogeの実装(#111)` というようなプルリクエストが立てられるようになるわけです。

次が `-a` です。
こちらはassigneesに該当する項目です。
`${{ github.actor }}`ではワークフロー実行をトリガーしたユーザのユーザ名が取得できます。
今回のトリガーはpushに応じてなのでpushしたユーザをassigneesに含めることができます。

そして最後に `--body-file` となります。
本当はここ `-F` でもよかったらしいです（なぜかここだけ短縮しなかった・・・）。
これで特定のファイルからプルリクエストの本文を読み込むことができます。
テンプレートがリリースの時は変えたいので今回は `./.github/RELEASE_WORKFLOW_TEMPLATE.md` としてリリース用のテンプレートを作りました。
ファイルの形式についてはプロジェクトや会社ごとで慣例的に使っているフォーマットを使っていただければ良いと思います。

以上です。
これでプルリクエストが自動で生成されるようになりました。

### Step4 - プルリクエストの編集
最後にプルリクエストの編集です。
基本的には１機能１PRでリリースするんですがサービスに関係ない資材や本当に軽微な修正についてはまとめて出す時があります。
そのためすでに上記で生成されたプルリクエストに対して編集機能を設ける必要があります。

```yml
- name: Edit Release Pull Request
  if: ${{ steps.check_pr.outputs.count != 0 }}
  run: |
    pr_data=$(gh pr list -S '本番反映'in:title \
      --json "title" \
      | jq -c .[])
    TITLE="$(echo $pr_data | jq -r '.title')"
    gh pr edit  ${{ github.ref_name }} \
      -t "${TITLE} / ${{ steps.check_pr.outputs.message }}"
```

if文は先ほどの作成時の条件と反対です。
件数が1件以上あれば編集側を実行します。

まずはgh pr listを使って先ほどと同じ条件でヒットしたプルリクエストを参照します。
やりたいことは、「既存のプルリエクスとのタイトル」を取得してくることです。
今回違うのは、`--json`のオプションと`jq`コマンドです。
`--json`で出力されるフォーマットをjsonに指定できます。
"title"とすることでタイトルだけのjsonを作成します、形式は下記のような感じです。
```json
[
  {
    "title": "【本番反映】hogeの実装(#111)"
  }
]
```
その後に続くのが[jqコマンド](https://stedolan.github.io/jq/manual/)です。
jqコマンドはJSONを扱いやすくするコマンドです。
-cのオプションをつけると1行でコンパクトな出力を得られます。
下記のようなイメージです。
```json:default.json
{
  "title": "【本番反映】hogeの実装(#111)"
}
```
```json:compact-output.json
{"title":"【本番反映】hogeの実装(#111)"}
```
また今回はgh pr listの結果が配列になるので.[]で配列のパースを指定しています。
jqコマンドについてはローカルで試していただくのもいいですが[オンライン上で試す](https://jqplay.org/)こともできます。

https://jqplay.org/

```shell
TITLE="$(echo $pr_data | jq -r '.title')"
```
そのあとは再度jqコマンドを使ってデータを取得します。
`.title`はtitleのデータを出力するという意味です（titleしかないんですけどね・・・）。
-rオプションを使うとJSONのダブルクォートを勝手にフォーマットしてくれます。
-rをつけなければダブルクォートが付与されたまま出力されるので気をつけましょう。

```shell
gh pr edit  ${{ github.ref_name }} \
      -t "${TITLE} / ${{ steps.check_pr.outputs.message }}"
```
そしてようやく本題です。
[gh pr edit](https://cli.github.com/manual/gh_pr_edit)でプルリクエストの編集を行います。

`${{ github.ref_name }}`でワークフローの実行をトリガーしたブランチ名が取得されます、といってもstagingでしか実行されないのでここはハードコードしてもよいです。
これで「どのプルリクエストを編集するか」という情報を設定しています。
他にはプルリクエストの番号とかURLも指定できます。
より厳密にプルリクエストの番号指定でやりたい場合は、JSONを取得する際にnumberも含めておくと良いかと思います。

`-t`のオプションではタイトルの生成を行なっています。
この場合は「元あったタイトル / 新しいコミットメッセージの1行目」という形式になります。
実際のイメージだと
`【本番反映】hogeの実装(#111) / fugaの実装(#113)`
という形になります。
さらに別のものを載せる場合は後ろに「 / 名前」という形式でタイトルが拡張されていきます。

今のところ何個も何個も相乗りして出すことはないのですが、今後そのようなことになったらタイトルの文字数が超過しそうなのでやり方を考える必要がありそうです。

# おわりに
これで一通りの要件を満たした実装となりました。
他にも色々できたらいいこともあるのですが、それはおいおい対応の拡張として増やしていこうと思っています。
例えば弊社だとCircleCIも使っているのですがこういうCI周りは一度誰かが構築してしまうとなかなか変更することもないので、触ることがなく機会を見つけて学び直すしかないんですよね。
今回は個人のtimesでこのような声が上がったんですが、ちょうど少し前に私がこういうできるのかなと個人で試していたので隙間時間を使って実装しました。

![](https://storage.googleapis.com/zenn-user-upload/7f75d6ac7e61-20221001.png)

こうやってtimesベースから改善しましょうと声があがったり相談できたりするのはとても良い文化だと思っています！

もし弊社に興味が湧きましたら現在一緒に働けるメンバーを募集中ですので、一緒に是非お気軽にお声かけください！

https://spacemarket.co.jp/recruit/engineer/

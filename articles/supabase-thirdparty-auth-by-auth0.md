---
title: "supabase × Auth0でLINEログインを実現し、RLSで安全にユーザーデータを扱う方法"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["supabase", "auth0", "liff"]
published: false
publication_name: "spacemarket"
---

どうも、LIFFアプリの構築時に「お。認証機能もあるやん」ってことでsupabaseを選んだものの、LINEログインに非対応だと後になって知り頭を抱えた人です。

カカオトークは あるけれど LINEの認証は見だごとァ 無ェ！

ということで、supabase × Auth0でLINE認証を実現しつつ、Row Level Security（RLS）で安全にユーザーデータを扱えるようにしたので、その実装をまとめておきます。

# 前提知識のおさらい

## supabaseの認証について

https://supabase.com/docs/guides/auth

supabaseにはありがたいことに[認証機能](https://supabase.com/docs/guides/auth)がついています。
ただし、特定のサービスでのログインには対応していないこともあり、特殊な要件がある際は困るケースが多々あります。
（はい、冒頭で書いたように私がまさにそれでした。）

今回はブラウザからのアクセスはなく、LIFF上で動くアプリケーションを開発していたので、どうしてもLINEログインは実現しなければならず、かなり悩みました。

そんな時に頼れるのがsupabaseのサードパーティ認証機能です。
自分が実装したのは1年前くらいの話なので、確かリリース直後あたりだったと記憶しています。
正直、マジで危なかった…笑

### supabaseのサードパーティ認証

https://supabase.com/docs/guides/auth/third-party/overview

私が実装した当時は3種類（Firebase Auth、Auth0、Cognito）でしたが、今はClerkが追加されて計4つに。

- Clerk
- Firebase Auth
- Auth0
- AWS Cognito(Amplify)

さすがに有名どころは揃っているので、ほとんどのケースはこれでカバーできると思います。
もしこの記事に辿り着いたけど対応していないとしたら…私には救えません。円環の理に導かれるしかないでしょう。

## Auth0のLINE認証

https://auth0.com/blog/jp-line-login-now-supported-with-auth0/#LINE-Login------------

Auth0はLINE認証に対応しています。
普通のWebアプリでLINEログインのボタンを設置して認証するのもOKです。

今回作ったアプリはLIFF専用アプリだったため、シームレスな認証が可能になっています。
（このあたりの詳細は、また別の機会にブログで書こうと思います。）

## Row Level Security（RLS）

https://supabase.com/docs/guides/database/postgres/row-level-security

supabaseのデータベースはPostgreSQLベースなので、PostgreSQLの Row Level Security（RLS） を活用できます。
RLSとは、「テーブル内の行単位でアクセス制御ができる」 という機能です。

これの何が嬉しいかというと、知らないうちに「ユーザーAがユーザーBの情報を見れるようになっていた…」みたいな事故を未然に防げるということです。

supabaseはクライアントから直接データベースを参照しにいけます。
また、RESTやGraphQLからも直接テーブルを操作できるため、うっかりミスで情報漏洩につながる設計も十分にあり得ます。

```ts
import { createBrowserClient } from '@supabase/ssr'

export const supabaseClient = createBrowserClient(
  SUPABASE_URL,
  SUPABASE_KEY,
  {
    db: {
      schema: 'public',
    },
  },
)

const getUser = (userId: string) => {
  return supabaseClient.from('users').select('*').eq('id', userId)
}

const 期待していた実装 = async () => {
  // 認証情報から自分のUserIDを元にデータをとってくる
  const { data } = supabaseClient.auth.getUser()
  const userId = data.user.id
  return getUser(userId)
}

const 問題のある実装 = async (selectUserId) => {
  // 選択されたユーザーIDで誰の情報でも取れてしまう（セキュリティ的に危険）
  return getUser(selectUserId)
}
```

このように「誰でも他人の情報を取得できる設計」は大変危険です。
個人情報を含むテーブルでこれをやってしまうと、サービスの信頼性に関わる事故につながります。

そこで、supabase（PostgreSQL）では テーブルごと、行ごとにセキュリティポリシーを設定できます。

例えば以下のような制御も可能です。

- 一般ユーザー: 自身の情報だけ見られるようにする
- 管理者ユーザー：すべてのユーザー情報を閲覧・操作できる

# 実際にユーザーテーブルのアクセスを制御してみる

前提知識を確認したところで、ここからは実際に supabase × Auth0 でユーザーアクセス制御を実装する方法を紹介していきます。

今回は Auth0 を例に進めますが、他のプロバイダー（Firebase Auth、Cognito）でも基本的な考え方は同じのはずです。
supabaseには各サードパーティ認証サービスのドキュメントがあるので参照しつつ読み替えながら読み進めてください。

## supabaseとAuth0を連携する

まずは、supabase側でAuth0との連携（サードパーティ認証の設定）を行いましょう。

supabaseの管理画面で以下の順に進みます。

- `Authentication` > `Sign In / Providers` > `Third Party Auth` 
- `Add Provider`から自分の使用するProviderを選択

今回はAuth0を選択します。

![](https://storage.googleapis.com/zenn-user-upload/ca2bc17d7b71-20250526.png)

選択後、Auth0との接続に必要な情報を入力する画面になります。  
ここでは Auth0のドメイン名 を入力します。

このドメイン名は、Auto0の管理画面から以下の順に進み確認できます。
- `Applications` > `使用するアプリケーション` > `Settings` > `Domain` 

その `Domain` に表示されている値をコピーして、supabase側に貼り付けましょう。

![](https://storage.googleapis.com/zenn-user-upload/092a18058456-20250526.png)

### Auth0でCustom ClaimのRoleの割り当てる

次に、Auth0側で「認証後のトークンに `role` を含める」設定を行います。 
これによって、supabase側で `authenticated` のロールが使えるようになり、認証されたと判断できるようになります。

supabase公式で設定すべき内容が記載されています。

```js
exports.onExecutePostLogin = async (event, api) => {
  api.accessToken.setCustomClaim('role', 'authenticated')
}
```

https://supabase.com/docs/guides/auth/third-party/auth0#use-an-auth0-action-to-assign-the-authenticated-role

Auth0側にどのように設定すべきかという点はAuto0のCustom Claimsのページを参考にすると分かりやすいです。
`Actions` から設定していきます。

https://auth0.com/docs/ja-jp/secure/tokens/json-web-tokens/create-custom-claims#create-custom-claims

たったこれだけで、ログイン後のトークンに role: authenticated が含まれるようになります。
supabase側ではこの情報を元にセキュリティポリシーを設定できるようになるので非常に便利です！

## ユーザーのテーブルと認証情報を紐づける

ユーザーがログインしたあと、認証情報とデータベース上のユーザー情報を紐づけられるようにしておきましょう。  
今回はLINEログインを使っているので、LINEのユーザーID をキーとして使います。

以下はシンプルなサンプル構成です。

| カラム名 | 説明 |
|---------|------|
| id      | ユーザーの内部ID（supabaseの主キー） |
| line_id | LINEのユーザーID（Auth0から取得） |
| name    | 表示名などの任意情報 |

実際のデータは以下のようになります。

| ID | LINE_ID | NAME     |
|----|---------|----------|
| 1  | U1      | 太郎さん |
| 2  | U2      | 次郎さん |
| 3  | U5      | 花子さん |

この `line_id` は、ログイン後のトークンから取得できます。  
supabaseのクライアントを使ってユーザー情報を取得する場合、以下のようにして `user.id` を取り出します。

```ts
const { data } = await supabaseClient.auth.getUser()
const userId = data.user.id
```

## テーブルに対してセキュリティポリシーを設定する

supabaseでは、管理画面からテーブル単位でセキュリティポリシー（RLS）を設定できます。  
GUIでの操作が中心なので、コードを書かずにアクセス制御のルールを組み立てられるのが魅力です。

まずは管理画面で以下の順に進みましょう.

- `Authentication` > `Policies` 

ここで、対象のテーブルに対して新しいポリシーを作成できます。  
設定項目としては以下のような内容を選びます。

- Policy Name（任意の名前）
- Table（対象となるテーブル。例: `public.users`）
- Policy Command（どの操作に対するポリシーか）
  - `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `ALL`
- Target Role（この操作を許可するロール）

このようにGUIから選択しながら進められるため、初めてでも比較的わかりやすい設計になっています。

それでは、具体的にCRUD操作ごとのセキュリティ設計を見ていきましょう。

### Insert（認証ユーザーのみ登録を許可）

まずは、ユーザーが自由にレコードを登録できないように、認証済みユーザーだけがINSERTできるように制御します。  
（誰でも登録できてしまうと、不正なアカウント作成やスパムの温床になります。）

| 設定項目 | 設定内容 |
|----------|----------|
| Policy Name | 任意の名前（例：`InsertPolicyForAuthenticated`） |
| Table       | `public.users` |
| Policy Command | `INSERT` |
| Target Roles   | `authenticated` |

ポリシーを作成すると、画面上には次のようなSQLが生成されるのでwith checkを書き換えます。

```sql
alter policy "InsertPolicyForAuthenticated"
on "public"."users"
to authenticated
with check (
  true
);
```

この with check (true) は「認証ユーザーなら無条件にINSERTを許可する」という意味です。
これで、未認証ユーザーによるデータ登録を防ぐことができます。

### SELECT / UPDATE / DELETE（自分のデータにだけ操作を許可）

次は、ユーザー自身にだけデータの閲覧・更新・削除を許可するポリシーを設定します。  
他人のデータを見たり書き換えたりできると、大きなセキュリティリスクになるため、ここは確実に制御しましょう。

supabaseのGUIから、`SELECT` / `UPDATE` / `DELETE` の各操作に対して個別にポリシーを設定していきます。  
（※残念ながら、GUI上では一括で複数操作を選んで設定することはできません。）

ここでは、`SELECT` の設定を例に見ていきます。

| 設定項目 | 設定内容 |
|----------|----------|
| Policy Name     | 任意の名前（例: `SelectOnlyOwnUser`） |
| Table           | `public.users` |
| Policy Command  | `SELECT` |
| Target Roles    | `authenticated` |

ポリシーを作成すると、画面上には次のようなSQLが生成されるのでwith checkを書き換えます。

```sql
alter policy "SelectOnlyOwnUser"
on "public"."users"
to authenticated
with check (
  auth.get_line_id() = line_id
);
```

この with check では、「認証情報から取得したLINE ID」と「テーブルの line_id」が一致しているかどうかを判定しています。

```sql
with check (
  auth.get_line_id() = line_id
)
```

このようにすることで、ユーザー自身の行にだけアクセスが許可される仕組みになります。
UPDATE や DELETE に関しても、同様の条件で設定すればOKです。

## SQLのFunctionで認証情報から一意のユーザー情報を取り出す

ここまでで、RLSポリシーで「ユーザー自身のデータだけを操作できるようにする」設定を行ってきました。  
その際に使った `auth.get_line_id()` 関数について、ここで詳しく解説します。

### なぜ関数が必要なのか？

https://www.postgresql.jp/docs/11/sql-createfunction.html

RLSの評価は PostgreSQLの中で実行されるため、通常のアプリ側コード（TypeScriptなど）では認証情報を直接参照できません。  
その代わり、supabaseではJWTトークンの中身（claims）をPostgreSQL側で参照する仕組みが提供されています。

```sql
current_setting('request.jwt.claims', true)
```

この値はJSON形式でJWTの中身（claims）を表しており、必要なキーを抽出して関数化することでRLS内でも安全に扱えるようになります。

### get_line_id() 関数の定義

以下の関数では、JWTの sub（Subject）からLINEログイン時のユーザーIDを取り出しています。

```sql
create or replace function auth.get_line_id() returns text as $$
  select nullif(current_setting('request.jwt.claims', true)::json->>'sub', '')::text;
$$ language sql stable;
```

- sub はLINEログイン時に付与される一意のユーザーID（例: "U5"）
- nullif(..., '') によって空文字列の場合は null を返すことで安全性を担保しています

この関数を使えば、RLSポリシーやSQLクエリの中で auth.get_line_id() を呼ぶだけで、ログインユーザーのIDをテーブルと照合可能になります。

JWTのイメージ

```json
{
  "sub": "U5",
  "role": "authenticated"
}
```

### 関数かのメリット

- ポリシー内の記述が簡潔にできる（長い式を繰り返さずに済む）
- 意図が明確になる（関数名で意味が伝わる）
- 管理者や特別ロールなど、ロジックごとに複数の関数を使い分けられる

たとえば管理者であれば、role = 'admin' のような条件付きで ALL 権限を与えることもできます。

### 補足：ユーザーIDを取得したい場合

この関数は 認証済みユーザーのID（内部ID）を取得するためにも使えます。
たとえば、line_id で users テーブルと突き合わせて内部IDを返す get_user_id() 関数を定義しておくと便利です。

```sql
create or replace function public.get_user_id() returns text as $$
  select
    id
  from
    users
  where line_id = auth.get_line_id()
  limit 1;
$$ language sql stable;
```

### Edge Functionでの使用例

このような関数は、supabaseの Edge Function から RPC（リモートプロシージャコール）として呼び出すことができます。

```ts
// 認証付きでsupabaseクライアントを生成
export const createSupabaseClient = (authorization) => {
  return createClient(
    SUPABASE_URL,
    SUPABASE_KEY,
    {
      global: {
        headers: {
          Authorization: authorization ?? `Bearer ${SUPABASE_CONFIG.API_KEY}`,
        },
      },
    }
  )
}

Deno.serve(async (req) => {
  const headers = req.headers;
  const authorization = headers.get('authorization');
  const supabaseClient = createSupabaseClient(authorization);

  // get_user_id 関数をRPCで呼び出す
  const { data } = await supabaseClient.rpc('get_user_id');
  const userId = data ?? '';
});
```

このように、DBの中でJWT情報を使ってユーザー情報を安全に取り出す仕組みを用意しておくと、RLSやサーバー処理の中でもブレない一貫性を保てます。

## まとめ

今回は、supabaseとAuth0を連携してLINEログインを実現し、 Row Level Security（RLS）でユーザーごとのアクセス制御を安全に行う方法を紹介しました。

1. supabaseでAuth0連携を行う
2. Auth0のカスタムクレームでロールを付与する
3. RLSで行単位の制御を設定する
4. JWT情報を扱う関数を用意する
5. Edge Functionなどでその関数を活用する

ここまで設定すれば、他人のデータが見えない・書き換えられない、しっかりした認証付きアプリが構築できます。

また実際に作ったプロダクトでは、

- 一般ユーザーはLIFFアプリを経由して使うためLINE認証をするのでAuth0を使う
- 管理者ユーザーはWeb上でブラウザ経由で管理画面が確認できれば良いので、Googleログインをsupabaseの認証を使って作る
  - supabaseでアカウント作ったユーザーはadmin扱いにする

と言った形で複数の認証を使い分けてたりもします。

制約と言えば料金・プランになりますが、実際に認証の仕組みを作ろうとしている方の力になれたなら幸いです。
（自分がやった時は記事があまりなかったので）

他にもsupabaseであれこれやっていたので今後もまとめていきます。

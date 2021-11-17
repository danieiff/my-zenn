---
title: "vte.cxのBFF開発_データベースからCSV出力"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vtecx,csv]
published: false
---
vte.cxのフロントエンド開発はViewと、ビジネスロジックのみを責務に持つBFFで完結します。
### BFFの開発手順
1. `/server`配下に *`{スクリプト名}.ts/tsx`*~~/js/jsx~~ をおく
2. `npm run login`　ログインしているサービスにデプロイされる
3. `npm run watch -- --env entry=/server/{目的のファイル名}` **ES5**へ変換
4. `GET|POST|PUT /s/{スクリプト名}`と呼び出す。 最大実行時間5分
  `_async`パラメータを付けると非同期リクエスト: 別スレッドが起動し、`202 Accepted`を返す。 バッチジョブサーバで、設定されたタイムアウト時間を最大実行時間として処理される。

### BFF開発例__データベースからcsv出力
↓ BigQueryのテーブルからCSV出力します。
![BigQueryのuserテーブル](/images/bq_user_table.png)

#### 手順通り
- csvを返すファイルは *`{任意}.csv.ts/tsx`* とする
- `vtecxapi.getBQ(sql)` vte.cxのAPIを通じてBigQueryに対して任意のSQLを実行できます。
- `vtecxapi.doResponseCsv([ headers[], ...entry[] ], '{csvのファイル名}')`
  レスポンスヘッダの`content-disposition: "attachment; filename=\"{csvのファイル名}\""`
```ts: user.csv.ts
import * as vtecxapi from 'vtecxapi'

const DATASET = '{BigQueryの該当データセット名}'
const TABLE = '{BigQueryの該当テーブル名}'
const SCHEMA = {userid:'ID',name:'名前',gender:'性別',age:'年齢',birth:'生年月日',email:'メールアドレス',tel:'電話番号',postcode:'郵便番号',address:'住所',memo:'備考欄'}
const ITEMS = Object.keys(SCHEMA)
const TITLES = Object.values(SCHEMA)
const ORDER = 'order by f.userid asc'
const sql = `
select
  ${ITEMS.join(',') /* =>'useid,name,...,memo' */}
from
  ${DATASET}.${TABLE} as f
  right join
    ( select
        key,
        max(updated) as updated
      from
        ${DATASET}.${TABLE}
        group by
          key
    ) as k
    on
      f.updated=k.updated and
      f.key=k.key
where
  f.deleted = false
${ORDER}
`
const feed = vtecxapi.getBQ(sql) // [ {userid: 1, name: '{名前}', ... }, ... ]
const body = feed.map((entry: any) => ITEMS.map(item => entry[item] ?? '未登録')) // [ ['{ID}', '{名前}', ..., '{備考欄}'], ["], ["], ... ]
const csv = [TITLES, ...body]

vtecxapi.doResponseCsv(csv, 'user.csv')
```
ビルド後、`GET '/s/user.csv'` `/s/{スクリプト名}` ~~ファイル名~~
```ts
axios.get('/s/user.csv', { responseType: 'blob' })
```

![csvファイル](/images/user_csv.png)
---
title: "vte.cxでPDFをSSR"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vtecx,pdf]
published: true
---
### PDFをSSR
BigQueryのテーブルからデータを取得して、PDFを出力します。
:::details ↓ファイルをデプロイ
1. `/server`配下に *`{スクリプト名}.ts/tsx`*~~/js/jsx~~ をおく
2. `npm run login`　ログインしているサービスにデプロイされる
3. `npm run watch -- --env entry=/server/{目的のファイル名}` **ES5**へ変換
4. `GET|POST|PUT /s/{スクリプト名}`と呼び出す。 最大実行時間5分
  `_async`パラメータを付けると非同期リクエスト: 別スレッドが起動し、`202 Accepted`を返す。 バッチジョブサーバで、設定されたタイムアウト時間を最大実行時間として処理される。
:::
- pdfを返すファイルは *`{任意}.pdf.tsx`* とする
- `vtecxapi.getBQ(sql)` vte.cxのAPIを通じてBigQueryに対して任意のSQLを実行できます。
- `vtecxapi.toPdf(page, html, outfilename)`
  page: ページ数
  html: テンプレートHTML
  outfilename: ファイル名 レスポンスヘッダの`content-disposition: "attachment; filename=\"{outfilename}\""`
```tsx: /server/user.pdf.tsx
import * as React from 'react'
import * as vtecxapi from 'vtecxapi'
import * as ReactDOMServer from 'react-dom/server'
import { _page, _table, _td } from '../pdf/pdfstyles'

const DATASET = '{BigQueryの該当データセット名}'
const TABLE = '{BigQueryの該当テーブル名}'
const SCHEMA = {userid:'ID',name:'名前',gender:'性別',age:'年齢',birth:'生年月日',email:'メールアドレス',tel:'電話番号',postcode:'郵便番号',address:'住所',memo:'備考欄'}
const ITEMS = Object.keys(SCHEMA)
const TITLES = Object.values(SCHEMA)
const ORDER = 'order by f.userid asc'
const sql = `
select
 ${ITEMS.join( ',')  /* =>'useid,name,...,memo' */}
from
  ${DATASET}.${TABLE} as f
    right join
      ( select
          key,
          max(updated) as updated
        from
          ${DATASET}.${TABLE}
          group by key
      ) as k
      on
        f.updated=k.updated and
        f.key=k.key
where
  f.deleted = false
${ORDER}
`
const feed = vtecxapi.getBQ(sql)
const pdf = (
  <html>
    <body>
      <div className="_page" style={_page}>
        <table style={_table}>
          <tr>
            {TITLES.map( title => <th style={_td}> {title} </th> )}
          </tr>
        {feed.map( (entry: any) =>
          <tr>
          {ITEMS.map( item => <td style={_td}>{entry[item] ?? '未登録'}</td> )}
          </tr>
        )}
        </table>
      </div>
    </body>
  </html>
)
const html = ReactDOMServer.renderToStaticMarkup(pdf)

vtecxapi.toPdf(1, html, 'user.pdf')
```
`<div class="_page">`要素には、`style`属性にページの大きさや向き、余白サイズ、暗号化や署名などを設定できます。
:::details /pdf/pdfstyles.ts
```ts
export const _page: any = {
  pagesize: 'A4',
  orientation: 'landscape'
}

export const _table: any = {
  frame: 'box',
  align: 'left',
  width: '100%',
  cols: '10',
  color: '333333',
  cellpadding: '2',
  widths: '1,3,2,2,2,3,4,2,4,2'
}

export const _td: any = {
  left: 'true',
  right: 'true',
  top: 'true',
  bottom: 'true'
}
```
:::
`GET '/s/user.pdf'` `/s/{スクリプト名}` ~~ファイル名~~
```ts
axios.get('/s/user.pdf', { responseType: 'blob' })
```
![pdf](/images/user_pdf.png)
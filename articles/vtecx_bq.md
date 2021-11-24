---
title: "vte.cxとBigQueryを連携"
emoji: "🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vtecx,bigquery]
published: false
---
vte.cxのデータストアをBigQueryにするだけ。
1. vte.cxにBigQueryと連携させる設定を記述
2. 指定したテーブルが作られ、データを登録される
`vtecxapi.postBQ()`
#### スキーマ定義
```xml: /setup/_settings/template.xml抜粋
<!-- 規約通りに記述します -->
<content>user
 name(string)!
 age(int){0~200}
 birth(date)
 gender(string)
 tel(string)
 email(string)
 postcode(string)
 address(string)
 memo(string)
 userid(int)</content>
 ```
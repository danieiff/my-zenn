---
title: "vte.cxのグループ権限"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vtecx]
published: false
---
グループの目的は、ユーザーの権限をグループ一括で管理することです。
ユーザがグループを作り、別ユーザーがそこへ参加するまで。

### ユーザーアカウントを作る
admin権限でユーザーを作成 `vtecxapi.addusersByAdmin()` できる

### グループ作成
1. サービスのリソース上にシステムフォルダ以外の任意のフォルダを用意し、これをグループとする
(1) 作成者(`UID:123`)が権限(`CRUD`)を持つ、
(2) グループ自体(`/groupS/group_123`)が権限(`CRUD`)を持つ、
(3) グループ(`/groupS/group_123`)を作成する
```ts
const group_creation_entry = {
  contributor: [
    { uri: 'urn:vte.cx:acl:123,CRUD' }, /*(1)*/
    { uri: 'urn:vte.cx:acl:/groupS/group_123,CRUD' } /*(2)*/
  ],
  link: [
    { ___rel: 'self', ___href: '/groupS/group_123' } /*(3)*/
  ]
}
```
2. グループに対して (1)参加するユーザを配下に登録 (2)それのエイリアスを登録
- 作成者をグループ配下に登録、エイリアス登録
```ts
const signature_entry = {
  link: [
    { ___rel: 'self', ___href: '/groupS/group_123/123'},
    { ___rel: 'alternate', ___href: '/_user/123/group/group_123'}
  ]
}
```
グループ作成者が下を`POST`する。 グループは作成され、グループ作成者はこれに所属します
```ts
const feed = [ group_creation_entry, signature_entry ]
```

### グループ招待/参加申請　を送る
`UID:456`のユーザを招待します
1. グループに対して権限を持つユーザ(作成者など)が[グループ作成の手順2]をユーザに対して実行します
```ts
const signature_entry = {
  link: [
    { ___rel: 'self', ___href: '/groupS/group_123/456'},
    { ___rel: 'alternate', ___href: '/_user/456/group/group_123'}
  ]
}
```
2. グループに対して権限を持つユーザ(作成者など)が　`PUT: /groupS/group_123/456?_signature`
`PUT: {signature_entry の self のURL}?_signature`　の形式

3. 招待を受けるユーザが　`PUT: /_user/456/group/group_123`
`PUT: {signature_entry の alternate のURL}?_signature`　の形式

手順2と3の後、URLのselfとalternateの対応が確認されると、ユーザーはグループに参加させられる
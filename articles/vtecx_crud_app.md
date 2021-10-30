---
title: "業務Webアプリ向けBaaS vte.cxに入門"
emoji: "️🖥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vtecx", "react", "typescript", "materialui"]
published: false
---

vte.cx(ブイテックス) [歩き方](https://qiita.com/stakezaki/items/e526ca061d8f004db7f5)

API設計は普遍的、直感的な規約に守られていて、フロントエンドエンジニアはBFFの領域で安全にサーバーリソースを扱えます。
APIサーバーの構築と、保守、拡張、運用に全く手がかかりません。
バックエンドについて最低限の知識で、多様なビジネスロジックを持つWebアプリケーションを構築してリリースすることができます｡

以下､ **vte.cx, React, TypeScript, Material UI による
Create, Read(+ページネーション, データベース検索), Update, Deleteアプリ**の実装記録です｡
## 構成
```
─ src/components/
  ├─ index.tsx 3つのページをルーティング            ・・・(3)
  ├─ Register.tsx 新規登録ページ '/register'       ・・・(1)
  ├─ tableComponents/
  │    ├─ Table.tsx データ一覧表示ページ '/'        ・・・(2)
  │    │   表示データGET通信(ページネーション・・・(5)､データベース検索・・・(6))
  │    ├─ Edit.tsx 既存データ編集ページ '/edit'     ・・・(4)
  │    └─ Delete.tsx 既存データ削除
  ├─ Pagination.tsx                               ・・・(5)
  ├─ SearchField.tsx
  │   ユーザーの入力から､表示データGET通信にのせる検索用パラメータを生成します｡
  │                                               ・・・(6)
  └ formComponents/{~Input}.tsx
     フォーム項目ごとに雛形コンポーネント､ファイルを作成
```
## (1)~(6)をさらっていきます
### (1) 新規登録フォーム Register.tsx
>vte.cx, Axios, Controlled Form, Yup
#### APIを用意~利用する
vte.cxを使ってAPIサーバーを用意し、**エンドポイント**と**スキーマ**を設定します｡
```ts
axios.post('/d/{エンドポイント}', [ {/* スキーマ定義通りの内容のオブジェクト */} ])
```
vte.cxではリクエストを送る際に以下の設定が必要です。
```ts
axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest'
```
#### 最後(6)まで(とりあえずフォームまで)コーディング
Controlled Form
親のRegisterコンポーネントが`state`と`onChange`でフォーム入力(→状態State)を管理(→100歩譲って"*上書き*")すること｡
ユーザーの入力に伴って再描画のたびに周りのUIを変えるパターン｡
```ts:Register.tsx
import { useState } from 'react'
const Register = () => {
  const [firstname, setFirstname] = useState(undefined /* もしくは '' */)
  // vte.cxスキーマ定義からファイル生成される型を使うこと｡
  // ...他項目
  const handleSubmit = ( event: React.FormEvent<HTMLFormElement> ) => {
    event.preventDefault()
    const user = { firstname /*,...*/ }
    // 前述の2行のaxiosのコード + 例外処理
  }
  return (
    <form onSubmit={handleSubmit} >
      <label>名</lable>
      <input value={firstname} onChange={setFirstname} />
      <input type="submit" />
    </form>
  )
}
```
Uncontrolled Form
各入力コンポーネントが入力を管理すること｡ 周りのUIを変えないので**useRef**を使い､再描画させないパターン｡
```ts
const Uncontrolled_Form = () => {
  const firstnameRef = useRef( {} as HTMLInputElement )
  // ...他項目
  const handleSubmit = (event/*:型*/) => {
    const user = { firstname: inputRef.current.value /*,...*/ }
    // ...
  }
  // ...
  <input ref={firstnameRef} />
}
```
ref. [Controlled and uncontrolled form](https://goshakkk.name/controlled-vs-uncontrolled-inputs-react/#conclusion)

入力項目ごとにPropsや挙動を変えるのでフォーム項目ごとに共通化しました。
各項目コンポーネントに値が`'register'|'edit'`の`mode`Propsを渡して､UIを出し分けます。
```ts:Firstname.tsx
type Props = { mode: 'register', register='登録' /*,...*/ } | { mode: 'edit' /*,...*/ }
const Firstname: React.VFC<Props> = props => {
  if ( props.mode === 'register' ) {
    const { register } = props
    // ...
    return // ...
  } else if ( props.mode === 'edit' ) {
    // ...
  } else null
}
```
↓と ライブラリ[React Final Form](https://final-form.org/react)がおすすめ
:::details バリデーションライブラリ Yup
```ts:Register.tsx
import * as yup from 'yup'
// コンポーネントの外側
const schema = yup.object().shape({
  firstname: yup.string().required('必須')
  middlename: yup.string(),
  lastname: yup.string().required('必須'),
  email: yup.string().email('無効なメールアドレス')
})
// 内側
const [firstname, setFirstname] = useState(undefined)
const [error, setError] = useState({ firstname: '' /*,...*/ })
const handleSubmit = async (event) => {
  try {
    const result = await schema.validate({ firstname /*,...*/ })
    // 送信
  } catch (error) {
    setError({ [error.path]: error.message }) //
  }
}
return (
  <form onSubmit={handleSubmit} >
    <label>名</label>
    <input onChange={setFirstname} />
    {error.firstname || <></>}
    // ...
  </form>
)
```
validation関数でthrowされるerrorオブジェクト
```ts
{
  name: 'ValidationError',
  value: {/* バリデーションされたオブジェクトのコピー*/},
  path: '最初にひっかかった項目の名前',
  type: 'yupライブラリ内部バリデーション処理で最初にひっかかったエラーの名前',
  errors: ['上のエラーの警告文'],
  inner: [] // validate(,第2引数{ abortEarly: false })と指定したとき
  // [ひっかかったすべての項目ごとに{第一階層の形式},{〃}, ...}
  message: '上コードのschemaに文errorsプロパティの文',
}
```
[Yup](https://github.com/jquense/yup)はスキーマをもとにパース､バリデーションするためのライブラリです｡
:::
### (2) データ一覧表示 Table.tsx
>Data Grid コンポーネント(Material UI)

```ts:Table.tsx
useEffect(()=>{ /* 一覧表示するデータをGETリクエストしてstateに入れる */ },[])
```
**失敗時**
```ts:Table.tsx
import { GridOverlay } from '@mui/x-data-grid'
<DataGrid
  // ...
  components={{
    NoRowsOverlay: ()=>
      <GridOverlay>{ state.errMsg || 'データ0件' }</GridOverlay>}
  }}
/>
```
**成功時**
```ts
// ↓のレスポンスを整形してstateに持たせた
{
  data: [
    {
      user,
      id,
      link: [ { ___href:'{更新先}', ___rel:'{←hrefの説明。selfだとかalias}'} /* 1 or 複数 ,... */ ]
      // ,...
    } // ,...
  ] // ,... ←ここは不要
}
// ↓ 整形
[ {...user, id, link /*,...*/ } ] // 型も定義する
```
```ts:Table.tsx
import { DataGrid } from '@mui/x-data-grid'
//Tableコンポーネントの外
// 一覧表示される項目(=列column)ごとの設定
const columns = [
  // 一覧表示されるstate.feed配列の中のオブジェクト(=行row)のageプロパティについての設定
  {
    field: 'age',
    headerName: '年齢',
    width: '100px',
    renderCell: catchUnregistered(
      (params: GridValueFormatterParams & { value: string }) => value + '歳'
    ) // 後述
  } // ,...
]
// Tableコンポーネント内
<DataGrid
  columns={columns}
  rows={state.feed} // 整形済み[{ ...user, id, link ,... }]
/>
```
:::details DataGridのProps columns の renderCell プロパティの説明
DataGridでは､行にプロパティがないと空欄になる｡
代わりに未登録を明示するために高階関数を使い､コードを読みやすくしました｡
DataGridのProps `columns`の`renderCell`プロパティの中身は､`GridValueFormatterParams`型のparamsが引数で､プリミティブ型やJSX.Element型が返り値の関数になります｡ この返り値が一覧表に表示されます｡

```ts:Table.tsx
import { GridCellValue, GridValueFormatterParams } from '@mui/x-data-grid'
// コンポーネントの外側
const catchUnregistered =
  (formatterFunc?: (params: GridValueFormatterParams) => GridCellValue) =>
  // このcatchUnregistered関数は以下を返す｡
  (params: GridValueFormatterParams) =>
    params.value && params.value !== '' ? ( // 未登録か?
      formatterFunc ? ( // 関数が渡されたか?
        formatterFunc(params) // 渡された関数でフォーマットされた登録値を返す関数
      ) : (
        params.value // 登録値をそのまま返す関数
      )
    ) : (
      <p style={{ color: 'silver' }}>Not Registered</p>
      // 未登録を明示するhtml要素を返す関数
    )
```
:::
### (3) (1)と(2)をルーティング index.tsx
>React Router
```ts:index.tsx
import { HashRouter, Switch, Route, Redirect } from 'react-router-dom'
// ...
 <HashRouter hashType="noslash">
   <Menu /> // メニューコンポーネント 省略
   <Switch>
     <Route exact path="/" >
       <Redirect to="/table" />
     </Route >
  // <indicatorContext.Provider
    // value={{ indicatorState, indicatorDispatch }}
  // >
       <Route exact path="/table" exact>
         <Table /> // 一覧コンポーネント
       </Route>
       <Route path="/register">
         <Register /> // 登録コンポーネント
        </Route>
        <Route path="/edit">
          <Edit />  // 編集コンポーネント (4)
        </Route>
   // </indicatorContext.Provider>
    </Switch>
 // ReactのReducerとContextを使って､CRUDアクションごとに通知を表示すること｡axios中に各ボタンをdisabledにすること｡ 省略
 // {indicatorState.isShow &&
   // <Indicator
     // message={indicatorState.msg}
     // onClick={indicatorDispatch({type:'hide'})}
   // />
 // }
// ...
```
`<Switch></Switch>`の外側の`<Menu />`はパスに関係なく表示されます｡
### (4) データ編集 Edit.tsx
>onSelectionModelChange (Data Grid Props)

一覧表UIの各行の最左列にチェックボックスがあり､`onSelectionModelChange` Propsで制御します｡
複数データを編集と削除できるようにしてみます｡
```ts:Table.tsx
import { button } from '@material-ui/core'
import { Link } from 'react-router-dom'
// ...
const [ids, setIds] = useState([])
const handleIdsChange = (ids: GridSelectionModel) =>
 setIds( Array.from( new Set(ids).values() ) )
const selection = state.feed.filter( entry => ids.has(entry.id) )
}
return (
  <DataGrid
    // ...
    selectionModel={ids}
    onSelectionModelChange={handleIdsChange}
    components={{
      Footer: ()=> // selectionを編集コンポーネントに渡すこと｡
        <div class="レイアウト調整">
          <button
            component={Link}
            to={{ pathname: '/edit', state: { selection }}
          >
            編集
          </button>
          <button onClick={/*削除処理*/} >削除</button>
        </div>
    }}
  />
)
```
まず削除処理
```ts:Table.tsx
const delete = async () => {
  if ( selection.length > 20 ) {
    alert('一度に削除できるのは20件まで｡') // サーバーへの負荷対策
    return
  }
  const feed = selection.map( ({ link }) => ({ link }) )
  try {
    const r = await axios.put('/d' + feed) // このように1度のトランザクションで複数削除できる
    return { success: `削除されました: ${feed.join(', ')}` }
  } catch (e) {
      if ( e === '500') { // エラー種類ごとに対応すること
        return { error: 'サーバーに問題があります'}
      } else // ...
      }
    })
  )
  const error_msg = results.flatMap((r: any)=>'error' in r.value? r.value.error: []) .join('\n')
  // 一覧データのGETリクエストを送り再描画すること｡ データベース更新までのラグがある
}
```
複数編集するEditコンポーネント
```ts:Edit.tsx
import { useLocation } from 'react-router-dom'

const initial/*:Tableコンポーネントのstate.feedの型*/ = {}

const Edit = () => {
  const { entries: _entries } = useLocation<{ entries/*:state.feedの型*/ }>().state || { entries: [] }
 // リダイレクトすること｡
  if (_entries.length < 1 || _entries.length > 10) history.push('/')
 // stateの初期値を動的に生成すること｡
  _entries.forEach( ({ entry }) => initial[entry.id] = entry )
  const [entries, setEntries] = useState(initial)
 // UI
  const entries_form = Object.values(entries).map(props =>
      <div class="レイアウト" key={props.id} >
        <div>{props.id}</div>
        <label>名</label>
        <input
          value={props.firstname}
          onChange={(e/*:型*/)=>
            setEntries(prev=>( { ...prev, [props.id]: { ...prev[id], firstname: e.target.value } } ))
          }
        />
        // ...
      </div>
    )
  return <form onSubmit={/*更新処理*/}> {entries_form} </form>
}
```
更新処理の構造は削除処理と同じ
```ts:Edit.tsx
axios.put('/d/', [
  { // スキーマ定義の形にすること｡
    user: {
      firstname,
      age: age || undefined
      // ,...
    },
    link
  } //,...複数のエントリ
])
```
### (5) ページネーション
>vte.cx クエリパラメータ

- ページネーションのページ数がブラウザ履歴に残るようにすること。
↓クエリパラメータにページ数を管理させる。
`https://{ドメイン名+ホスト名}/table` `?page={ページ数}`
- 3ステップのリクエストごとにエラーを表示し､それぞれリトライすること｡
```ts:Table.tsx
import { useLocation } from 'react-router-dom' // ←URLが変わるたびにコンポーネントを再レンダーします。
// Pagination Config
const RANGE = 50 // カーソルの範囲が50 →範囲50ページ
const LENGTH = 5 // 1回のレスポンスに入るエントリの数 →1ページに5件

const query = new URLSearchParams( useLocation().search ) // URL'.../table?page={ページ数}'の{ページ数}を取得する。
const page = Number( query.get('page') ) || 1 // ない時は1。 number型キャストして数値比較できるようにすること。
const total = useRef(0)
const cursorEnd = useRef(1)

const getFeed = async () => {
 // 総件数
  await fallback( async ()=>{
    const count = await axios.get(`d/{エンドポイント}?c`)
    total.current = count // 本当はcount.data.feed.title
  })
 // "カーソルを作る"
  await fallback( async ()=>{
  // 最初と､ページがカーソルを超えるとき
    if (cursorEnd.current === 1 || cursorEnd.current < page) {
      // カーソルを更新する
      // 前のカーソル終わり位置cursorEndを次のカーソル始め位置にする _pagination={始め,終わり} 最初は{1,50程度}にすること
      // 総件数がLENGTH*RANGEより少なければ､Math.総件数カーソルが作られる
      // RANGE(→50)ずつ上げていく
      const cursor = await axios.get( `/d/{エンドポイント}?_pagination=${cursorEnd.current},${ page + RANGE - 1 }&l=${LENGTH}`
      )
      cursorEnd.current = cursor.cursorEnd // 本当はcursor.data.feed.subtitle
    }
  })
 // フィードを取得する｡
  await fallback( async ()=>{
    const feed = await axios.get(`/d/{エンドポイント}?${query}&n=${page}&l=${LENGTH}`)
    setState({ feed: SEIKEI(feed.data) }) // 整形する (2)参照
  })
}

const fallback = async (func: ()=>Promise<void>) => {
  const LIMIT = 10
  let retry = 0
  while (retry++ < LIMIT) {
    try {
      await func()
      retry = LIMIT
    } catch(e) {
      if (LIMIT < retry) {
        setState({ errMsg: e.response.message }) // エラー処理 (2)参照
        throw 'Error: 試行回数10超え'
      }
      : await new Promise(resolve => setTimeout(resolve, 500))
    }
  }
}
```
ページを移るときにクエリパラメータを生成する。
```ts:Table.tsx
import { useHistory } from 'react-router-dom'
const history = useHistory()
useEffect(getFeed, [page])

<button onClick={()=>history.push(`?page=${page+1}`)}>進む</button>
<button onClick={()=>history.push(`?page=${page-1}`)}>戻る</button>
```
### (6) 検索
> vte.cxクエリパラメータ

- 検索結果が履歴に残るようにすること。クエリパラメータに検索句を管理させる↓
`https://{ドメイン+ホスト}/{エンドポイント}/table?page={ページ数}` `&search={検索句}`
- (5)の3つのaxiosリクエストのクエリパラメータに検索用パラメータを追加します。
1. 総数を取得するクエリパラメータ
`'/d/{エンドポイント}?c` `&{検索用パラメータ}'`
2. カーソルを作るクエリパラメータ
 `'/d/{エンドポイント}?_pagination=${cursorEnd.current},${ page + RANGE - 1 }&l=${LENGTH}` `&{検索用パラメータ}'`
3. フィードを取得するクエリパラメータ
`'/d/{エンドポイント}?n=${page}&l=${LENGTH}` `&{検索用パラメータ}'`
```diff ts:Table.tsx
  const query = new URLSearchParams( useLocation().search )
- const page = Number( query.get('page') ) || 1
+ const _page = Number( query.get('page') ) || 1

+ const search = query.get('search') || '' // URL'https://.../table?search={検索句}'の{検索句}を取得する。
+ const searchRef = useRef('')

  const getFeed = () => {
+   if ( search !== searchRef.current ) { // 検索句が新しいとき
+     cursorEnd.current = 1 // カーソルを新しくします。
+     searchRef.current = search
+   }
   // search(→検索句)から検索用パラメータを生成する
+   const search_name = `user.name-rg-.*${search}.*`
    fallback(()=>{
-     const count = await axios.get('d/{エンドポイント}?c')
+     const count = await axios.get(`d/{エンドポイント}?c&${search_name}`)
    // ...
```
vte.cx検索用パラメータの詳細は[歩き方の4章](https://qiita.com/stakezaki/items/e83dd86f1db5833e6ac1)を参照。

↑ vte.cxエンドポイントへのリクエストのクエリパラメータの値(→検索用パラメータ) を
↓ 検索バーの入力値→URLのクエリパラメータsearchの値(→検索句)　から生成すること。
```ts:Table.tsx
const history = useHistory()
const [search, setSearch] = useState('')
const handleSearch = () => history.push(`?page=1&search=${search}`)
useEffect(getFeed, [page, search])

<input value={search} onChange={setSearch} />
<button onClick={handleSearch}>検索</button>
```
![](/images/crud_table.png)
完成
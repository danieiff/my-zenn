---
title: "[メモ]import * as React from 'react'"が不要-React17"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript"]
published: false
---

:::message alert
vte.cxのサーバーサイドJS実行環境**nashorn**が、以下のtsconfigの`react-jsx`,`react-jsxdev`を通して出力されるコードに
対応していないようだ。
:::
#### 概要

React17-JSXの変換に`import * as React from 'react``が不要になった｡  [まとめ](https://zenn.dev/uhyo/articles/react17-new-jsx-transform)
---
以下の通りに設定する
1. tsconfig.json
2. jest.config.js

### 1. tsconfig.json
```ts:tsconfig.json
{
  "compilerOptions": {
    // ...
    "jsx": "react-jsx" // ←production, development→ "react-jsxdev"
  }
```
react-jsx, react-jsxdev
https://github.com/microsoft/TypeScript/pull/39199

:::detailed tsconfigをproduction, developmentで分ける
```js:webpack.config.js
module.exports = (_, argv) => {
  return {
    module: {
      rules: [
        {
          test: /\.tsx?$/,
          use: {
            loader: "ts-loader",
            options: {
              transpileOnly: true,
              configFile:
                argv.mode === "production"
                  ? "webpack.tsconfig.prod.json"
                  : "webpack.tsconfig.json",
            }
          }
        }
      ]
    },
    resolve: {
      extensions: [ ".js", ".ts", ".tsx" ]
    }
  }
}
```
```:webpack.tsconfig.prod.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "jsx": "react-jsx"
  }
}
```
```:webpack.tsconfig.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "jsx": "react-jsxdev"
  }
}
```

:::
既存コードのreact import文を消す
`npx react-codemod update-react-imports`


### 3. jest.config.js
[jestの設定](https://zenn.dev/panyoriokome/scraps/78aecf55ba5a38
)
```js:jest.config.js
module.exports = {
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json'],
  globals: {
    'ts-jest': {
      tsconfig: {
        jsx: 'react-jsx',
      },
    },
  },
}
```
:::detailed eslintエラーが出たら
```.eslintrc.json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript",
    "plugin:react/recommended",
    "plugin:react/jsx-runtime",
    "prettier"],
  ]
}
:::
以上
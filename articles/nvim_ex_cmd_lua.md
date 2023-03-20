---
title: "Vim/NeoVim ExCommand ':lua' で '|'(bar) は使えない"
emoji: "📘"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim"]
published: false
---

ExCommandでの`|`(bar)の挙動について `:help :|`
`:echo "foo" | pwd` →通常通り
`:lua print("foo") | pwd` →エラー

### `:lua` の内部の仕組み
- Ex Commandの定義一覧 https://github.com/neovim/neovim/blob/master/src/nvim/ex_cmds.lua#L1607-L1612
```lua
module.cmds = {
  {
    command='lua',
    flags=bit.bor(RANGE, EXTRA, NEEDARG, CMDWIN, LOCK_OK),
    addr_type='ADDR_LINES',
    func='ex_lua',
  },
}
return module
```
`flags=bit.bor(RANGE, EXTRA, NEEDARG, CMDWIN, LOCK_OK)` ExCommand`:lua`のフラグが設定される。引数(RANGE...)はそれぞれビット列。XOR演算されてメモリに格納される。 **`見ての通りTRLBAR`は設定されてない**

### `TRLBAR`フラグが無効のExCommand
`|`がコマンド分割として働かない(=`|` 自体がExCommandの引数の一部として扱われる)

内部処理の続き
- ↑↑`func='ex_lua'`... ExCommand`:lua`で`ex_lua関数`が呼ばれる https://github.com/neovim/neovim/blob/master/src/nvim/lua/executor.c#L1610-L1639

- ↑↑ から`build/src/nvim/auto/ex_cmds_defs.generated.h`を生成するスクリプト ttps://github.com/neovim/neovim/blob/master/src/nvim/ex_cmds.lua 
(NeoVim コンパイル時に呼ばれる。 https://github.com/neovim/neovim/blob/master/src/nvim/CMakeLists.txt#L263)
```lua
local defsfname = autodir .. '/ex_cmds_defs.generated.h'

local ex_cmds = require('ex_cmds')
local defs = ex_cmds.cmds

for _, cmd in ipairs(defs) do
  defsfile:write(string.format([[
  [%s] = {
    .cmd_name = "%s",
    .cmd_func = (ex_func_T)&%s,
    .cmd_preview_func = %s,
    .cmd_argt = %uL,
    .cmd_addr_type = %s
  },
]], enumname, cmd.command, cmd.func, preview_func, cmd.flags, cmd.addr_type))
end
```
生成された ExCommand "lua"の定義 (`build/src/nvim/auto/ex_cmds_defs.generated.h`)
```h
  [CMD_lua] = {
    .cmd_name = "lua",
    .cmd_func = (ex_func_T)&ex_lua,
    .cmd_preview_func = NULL,
    .cmd_argt = 17301637L,
    .cmd_addr_type = ADDR_LINES
  },

```

- `parse_cmdline`でフラグごとの処理が定義されている https://github.com/neovim/neovim/blob/84027f7515b8ee6f818462f105882fc0052783c4/src/nvim/ex_docmd.c#L1375-L1385
(NeoVimコマンドラインへの入力をパースする`nvim_parse_cmd`の中で呼ばれる https://github.com/neovim/neovim/blob/master/src/nvim/api/command.c#L113)
フラグ`TRLBAR`の処理 https://github.com/neovim/neovim/blob/84027f7515b8ee6f818462f105882fc0052783c4/src/nvim/ex_docmd.c#L1479-L1483
```c
  // Initialize eap
  *eap = (exarg_T){
  .line1 = 1,
  .line2 = 1,
  .cmd = cmdline,
  .cmdlinep = &cmdline,
  .getline = NULL,
  .cookie = NULL,
  };

  // Parse arguments.
  if (!IS_USER_CMDIDX(eap->cmdidx)) {
  eap->argt = cmdnames[(int)eap->cmdidx].cmd_argt;
  }

  // Check for '|' to separate commands and '"' to start comments.
  // Don't do this for ":read !cmd" and ":write !cmd".
  if ((eap->argt & EX_TRLBAR)) {
  separate_nextcmd(eap);
  }
```
↓ T`RLBAR`が`|`の扱いを決めていることが分かった
```c:src/nvim/ex_cmds_defs.h
#define EX_TRLBAR          0x100u
```
```lua:src/nvim/ex_cmds.lua
local TRLBAR     =     0x100
```

---

- luaコマンド定義のflagsに`TRLBAR`を追加してNeoVimをビルドすると望んだ`|`の動作になった
`++ flags=bit.bor(RANGE, EXTRA, NEEDARG, CMDWIN, LOCK_OK, TRLBAR)` 

- ユーザー定義コマンドでは、これらのフラグを設定できる

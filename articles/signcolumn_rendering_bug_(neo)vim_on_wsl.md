---
title: "WSL Windows Terminal上のNeoVimで行のテキストが画面上に重なり合うバグ"
emoji: "📘"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["windows", "wsl", "windows terminal", "vim", "neovim"]
published: false
---

>> Windows Terminal最新版で解決 https://github.com/microsoft/terminal/releases/

## 症状

行のテキストが画面上に重なり合い、読みにくくなる。

## 原因

Vim/NeoVimのVirtual Textが行末に到達した場合に、画面の右側に存在するテキストの描画が正しく処理されないことが原因

### 公式Repositoryで修正までの流れ

- バグ報告される https://github.com/microsoft/terminal/issues/6865
- 原因特定される https://github.com/microsoft/terminal/issues/6987#issuecomment-1433461701
- Collaborator様による修正がmerge https://github.com/microsoft/terminal/pull/14735
- 2/27 Release https://github.com/microsoft/terminal/pull/14735

## 対応

1. 最新(3週間前)のWindows Terminal(https://github.com/microsoft/terminal/releases/tag/v1.16.10261.0)をインストールする

2. VimをWindows Terminal以外のターミナルエミュレーターで使用する。

3. Vimの設定を変更し、Virtual Textを出力しないようにする

- 効果なかったもの
Windows Terminalの各「描写設定」on/off

---

- ちょうど3週間前に修正してくださっていてtskr
- 日本語入力中に頻繁に落ちるバグは続いている

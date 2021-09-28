---
title: 如何避免 Commit Message 拼錯字？
date: 2021-09-29 01:43:50
categories: [tips]
tags: [vim, git, spell]
---

文件打錯字還好，隨時可以修。
但在 commit message 中打錯字，可是會流傳千古。

身為一個 RD，有個極簡的解決方法..
把底下這行設定放到 `.vimrc` 內即可。 (O

```
autocmd FileType gitcommit setlocal spell
```

(對於屬 Git commit message 的 buffer 自動啟用 spell check 功能)

設定完之後，當出現 `vim` 不認識的單字時，就會有醒目的顏色提示，
提醒自己該回頭看一下是不是又拼錯字了。

當然，要有舒適的拼字檢查體驗，字典檔的維護也是很重要的一環。
不過那又是另一個話題了..

## Reference

- [Vim Spell-Checking](https://thoughtbot.com/blog/vim-spell-checking)
- [Vim: spell.txt](https://vimhelp.org/spell.txt.html#spell)

---
title: "利用 GitHub Page 經營 Blog"
date: 2019-01-12 08:00:00
categories: [notes]
tags: [github, blog, jekyll]
---

如果要用一句話來簡單說明 GitHub Page，那基本上就是

> 指定一個 Git 版本庫來作為存放網站資源的地方，然後讓 GitHub 幫你把網站架起來。

任何人只要申請一個 GitHub 帳號，都可以免費的享有這個服務。

當然，考量到 GitHub 只是把我們放在版本庫上的檔案，讓別人透過瀏覽器瀏覽，
那種需要用到資料庫的可互動網站基本上是很難達成。
但若我們只是要經營一個部落格，或是存放專案文件等靜態網站時，GitHub Page
就會是個很合適且方便的選擇。

本文會粗淺的介紹如何利用 GitHub Page 來經營自己的 Blog，
以省去自行架設機器的各種煩惱。 :D

## GitHub Page 的類別

目前 GitHub Page 有兩類的站台，一類是 User Page、另一類是 Project Page。
(其實還有 Organization Page，但這邊就不花時間贅述) User Page 與 Project Page
最主要的差別在於專案名稱的限制，與網站的 URL 格式這兩點。簡單整理如下

User Page 特點:

- 專案名稱須為 `<username>.github.io`，其中 `<username>` 即為 GitHub 帳號的使用者名稱。
- 站台會擺在 `http(s)://<username>.github.io` 供他人瀏覽

Project Page 特點:

- 專案名稱沒有限制。 若假設專案名稱為 `<projectname>` 則
- 站台會擺在 `http(s)://<username>.github.io/<projectname>` 供他人瀏覽

因為專案名稱的限制，一個 GitHub 帳號只能有一個 User Page 但可以有多個 Project Page。

更詳細的介紹請參考 [官方網站的說明](https://help.github.com/articles/user-organization-and-project-pages/)

## GitHub Page 使用方式

GitHub Page 的使用方式也可以簡單分成兩種。

第一種是直接建置好的整個網站直接 push 到 GitHub 上，供使用者瀏覽。
若我們需要架一個 Blog，可以先用 Markdown 等 markup language 撰寫文章，
之後利用 Jekyll(Ruby)、Hugo(Golang) 或 Hexo(JS) 等靜態網站生成工具，
建出一個 Blog 網站並 push 上去。
又或我們需要 host 一個專案文件站台時，可以將 Doxygen 或 Sphinx 等工具
產生出的網站推上 GitHub。

```
                                      +
            Local Project             |    GitHub Project
                                      |    github.io site
                                      +
  +----------+          +--------+         +------+  User
  | Markup   |  Build   | Site   |  Push   | Site |  Browse
  | config.. | +------> |        | +-----> |      | +------->
  +----------+          +--------+         +------+
```

如果我們是使用 Jekyll 來建置我們的網站，那 GitHub Page 有提供我們第二種用法。
我們可以將 Markup 和其他 Jekyll 需要的設定檔 push 上 GitHub，讓 GitHub 幫我們
建置網站，並在 github.io 網域上放出網站供人瀏覽。

```
                 +                    +
  Local Project  |   GitHub Project   |    github.io site
                 |                    |
                 +                    +
  +----------+        +----------+         +------+  User
  | Markup   |  Push  | Markup   |  Build  | Site |  Browse
  | config.. | +----> | Config.. | +-----> |      | +------->
  +----------+        +----------+         +------+
```

第二個做法的缺點是，GitHub 只支援 Jekyll 這套工具，其他同性質的工具的不支援。
但相對來說也有優點，即是我們不須把工具建出的網站內的所有檔案都進到 commit 中。
(在 git project 中看到許多無意義 diff 實在不是工程師所樂見的事情 XD)

針對這兩個方式的更詳細說明，也請見官方文件

- [使用 Jekyll](https://help.github.com/articles/about-github-pages-and-jekyll/)
- [不使用 Jekyll](https://help.github.com/articles/using-a-static-site-generator-other-than-jekyll/)

## 簡單的流程說明

接下來會介紹使用 Jekyll，並讓 GitHub 幫忙 build 與 host 網站的簡單步驟。

參照 [官方介紹](https://pages.github.com/) 的說明，最簡單的方式，其實只需要
我們點開 project 的 GitHub 設定頁面，找到 GitHub Page 的設定選項，設定一個
Jekyll 使用的主題，並用 Markdown 寫一個首頁文章即可。

用此方法會在專案內產生首頁的 `index.md` 檔案及一個 Jekyll 的設定檔 `_config.yml`。
檔案內僅一行你選的主題名稱

``` yaml
theme: jekyll-theme-minimal
```

但基本上，一個 [完整的 Jekyll 專案](https://jekyllrb.com/docs/structure/)
不會只有這兩個檔案，到最後我們還是得把其他需要的檔案生出來。
所以個人推薦使用下述方法建立我們的專案。

(假設我們已經裝好 Git 和 Jekyll 等工具。)

建立 Git 專案

``` sh
mkdir website && cd website
git init
```

在專案資料夾建立 Jekyll 的 template 檔案

```
jekyll new .
```

此時應該會看到 jekyll 預設產生的檔案

```
$ ls
404.html  about.md  _config.yml  Gemfile  Gemfile.lock  index.md  _posts
```

將所有產生的檔案 add 並 commit 起來
(要不要略過 `Gemfile.lock` 看個人需求)

``` sh
git add .
git commit
```

之後將專案 push 上 GitHub，並至專案設定內啟用 GitHub Page 即可。
沒意外的話，大概十秒內就可以在對應的 URL 看到生成好的網站了。

有關 Jekyll 的安裝說明或其他細部設定，可參考 [官方網站](https://jekyllrb.com/)。

## 在 GitHub Page 服務上使用個人客製的網址

如果不想使用 `<username>.github.io` 來提供自己的網站，而是透過自己購買的域名，
所需的麻煩差事 GitHub Page 也幫我們做得好了。

在 GitHub 專案開啟 GitHub Page 功能後，可以看到一個額外的選項 Custom domain，
可以填入我們可控制的 DNS hostname。

假設我們想在 `blog.example.com` 提供我們的網站，只需要在 DNS 設定中加入一筆
`CNAME`，將 `blog.example.com` 指向 `<username>.github.io`。並去 GitHub Page
所用的 GitHub 專案設定頁面內，在 Custom domain 欄位內填入 `blog.example.com` 即可。

設定完後，即可透過 `blog.example.com` 瀏覽我們要的網站。
同時 GitHub 也會在一天內生出對應的 SSL 憑證，即使透過 `blog.example.com` 瀏覽，
也可以享有 HTTPS protocol 帶來的安全性。 :D

## 雜談

大概從 2018 十月開始，小弟我在與朋友以及公司同事談話後，漸漸有了經營自己 Blog 的想法。

經歷了數週的拖拉散漫後，終於在 2018 十二月底刷卡買了自己的 domain，並利用
GitHub Page 架設好 Blog。但因為一直沒想好要寫什麼文章，於是第一篇就先來寫寫
我自己的架站筆記。

期許自己未來能不斷產出新文章，成為一位散發正面能量的一倍工程師。

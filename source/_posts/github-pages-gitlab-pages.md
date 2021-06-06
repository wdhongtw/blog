---
title: GitHub Pages 與 GitLab Pages 架設 Blog
date: 2021-06-06 21:00:00
categories: [notes]
tags: [github, gitlab, blog]
---

筆者最近把個人 blog 的產生工具從 GitHub Pages 預設的 [Jekyll](https://jekyllrb.com/)
換成 [Hexo](https://hexo.io/)，有了一點心得。 而且不只 GitHub Pages，
筆者在公司業務中也有大量使用 GitLab Pages 來產生文件及測試報表，算是有累積不少經驗。

趁著印象還深刻時，寫點筆記，替這兩個相同性質的服務做基本的介紹。

## Pages 服務與 Static Site Generator

GitHub / GitLab Pages 可以將一組靜態網頁內容 (html, css, js 等)，透過 GitHub / GitLab
的伺服器，host 在某個 URL 底下。 網頁產生工具 (Static Site Generator, 下稱 SSG) 則是
一個可以將用 Markdown 撰寫的文章，轉化成漂亮的靜態網頁內容的工具。常見的 SSG 有 Jekyll(Ruby), Hugo(Go),
Hexo(JavaScript) 等。

若將 SSG 工具與 GitHub / GitLab Pages 服務，搭配使用，
**寫作者只需要寫寫簡單的 Markdown 並 push commit，就能得到一個漂亮的 blog 或是文件網頁。**
筆者的個人 blog 及公司的工作筆記即是使用這類流程架設。

整體流程大概如下圖所示:

```
                +   GitHub            +     github.io
 Local Project  |          Project    |               site
                |   GitLab            |     gitlab.io
                +                     +

 +----------+        +----------+  Build &  +------+  User
 | Markup   |  Push  | Markup   |  Deploy   | Site |  Browse
 | config.. | +----> | Config.. | +-------> |      | +------->
 +----------+        +----------+           +------+
```

## GitHub Pages

[GitHub Pages](https://pages.github.com/) 基本上會有兩種主要的使用方式。
可以直接使用 GitHub Pages，或是透過 GitHub Pages 的 Jekyll 整合功能。
前者需要的技術背景與設定步驟均較複雜，後者較簡單但缺少了根據個別需求調整的機會。

### Native GitHub Pages

若直接使用 GitHub Pages，使用方式是: 將 SSG 產生的網頁擺放至某 branch (預設為 `gh-pages`)
的 `/` 或 `/docs` 目錄。 每次該 branch 被更新時，GitHub 就會將最新版本的網頁內容，
呈現在 `https://<username>.github.io/<project>` 連結下。

早期這個 push brach 的動作是蠻麻煩的，但後來有了 GitHub Action 之後，
產生網站和後 push branch 的動作都可以在 GitHub 提供的環境完成，非常方便。

筆者個人使用的 job 描述檔如下:

```yaml
# .github/workflows/blog.yaml

name: build-and-deploy-blog

on:
  push:
    branches: [ "master" ]
  pull_request:

jobs:
  blog:
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout
        uses: actions/checkout@v2.3.4
      - name: Setup Node.js environment
        uses: actions/setup-node@v2.1.5
        with:
          node-version: 12.x
      - name: Install dependent packages
        run: npm install
      - name: Build blog posts
        run: npm run build
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

若不想使用 GitHub 提供的 domain，也可以參照
[官方文件](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site)，
使用自己購買的 domain 來架設網站。
只要設定完成，GitHub 也可以一併幫使用者申請 custom domain 需要的 HTTPS 憑證。

比方說筆者的 blog 原本可存取的位置應是 `https://wdhongtw.github.io/blog`，但有設定
custom domain 後，目前是透過 `https://blog.bitisle.net` 來存取架在 GitHub Pages 上的 blog。

![](/assets/images/f0b7ac13-d964-496f-ab78-1fed9a8a742b.png)


### GitHub Pages Jekyll

前述 (Native) GitHub Pages 的使用方式會需要自己 push branch。
但若 GitHub 偵測到 project 使用的 SSG 是 Jekyll，GitHub 會自動處理產生網頁以及
後續部屬到 `https://<username>.github.io/<project>` 的工作。
(連 `gh-pages` branch 都省了，整個 project 會非常乾淨。)

此方法相當容易上手，也是 GitHub Pages 教學文件預設的使用方式。但因為產生網站的
環境是由 GitHub 協助處理，能使用的 Jekyll plugin 自然也是受到限制。

說是受到限制，但以一般使用情境應該也很夠了。諸如 RSS feed, sitemap, Open Graph metadata
等現代 blog 必備的功能都有自帶。
若想知道有那些 Jekyll plugin 可用，可至 [Dependency versions | GitHub Pages](https://pages.github.com/versions/)
查閱。

### Compare Native GitHub Page with GitHub Pages and Jekyll

簡單比較上述兩種方式如下

|               | GitHub Page         | GitHub Page with Jekyll    |
|---------------|---------------------|----------------------------|
| SSG Tool      | on your choice      | only Jekyll                |
| Deployment    | push generated site | done by GitHub             |
| Customization | any plugins of SSG  | limited Jekyll plugin list |

## GitLab Pages

[GitLab Pages](https://about.gitlab.com/stages-devops-lifecycle/pages/)
與 GitHub Pages 一樣，有將 SSG 產生的網頁 host 在特定網址的能力。
以 GitLab 官方 host 的站台來說，網站會放在 `https://<username>.gitlab.io/<project>` 下。
(私人或公司 host 的 GitLab instance 就要看各自的設定)

與 GitHub 不同的是，GitLab Pages 並不是透過 push branch 的方式部屬，
且沒有針對特定 SSG 提供更進一步的自動部屬功能。

GitLab Pages 的使用方式是基於 GitLab CI Pipeline 的架構設計的。若想要部屬
網站，一般的使用方式是在 pipeline 內產生網頁，接著將網頁內容擺放至為特定 job (`pages`) 的特定
artifacts 目錄 (`public`) 中。 一旦有 pipeline jobs 達成此條件。 GitLab 就會
把網頁內容部屬到對應的網址下。

筆者個人使用的 job 描述檔如下:
(因為 GitHub Action 與 GitLab CI 的架構差異，寫起來比較簡潔)

```yaml
# .gitlab-ci.yml
pages:
  image: ruby:3.0.1
  stage: deploy
  before_script:
    - bundle install
  script:
    - bundle exec jekyll build --destination public --baseurl "$CI_PAGES_URL"
  artifacts:
    paths:
      - public
  only:
    - master
```

至於其他 GitHub Pages 有的功能，如 custom domain，自動申請 HTTPS 憑證等，GitLab Pages
也都有。 記得去 project 設定頁面設定即可。

![](/assets/images/49f9611b-3d40-4425-beb6-317aeae0aa6e.png)

## Conclusion

在 2008 年 GitHub Pages 剛推出時，使用者都要自己手動 push `gh-pages` branch。
後來 GitLab 推出基於 GitLab CI 的 Pages 服務之後，GitHub Pages
使用體驗相較之下可說是非常糟糕。

但後來隨著 GitHub Actions 服務推出，以及社群維護的高品質
[Pages 部屬 Action](https://github.com/marketplace/actions/github-pages-action) 出現。
GitHub / GitLab Pages 的使用體驗已經變得相當接近。
其他像是 custom domain 以及 HTTPS 的支援也都是免費的基本功能。

基於上述原因，許多早期的 [比較文章](https://mjswensen.com/blog/github-pages-vs-gitlab-pages/)
其實已經沒什麼參考價值。 若現在想架設新的 blog 等站台，只要選擇自己習慣的平台即可。

## References

- [GitHub Pages Documentation - GitHub Docs](https://docs.github.com/en/pages)
- [GitLab Pages | GitLab](https://docs.gitlab.com/ee/user/project/pages/)

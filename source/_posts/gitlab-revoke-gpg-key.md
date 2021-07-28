---
title: GitLab 更換自家 GPG Key
date: 2021-06-17 11:10:00
categories: [news]
tags: [gitlab, gnupg]
---

今天 GitLab 在自家 blog 上公告 revoke 簽署 package 的 GPG key。

> We recently became aware of an instance where this key and other tokens
> used to distribute official GitLab Runner packages and binaries were not
> secured according to GitLab’s security policies.

> We have not found any evidence of unauthorized modification of the packages
> or access to the services storing them.

並不是因為 key 被 compromise，僅是因為 key 不符合公司的安全規範，所以就進行了一次 rekey。

GPG key rekey 並不如換憑證一樣，只要重簽一張就好 (因為信賴建立在已知的第三方 CA 上)。
GPG key rekey 需要透過可信管道重新宣告 fingerprint 並請大家 import 新的 key。
這個轉換的成本，相較換憑證應是高非常多且難以量化的。

沒想到居然僅為了不合安全規範就進行 rekey，不愧是國際一線的軟體公司！

See: [The GPG key used to sign GitLab Runner packages has been rotated | GitLab](https://about.gitlab.com/blog/2021/06/16/gpg-key-used-to-sign-gitlab-runner-packages-rotated/)

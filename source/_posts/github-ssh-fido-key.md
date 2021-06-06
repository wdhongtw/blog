---
title: GitHub 即日起支援使用 Security Key 進行 Git 操作
date: 2021-05-11 08:00:00
categories: [notes]
tags: [github, fido, git]
---

GitHub 開始支援使用 security key 進行 Git 操作啦！

這應該是各家科技巨頭當中，第一個支援 security key 進行 SSH login 的服務吧。
筆者昨天 5/10 才在公司內分享如何使用 security key 來做 SSH login，
沒想到 Yubico 和 GitHub 也剛好在昨天一起同步更新 blog 文章，通知大家這個新功能。
喜極而泣..

- [Security keys are now supported for SSH Git operations - The GitHub Blog](https://github.blog/2021-05-10-security-keys-supported-ssh-git-operations/)
- [GitHub now supports SSH security keys - Yubico](https://www.yubico.com/blog/github-now-supports-ssh-security-keys/)

以下簡單介紹如何使用這個新功能。
為了方便解說及避免誤會，後述內容均以正式名稱 authenticator 代稱 security key。

## 如何使用

首先當然要有一把 authenticator，如果還沒有，趕快去買一把囉。 :D

**第一步**，在 authenticator 內產生新的 key pair。

產生 key pair 的流程和傳統放在檔案系統上的差不多，只是 key type 要指定代表 authenticator
的 type。

```shell
ssh-keygen -t ecdsa-sk
```

產生的過程，根據不同的 authenticator，會有要求按一下 且/或 輸入 PIN code 驗證身分。
此步驟會在 authenticator 內產生一組 key pair，並將 public key 寫到檔案系統上。
平常放 private key 的那個檔案還是會產生，不過這次裡面放的會是一個 key handle。

**第二步**，透過 GitHub 的 web UI 上傳 public key

上傳完之後，沒意外就可以順利使用了，可以試著 clone 一些 project

```
$ git clone git@github.com:github/secure_headers.git
Cloning into 'secure_headers'...
Confirm user presence for key ECDSA-SK SHA256:........
User presence confirmed
```

若確認 OK，之後的 Git 操作都可以透過隨身攜帶的 authenticator 保護，
只要按一下 authenticator 即可。

設定完之後，**若沒有拿出 authenticator，就不能進行 push / pull 操作。**
只要確保 authenticator 還在身上，就可以安心睡大覺。
(再也不用擔心 private key 放在公司電腦上會被摸走了！)

## 進階使用方式

FIDO 2 authenticator 博大精深，除了上述的基本使用流程外，還有些細節可以設定。

以下分別介紹三個進階使用技巧：

### 要求身分驗證 (User Verification) 才能使用 Authenticator

根據每個人平常保管鑰匙習慣的差異，可能有人會擔心 authenticator 真的被摸走。

但 authenticator 也有支援一定要驗甚身分 (e.g. PIN code or 指紋辨識) 後才能
使用內部的 key pair 的功能。

要如何使用呢？ 只要在產生 key pair 的過程中多下一個 flag 即可。

```shell
ssh-keygen -t ecdsa-sk -O verify-required
```

若之後要使用由此方式產生的 key pair，除了手指按一下 authenticator 之外，還會要求
使用者輸入 PIN code 才能順利完成操作。如此即使 authenticator 被偷走了，也不用太緊張。

### 全自動使用 Authenticator (避免 User Presence Check)

若把 authenticator 插上電腦後，想要隨時隨地都進行 push / pull，
但不要每次都手按一下 authenticator。這種自動化的使用情境 OpenSSH 其實也是有支援的。

使用的方式也是在產生 key pair 時多帶一個參數。

```shell
ssh-keygen -t ecdsa-sk -O no-touch-required
```

同時在將 public key 部屬到目標機器上時，在 public key 該行前面多下 `no-touch-required`
即可。 (詳情請見 `ssh-keygen(1)` 及 `sshd(8)`)

以此方始產生的 key pair 在使用時就不需要每次都手按一下，可以全自動使用。

不過，雖然 OpenSSH 有支援此種使用情境，**但目前 GitHub 禁止這種使用方式**。

節錄自上述 blog 文章

> While we understand the appeal of removing the need for the taps,
> we determined our current approach to require presence and intention
> is the best balance between usability and security.

所以在 GitHub 上，若想要全自動操作，只能回去用一般的 SSH key 或 API token 囉。

### 避免手動複製 Key Handle

前面有提到，原先檔案系統上用來放 private key 的檔案會變成拿來擺放 key handle。

這意味著，當我們想在新的機器上透過 SSH 進行 Git 操作時，除了拿出 authenticator 之外，
也需要把原先的 key handle 檔案複製到新的機器上。
且若 key handle 檔案掉了，該組 key pair 就不能使用了。

若要避免此問題，就要用上 authenticator 的另一個進階功能
discoverable credentials / resident keys 。

```shell
ssh-keygen -t ecdsa-sk -O resident
```

使用此類型的 key pair 時，會在 authenticator 上消耗一些儲存空間。但換來的好處是，
使用者可以在新的機器上，把 key handle 從 authenticator 內抽出來。

```shell
ssh-keygen -K # extract discoverable credentials
```

如此就不用手動複製擺放 key handle 的 private key 檔案了。

但要注意的是此類型 key pair 會消耗 authenticator 的空間。
每把 authenticator 可以放多少 key pair 要再自行查閱官方文件。

## 結語

以上介紹了基本使用情境及三種可能的進階使用方式。

筆者在 2014 年第一次注意到 FIDO (U2F) 標準，當時就在想像沒有密碼的世界。
如今，藉由 FIDO 2 security key 的普及，當初所想的美好願景似乎在慢慢地實現中..

希望未來能看到 security key 運用在更多場景上！

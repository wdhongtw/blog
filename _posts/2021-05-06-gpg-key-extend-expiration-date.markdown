---
layout: post
title: 延長 / 縮短 GPG 金鑰的過期時間 (Expire Time)
---

筆者在 2018 年的時候開了第一個真的有在長期使用的 GPG 金鑰。

因為年少輕狂不懂事，當時特別把 primary key 設定成永遠不會過期。
但演算法可能在未來被發現漏洞，電腦的運算能力也會越來越好，
一把不會過期的 GPG 金鑰是先天上不合理的存在。
考量到此問題，筆者後來又再修正了該金鑰的過期時間，
以及整理這篇筆記...

## GPG Key 可以延展過期時間？

我想這應該是熟悉 X.509 憑證生態系的人最為驚訝的一件事情了。

我發現幾位公司主管並不知道這件事情，也促使我在經過一段時間後回來整理這篇文章。
事實上，GPG key 不只是可以延展過期時間，這也是一般推薦的最佳慣例。

> People think that they don’t want their keys to expire,
> but you actually do. Why? Because you can always extend your expiration date,
> even after it has expired!

使用者應該設一個較短的有效時間，並在後續有需要時延展過期時間。
See: [OpenPGP Best Practices - riseup.net](https://riseup.net/en/security/message-security/openpgp/best-practices#use-an-expiration-date-less-than-two-years)

GPG key 可以自己修改金鑰的過期時間，是因為 GPG key 和 X.509
憑證有著本質上的區別。

> GPG key 的產生是透過 primary key 的 self-signature，
> 而 X.509 憑證的簽署是由公正的第三方 CA 進行。

X.509 憑證的過期時間是 CA 幫你簽署憑證時決定，自然無法隨意修改，
大家也很習慣這件事情，但 GPG key 就不一樣了。
GPG key 的有效時間是透過 key 的 self-signature 內所記載的時間決定。
只要 primary (private) key 沒有遺失，持有者隨時可以重新自簽並修改時間。

只要認知到兩者本質上的差異，可以修改過期時間這件事情也就很好理解了。

## 他人如何認定過期時間？

既然 GPG key 可以隨時重簽修改過期時間，那對他人來說，
該如何判定某把 key 究竟什麼時候過期呢？

規則很簡單

> The latest self-signature takes precedence

See: [Key Management](https://www.gnupg.org/gph/en/manual/c235.html)

若是透過 `gpg` tool 修改過期時間，舊的 self-signature 會被刪掉。
因為只有一個 self-signature，修改完之後，只要重新把 key export 給他人，
他人就可以知道新的過期時間。

若不是透過信賴管道直接把新簽的 key 給他人，而是透過 GPG key server，
狀況會有點不一樣。

基於安全考量，GPG key server 是不允許部分或完全刪除 key 的，MIT 名下的 key server
還特別寫了一篇 [FAQ](https://pgp.mit.edu/faq.html) 來說明這件事。
對於一把已存在的 key，使用者只能推新的 sub key 或新的 signature 上去。

因此，他人透過 key server 取得 key 時，也會拿到多個 signature。
好在 signature 本身也有時戳，根據上述 "後者為準" 的規則，他人就可以知道
正確的過期時間是何時。

有興趣的可以查看筆者的 GPG key 來確認這個行為

```shell
gpg --keyserver keys.gnupg.net --recv-keys C9756E05
# Get key from key server

gpg --export C9756E05 | gpg --list-packets
# One signature has "key expires after ..." while another doesn't

gpg -k C9756E05
# Validate that the key indeed expires at some time
```

或是可以直接去 GnuPG 官方的 key server 查看:
[Search results for '0xc728b2bdc9756e05'](http://keys.gnupg.net/pks/lookup?op=vindex&fingerprint=on&search=0xC728B2BDC9756E05)

## 結語

翻閱文件研究的過程，慢慢感受到到 GPG 這個扣除 X.509 之外唯一成熟的 PKI
生態系，究竟有多麼偉大。同時也看到很多值得細讀的 guideline 文件。

若有時間，真的該來好好吸收整理。

---
layout: post
title:  "關於電子報一鍵退訂"
---

## 前言

前先日子在 Gmail 內整理信件時，赫然注意到了一件事情：

某些訂閱來的電子報，在寄件者名字的右邊有個退訂的文字可以按！
但讓我訝異的其實不只是這個退訂文字..  按下退訂文字後，Gmail 會彈出一個視窗給我，裡頭會問我是不是確定要退訂此電子報，以及一個藍色的退訂按鈕。 神奇的是，當我按下這個按鈕後，Gmail 直接告訴我，你已經退訂了此電子報！ "You unsubscribed from xxxxxx.."

![](/assets/images/6f64025c-3482-48f7-a029-e459061a200f.png)

![](/assets/images/d068304f-8a52-49cd-ab90-9c1a9e2b2624.png)

某人: 奇怪... 一般不是會連到某個電子報寄送方的頁面，然後讓我點個確認之類的嗎？

既然 Gmail 有辦法在不離開 web mail 介面的情況下，幫我完成退訂的動作，那大概是有某種標準程序，可以讓 mail 供應商自動化的幫我處理吧？ 於是乎，本著實事求是的精神，就有了今天這篇文章。

## 工程師的直覺

注意到有這個退訂按鈕可以按之後，我就開始一封信一封信看，有退訂按鈕的都來給他點看看，看會發生什麼事。但我很快地就注意到... 並不是每個有按鈕可按的 Gmail，都可以直接幫我完成退訂。 像是 Linkin 和 Google Map 寄來的信，都是會幫你打開某連結，讓你到該頁面去做後續處理。

![](/assets/images/e1bc655c-5396-4651-86d1-b7275e90b131.png)

但不管如何，既然 Gmail 可以幫我把這個連結取出來，大概是有某種標準的 mail header 來記錄這些資訊吧？把 Linkin 那封原始信件點開來一看，發現了這個東西

```
List-Unsubscribe: <https://www.linkedin.com/e/v2?e=6f948d7f&t=ad69-4827&...>
```

再把這個 `List-Unsubscribe` 拿去查，就找到一篇 RFC 了

```
Network Working Group                                      G. Neufeld
Request for Comments: 2369                                      Nisto
Category: Standards Track                                     J. Baer
                                                 SkyWeyr Technologies
                                                            July 1998


        The Use of URLs as Meta-Syntax for Core Mail List Commands
            and their Transport through Message Header Fields
```

RFC 2369 "The Use of URLs as Meta-Syntax for Core Mail List Commands"。仔細一讀後發現，以前的人為了用 mail 標準化 mail list 的各種處理動作，還下了不少苦心。 XD  光是這份 RFC 內有提到的部分 header 就有:

- `List-Help`: 所有和此 mail list 相關的資訊都從這邊取得
- `List-Unsubscribe`: 使用者快速退訂的方式
- `List-Subscribe`: 使用者快速訂閱的方式
- `List-Post`: 使用者發表文章至此 mail list 的方式

其中的 `List-Unsubscribe` 就是我們要的東西。參照 RFC 說明，這個 header 內可以放 HTTP 的連結或 `mailto` 的連結。這樣看起來，如果是 HTTP 連結，Gmail 就會幫我們連上該頁面。而那些沒有額外跳出頁面的，大概就是 Gmail 直接幫我們寄退訂信件了吧。

```
List-Unsubscribe: <http://www.host.com/list.cgi?cmd=unsub&lst=list>,
    <mailto:list-request@host.com?subject=unsubscribe>
```

### 開始驗證

為了確認上面的猜想，我又點了封有退訂按鈕且不會跳出額外頁面的信件。這次的實驗對象是 The Hacker News。在點擊退訂之後，在我的寄件信箱內找到 Gmail 自動幫我產生的信件，Gmail 不只有幫我寄這封信，連信件主旨和內文都幫我填好好的。 XD

```
# 按下退訂後，我寄出的信 (省略部分 header)
To: f081fcde-0c7e-4617-afaf-c0c35eeea170@unsubscribe.netline.com
Subject: Unsubscribe
Content-Disposition: inline

$You will be unsubscribed from this list within ten days of sending this reply
```

在回去看一下原本納封電子報的原始信件，果然在 header 內找到這對應的 `List-Unsubscribe` `mailto` 連結，且信件主旨和信件內文和這個連結後方帶的資訊完全吻合。

```
# The Hacker News 信件的 List-Unsubscribe header
List-Unsubscribe: <mailto:f081fcde-0c7e-4617-afaf-c0c35eeea170@unsubscribe.netline.com?subject=Unsubscribe&body=$You%20will%20be%20unsubscribed%20from%20this%20list%20within%20ten%20days%20of%20sending%20this%20reply>
```

看到這裡，似乎是真相大白了。參與制定 RFC 的人們真偉大！趴機趴機趴機！

用 HTTP(S) 連結做退訂的會開一個 HTTP Get 請求，讓使用者到某頁面按退訂；而那些用 `mailto` 連結的，mail agent 可以幫我自動寄出退訂信件，於是達成一鍵退訂的功能！

不過... 我剛剛好像也一鍵退訂了 Pinkoi 的電子報，但好像沒有看到自動寄出的信件？

## 案外案: RFC 8058

原本以為該懂得都懂了，一切就是那麼的單純，都在我的掌握之中。
直到我注意到 Pinkoi 的電子報 (X

在實驗的過程中，Pinkoi 也像 The Hacker News 一樣，可以在 mail 介面中直接完成退訂。但不一樣的是，mail 系統沒有自動幫我產生並寄出退訂用的信件。 ...看來這當中一定還有些我不知道的東西！

有了先前的經驗，這次很快地把 Pinkoi 電子報的原始信件打開來看，並直接搜尋 **unsubscribe** 字眼。一搜不得了，看到了一個沒在 RFC 2369 中出現的 header: `List-Unsubscribe-Post` 。拿這個 header 去找 RFC，結果找到了這個東西

```
Internet Engineering Task Force (IETF)                         J. Levine
Request for Comments: 8058                          Taughannock Networks
Category: Standards Track                                     T. Herkula
ISSN: 2070-1721                                              optivo GmbH
                                                            January 2017


        Signaling One-Click Functionality for List Email Headers

Abstract

    This document describes a method for signaling a one-click function
    for the List-Unsubscribe email header field.  The need for this
    ...
```

"Signaling One-Click Functionality for List Email Headers"，嗯.. 看來這就是我要的東西了...

在經過快速地閱讀之後，這個 RFC 的部分動機大概是這樣的

> 防毒軟體一般會掃過所有在信件內的 HTTP(S) 連結。電子報供應商為了避免防毒軟體不小心幫使用者退訂，通常會把連結做成需要使用者互動的網頁，像是在頁面中放個額外的確認按鈕等。但此作法又會造成信件軟體或信件服務商，無法在取得使用者的同意後，自動化的幫使用者退訂電子報。因此，需要訂出一個標準的方法，讓 HTTPS 的退訂連結也可以達成一鍵退訂。

這動機看起來是很清楚了.. (其實還有部分有關垃圾信的處理問題，這邊就不翻譯了)。不過究竟該怎麼做一鍵退訂呢？

根據 RFC 8058 的描述，信件若要用 HTTPS 連結做一鍵退訂，至少需要滿足以下幾點:

1. `List-Unsubscribe` header 內至少有一 HTTPS 連結
2. 需要額外有 `List-Unsubscribe-Post` header，且其值必須為 `List-Unsubscribe=One-Click`
3. 必須有 DKIM 簽章來驗證上述兩個欄位

第一點是挺合理的，這個 RFC 是 2017 年出來的，大概不會有人還想推 HTTP 連結了。而第二點的 List-Unsubscribe-Post header，是要告訴 mail 軟體說 "我這個連結可以吃 HTTP POST 請求喔！喔對了記得 POST 過來時內容要帶 List-Unsubscribe=One-Click 喔"。至於第三點，單純是要確保上述兩個 header 是沒有被竄改過的。

因為有講好 client 應該用 POST 方法去戳這個連結，於是電子報的 server 就可以很清楚的分辨，哪些請求是防毒軟體不小心誤發的，哪些請求是使用者真的想退訂才發的。且因為有清楚的表達意圖，這個 HTTPS POST 請求也不需要回一個要使用者額外互動的頁面，server 可以在收到請求後，直接處理使用者的退訂動作。

```
# 假設信件內有以下 header
List-Unsubscribe: <https://example.com/unsubscribe/opaquepart>
List-Unsubscribe-Post: List-Unsubscribe=One-Click

# 那 mail 軟體(提供商) 可以簡單透過以下 HTTPS POST 幫使用者退訂
POST /unsubscribe/opaquepart HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 26

List-Unsubscribe=One-Click
```

可喜可賀.. 可喜可賀 XDD

### 其他軟體的支援性和 Google 的工人智慧

發現有這個標準後，其實很好奇其他的 web mail 或信件軟體對這些 header 的支援度如何。無奈的是，沒找到比較完整的整理結果，似乎也沒多少人在意這個東西。 XD 不過從[唯一一份找到的資料](https://litmus.com/blog/the-ultimate-guide-to-list-unsubscribe)來看，在 iOS Mail, Gmail, Outlook 與 Yahoo Mail 四者中，mailto 的退訂連結都有支援，而 RFC 8058 所定義的一鍵退訂則是只有 Gmail 可以做到。(至少在 2018 年 11 月還是如此)

另外，我也發現到，即使有些信件完全沒有 `List-Unsubscribe` header，Gmail 仍然可以生出退訂的按鈕給使用者按，Twitter 的通知信件即是一例。Twitter 的信件 header 內沒有相關的資訊，但 Gmail 可以從信件內文內把退訂的連結 parse 出來。至於這部分是 Google 偉大的工人智慧，還是有一些我還不知道的標準可參考，這我就還沒研究到了。

## 結論

這篇文章基本上是把某個週末因為三分鐘熱度而去學的東西記錄下來。但過了兩週之後再回來看，其實好像也不是多重要的東西。就算今天我們不知道有這個標準，或是根本沒注意到有這個功能，日子也還是過得很好 (?

但是.. 小工程師滿足自己的好奇心後，心中所獲得的那種成就感，是無可取代的！

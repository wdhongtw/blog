---
layout: post
title:  "Samba 內檔案的異常執行權限"
date:   2018-04-17 11:10:11 +0800
---

許多場合會利用 Samba 來建立給所有工作者共用的 SMB 工作目錄。

SMB 是 Microsoft 針對 Windows 體系設計的 protocol，但因為種種因素，
目前 Mac 及各 Linux 桌面發行版也都有不錯的 SMB client 支援。
已 SMB 提供共用的工作目錄，似乎已成為最常見的協同工作方法。

在另一方面，Samba 是一套基於 Unix 環境開發的開源 SMB server 實作。
但因為 Windows 與 Unix 在檔案系統上設計的根本性差別，儘管 Samba 歷史悠久且功能齊全，
仍然會有一些先天性的問題。

為了解釋問題到底從何而來，以下要簡單介紹一下 Windows 與 Unix 的檔案相關特性。

## Windows File Attribute 與 Unix File Mode

傳統上，Windows 系統下的每個檔案都會需要有以下四個屬性:

- Archive: 紀錄此檔案在上次備份後是否更動過。
- Hidden: 紀錄此檔案是否要隱藏。Windows 內建 `dir` 或檔案總管均會遵從此設定。
- System: 紀錄此檔案是否爲系統檔案。
- Read-only: 紀錄此檔案是否只能讀取。

Unix 作業系統採取了與 Windows 不同的設計。
在 Unix 內，每個檔案紀錄針對檔案擁有者、檔案擁有群組及其他人分別紀錄三組權限

- Readable: 檔案是否可讀取
- Writable： 檔案是否可寫入
- Executable： 檔案是否可執行

現在我們可以知道，Windows 與 Unix 對於檔案應該紀錄的特性/模式其實有不一樣的要求。
許多和檔案系統相關的功能也會用不同的方式來達成。

比方說，有關檔案 **是否隱藏** 這件事情，在 Windows 上會有一個獨立的 attribute 來處理。
而在 Unix 上，則是依據 "`.` 開頭的檔案應被隱藏" 的常規。除了檔名看起來有點不一樣之外，
隱藏檔案和一般檔案在檔案系統裡沒有差別。

另外我們也可以注意到Windows 和 Unix 對於 **可執行** 這個概念的處理方式也不同。
在 Unix 的世界中，每個檔案的 mode 中會紀錄這個檔案是否可執行。而在 Windows 中，
檔案並沒有是否可執行的概念，而是讓作業系統維護一個表單，裡面紀錄各種副檔名的檔案應該如何開啟或執行。

## Samba 的設計與產生的問題

Samba 是個 SMB 的 server 實作，作為在 Unix 環境上跑的服務，但同時也要支援 Windows 的
file attribute，亦即前述提到的 Archive、Read-only 等。這些 attribute 是必得以某種
方式存在 Samba server 上。在這裏 Samba 的實作非常有趣：

> 利用 Unix 環境中 Windows 用不到的 executable bit 來存放 Windows 需要的 file attribute。

具體來說，Archive、System 與 Hidden 屬性會分別存在擁有者、擁有群組與其他人的 file mode
的 executable bit 當中。而 Read-only 則是影響檔案擁有者的寫入權限是開啟。
簡單圖示如下:

```
      Owner    Group    Others
    +----------------------------+
    | r  w  x  r  w  x  r  w  x  |
    +-^--^--^--------^--------^--+
      |  |  |        |        |
  ReadOnly  Archive  System   Hidden
```

*但此做法舉其實會導致其他問題*。這會其他在 Mac 或 Unix 環境下掛載 SMB share 的使用者，
會看到檔案有時莫名的變成可執行，或原本可執行的 script 突然變成不可執行。

於是乎，在 Window 與 Linux 混用的工作環境中，RD 的 terminal 下常會看到一片花花綠綠的
source code 資料夾。(一般 shell 都會預設開 `ls` 的 colorize 選項)

## Executable Bit 異常的解決方式

若要避免 Samba 的這類行為，可以在 `smb.conf` 中加入以下設定

```
map archive = no
map system = no
map hidden = no
map read only = no
```

如此 Samba 就不會嘗試利用 Unix 的 file mode 來存放這些 attribute。

又或是如果想支援 Windows 的 attribute，但又不想影響 Unix 下的執行權限，可以將這些
attribute 寫進 extended attributes 裡。這需要使用以下設定

```
store dos attributes = yes
```

## 碎碎唸

作為一個軟體工程師，**Every detail matters** 或其他類似精神的標語。
魔鬼蔵在細節裏，確保每個細節的正確(或至少看起來正確)是每個工程師都應該追求的境界。
而這個追求細節的精神，必須從乾淨的工作環境開始做起。

為什麼會寫出這邊文章？其實就只是在工作時，看到公司的 VCS 內各種怪異的檔案權限，
感到困惑而已。清楚的變數命名、符合直覺的 API 設計有助於開發人員理解專案。正確的檔案權限
其實也是如此。設定檔應該是 `rw-`，script 應該是 `r-x`，不違背直覺的專案狀態才不會
阻礙工程師工作。

當然，一切的根本原因還是在於 Samba 的預設設定會把 map Archive attribute 到
file mode 裡面。在 Window 環境工作的工程師一般都是在無意的情況下把 file mode 異動
寫道 VCS 裡面。這時只能抱怨爲何當年 Samba 的作者要做出這種設計了。

## 相關連結

- [Why are files in a smbfs mounted share created with executable bit set?](https://unix.stackexchange.com/questions/103415/why-are-files-in-a-smbfs-mounted-share-created-with-executable-bit-set)
- [File Permissions and Attributes on MS-DOS and Unix](http://www.oreilly.com/openbook/samba/book/ch05_03.html)
- [Document: `attrib` command](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/attrib)
- [Document: File attribute API](https://docs.microsoft.com/en-us/windows/desktop/fileio/file-attribute-constants)
- [Official Samba Config Document](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)

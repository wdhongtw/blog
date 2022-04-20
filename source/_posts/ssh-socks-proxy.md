---
title: 利用 SSH 建立 SOCKS Proxy
date: 2022-04-20 10:58:00
categories: [tips]
tags: [ssh, socks]

---

最近因為疫情又開始 WFH 了。 公司有提供一些 VPN solution 讓員工存取公司內網路，
但有一些架在 public cloud 上的服務後台因為有擋來源 IP，無法在家直接存取。

這時候 SSH 內建的 SOCKS proxy server 功能就可以派上用場了！

SSH 可以在建立連線時，一併在本機端開出一個 SOCKS (version 4 and 5) 的 server，
接下來任何應用程式都可以將任意的 TCP 連線透過這個 SOCKS server，轉送到 SSH server
後再與目標站台連線。
因為大家一定在公司裡有台可以 SSH 的機器(?)，於是這種限制公司 IP 的管理後台就可以順利存取。 :D

使用方式很簡單，SSH 連線時多下參數即可。

```shell
ssh "target-machine" -D "localhost:1080" -N
```

- `-D localhost:1080`: 決定要開在 local 的 SOCKS port，RFC 建議是 1080
- `-N`: 如果不需要開一個 shell，只是要 SOCKS proxy 功能，那可以多帶此參數

Note: SSH 有支援 SOCKS5 (可做 IPv6 proxy) 但不支援 authentication，不過因為 SOCKS server
可以如上述設定只開在 `localhost` 上，所以沒麼問題。

接著我們就可以設定 OS 層級或是 application 層級的 proxy 設定來使用這個 proxy 了！
以我一開始遇到的問題來說，通常我會多開一個 Firefox 並設定使用 proxy 來存取公司的各種管理後台。
這樣就可以保持其他網路流量還是直接往外打，不需要過 proxy。 :D

若要快速啟動 proxy，可以使用 [Windows Terminal](https://www.microsoft.com/zh-tw/p/windows-terminal/9n0dx20hk701)
並設定一個 profile，執行上述 SSH 指令。

PuTTY 作為 Windows 上最多人使用的 SSH client，也有支援 SOCKS proxy 功能，
詳見: [How To Set up a SOCKS Proxy Using Putty & SSH - Security Musings](https://securitymusings.com/article/462/how-to-set-up-a-socks-proxy-using-putty-ssh)

## Reference

- [ssh(1): OpenSSH SSH client - Linux man page](https://linux.die.net/man/1/ssh)
- [linux - How can I setup a SOCKS proxy over ssh with password based authentication on CentOS? - Server Fault](https://serverfault.com/questions/336067/how-can-i-setup-a-socks-proxy-over-ssh-with-password-based-authentication-on-cen)

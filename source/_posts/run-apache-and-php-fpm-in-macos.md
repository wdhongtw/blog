---
title: "在 macOS 上架設 Apache 與 PHP-FPM"
date: 2019-11-09 08:00:00
categories: [notes]
tags: [github, fido, security]
---

工作上因為一些特殊需求，需要在 macOS 環境下架設 Apache + PHP-FPM
的使用環境。好在 macOS 本來就有預裝 Apache 以及 PHP-FPM，並提供 Apache 的 `launchd` 設定檔，要在 macOS 上架設這個服務並不困難。

本文介紹如何以最低限度的設定，在 macOS 上跑 Apache + PHP-FPM。以筆記的方式呈現，不會有太多的講解。

## Notes

- 筆者是在 macOS 10.14 與 10.15 上測試此流程
- macOS 系統上，`/etc` 是一個 symblic link 連至 `/private/etc`，
`/var`, `/tmp` 也有相同行為。

## 設定與啟用 PHP-FPM

### 複製並修改 PHP-FPM 設定檔

系統內有會自帶 PHP-FPM 的 default 設定檔，將其複製一份出來，並修改內容。

```
$ sudo cp /etc/php-fpm.conf.default /etc/php-fpm.conf
$ sudo cp /etc/php-fpm.d/www.conf.default /etc/php-fpm.d/www.conf
```

將執行身分從 `nobody` 修改為 `_www` (與 Apache httpd一致)。


```
$ sudo vim /etc/php-fpm.d/www.conf
```

```
; Unix user/group of processes
; Note: The user is mandatory. If the group is not set, the default user's group
;       will be used.
user = _www
group = _www
```

修改 `error_log`，調整 log file 的路徑。

```
$ sudo vim /etc/php-fpm.conf
```

```
; Error log file
; If it's set to "syslog", log is sent to syslogd instead of being written
; into a local file.
; Note: the default prefix is /usr/var
; Default Value: log/php-fpm.log
error_log = /var/log/php-fpm.log
```

### 新增 PHP-FPM 的 launchd 設定檔並啟用

創一個 launchd daemon 設定檔給 PHP-FPM 使用，
此舉目的為讓 PHP-FPM daemon 可以在 macOS 開機時自己啟用。

建議將設定檔放在 `/Library/LaunchDaemons` 下，參照 launchd 的文件，
此位置是供第三方軟體擺放 daemon 設定使用。

```
$ sudo vim /Library/LaunchDaemons/com.example.php-fpm.plist
```

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>Disabled</key>
        <true/>
        <key>Label</key>
        <string>com.example.php-fpm</string>
        <key>ProgramArguments</key>
        <array>
                <string>/usr/sbin/php-fpm</string>
                <string>--nodaemonize</string>
        </array>
        <key>OnDemand</key>
        <false/>
</dict>
</plist>
```

做好設定檔之後，用 `launchctl` 指令 load 此設定檔，並下參數告訴 macOS
之後此 daemon 要在開機時預設啟用。

```
$ sudo launchctl load -w /Library/LaunchDaemons/com.example.php-fpm.plist
```

上述指令執行完後， launchd 會把 PHP-FPM daemon 叫起。

```
$ ps aux | grep php
_www              515   0.0  0.0  4297608    648   ??  S     6:18PM   0:00.00 /usr/sbin/php-fpm
_www              514   0.0  0.0  4305800    628   ??  S     6:18PM   0:00.00 /usr/sbin/php-fpm
root              513   0.0  0.0  4305800    784   ??  Ss    6:18PM   0:00.00 /usr/sbin/php-fpm
```

## 設定並啟用 Apache Web Server

修改設定檔，讓 Apache 使用 `proxy_module` 與 `proxy_fcgi_module`，
並確認 `php7_module` 沒被啟用。

需要本文的讀者應該不至於把 Apache PHP module 與 PHP-FPM 搞混.. XD

```
$ sudo vim /etc/apache2/httpd.conf
```

```
LoadModule proxy_module libexec/apache2/mod_proxy.so
LoadModule proxy_fcgi_module libexec/apache2/mod_proxy_fcgi.so
# LoadModule php7_module libexec/apache2/libphp7.so
```

在 `<Directory "/Library/WebServer/Documents">` 或其他需要的地方內，
加入 PHP 的 handler，指向 PHP-FPM 預設提供服務的 socket。

``` xml
<Directory "/Library/WebServer/Documents">
... 上略
    <FilesMatch \.php$>
        SetHandler "proxy:fcgi://localhost:9000/"
    </FilesMatch>
</Directory>

```

Apache 的 daemon config 本來就存在於系統目錄內，但 Disable 的值被設為 true，
用下述 command 將 Apache daemon load 進 launchd 內，並讓 launchd 記錄此 daemon 應被啟用。

```
sudo launchctl load -w /System/Library/LaunchDaemons/org.apache.httpd.plist
```

與 PHP-FPM 相同，此指令下去之後，launchd 就會把 Apache 拉起。

此時可去 http://localhost/ 確認，如果看到大大的標題寫著

> It works!

即代表 Apache 有順利執行。

## 確認 PHP-FPM 運作正常

丟一個 `phpinfo` 到 web server 的預設根目錄下。

```
$ sudo vim /Library/WebServer/Documents/phpinfo.php
```

``` php
<?php
   phpinfo();
?>
```

之後連上 http://localhost/phpinfo.php ，看到 `Server API` 為
`FPM/FastCGI` 即可。 :D


## Useful Links

- [A launchd Tutorial](https://www.launchd.info/)

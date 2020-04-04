---
layout: post
title:  在 Ubuntu Server 上自動啟用 SSH Agent
---

當 我們的 SSH private key 有上 pass phrase 保護時，
SSH agent 是個方便的好東西。因為它可以幫我們記住已經解鎖過的 private key。 

可惜的是，Ubuntu server 18.04 的環境預設並不會幫你生一個 SSH agent 出來。

本文章記錄一點摸索的過程...

## 系統自帶的 SSH agent systemd unit

> 我看別人的 Ubuntu 登入之後就有 SSH agent 可以用啊？

很可惜的是我的環境沒有。研究一陣子之後，發現 SSH agent 應是在有圖形介面
的情況下才會被自動帶起。

在 `dpkg --listfiles openssh-client` 下可看到幾個重要的檔案

- `/usr/lib/openssh/launch-agent`
- `/usr/lib/systemd/user/ssh-agent.service`
- `/usr/lib/systemd/user/graphical-session-pre.target.wants/ssh-agent.service`

看了這幾個檔案的內容後可得知

1. 這是設計給圖形介面的登入 session 使用的 service
2. 即使想要直接 enable `ssh-agent.service` 也無法，因為裡面沒有寫任何的 `[Install]` 參數

## 自行撰寫並啟用一個 SSH agent 服務

為了解決沒有 SSH agent 的問題，我們可以自己寫一個 systemd 的 user service，
讓系統在發現我登入之後，自動幫我把 SSH agent 拉起來。

首先編輯 `~/.local/share/systemd/user/ssh-agent.service` (參考 `man systemd.unit` 此為預設的 user unit 路徑)

``` ini
[Unit]
Description=SSH authentication agent

[Service]
ExecStart=/usr/bin/ssh-agent -a %t/ssh-agent.socket -D
Type=simple

[Install]
WantedBy=default.target
```

注意 `ssh-agent` 的 `-D` 參數與 `Type=simple` 設定。

接著執行 `systemctl --user enable ssh-agent.service`。
這一步會在 `.config/systemd/user/default.target.wants` 資料夾下創出一個 symbolic link，
連回剛剛我們寫的 service file，表示要在登入時自動啟用此 unit。

接著重新登入該機器，應該就可以看到一個 `ssh-agent` process 跑起來了。

## 設定 SSH agent 所需的的環境變數

雖然 SSH agent 起來了，但此時若下 `ssh-add -L` 依然會發現無法連上 SSH agent。

> Could not open a connection to your authentication agent.

這是因為 `ssh` 以及 `ssh-add` 等工具預設都是看 `SSH_AUTH_SOCK` 環境變數來得知
要透過哪個 Unix socket 與 agent 溝通。

為了處理此問題，我們需在 `~/.profile` 內加入一行環境變數設定，確保在登入時能自動設定完成。

``` shell
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"
```

註: `$XDG_RUNTIME_DIR/ssh-agent.socket` 與前述 unit file 內的 `-a %t/ssh-agent.socket` 對應。詳細可參考 `man systemd.unit`

下次登入重新讀取 profile 之後即可正常使用 SSH agent 囉。 :D

## Alternative Solution

尋找解決方式的過程中，注意到了一些解法，透過純 shell script 的方式處理重複登入的問題

``` bash
SSH_ENV="$HOME/.ssh/environment"

function start_agent {
    /usr/bin/ssh-agent | sed 's/^echo/#echo/' > "${SSH_ENV}"
    chmod 600 "${SSH_ENV}"
    . "${SSH_ENV}" > /dev/null
    /usr/bin/ssh-add;
}

if [ -f "${SSH_ENV}" ]; then
    . "${SSH_ENV}" > /dev/null
    ps -ef | grep ${SSH_AGENT_PID} | grep ssh-agent$ > /dev/null || {
        start_agent;
    }
else
    start_agent;
fi
```

- Ref: <https://stackoverflow.com/questions/18880024/start-ssh-agent-on-login/18915067#18915067>

若不考慮 race condition，該作法其實也很值得參考。可以在沒有 systemd 輔助的的生態系底下使用。

## 雜談

看 systemd 的文件時，發現 systemd 的 user mode 會非常遵守 `XDG_` 系列的環境變數。
不過因為我們是在 Ubuntu server edition 下，所以大部分都略過不看。 :D

但 `XDG_RUNTIME_DIR` 這個變數除外，此變數雖然也是由
[XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
所規範，但在一般 Linux 發行版，此變數是由 `pam_systemd` 直接維護的。所以即使是在 server
環境也會有此變數存在。

## References

- [systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
- [command line - What is XDG_RUNTIME_DIR? - Ask Ubuntu](https://askubuntu.com/questions/872792/what-is-xdg-runtime-dir)
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
- [Add user ssh-agent as daemon to Ubuntu 18.04LTS server.](https://gist.github.com/magnetikonline/b6255da90606fe9c5c25d3333c98c90d)
- [git - Start ssh-agent on login - Stack Overflow](https://stackoverflow.com/questions/18880024/start-ssh-agent-on-login)

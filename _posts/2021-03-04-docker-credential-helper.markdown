---
layout: post
title: 保護存在檔案系統上的 Docker 登入密碼
---

在企業內部的工作環境中，常會碰到需要存取 private registry 上的 image 的狀況。
以 Docker 的工作流程來說，一般要透過執行 `docker login` 來存取 private registry。
不過，若事先毛設定好 *docker credential helper*，執行 `docker login` 會導致
我們的密碼 / API token 直接以明文的方式寫在檔案系統上。

這篇筆記說明如何在 Linux 環境下安裝與設定 *docker credential helper*。

## Installation

至 docker/docker-credential-helpers 的 GitHub release 頁面下載最新版本的
`docker-credential-secretservice`。

```shell
curl -L https://github.com/docker/docker-credential-helpers/releases/download/v0.6.3/docker-credential-secretservice-v0.6.3-amd64.tar.gz >secretservice.tar.gz
```

解壓縮並把執行檔放到任意一個 PATH 資料夾內。

```shell
tar -zxv -f secretservice.tar.gz
chmod +x docker-credential-secretservice
mv docker-credential-secretservice ~/.local/bin
```

## Configuration

為了讓 `docker` 工具知道我們要用 credential helper，需要調整家目錄下的設定檔。

在設定檔 `~/.docker/config.json` 內加入 `credsStore` 設定。

```shell
$EDITOR ~/.docker/config.json
```

```json
{
    "credsStore": "secretservice"
}
```

註: 此資料夾和 JSON 檔案可能不存在。若沒有自己創一個即可。

註: 根據文件，此欄位的值與是 helper binary 的後綴對齊，因為 Linux 環境使用的 binary 是
`docker-credential-secretservice` 所以需要填入的值爲 `secretservice`

## Usage

如果已經有登入過某 registry，需要手動登出。

```shell
docker logout registry.example.com
```

(重新) 登入該 registry。

```shell
docker login registry.example.com
```

檢視 `~/.docker/config.json` 並確認對應的身分紀錄是空白的。

```json
{
    "auths": {
        "registry.example.com": {}
    }
}
```

若有安裝 Seahorse 程式的話，此時可以看到 secret 被放在 Login keyring 中。

如果設定錯誤的話，登入資訊會以編碼過的方式呈現在該紀錄中。

```json
{
    "auths": {
        "registry.example.com": {
            "auth": "c3R...zE2"
        }
    }
}
```

## Future Reading

- [docker login - Docker Documentation](https://docs.docker.com/engine/reference/commandline/login/)
- [docker/docker-credential-helpers](https://github.com/docker/docker-credential-helpers)
- [GNOME/Keyring - ArchWiki](https://wiki.archlinux.org/index.php/GNOME/Keyring)

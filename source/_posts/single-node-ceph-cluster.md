---
title: 單台機器的 Ceph 部署
date: 2021-08-08 11:52:24
categories: [notes]
tags: [ceph]
---

## 原由

Ceph 的預設設定對資料的 replica 行為要求嚴格。
若只有單台機器或是硬碟數量受限，往往架設起來的 Ceph 無法順利存放資料。

此篇筆記關注的目標如下

> 若想要用最少資源，建立可用的 Ceph 環境，需要做哪些額外的調整？

## 背景知識

Ceph 是一套開源的儲存叢集 solution。 可以整合多個儲存設備並在其上提供
RADOSGW, RBD, Ceph FS 等不同層級的存取介面。

![ceph-stack](https://docs.ceph.com/en/latest/_images/stack.png)

對於每個儲存設備 (HDD, SSD)，Ceph 會建立對應的 OSD 來管理儲存設備。
有了儲存設備之後，Ceph 會建立邏輯上的 pool 作為管理空間的單位。
Pool 底下會有多個 PG(placement group) 作為實際存放資料至 OSD 中的區塊。

在 Ceph 的預設設定中，一般 pool 的 replica 行為如下

- 要有三份 replica
- replica 要分散在不同的 host 上

在開發環境中，資料掉了其實並無傷大雅，三份 replica 意味著儲存空間的浪費。
且若資料真的要放在不同的 host 上，連同 replica 三份這點，我們就至少要開三台機器，
增加無謂的管理成本。

## 解決方式

假設我們都是透過 `cephadm bootstrap` 來架設 Ceph。

Ceph cluster 建立好，也設定完需要的 OSD 之後，就可以來建立 pool。

根據 pool 的目的不同，要解決單台機器部署 Ceph 的限制，大概會有兩種做法。

### 降低 Pool Size

Pool size 此術語意味著該 pool 下的 PG 要 replica 幾份。
若某 pool 是拿供他人存放資料，或是會使用較多空間的，可以把 size 降為 1。

調整完之後就相當於該 pool 內的所有資料都不會有 replica。

範例如下:

```shell
ceph osd pool create "<pool name>"
ceph osd pool set "<pool name>" size 1
ceph osd pool application enable "<pool name>" rbd
```

### 調整 Choose Leaf 行為

Ceph 有定義不同層級的資料分散設定。
預設值為 `host`，意味著只有一台機器的情況下，資料會無法複製。
若調整為 `osd`，只要該機器上有多顆硬碟即可滿足複製條件。
若是針對 Ceph 自行建立出來，管理 meta data 的 pool (e.g. `device_health_metrics`)
可以考慮使用此方式處理。

設定方式大概有兩種。

**方法一**: 調整 Ceph global 設定

編輯 `/etc/ceph.conf` 並在 global section 下加入 `osd_crush_chooseleaf_type` 設定

```
[global]
...
    osd_crush_chooseleaf_type = 0
```

或是直接執行 command

```shell
ceph config set global osd_crush_chooseleaf_type 0
```

這邊的 `0` 代表 OSD。預設的對應列表如下

```
type 0 osd
type 1 host
...
type 9 zone
type 10 region
type 11 root
```

**方法二**: 修改 crush map 內容

筆者有注意到有時即使有執行方法一，pool 還是不會受到設定影響。 (相關知識還太少，
不太確定具體原因) 不過針對此狀況，還有第二個方法可以使用。

此方法會用到 `crushtool` 指令 (Ubuntu 中需要額外安裝 `ceph-base` 套件)

首先執行指令將目前的 crush map 撈出來

```shell
ceph osd getcrushmap -o "compiled-crush-map"
crushtool -d "compiled-crush-map" -o "crush-map"
```

接著修改 `crush-map` 檔案內容，應該會有一行有 `step chooseleaf` 開頭的設定，把最後的
type 從 `host` 調整為 `osd`。

```shell
# Before
step chooseleaf firstn <number> type host
# After
step chooseleaf firstn <number> type osd
```

最後將修改好的 crush map 設定塞回去。

```shell
crushtool -c "crush-map" -o "compiled-crush-map"
ceph osd setcrushmap -i "compiled-crush-map"
```

相關 reference link

- [Common Settings — Ceph Documentation](https://docs.ceph.com/en/latest/rados/configuration/common/)
- [Chapter 4. Create a Cluster Red Hat Ceph Storage 1.2.3 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/1.2.3/html/installation_guide_for_centos_x86_64/create_a_cluster)
- [CRUSH Maps — Ceph Documentation](https://docs.ceph.com/en/latest/rados/operations/crush-map/)
- [Manually editing a CRUSH Map — Ceph Documentation](https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/)
- [CRUSH Maps — Ceph Documentation](https://docs.ceph.com/en/latest/rados/operations/crush-map/)

## 結語

筆者在公司業務並不負責維護 production 的 Ceph cluster，僅是為了建立 Kubernetes
開發環境，需要有個基本會動的 Ceph。

為了用最少資源建立 Ceph 環境，需要調整相關設定來改變 Ceph 行為。
只可惜相關的資源不是很夠，一路跌跌撞撞下來，決定寫下這篇筆記，希望造福未來的自己，也同時照顧他人。

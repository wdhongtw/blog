---
layout: post
title:  Kubernetes Audit Log 使用筆記
---

我在公司的工作環境中，有些業務需要部屬服務在 Kubernetes (下稱 K8s) 上。
因此在專案早期，部門內的同事自架了 K8s cluster 來開發。

隨著時間流逝，各個 RD 開始上手 K8s 操作後，每天都有人在對 K8s 的 master 開發環境做修改。
於是部門內開始產生一些令人煩躁的對話

- 我看 K8s 上面有裝了某個 CRD，但沒有裝對應的 service 來用這個 CRD，這個是你裝的嗎？
- Test namespace 裝了一個 Ingress rule 產生衝突了，那個 rule 是誰裝的？
- ...

這些對話的共通點是：想知道 K8s 的狀態改變是誰造成的。
但在部門自架的環境內，因為大家共用了一個 kubeconfig，所以根本無從找起..

於是我想辦法把開發用的 K8s 環境設定好 auditing log 的功能，並留下這篇筆記

## Audit 目標

要做 audit 來確認每個人做了什麼操作，我需要達到兩個目標

1. 不同人員需要使用不同的身分存取 K8s API server
2. API server 需開啟 log 且 log 需保存在 persistent storage 上

## 身分驗證方式比較

參考 K8s 的 [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
官方文件，在不依賴外部服務的情況下，大概有三種身分驗證的方式

- X509 Client Certificate
- Static Token File
- Service Account Tokens

以下分別介紹各方式的優缺點

### X509 Client Certificate

此方式依賴 TLS 的 client verification 功能，只要你有正確的憑證塞在 kubeconfig 裡即可使用。
一般在做 cluster 初始化過程中拿到的 admin kubeconfig ，其內容即屬這一類。

此方式的優點為

- 若採取嚴謹的使用者自行產生 key-CSR pair 再給 CA 簽署流程，因為僅使用者有 private key，出事時有高度信心一定是該使用者所為
- 除了自己的 user name 外，使用者可以從自己的憑證中直接確認操作 K8s 時會有那些 group 身分
  - 憑證內 subject 的 *CN* 對應 K8s user name, *O* 對應 K8s group name

此方式的缺點為

- K8s 不支援 X509 原生的 certificate revocation 功能，若有特定 client 憑證有問題，得整個 CA 換掉重來
  - Upstream issue: [Support for managing revoked certs](https://github.com/kubernetes/kubernetes/issues/18982) (opened for 5 years)

### Static Token File

K8s API server 在開啟時，可以設定一個檔案來記錄 token 與 user(group) 的 mapping 關係。
Client 連上 API server 時，只要能拿出此 token，便會被視為對應的 user 進行後續權限檢查。

此方式的優點為

- 設定簡單。需要新增/刪除使用者或修改 token 時，只需修改一個檔案
- Token 可長可短，可以做出較為可讀的 kubeconfig 檔案 (行寬 80 字元以內)

此方式的缺點為

- static token file 設定有異動時需要重開 server

### Service Account

Service Account 是 K8s 原生設計給 K8s 內的 service 做 K8s 自我管理的機制。

此方式的優點為

- 彈性極高，可在 runtime 直接透過 K8s API 產生新的 service account

此方式的缺點為

- service account 屬 namespaced resource，若有多個 namespace 要相同 user，需要重複設定
- 產生的 audit log 較難做事後梳理
  - K8s 有大量利用 service account 的自我管理行為，因此難以區隔使用者操作和 K8s 自身操作
- 相較於 X509 或 static token 方式，service account 不能直接設定群組

## 環境說明

若使用 `kubeadm` 安裝設定 K8s cluster，只有 `kubelet` 會作為一個 system service 運行在 host 中。
其他如 K8s API server, scheduler 及 etcd 等都是跑在 master node 的 Docker container 環境中

以下說明均假設為此類環境進行操作。

## 設定 Static Token File

### K8s Master Node 設定修改

新增 user token file `/etc/kubernetes/tokens.csv` (路徑可自行調整)

``` csv
user-token,user-name,uid,"optional-group,another-group"
fc27911e-73dd-46b0-8c57-86f2fe5fdd21,alice,alice@example.com,"developer"
```

檔案為單純的 CSV 格式，包含四個欄位

- User Token: 任意字串，不一定要使用 UUID 格式
- User Name: 使用此 token 身分驗證完成後得到的 user name
- UID: 用途不明，會出現在 audit log 中
  - > identifies the end user and attempts to be more consistent and unique than username
- List of Group Name: (Optional) 使用此 token 身分驗證完成後得到的 group 身分

設好 static token file 後，修改 API server 的 static pod 描述 `/etc/kubernetes/manifests/kube-apiserver.yaml`。
`user-tokens` 的 path 與前述設定對齊。

``` diff
diff --git a/root/manifests/kube-apiserver.yaml b/root/token-api-server.yaml
index 31c5f40..d4511ae 100644
--- a/root/manifests/kube-apiserver.yaml
+++ b/root/token-api-server.yaml
@@ -37,6 +37,7 @@ spec:
     - --service-cluster-ip-range=10.96.0.0/12
     - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
     - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
+    - --token-auth-file=/etc/kubernetes/tokens.csv
     image: k8s.gcr.io/kube-apiserver:v1.17.4
     imagePullPolicy: IfNotPresent
     livenessProbe:
@@ -71,6 +72,9 @@ spec:
     - mountPath: /usr/share/ca-certificates
       name: usr-share-ca-certificates
       readOnly: true
+    - mountPath: /etc/kubernetes/tokens.csv
+      name: user-tokens
+      readOnly: true
   hostNetwork: true
   priorityClassName: system-cluster-critical
   volumes:
@@ -98,4 +102,8 @@ spec:
       path: /usr/share/ca-certificates
       type: DirectoryOrCreate
     name: usr-share-ca-certificates
+  - hostPath:
+      path: /etc/kubernetes/tokens.csv
+      type: FileOrCreate
+    name: user-tokens
 status: {}
```

上述修改內容的重點為

- 將 master node 上的 user token 設定檔 mount 至 API server 的 container 內
- 設定 API server 去使用此 token 檔案

### User Token File 後續維護

若之後需要修改 user token file，因為一些[上游的限制](https://github.com/kubernetes/kubernetes/issues/44713)，
API server pod 無法觀測到檔案的修改，即使 kill pod 再重啟也無法使用新的 token file。

不過我們可以透過修改 API server 描述檔的方式，穩定地重新部屬 API server，讓新的 token file 生效。

- 編輯 `/etc/kubernetes/tokens.csv`
- 修改 API server 描述檔 `/etc/kubernetes/manifests/kube-apiserver.yaml`
  - 加入或修改 `metadata.annotations.lastModify` 欄位，填入合適字串
- 修改後 kubelet 會偵測到檔案異動，並重新 apply `apiserver` pod

### User kubeconfig 設定

使用 `kubectl` 設定 user token

`kubectl config set-credentials <user-name> --token=<token>`

或是直接修改 kubeconfig 內的 user object

``` yaml
- name: alice
  user:
    token: fc27911e-73dd-46b0-8c57-86f2fe5fdd21
```

## Log 設定

當各個使用者操作 K8s 的身分確實有被切分開之後，即可進行後續的 audit log 設定動作。

Audit log 必須在吻合事先設定的 match rule 才會被記錄下來。
根據 [Auditing 文件](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/) 說明，
server 在判斷每個事件的 log level 時，是採取 first match 的規則進行。第一個吻合的規則會決定此事件是否紀錄以及紀錄的詳細程度。

> The first matching rule sets the "audit level" of the event.

### API Server Audit 設定

在 master node 上設定 audit policy `/etc/kubernetes/audit-policy.yaml` (路徑可自行調整)

``` yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- "RequestReceived"
rules:
- level: Metadata
  userGroups:
  - "developer"
  verbs: ["create", "update", "patch", "delete", "deletecollection"]
- level: Metadata
  userGroups:
  - "developer"
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
```

此設定有幾個重點

- global 的 *omitStages* 設定
  - 所有 API request 都會經過 *RequestReceived* stage
  - 省略此 stage 可以避免所有的 request 都產生兩筆 log
- Rule 以 *userGroups* 進行篩選
  - 若已知要紀錄的 user group 範圍，明定 group 可避免記錄到大量的 K8s 自身維護的事件
- 設定動詞範圍記錄所有的 modify 操作
- 設定敏感的 resource 種類 (e.g. secrets & configmaps) 記錄所有操作

接著修改 API server 的 static pod 描述 `/etc/kubernetes/manifests/kube-apiserver.yaml`。
`audit` hostPath volume 需與前述設定對齊

``` diff
diff --git a/root/token-api-server.yaml b/root/audit-token-api-server.yaml
index d4511ae..0e07f7f 100644
--- a/root/token-api-server.yaml
+++ b/root/audit-token-api-server.yaml
@@ -38,6 +38,10 @@ spec:
     - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
     - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
     - --token-auth-file=/etc/kubernetes/tokens.csv
+    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
+    - --audit-log-path=/var/log/kubernetes/audit.log
+    - --audit-log-maxsize=1
+    - --audit-log-maxbackup=6
     image: k8s.gcr.io/kube-apiserver:v1.17.4
     imagePullPolicy: IfNotPresent
     livenessProbe:
@@ -75,6 +79,12 @@ spec:
     - mountPath: /etc/kubernetes/tokens.csv
       name: user-tokens
       readOnly: true
+    - mountPath: /etc/kubernetes/audit-policy.yaml
+      name: audit
+      readOnly: true
+    - mountPath: /var/log/kubernetes
+      name: audit-log
+      readOnly: false
   hostNetwork: true
   priorityClassName: system-cluster-critical
   volumes:
@@ -106,4 +116,12 @@ spec:
       path: /etc/kubernetes/tokens.csv
       type: FileOrCreate
     name: user-tokens
+  - name: audit
+    hostPath:
+      path: /etc/kubernetes/audit-policy.yaml
+      type: File
+  - name: audit-log
+    hostPath:
+      path: /var/log/kubernetes
+      type: DirectoryOrCreate
 status: {}
```

Note: 開 `/var/log/kubernetes` 資料夾而非單一 log 檔案，是為了避免 log rotate 時因權限不足無法正確 rotate

設定完之後即可在 master node 的 `/var/log/kubernetes` 看到 access log

Sample 如下

command: `kubectl apply -f services/tasks/redis-cluster-proxy.yml`

log: (Log 檔內會寫成一行，beautify 後如下)
``` json
{
   "kind":"Event",
   "apiVersion":"audit.k8s.io/v1",
   "level":"Metadata",
   "auditID":"f09f32f4-a93f-41ee-b2b9-2f3acf3aa963",
   "stage":"ResponseComplete",
   "requestURI":"/api/v1/namespaces/alice/services",
   "verb":"create",
   "user":{
      "username":"alice",
      "uid":"alice@example.com",
      "groups":[
         "developer",
         "system:authenticated"
      ]
   },
   "sourceIPs":[
      "10.300.400.512"
   ],
   "userAgent":"kubectl/v1.18.2 (linux/amd64) kubernetes/52c56ce",
   "objectRef":{
      "resource":"services",
      "namespace":"alice",
      "name":"redis-cluster-proxy",
      "apiVersion":"v1"
   },
   "responseStatus":{
      "metadata":{},
      "code":201
   },
   "requestReceivedTimestamp":"2020-10-21T12:27:30.252440Z",
   "stageTimestamp":"2020-10-21T12:27:30.272401Z",
   "annotations":{
      "authorization.k8s.io/decision":"allow",
      "authorization.k8s.io/reason":"RBAC: allowed by RoleBinding \"super-user-role-binding-alice/alice\" of Role \"super-user\" to User \"alice\""
   }
}
```

## 疑難排解

### 設定檔位置

kubelet Static Pod 設定資料夾不一定在 `/etc/kubernetes/manifests` 位置，
須從 `kubelet` 啟動設定中的 `staticPodPath` 欄位找到真實位置。

### 備份設定檔

若要備份 static pod 設定資料夾內的任何檔案，不能備份在相同資料夾內，否則會導致 `kubelet` 行為怪異。

### Reload K8s API server 設定

`kubelet` service 一般會自動偵測 static pod 資料夾內的檔案異動，並重新佈署該 pod，但偶爾還是會碰上意外..

發生意外時，以下方式可能可以回到正常狀態

- 刪除對應的 pod, e.g. `kubectl delete -n kube-system pod kube-apiserver-<cluster name>`
  - 刪除後 `kubelet` 會馬上重新佈署一個新的 API server
  - `Controlled By:  Node/k8s-master`: 意味者此 pod 不是由 deployment 等 K8s object 控制，是直接由 master node 控制
- 或是重啟 `kubelet` systemd service

## 後續

此篇筆記紀錄 static token 的身分驗證機制，但若有企業規模的身分驗證需求時，這顯然不是個好方法。

Kubernetes 也有原生支援 OpenID 的身分驗證方式來應付更進一步的需求，不過這部分就等未來有空再來研究了。

## References

- [Authenticating \| Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [kubernetes - How can kube-apiserver be restarted? - Stack Overflow](https://stackoverflow.com/questions/51666507/how-can-kube-apiserver-be-restarted)
- [Auditing \| Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)
- [Authorization Overview \| Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [kube-apiserver \| Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- [kube-apiserver audit log rotation throwing permission denied · Issue #70664 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/issues/70664)

## Appendix

Request Stages:

```
 +-----------------+
 | RequestReceived +----+
 +---+-------------+    |
     |                  |
     |       +----------v------+
     |       | ResponseStarted |
     |       +----------+------+
     |                  |             +-------+
     |                  |             | Panic |
+----v------------+     |             +-------+
| ResponseComplete<-----+
+-----------------+
```

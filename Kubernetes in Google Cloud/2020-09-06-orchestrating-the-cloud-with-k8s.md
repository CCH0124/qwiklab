---
layout: post
title: Kubernetes in Google Cloud- 03
date: 2020-09-06
excerpt: "Orchestrating the Cloud with Kubernetes"
tags: [qwiklab, k8s, google]
comments: true
---


- 使用 `Kubernetes Engine` 設置完整的 `Kubernetes` 集群。
- 使用 `kubectl` 部署和管理 `Docker` 容器。
- 使用 `Kubernetes` 的 `Deployments` 和 `Services` 將應用程序分解為微服務。

將使用 [App](https://github.com/kelseyhightower/app) 進行練習。

`Kubernetes` 是一個開源項目，它可以在許多不同的環境中運行，從可攜式計算機到高可用性多節點群集，從公共雲到本地部署，從虛擬機到裸機。


## Get the sample code

將實驗專案從 `github` 拉取
```shll
$ git clone https://github.com/googlecodelabs/orchestrate-with-kubernetes.git
$ cd orchestrate-with-kubernetes/kubernetes
$ ls
deployments/  /* Deployment manifests */
  ...
nginx/        /* nginx config files */
  ...
pods/         /* Pod manifests */
  ...
services/     /* Services manifests */
  ...
tls/          /* TLS certificates */
  ...
cleanup.sh    /* Cleanup script */
```

## Quick Kubernetes Demo

`Kubernetes` 入門的最簡單方法是使用 `kubectl create`。這邊使用它來啟動 `nginx` 容器實例

```shell
$ kubectl create deployment nginx --image=nginx:1.10.0
```

`Kubernetes` 已經建立了一個部署，後面會介紹 `deployment`，但是到目前為止，只需要知道即使在其 `POD` 上運行的節點發生故障時，部署也可以保持`POD` 正常運行。

在 `Kubernetes` 中，所有容器都在 `POD` 中運行。使用 `kubectl get pods` 命令查看正在運行的 `nginx` 容器

```shell
$ kubectl get pods
```

`nginx` 容器運行後，可以使用 `kubectl expose` 將其暴露在 `Kubernetes` 之外

```shell
$ kubectl expose deployment nginx --port 80 --type LoadBalancer
```

在幕後，`Kubernetes` 創建了一個外部負載均衡器，並為其附加了公有 `IP` 地址。任何請求該公有 `IP` 地址的客戶端都將被路由到該服務後面的 `POD`。在這種情況下，這將是 nginx pod。使用 `kubectl get services` 指令列出我們的服務

```shell
$ kubectl get services
```

將服務分配 `ExternalIP` 字段可能需要幾秒鐘。這是正常現象-每隔幾秒鐘重新運行一次 `kubectl get services` 命令，直到填充該字段。

```shell
$ curl http://<External IP>:80
```

`Kubernetes` 支援使用 `kubectl run` 和 `expose` 直接使用，這是一種易於使用的工作流程。


## Pods

Kubernetes 的核心是 [POD](http://kubernetes.io/docs/user-guide/pods/)。`POD` 代表有一個或多個容器的集合。通常，如果有多個相互依賴的容器，則將這些容器打包在一個容器中。

![](https://cdn.qwiklabs.com/tzvM5wFnfARnONAXX96nz8OgqOa1ihx6kCk%2BelMakfw%3D)

在此範例中，有一個包含 `monolith` 容器和 `nginx` 容器。

`pod` 也有 [Volumes](http://kubernetes.io/docs/user-guide/volumes/)。`Volume` 是與 `POD` 生命週期一樣長的數據磁盤，可以被該 `POD` 中的容器使用。`POD` 為它們的內容提供了一個共享的 `namespace`，這表示我們範例 `POD` 中的兩個容器可以相互通訊，並且還共享附加的 `Volume`。`POD` 也共享一個網路 `namespace`，表示每個 `POD` 有一個 `IP` 地址。


## Creating Pods

```shell
$ cat pods/monolith.yaml
```
將嘗試理解 monolith 的 `yaml` 配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
```

- `POD` 由一個容器(monolith)組成
- 在容器啟動時將一些參數傳遞給容器
- 開放端口 `80` 進行 `HTTP` 通訊

建立 `POD`
```shell
$ kubectl create -f pods/monolith.yaml
```
查看 `POD`，使用 `kubectl get pods` 列出在預設 `namespace` 中運行的所有 `POD`

```shell
$ kubectl get pods
```

>整體 `POD` 啟動並運行可能需要幾秒鐘的時間。容器鏡像需要從 `Docker Hub` 中拉取，然後才能運行。

`POD` 運行後，使用 `kubectl describe` 獲取有關 `POD` 的更多訊息

```shell
$ kubectl describe pods monolith
```
會看到很多有關容器的訊息，包括容器 `IP` 地址和事件日誌，在做故障排除時，此訊息會派上用場。`Kubernetes` 透過在資源清單中對其進行描述來輕鬆創建 `POD`，並在它們運行時輕鬆查看有關它們的訊息。

## Interacting with Pods
預設情況下，會為 `POD` 分配一個專用 `IP` 地址，並且該 `IP` 無法在群集外部直接訪問 `POD`。使用 `kubectl port-forward` 將本機端口映射到容器內的端口。

打開兩個 `Cloud Shell` 終端。一個運行 `kubectl port-forward` ，另一個下 `curl`。
```shell
$ kubectl port-forward monolith 10080:80
```

```shell
curl http://127.0.0.1:10080
```

現在，使用 `curl` 查看請求 `secure` 端點時會發生什麼
```shell
curl http://127.0.0.1:10080/secure
```
嘗試登錄以從 `POD` 中容器(monolith)獲取身份驗證 token，在登錄提示符下，使用 `password` 登錄。

```shell
$ curl -u user http://127.0.0.1:10080/login
```

登錄觸發後打印出 `JWT` 令牌(token)。由於 `cloud shell` 無法很好處理長字串的複制，因此創建一個環境變量以儲存。下面會出發提示窗再次輸入 `password`。
```shell
$ TOKEN=$(curl http://127.0.0.1:10080/login -u user|jq -r '.token')
```

再次使用 `curl` 請求 `secure` 端點，這次帶有 `token`，成功後會有響應資訊。
```shell
$ curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure
```


使用 `kubectl logs` 命令可查看 `POD` 的 `log`，要即時不斷接收則需要帶 `-f` 選項。
```shell
$ kubectl logs monolith
```

在開啟一個終端，執行以下。

```shell
$ kubectl logs -f monolith
```
在另一個終端執行以下，執行後上面使用 `logs` 的終端會不斷給出新的 `log` 訊息。
```shell
$ curl http://127.0.0.1:10080
```

使用 `kubectl exec` 在 `POD` 內部執行交互式 `shell`。當要從容器內進行故障排除時可以派上用場

```shell
$ kubectl exec monolith --stdin --tty -c monolith /bin/sh
```
當我們進入該容器後，可執行以下

```shell
$ ping -c 3 google.com
```

完成此交互式 `shell` 程序後，離開方式使用 `exit`。與 `POD` 交互就像使用 `kubectl` 一樣容易，如果您需要遠端存取某個容器或獲取登錄 `shell`，`Kubernetes` 將提供所需的一切。

## Services

`pod` 並不是持久的，可以出於多種原因停止或啟動它們，例如失敗的 `liveness` 或 `readiness` 檢查，這會導致問題。如果想與一組 `POD` 通訊會怎樣？重新啟動時，它們可能具有不同的 `IP` 地址。這就是為什麼要 [Service](http://kubernetes.io/docs/user-guide/services/) 的來源，`Service` 為 `POD` 提供穩定的終端。

![](https://cdn.qwiklabs.com/Jg0T%2F326ASwqeD1vAUPBWH5w1D%2F0oZn6z5mQ5MubwL8%3D)

`Service`使用*標籤*來確定它們在 `POD` 上進行操作。如果 `POD` 具有正確的標籤，`Service` 會自動將其整合並暴露出來。`Service` 提供給一組 `POD` 的訪問級別，取決於服務的類型。當前有三種類型：

- `ClusterIP` (internal) -- the default type means that this Service is only visible inside of the cluster,
- `NodePort` gives each node in the cluster an externally accessible IP and
- `LoadBalancer` adds a load balancer from the cloud provider which forwards traffic from the service to Nodes within it.


### Creating a Service
在創建 `Service` 之前，讓我們首先創建一個可以處理 `https` 流量的 `POD`。

切換至 `~/orchestrate-with-kubernetes/kubernetes` 目錄下，探索 `Service` 配置檔案

```shell
$ cat pods/secure-monolith.yaml
```

創建 `secure-monolith` POD 及其配置數據：

```shell
$ kubectl create secret generic tls-certs --from-file tls/
$ kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf
$ kubectl create -f pods/secure-monolith.yaml
```

現在有了一個安全的 `POd`，是時候在外部公開 `secure-monolith` 的 `POD` 了，為此，創建一個 `Kubernetes` 服務。

探索 `monolith` `Service` 配置檔案

```shell
$ cat services/monolith.yaml
```

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "monolith"
spec:
  selector:
    app: "monolith"
    secure: "enabled"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443
      nodePort: 31000
  type: NodePort
```

- 有一個 `selector`，用於自動查找和公開帶有標籤`app = monolith` 和 `secure = enabled` 的任何標籤
- 現在必須在此處暴露節點 `port`，因為這是將外部流量從端口 `31000` 轉發到 `nginx`端口 `443` 上的方式

使用 `kubectl create` 從 `monolith service` 配置檔案創建 `monolith service`

```shell
$ kubectl create -f services/monolith.yaml
```

我們正在使用端口暴露服務，如果另一個應用嘗試綁定到一台服務器上的端口 `31000`，則可能會發生端口衝突。通常，`Kubernetes` 將處理端口分配。使用`gcloud compute firewall-rules` 允許流量流向暴露 `nodeport`上的 `Monolith` 服務

```shell
$ gcloud compute firewall-rules create allow-monolith-nodeport \
  --allow=tcp:31000
```

現在，一切都已設置完畢，應該能夠在不使用端口轉發的情況下從集群外部訪問 `secure-monolith ` 服務。首先，獲取其中一個節點的外部 `IP` 地址，再使用 `curl` 請求
```shell
$ gcloud compute instances list
$ curl -k https://<EXTERNAL_IP>:31000 # timed out
```

我們來查看 `Service` 資源，最後原因是它與*標籤*有關。

```shell
$ kubectl get services monolith
$ kubectl describe services monolith
```


## Adding Labels to Pods
當前，`monolith` 服務沒有端點。解決此類問題的一種方法是對標籤查詢使用 `kubectl get pods`。

```shell
$ kubectl get pods -l "app=monolith"
$ kubectl get pods -l "app=monolith,secure=enabled"
```

上面的標籤查詢不無任何結果，似乎需要在其上添加`secure = enabled`標籤。使用 `kubectl label` 將缺少的 `secure=enabled` 標籤添加到 `secure-monolith` `POD` 中。之後，可以檢查並查看標籤是否已更新。

```shell
$ kubectl label pods secure-monolith 'secure=enabled'
$ kubectl get pods secure-monolith --show-labels
```
現在，`POD` 已正確標記，查看 `Monolith` 服務上的端點列表

```shell
$ kubectl describe services monolith | grep Endpoints
```

此時將會有一個端點，我們再做一次請求

```shell
$ gcloud compute instances list
$ curl -k https://<EXTERNAL_IP>:31000
```

## Deploying Applications with Kubernetes
此實驗室的目的是為擴展和管理生產中的容器做好準備。這就是 [Deployment](http://kubernetes.io/docs/user-guide/deployments/#what-is-a-deployment)的用武之地，部署是一種*聲明式*方法，可確保運行的 `POD` 數量等於用戶指定的所需 `POD` 數量。

![](https://cdn.qwiklabs.com/1UD7MTP0ZxwecE%2F64MJSNOP8QB7sU9rTI0PSv08OVz0%3D)

`Deployment` 的主要好處是可以抽象化管理 `POD` 的底層細節，在後台部署使用 [Replica Sets](http://kubernetes.io/docs/user-guide/replicasets/)來管理 `POD` 的啟動和停止。如果需要更新或擴展 `POD`，則 `Deployment` 將進行處理，如果 `POD` 由於某種原因而崩潰，則 `Deployment` 還可以重新啟動 `POD`。

![](https://cdn.qwiklabs.com/fH4ZxGNxg5KLBy5ykbwKNIS9MIJ9cgcMEDuhB0a9uBo%3D)

上圖中，`POD` 與創建它們的 `Node` 的生命週期息息相關，圖中 `Node3` 掉線了，當中帶了一個 `POD`。`Deployment` 無需手動創建新的 `POD` 並為其找到節點，而是創建了一個新的 `POD` 並在 `Node2` 上啟動它。現在該結合上述所學到的關於 `POD` 和 `Services` 的所有知識，使用 `Deployments` 將整體應用程式分解為更小的服務。


## Creating Deployments
我們將把整體應用程式分成三部分：
- auth 
    - Generates JWT tokens for authenticated users.
- hello 
    - Greet authenticated users.
- frontend 
    - Routes traffic to the auth and hello services.

我們準備建立 `Deployment` 資源，每個服務一個。然後，我們將 `auth` 和 `hello` 部署定義內部服務，並為 `frontend` 部署定義外部服務。完成後，將可以與微服務進行交互，就像使用 `Monolith` 一樣，只不過現在每個元件都可以獨立擴展和部署！

查看 `auth` `Deployment`資源配置

```shell
cat deployments/auth.yaml\apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: auth
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:2.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
...
```

該資源清單將創建 1 個副本，而我們使用的是 `auth` 容器的 `2.0.0` 版本。當運行 `kubectl create` 命令創建 `auth` 時，它將創建一個與 `Deployment` 清單中的數據一致的容器。這表示可以透過更改 `replicas` 字段中指定的數量來縮放 `POD` 的數量。

```shell
$ kubectl create -f deployments/auth.yaml
```

現在為 `auth` 創建 `Service`，使用 `kubectl create` 創建 `auth` 服務：

```shell
$ kubectl create -f services/auth.yaml
```

現在，執行相同的操作來創建和暴露 `hello` 部署

```shell
$ kubectl create -f deployments/hello.yaml
$ kubectl create -f services/hello.yaml
```

建立並暴露 `frontend` 部署。
```shell
$ kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
$ kubectl create -f deployments/frontend.yaml
$ kubectl create -f services/frontend.yaml
```

創建 `frontend` 還有另外一步，因為需要將一些配置數據與容器一起儲存。取得 `frontend` 的外部 `IP`，然後 `curl` 到前端與前端進行交互
```shell
$ kubectl get services frontend
$ curl -k https://<EXTERNAL-IP>
```
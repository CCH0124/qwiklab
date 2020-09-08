---
layout: post
title: Kubernetes in Google Cloud- 02
date: 2020-09-06
excerpt: "Kubernetes Engine: Qwik Start"
tags: [qwiklab, k8s, google]
comments: true
---

`Google Kubernetes Engine(GKE)` 提供了一個託管環境，用於使用 `Google` 基礎架構部署、管理和擴展容器化應用程序。`Kubernetes Engine` 環境由多台機器(`Compute Engine`實例)組合在一起構成一個容器集群。


### Cluster orchestration with Kubernetes Engine
`Kubernetes Engine` 群集由 `Kubernetes` 開源群集管理系統提供支援。`Kubernetes` 提供了與容器集群進行交互的機制，可以使用 `Kubernetes` 操作和資源來部署和管理應用程式、執行管理任務和設置策略以及監視已部署工作負載的運行狀況。`Kubernetes` 遵循運行受歡迎的 `Google` 服務的相同設計原則，並提供相同的好處，針對應用程式容器的自動管理、監視和活動探針、自動縮放、滾動更新等。


## Setting a default compute zone
[計算區域](https://cloud.google.com/compute/docs/regions-zones/#available)是群集及其資源所在的大致區域位置，例如 `us-central1-a` 是 `us-central1` 區域中的區域。在 `Cloud Shell` 中啟動終端機，運行以下指令將預設計算區域設置為 `us-central1-a`：

```shell
gcloud config set compute/zone us-central1-a
```

## Creating a Kubernetes Engine cluster
一個[群集](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture)至少由一個群集 *master*和多個稱為節點的計算機組成。節點是運行 `Kubernetes` 的 `Compute Engine virtual machine (VM)` 實例，該實例需要使它們成為集群的一部分。

要創建集群，運行以下指令，將 `[CLUSTER-NAME]` 替換為為集群選擇的名稱。

```shell
gcloud container clusters create [CLUSTER-NAME] 
```
建立完後，會有以下訊息

```shell
NAME        LOCATION       ...   NODE_VERSION  NUM_NODES  STATUS
my-cluster  us-central1-a  ...   1.13.11-gke.9  3          RUNNING
```

```shell
$ gcloud container clusters create cch-test
....
NAME      LOCATION       MASTER_VERSION  MASTER_IP     MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
cch-test  us-central1-a  1.15.12-gke.2   35.222.70.13  n1-standard-1  1.15.12-gke.2  3          RUNNING
```
## Get authentication credentials for the cluster
創建集群後，需要獲取身份驗證憑證才能與集群交互。要驗證集群，運行以下指令，將 `[CLUSTER-NAME]` 替換為集群的名稱：

```shell
gcloud container clusters get-credentials [CLUSTER-NAME]
```

執行完後，會輸出以下訊息

```shell
Fetching cluster endpoint and auth data.
kubeconfig entry generated for my-cluster.
```


```shell
$ gcloud container clusters get-credentials cch-test
Fetching cluster endpoint and auth data.
kubeconfig entry generated for cch-test.
```

## Deploying an application to the cluster

上面步驟已經創建了集群，可以將[容器化的應用程式](https://cloud.google.com/kubernetes-engine/docs/concepts/kubernetes-engine-overview)部署到該集群了。`Kubernetes Engine` 使用 `Kubernetes` 物件來創建和管理集群的資源。`Kubernetes` 提供了 `Deployment` 資源，用於部署無狀態應用程序，如 `Web` 服務器。`Service` 資源定義規則和負載平衡，以從 `Internet` 訪問應用程序。

在 `Cloud Shell` 中運行 `kubectl create` 從 `hello-app` 容器鏡像創建新的 `Deployment` `hello-server`

```shell
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
deployment.apps/hello-server created
```

上述創建一個代表 `hello-server` 的 `Deployment` 資源。在這種情況下，`--image` 指定要部署的容器鏡像。該命令從 `Google Container Registry` 儲存桶中拉取鏡像。`gcr.io/google-samples/hello-app:1.0` 表示要提取的特定鏡像版本，如果未指定版本，則使用 `latest` 版本。


在創建一個 `Kubernetes` 服務，它是一個 `Kubernetes` 資源，透過運行以下 `kubectl expose`，可以將應用程序暴露給外部流量

```shell
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
service/hello-server exposed
```

- `--port` 指定容器暴露的 Port
- `type="LoadBalancer"` 為容器創建一個 `Compute Engine` 負載均衡器

查看 `Service` 資源

```shell
$ kubectl get svc
NAME           TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
hello-server   LoadBalancer   10.3.246.34   <pending>     8080:31198/TCP   11s
kubernetes     ClusterIP      10.3.240.1    <none>        443/TCP          3m45s
```

在此命令的輸出中，從 `EXTERNAL IP` 該欄位複製服務的外部 `IP` 地址，並請求 `http://[EXTERNAL-IP]:8080`，頁面如下圖

![](https://i.imgur.com/sBcDV4F.png)

>生成外部 IP 地址可能需要一分鐘。如果 `EXTERNAL-IP` 列處於 `pending` 狀態，請再次運行指令


## Clean Up

刪除叢集資源

```shell
gcloud container clusters delete [CLUSTER-NAME]
```

更多訊息可查看[文件](https://cloud.google.com/kubernetes-engine/docs/how-to/deleting-a-cluster)。
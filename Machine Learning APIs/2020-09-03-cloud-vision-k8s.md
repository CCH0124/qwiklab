---
layout: post
title: Machine Learning APIs - 06
date: 2020-09-03
excerpt: "Awwvision: Cloud Vision API from a Kubernetes Cluster"
tags: [qwiklab, ML, google]
comments: true
---
## Create a Kubernetes Engine cluster

本實驗中，將使用`Google Cloud`的命令行工具[gcloud](https://cloud.google.com/sdk/gcloud)設置[Kubernetes Engine](https://cloud.google.com/kubernetes-engine)集群。

在 `Cloud Shell` 中，運行以下命令在 `us-central1-a` 區域中創建集群：

```shell
gcloud config set compute/zone us-central1-a
```

設置集群

```shell
gcloud container clusters create awwvision \
    --num-nodes 2 \
    --scopes cloud-platform
```

使用容器的認證

```shell
gcloud container clusters get-credentials awwvision
```

查看集群資訊

```shell
kubectl cluster-info
```

## Create a virtual environment

```shell
sudo apt-get update
sudo apt-get install virtualenv
virtualenv -p python3 venv
source venv/bin/activate
```

## Get the Sample

```shell
git clone https://github.com/GoogleCloudPlatform/cloud-vision
```

## Deploy the sample

```shell
cd cloud-vision/python/awwvision
make all # make all 來構建和部署所有內容
```

將構建 `Docker` 映像並將其上傳到 [Google Container Registry](https://cloud.google.com/container-registry/docs) 私有容器倉庫。此外，`yaml` 檔案將從模板生成，並填充有項目特定的訊息，並用於為實驗室部署 `redis`、`webapp` 和 `worker` Kubernetes 資源。

## Check the Kubernetes resources on the cluster

```shell
kubectl get pods # 列出部署的 POD，確認是否運行
NAME                     READY     STATUS    RESTARTS   AGE
awwvision-webapp-vwmr1   1/1       Running   0          1m
awwvision-worker-oz6xn   1/1       Running   0          1m
awwvision-worker-qc0b0   1/1       Running   0          1m
awwvision-worker-xpe53   1/1       Running   0          1m
redis-master-rpap8       1/1       Running   0          2m
```

```shell
kubectl get deployments -o wide # 取得 deployments 資源
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS         IMAGES                                SELECTOR
awwvision-webapp   1         1         1            1           1m        awwvision-webapp   gcr.io/your-project/awwvision-webapp   app=awwvision,role=frontend
awwvision-worker   3         3         3            3           1m        awwvision-worker   gcr.io/your-project/awwvision-worker   app=awwvision,role=worker
redis-master       1         1         1            1           1m        redis-master       redis                                 app=redis,role=master
```

```shell
kubectl get svc awwvision-webapp # 取得 Service 資源
NAME               CLUSTER_IP      EXTERNAL_IP    PORT(S)   SELECTOR                      AGE
awwvision-webapp   10.163.250.49   23.236.61.91   80/TCP    app=awwvision,role=frontend   13m
```

## Visit your new web app and start its crawler

將 `awwvision-webapp` 服務的外部 IP 複製，在瀏覽器中輸入該 IP，以打開 `webapp`，然後單擊` Start the Crawler`按鈕。



## 結論

從這一系列的練習，我知道 `API` 的定義和一些知識，以及使用 `API` 方式去做像是影像識別和 `NLP` 等的請求。
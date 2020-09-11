本實驗將使用 `Kubernetes Engine` 部署分散式負載測試框架，該框架使用多個容器將 `REST` 的簡單 `API` 創建負載測試流量。此解決方案可以測試簡單的 `Web` 應用程序，但可以使用相同的模式來創建更複雜的負載測試方案，如：遊戲或物聯網(IoT)應用程式。該解決方案討論了基於容器的負載測試框架的一般體系結構。

##### System under test
實驗中，被測系統是一個部署在 `Google App Engine` 的小型 `Web` 應用程式。該應用程式暴露了基本的 `REST` 樣式的端點，以獲取傳入的 `HTTP POST` 請求。

##### Example workloads
要部署的應用程式是根據許多物聯網(IoT)部署中的後端服務元件建模的。設備會先向該服務註冊，然後開始回傳指標或讀取傳感器，同時還定期向該服務重新註冊。

常見的後端服務元件交互如下：

![](https://cdn.qwiklabs.com/snd%2F3SIGwp84rlFJFEtf5EU582Ro%2FXy430FH6L%2B12n8%3D)

為了對此交互進行建模，將使用 `Locust`，是一個基於 `Python` 的分散式負載測試工具，能夠在多個目標路徑之間分配請求。例如 `Locust` 可以將請求分發到 `/login` 和 `/metrics` 路徑。工作負載基於上述的交互，會被建構為 `Locust` 中的一組任務。


##### Container-based computing
- `Locust` 容器鏡像是包含 `Locust` 軟體的 `Docker` 鏡像
- 一個容器集群由至少一個集群 `master` 和多個 `node` 運算電腦組成，這些 `master` 和 `node` 運行 `Kubernetes` 集群業務流程系統
- `POD` 是在一個主機上部署的一個或多個容器，可以定義為部署和管理的最小單元
- `Deployment controller` 為 `POD` 和 `ReplicaSet` 提供聲明式更新，實驗會有兩個部署：一個是 `locust-master` 其他則是 `locust-worker`
- Services

特定的 `POD` 可能會由於多種原因而異常，包括節點故障、更新等。`POD` 的 `IP` 地址不能為該 `POD` 提供可靠的接口。一種更可靠的方法是使用該接口的抽象表示形式，該表示形式永遠不變，即使容器異常被具有不同 IP 地址的新容器替換也是如此。`Kubernetes Engine` 的 `Service` 透過定義 `POD` 的邏輯集和訪問它們的策略來提供這種抽象接口。

實驗中，有幾種服務代表 `POD` 或 `POD `集合。例如，有一個 `DNS` 的 `POD` 服務，另一個用於 `Locust master` 的 `POD` 服務，以及一個代表所有 10 個 `Locust worker` 的 `POD`。下圖顯示了 `master` 和 `worker` 節點的內容

![](https://cdn.qwiklabs.com/YuSloHJWYCRVD%2FYqXWXjxQrzz%2F57LhROxH4fgqwxD%2Fs%3D)


## Set project and zone
使用環境變數儲存
```shell
PROJECT=$(gcloud config get-value project)
REGION=us-central1
ZONE=${REGION}-a
CLUSTER=gke-load-test
TARGET=${PROJECT}.appspot.com
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```


## Get the sample code and build a Docker image for the application
```shell
git clone https://github.com/GoogleCloudPlatform/distributed-load-testing-using-kubernetes.git
cd distributed-load-testing-using-kubernetes/
```

構建 `docker` 鏡像並將其儲存在容器倉庫中
```shell
gcloud builds submit --tag gcr.io/$PROJECT/locust-tasks:latest docker-image/.
latest: digest: sha256:03b7d06b262dd2d55f698a0a823e9ae0b46ec2b41f55421c7d78559886603bb5 size: 3053
DONE
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

ID                                    CREATE_TIME                DURATION  SOURCE                                                                                                      IMAGES                                                      STATUS
7c530954-8846-4ac8-950b-0d93b71aa5d3  2020-09-11T13:29:35+00:00  2M1S      gs://qwiklabs-gcp-00-dbe7f4452e89_cloudbuild/source/1599830972.535606-e82dfb11e7f54533835acb1327850b5e.tgz  gcr.io/qwiklabs-gcp-00-dbe7f4452e89/locust-tasks (+1 more)  SUCCESS
```


## Deploy Web Application
`sample-webapp` 包含一個簡單的 `Google App Engine` 的 `Python` 應用程序，作為*被測系統*。使用 `gcloud app deploy` 將應用程序部署

```shell
gcloud app deploy sample-webapp/app.yaml
```

## Deploy Kubernetes Cluster
```shell
gcloud container clusters create $CLUSTER \
  --zone $ZONE \
  --num-nodes=5
kubeconfig entry generated for gke-load-test.
NAME           LOCATION       MASTER_VERSION  MASTER_IP       MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
gke-load-test  us-central1-a  1.15.12-gke.2   104.154.202.70  n1-standard-1  1.15.12-gke.2  5          RUNNING
```

## Load testing master
部署的第一個組件是 `Locust master`，它執行負載測試任務的入口點。`Locust master` 使用單個副本部署，因只需要一個 `master`。`master` 部署的配置指定了幾個元素，包括需要容器暴露的端口，有用於 `Web` 界面的 `8089`和用於與 `workers` 通訊的 `5557` 和 `5558`，該資訊將用於配置 `Locust worker`。

```shell
ports:
   - name: loc-master-web
     containerPort: 8089
     protocol: TCP
   - name: loc-master-p1
     containerPort: 5557
     protocol: TCP
   - name: loc-master-p2
     containerPort: 5558
     protocol: TCP
```

### Deploy locust-master
將 `locust-master-controller.yaml` 和 `locust-worker-controller.yaml` 中 `[TARGET_HOST]` 和 `[PROJECT_ID]` 分別替換為已部署的端點和專案 `ID`。

```shell
sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-master-controller.yaml
sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-worker-controller.yaml
sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-master-controller.yaml
sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-worker-controller.yaml
```

部署 `Locust master`
```shell
kubectl apply -f kubernetes-config/locust-master-controller.yaml
kubectl get pods -l app=locust-master
NAME                             READY   STATUS    RESTARTS   AGE
locust-master-75b8c5bf69-9vmfc   1/1     Running   0          40s
```
部署 `Service` 資源
```shell
kubectl apply -f kubernetes-config/locust-master-service.yaml
```

此步驟將使用內部 `DNS` 名稱 `locust-master` 和端口 `8089`、`5557` 和 `5558` 暴露 `POD`。作為此步驟的一部分，`locust-master-service.yaml` 中的 `T​​ype:LoadBalancer` 告訴 `Google Kubernetes Engine` 創建一個可以公開可用 `IP` 地址到 `locust-master POD` 的 `Compute Engine` 轉發規則。

```shell
kubectl get svc locust-master
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                        AGE
locust-master   LoadBalancer   10.3.251.175   <pending>     8089:31506/TCP,5557:32327/TCP,5558:31603/TCP   26s
```

## Load testing workers
部署的下一個組件包括 `Locust worker`，它們執行上述負載測試任務。`Locust workers` 創建多個 `POD` 藉由一個 `deployment` 進行部署，`POD` 分佈在 `Kubernetes` 集群中。每個 `POD` 使用環境變數來控制重要的配置訊息，例如被測系統的主機名和 `Locust master` 的主機名。

部署 `Locust worker` 後，可以返回 `Locust` 主 `Web` 界面，並查看數量與已部署的 `workers` 程序的數量相對應。

```shell
apiVersion: "extensions/v1beta1"
kind: "Deployment"
metadata:
  name: locust-worker
  labels:
    name: locust-worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: locust-worker
  template:
    metadata:
      labels:
        app: locust-worker
    spec:
...
```

### Deploy locust-worker
部署 `locust-worker-controller.yaml`
```shell
kubectl apply -f kubernetes-config/locust-worker-controller.yaml
```

`locust-worker-controller` 設置為部署 5 個 `locust-worker` 的 `POD`
```shell
kubectl get pods -l app=locust-worker # 驗證
NAME                             READY   STATUS              RESTARTS   AGE
locust-worker-79c789f688-b9pkk   0/1     ContainerCreating   0          11s
locust-worker-79c789f688-bc4x2   0/1     ContainerCreating   0          11s
locust-worker-79c789f688-nbpff   0/1     ContainerCreating   0          11s
locust-worker-79c789f688-tncsw   1/1     Running             0          11s
locust-worker-79c789f688-wcqxr   0/1     ContainerCreating   0          11s
```

擴大模擬用戶的數量將需要增加 `Locust worker` 的 `POD` 數量。為了增加 `deployment` 所部署的 `POD` 數量，`Kubernetes` 提供了無需重新部署即可調整部署大小的功能。
```shell
kubectl scale deployment/locust-worker --replicas=20
kubectl get pods -l app=locust-worker # 驗證
NAME                             READY   STATUS              RESTARTS   AGE
locust-worker-79c789f688-2vwqb   1/1     Running             0          7s
locust-worker-79c789f688-87zt6   1/1     Running             0          7s
locust-worker-79c789f688-8ghgl   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-8twg5   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-b9pkk   1/1     Running             0          47s
locust-worker-79c789f688-bc4x2   1/1     Running             0          47s
locust-worker-79c789f688-bkb85   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-cjttv   1/1     Running             0          7s
locust-worker-79c789f688-dljmd   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-f67lb   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-f9ss4   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-lx55n   1/1     Running             0          7s
locust-worker-79c789f688-nbpff   1/1     Running             0          47s
locust-worker-79c789f688-rsjhz   1/1     Running             0          7s
locust-worker-79c789f688-t5stx   1/1     Running             0          7s
locust-worker-79c789f688-tncsw   1/1     Running             0          47s
locust-worker-79c789f688-vs7df   1/1     Running             0          7s
locust-worker-79c789f688-wcqxr   1/1     Running             0          47s
locust-worker-79c789f688-zg578   0/1     ContainerCreating   0          7s
locust-worker-79c789f688-zxm5c   1/1     Running             0          7s
```


下圖顯示了 `Locust master` 和 `Locust workers` 之間關係

![](https://cdn.qwiklabs.com/QYSigq8YwCYxOoT4X8EHZuugeOjPlOlz5kOY57CMR8I%3D)


## Execute Tests
要執行 `Locust` 測試，藉由以下獲取對外 IP 地址
```shell
EXTERNAL_IP=$(kubectl get svc locust-master -o yaml | grep ip | awk -F": " '{print $NF}')
echo http://$EXTERNAL_IP:8089
http://34.123.127.42:8089
```

`Locust master` Web 界面能夠針對被測系統執行負載測試任務，如以下示例圖像所示：
![](https://cdn.qwiklabs.com/%2BDaEkQJ83auIGFBoUyCB6LUPXohJOlTgJnbHqBEn7ME%3D)

首先，指定要模擬的用戶總數以及應產生每個用戶的速率。接下來，點擊 `Start swarming` 以開始模擬，可以將用戶數量指定為 300，將速率指定為 10。

![](https://i.imgur.com/OyfPjoi.png)

隨著時間的變化和用戶的產生，將看到統計信息開始匯總模擬指標，如*請求數*和*每秒請求數*。點擊 `Stop` 即可停止模擬。
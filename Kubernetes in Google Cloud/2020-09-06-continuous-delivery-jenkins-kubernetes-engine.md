---
layout: post
title: Kubernetes in Google Cloud- 05
date: 2020-09-06
excerpt: "Continuous Delivery with Jenkins in Kubernetes Engine"
tags: [qwiklab, k8s, google, jenkins]
comments: true
---

`Jenkins` 是經常將其代碼集成到共享倉庫(repository)中的開發人員使用的自動化服務。本實驗中構建的解決方案類似於下圖：

![](https://cdn.qwiklabs.com/1b%2B9D20QnfRjAF8c6xlXmexot7TDcOsYzsRwp%2FH4ErE%3D)


可以在[此處](https://cloud.google.com/solutions/jenkins-on-container-engine)找到有關在 `Kubernetes` 上運行 `Jenkins` 的更多詳細訊息


### What is Jenkins?
[Jenkins](https://jenkins.io/) 是一個開源自動化服務，可靈活的協調構建、測試和部署管道。`Jenkins` 允許開發人員快速迭代項目，而不必擔心可能由於持續交付而產生的開銷問題。

### What is Continuous Delivery / Continuous Deployment?
當需要設置持續交付(Continuous Delivery, CD)管道時，與基於 `VM` 的標準部署相比，在 `Kubernetes Engine` 上部署 `Jenkins` 具有重要的好處。當構建過程使用容器時，一台虛擬主機可以在多個作業系統上運行作業。`Kubernetes Engine` 提供了臨時構建執行器(ephemeral build executors)，僅在構建正在運行時才使用這些執行器，這為其他集群任務，如：批處理作業等留下了資源。臨時構建執行程序的另一個好處是*速度*，它們只需幾秒鐘即可啟動。

`Kubernetes Engine` 還預先配置 `Google` 的全域負載均衡器，可以使用該負載均衡器自動將網路流量路由到應用程式(實例)。負載均衡器處理 `SSL` 終止(SSL termination)，並利用配置有 `Google` 骨幹網路的全域 `IP` 地址，結合 `Web` 前端，此負載均衡器將使客戶端使用最快的路徑訪問應用程式實例。


## Clone the repository
進行設置，在 `Cloud Shell` 中打開一個終端機，然後運行以下將區域設置為 `us-east1-d`
```shell
$ gcloud config set compute/zone us-east1-d
```

```shell
$ git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
$ cd continuous-deployment-on-kubernetes
```

## Provisioning Jenkins
### Creating a Kubernetes cluster

配置 `Kubernetes` 集群，額外的範圍使 `Jenkins` 可訪問 `Cloud Source Repository` 和 `Google Container Registry`。

```shell
$ gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"
```

透過執行以下命令來確認集群正在運行
```shell
$ gcloud container clusters list
NAME        LOCATION    MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
jenkins-cd  us-east1-d  1.15.12-gke.2   35.237.45.224  n1-standard-2  1.15.12-gke.2  2          RUNNING
```
取得認證
```shell
$ gcloud container clusters get-credentials jenkins-cd
```

`Kubernetes Engine` 使用上述憑證來存取配置的群集，確認可以透過運行以下來連接到該群集
```shell
$ kubectl cluster-info
```

## Install Helm
將使用 `Helm` 從 `Charts` 儲存庫安裝 `Jenkins`。`Helm` 是一個軟體管理器，可以輕鬆配置和部署 `Kubernetes` 應用程式。一旦，安裝 `Jenkins` 就可以設置 `CI/CD` 管道。

1. 下載並安裝 `Helm` 的 `binary` 檔案
```shell
$ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.14.1-linux-amd64.tar.gz
```
2. 在 `Cloud Shell` 中解壓縮檔案
```shell
$ tar zxfv helm-v2.14.1-linux-amd64.tar.gz
$ cp linux-amd64/helm .
```
3. 在群集的 `RBAC` 中將自己添加為群集管理員，以便可在群集中授予 `Jenkins` 權限
```shell
$ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```
4. 允許 `Tiller`、`Helm` 的服務器端，集群中的 `cluster-admin` 角色
```shell
$ kubectl create serviceaccount tiller --namespace kube-system
$ kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```
5. 初始 `Helm`，這樣可確保在群集中正確安裝了 `Helm(Tiller)` 的服務器端
```shell
$ ./helm init --service-account=tiller
$ ./helm update
```
6. 透過以下指令，確保正確安裝了 `Helm`。應該看到顯示 `v2.14.1` 的服務器和客戶端的版本
```shell
$ ./helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```

## Configure and Install Jenkins
將使用自定義值檔案添加特定於 `Google Cloud` 的插件，該插件是使用 `service account` 憑證訪問 `Cloud Source Repository` 所必需的。

1. 用 `Helm` CLI 使用配置設置來部署 `chart`
```shell
$ ./helm install -n cd stable/jenkins -f jenkins/values.yaml --version 1.2.2 --wait
```

```yaml
# jenkins/values.yaml
master:
  installPlugins:
    - kubernetes:latest
    - workflow-job:latest
    - workflow-aggregator:latest
    - credentials-binding:latest
    - git:latest
    - google-oauth-plugin:latest
    - google-source-plugin:latest
    - google-kubernetes-engine:latest
    - google-storage-plugin:latest
  resources:
    requests:
      cpu: "50m"
      memory: "1024Mi"
    limits:
      cpu: "1"
      memory: "3500Mi"
  javaOpts: "-Xms3500m -Xmx3500m"
  serviceType: ClusterIP
agent:
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "512Mi"
persistence:
  size: 100Gi
serviceAccount:
  name: cd-jenkins
```
2. 確認 `Jenkins` 進入`RUNNING`狀態，並且容器處於`READY`狀態
```shell
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
cd-jenkins-7b7656d7f6-ww2h6   1/1     Running   0          4m4s
```
3. 配置 `Jenkins` 的 `service account`，讓它能夠部署到集群
```shell
$ kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
clusterrolebinding.rbac.authorization.k8s.io/jenkins-deploy created
```
4. 運行以下以設置從 `Cloud Shell` 到 `Jenkins UI` 的 `Port` 轉發
```shell
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```
5. 確認是否正確創建了 `Jenkins` 服務
```shell
$ kubectl get svc
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
cd-jenkins         ClusterIP   10.3.246.4     <none>        8080/TCP    5m42s
cd-jenkins-agent   ClusterIP   10.3.252.159   <none>        50000/TCP   5m42s
kubernetes         ClusterIP   10.3.240.1     <none>        443/TCP     9m34s
```
這邊正在使用 [Kubernetes 套件](https://plugins.jenkins.io/kubernetes/)，以便在 `Jenkins` 的主服務請求時根據需求自動啟動我們的構建器節點。完成工作後，將自動關閉，並將資源重新添加到群集資源池中。此服務為匹配 `selector` 的任何 `POD` 暴露端口 `8080` 和 `50000`。這將公開 `Kubernetes` 集群中的 `Jenkins Web UI` 和 `builder/agent` 的端口。此外，`jenkins-ui` 服務是使用 `ClusterIP` 公開的，因此無法從集群外部進行存取。

## Connect to Jenkins

1. `Jenkins` chart 將自動創建一個管理員密碼
```shell
$ printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
Czjd8hyfLE
```

2. 要進入 `Jenkins` 界面，在 `Cloud shell` 中點擊 `Web Preview` 按鈕，然後在端口 `8080` 上單擊 `Preview`

![](https://cdn.qwiklabs.com/wy13PEPdV6ZbYMJR2tmk3iKe%2FEyVDXVWtrWFVJeZUXk%3D)

3. 現在，應該可以使用客戶端 `admin` 和自動生成的密碼登錄

現在，已經在 `Kubernetes` 集群中設置了 `Jenkins`！


## Understanding the Application

這邊會在持續部署管道中部署範例應用程式 `gceme`，該應用程式是用 Go 語言編寫的，位於倉庫的 `sample-app` 目錄中。當在 `Compute Engine` 實例上運行 `gceme` 時，該應用程式會在訊息卡中顯示該實例的元數據。

![](https://cdn.qwiklabs.com/ceqFX6Vwtd12NSBtnNhrkemKfRSLHbCmFZiLn8WmC98%3D)

該應用程式透過支持兩種操作模式來模仿微服務。
- 在**後端模式**下，`gceme` 監聽端口 `8080` 並以 `JSON` 格式回傳 `Compute Engine` 實例元數據。
- 在**前端模式**下，`gceme` 查詢後端 `gceme` 服務，並在用戶界面中呈現 `JSON`

![](https://cdn.qwiklabs.com/P1T5JBWWprA4iLf%2FB5%2BO6as7otLE25YBde57gzZwSz4%3D)



## Deploying the Application
將應用程式部署到兩個不同的環境中

- Product
    - 客戶端訪問的即時站點
- Canary
    - 一個容量較小的網站，僅接收客戶端流量的一部分。在將軟體發布給所有客戶端之前，使用此環境透過即時流量驗證軟體

在 `Cloud shell` 執行

```shell
$ cd sample-app
```

在 `Kubernetes` 建立 `namespace` 讓邏輯上是隔離部署

```shell
$ kubectl create ns production
```

創建 `Product` 部署和 `Canary` 部署，並使用 `kubectl apply` 執行部署

```shell
$ kubectl apply -f k8s/production -n production
$ kubectl apply -f k8s/canary -n production
$ kubectl apply -f k8s/services -n production
```

預設情況下，僅部署*前端*一個副本，使用 `kubectl scale` 確保至少 4 個副本在運行。透過以下來擴展 `Product` 環境前端

```shell
$ kubectl scale deployment gceme-frontend-production -n production --replicas 4
```

現在，確認有 5 個 `POD` 用於前端，4 個用於 `Product` 流量，1 個用於 `Canary` 版本，對 `Canary` 版本的更改僅會影響 5 個客戶端中的 1 個，機率為 20%

```shell
$ kubectl get pods -n production -l app=gceme,role=frontend
NAME                                         READY   STATUS    RESTARTS   AGE
gceme-frontend-canary-5f996dd744-cl2sv       1/1     Running   0          81s
gceme-frontend-production-76bb445bbc-b99d5   1/1     Running   0          6s
gceme-frontend-production-76bb445bbc-dk7v9   0/1     Running   0          6s
gceme-frontend-production-76bb445bbc-h9t6b   1/1     Running   0          93s
gceme-frontend-production-76bb445bbc-z557d   0/1     Running   0          6s
```
還要確認有 2 個用於後端的 `POD`，1 個用於 `Product`，1 個用於 `Canary`

```shell
$ kubectl get pods -n production -l app=gceme,role=backend
NAME                                       READY   STATUS    RESTARTS   AGE
gceme-backend-canary-d6f654f8b-w5nxn       1/1     Running   0          108s
gceme-backend-production-d6ff496f9-pgbsw   1/1     Running   0          2m
```

取得 `Product` 服務的對外 IP

```shell
$ kubectl get svc gceme-frontend -n production
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
gceme-frontend   LoadBalancer   10.3.243.131   34.75.239.62   80:30078/TCP   2m6s
```

將對外 `IP` 輸入到瀏覽器中以查看網頁上顯示的訊息卡，如下所示

![](https://i.imgur.com/d1Xud73.png)

現在，將前端服務負載均衡器 IP 設在一個環境變量中，後面方便使用

```shell
$ export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
```

透過在瀏覽器中打開前端對外 `IP` 地址，確認兩種服務均正常運行。透過運行以下檢查服務的版本輸出，應顯示為 `1.0.0`
```shell
$ curl http://$FRONTEND_SERVICE_IP/version
```

上述都順利的話，表示成功部署了範例應用程序！

## Creating the Jenkins Pipeline
### Creating a repository to host the sample app source code
建立 `gceme` 範例應用程式的複製並將其推送到 [Cloud Source Repository](https://cloud.google.com/source-repositories/docs/)

```shell
$ gcloud source repos create default
$ git init # 初始化 sample-app
$ git config credential.helper gcloud.sh
$ git remote add origin https://source.developers.google.com/p/$DEVSHELL_PROJECT_ID/r/default
$ git config --global user.email "[EMAIL_ADDRESS]"
$ git config --global user.name "[USERNAME]"
$ git add .
$ git commit -m "Initial commit"
$ git push origin master
```

### Adding your service account credentials
配置憑據以允許 `Jenkins` 存取代碼倉庫，`Jenkins` 將使用群集的 `service account` 憑據，以便從 `Cloud Source Repository` 下載代碼。

##### Step 1
在 `Jenkins` 界面中，單擊左側選單欄中的 `Manage Jenkins`，然後點擊 `Manage Credentials`
![](https://i.imgur.com/drY41vY.png)
##### Step 2
點擊 `Jenkins`
![](https://cdn.qwiklabs.com/SiqLluBt531RSaLWnUZm28LRRVJCVBbINUZz7vaSyfo%3D)
##### Step 3
點擊 `Global credentials (unrestricted)`
##### Step 4
點擊左側選單的 `Add Credentials`
##### Step 5
從 `Kind` 下拉列表中的`Google Service Account from metadata`，然後點擊 `OK`。

![](https://i.imgur.com/5Akvvb1.png)


全域憑證已添加，憑證的名稱在會在的 `CONNECTION DETAILS` 部分中找到的 `Project ID`

![](https://i.imgur.com/XYAjfhj.png)


### Creating the Jenkins job
跳到 `Jenkins` 界面，然後按照以下步驟配置管道作業。

##### Step 1
左側選單點擊 `Jenkins > New Item`
![](https://cdn.qwiklabs.com/KXB9gx8B77F27XAgu6bgXle0HvnoERwlPf9f2WYVqQM%3D)

##### Step 2
將專案命名為 `sample-app`，然後選擇 `Multibranch Pipeline` 選項，然後點擊 `OK`
![](https://i.imgur.com/j6uGZdS.png)
##### Step 3
在下一頁的 `Branch Sources` 部分中，點擊 ` Add Source` 並選擇 `git`
![](https://i.imgur.com/6CsS26y.png)
##### Step 4
將 `Cloud Source Repositories` 中的範例應用程式倉庫的 `HTTPS clone URL` 貼到 `Project Repository` 字段中，將[PROJECT_ID]替換為此實驗的專案名稱

```shell
https://source.developers.google.com/p/[PROJECT_ID]/r/default
```

##### Step 5
從 `Credentials` 下拉列表中，選擇在前面的步驟中添加 `service account` 時創建的憑證的名稱

![](https://i.imgur.com/mGiTyU8.png)

##### Step 6
在 `Scan Multibranch Pipeline Triggers` 部分下，選則 `Periodically if not otherwise run` 框，並將*間隔(Interval)*值設置為 1 分鐘
![](https://i.imgur.com/qvjjvJx.png)

##### Step 7
點擊 `Save`，其它選項保留為默認值。


完成這些步驟後，將執行名為 `Branch indexing` 的作業。此 `meta-job` 標識倉庫中的分支，並確認現有分支中未發生更改。



## Creating the Development Environment
開發分支是一組環境，開發人員可以使用這些環境在提交代碼更改以集成到即時網站之前對其進行測試。這些環境是應用程式的較小版本，但需要使用與即時環境相同的機制進行部署。

### Creating a development branch
要從 `feature` 分支建立開發環境，可將該分支推送到 `Git` 服務器，並讓 `Jenkins` 部署環境。

創建一個開發分支並將其推送到 `Git` 服務器
```shell
$ git checkout -b new-feature
```

### Modifying the pipeline definition
定義管道的 `Jenkinsfile` 是使用 [Jenkins Pipeline Groovy](https://jenkins.io/doc/book/pipeline/syntax/) 語法編寫。使用 `Jenkinsfile` 允許整個構建管道以與源代碼一起儲存在相同檔案表示。管道支持強大的功能，如 `parallelization`，但需要客戶端的手動批准。

修改 `Jenkinsfile` 來設置專案 `ID`

```shell
$ vi Jenkinsfile
def project = 'REPLACE_WITH_YOUR_PROJECT_ID'
def appName = 'gceme'
def feSvcName = "${appName}-frontend"
def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
```

檔案中是這樣
```shell
environment {
    PROJECT = "qwiklabs-gcp-03-c5c35de82f0a"
    APP_NAME = "gceme"
    FE_SVC_NAME = "${APP_NAME}-frontend"
    CLUSTER = "jenkins-cd"
    CLUSTER_ZONE = "us-east1-d"
    IMAGE_TAG = "gcr.io/${PROJECT}/${APP_NAME}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
    JENKINS_CRED = "${PROJECT}"
  }
```
### Modify the site
為了實驗如何更改應用程序，將把 `gceme` 卡從藍色更改為橙色。
```shell
$ vi html.go
const version string = "1.0.0" # 換成 2.0.0
```


## Kick off Deployment
使用 `git` 提交並推送更改
```shell
$ git add Jenkinsfile html.go main.go
$ git commit -m "Version 2.0.0"
$ git push origin new-feature
```

將啟動開發環境的構建。將更改推送到 `Git` 倉庫後，跳到 `Jenkins` 用戶界面，可以在當中看到針對 `new-feature` 分支的構建已開始。

![](https://i.imgur.com/iZ7EUhJ.png)

運行構建後，點擊左側選單該構建旁邊的向下箭頭，然後選擇 `Console Output`

![](https://cdn.qwiklabs.com/3SL2TROBOgHPAAXRLhLCMEmyJ4h0LJGZAH2aNrLsToQ%3D)

不知道為何點箭頭沒反應，改點該箭頭下面的文字

![](https://i.imgur.com/EpE9VWB.png)

追蹤構建的輸出幾分鐘後，觀察 `kubectl --namespace = new-feature apply ...` 訊息是否觸發，該 `new-feature` 分支現在將部署到集群。如果在 `Build Executor` 中沒有看到任何內容，只需轉到 `Jenkins homepage --> sample app`。驗證是否已創建 `new-feature` 管道。

![](https://i.imgur.com/0WH7XpR.png)

在背景運行 `proxy`

```shell
$ kubectl proxy &
```
如果當掉，按 `Ctrl + C` 退出。將透過將請求發送到 `localhost` 並讓 `kubectl` 代理將其轉發到服務來驗證應用程式是否可訪問
```shell
$ curl \
http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/proxy/version
# 應該看到它以 `2.0.0` 版本回應
```

如果出現以下錯誤，表示前端端點尚未暴露，等個幾秒或分再次執行 `curl`。
```shell
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
  },
  "status": "Failure",
  "message": "no endpoints available for service \"gceme-frontend:80\"",
  "reason": "ServiceUnavailable",
  "code": 503
```



>在開發場景中，不會使用面向公眾的負載均衡器。為了保護應用程式的安全，可以使用 `kubectl proxy`。代理通過 `Kubernetes API` 對其自身進行身份驗證，並在不將服務暴露給 `Internet` 的情況下，將本地運算機器對集群中服務的請求進行代理。


## Deploying a Canary Release

我們已經驗證了應用程式在開發環境中正在運行最新的代碼，因此現在將該代碼部署到 `canary` 環境中。

建立一個 `canary` 分支並推送到 `Git` 服務器
```shell
$ git checkout -b canary
$ git push origin canary
```

在 `Jenkins` 應該看到 `canary` 管道已經啟動。完成後，可以檢查服務 `URL`，確認新版本正在處一些流量，應該可觀察到到大約五分之一的請求返回版本`2.0.0`。

```shell
$ export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
$ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done # 觀察請求，退出使用 `Ctrl + C`
```

## Deploying to production
上一步金絲雀發布成功了，且假設沒有聽到任何客戶投訴，我們將部署到其餘生產團隊中。

使用 `git` 做合併
```shell
$ git checkout master
$ git merge canary
```

在 `Jenkins`，應該看到主管道已啟動。當完成後，可以檢查服務 `URL` 以確保新版本 `2.0.0` 可以回應所有請求流量。
```shell
$ export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
$ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
```


同時跳到 `gceme` 應用程式網站在其上顯示訊息卡的顏色從藍色變為橙色。再次使用以下命令獲取對外 `IP` 地址
```shell
$ kubectl get service gceme-frontend -n production
```
我指改到著個的 `div`...

![](https://i.imgur.com/Zzrj8i4.png) 


## Other

`Jenkins` 建置的流程

![](https://i.imgur.com/svKHU1C.png)
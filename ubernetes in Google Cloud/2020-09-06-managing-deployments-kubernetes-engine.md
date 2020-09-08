---
layout: post
title: Kubernetes in Google Cloud- 04
date: 2020-09-06
excerpt: "Managing Deployments Using Kubernetes Engine"
tags: [qwiklab, k8s, google]
comments: true
---

`DevOps` 的實踐將定期使用多個部署來管理應用程式部署方案，如 `Continuous Deployment`、`Blue-Green Deployments`、`Canary Deployments`等。


## Introduction to deployments
異構部署通常涉及連接兩個或多個不同的基礎架構環境或區域，以解決特定的技術或維運需求。異構部署稱為`hybrid`、`multi-cloud` 或 `public-private`，具體取決於部署的細節。就本實驗而言，異構部署包括跨越單個雲環境、多個公有雲環境（多雲）或本地和公有雲環境（混合或公共-私有）組合的區域的那些部署。

限於單個環境或區域的部署中可能會遇到各種業務和技術挑戰：

- Maxed out resources
    - In any single environment, particularly in on-premises environments, you might not have the compute, networking, and storage resources to meet your production needs.
- Limited geographic reach
    - Deployments in a single environment require people who are geographically distant from one another to access one deployment. Their traffic might travel around the world to a central location.
- Limited availability
    - Web-scale traffic patterns challenge applications to remain fault-tolerant and resilient.
- Vendor lock-in
    - Vendor-level platform and infrastructure abstractions can prevent you from porting applications.
- Inflexible resources
    - Your resources might be limited to a particular set of compute, storage, or networking offerings.

異構部署可以幫助解決這些挑戰，但是必須使用編程和確定性的流程和程序來進行架構設計。一次性或臨時部署程序可能導致部署或流程脆弱且無法容忍故障，臨時程序可能會丟失數據或減少流量。良好的部署過程必須是可重複的，並使用行之有效的方法來管理供應、配置和維護。異構部署的三種常見方案是 `multi-cloud deployments`、`fronting on-premises data`和 `continuous integration/continuous delivery (CI/CD)` 流程。

## Setup
```shell
$ gcloud config set compute/zone us-central1-a
$ git clone https://github.com/googlecodelabs/orchestrate-with-kubernetes.git
$ cd orchestrate-with-kubernetes/kubernetes
$ gcloud container clusters create bootcamp --num-nodes 5 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

## Learn about the deployment object
首先，讓我們看一下 `Deployment` 對象。`kubectl` 中的 `explain` 可以告訴我們有關 `Deployment` 對象訊息。也可以使用 `--recursive` 選項查看所有字段。

```shell
$ kubectl explain deployment
$ kubectl explain deployment --recursive
```

使用 `explain` 可幫助了解 `Deployment` 對象的結構並了解各個字段的作用。
```shell
$ kubectl explain deployment.metadata.name
```

## Create a deployment

更新 `Deployments/auth.yaml` 檔案

```shell
$ vi deployments/auth.yaml
...
containers:
- name: auth
  image: kelseyhightower/auth:1.0.0 # 變更鏡像
...
```

修改完後

```shell
$ cat deployments/auth.yaml 
apiVersion: extensions/v1beta1
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
          image: "kelseyhightower/auth:1.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
...
```

當運行 `kubectl create` 建立 `auth` 部署時，它將創建一個與 `Deployment` 清單中的數據一致的容器。這表示可以透過更改 `replicas` 字段中指定的數量來縮放 `POD` 的數量。

建立
```shell
$ kubectl create -f deployments/auth.yaml
```
驗證是否建立
```shell
$ kubectl get deployments
```

創建部署後，`Kubernetes` 將為該部署結果創建一個 `ReplicaSet`。可以驗證是否為我們的部署創建了 `ReplicaSet`

```shell
$ kubectl get replicasets
NAME              DESIRED   CURRENT   READY   AGE
auth-7b97477d9f   1         1         0       3s
```

最後，可以查看在部署過程中創建的 `POD`。創建 `ReplicaSet` 時，`Kubernetes` 將創建單個 `POD`。
```shell
$ kubectl get pods
```

是時候為 `auth` 部署創建 `Service`了。你已經看過 `Service` 清單檔案，在此不再贅述。使用 `kubectl create` 命令創建 `auth` 的 `Service`，同樣執行相同的操作來創建 `Hello` 的 `Deployment` 和 `Service`，`frontend` 同樣也是。
```shell
$ kubectl create -f services/auth.yaml
$ kubectl create -f deployments/hello.yaml
$ kubectl create -f services/hello.yaml
$ kubectl create secret generic tls-certs --from-file tls/
$ kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
$ kubectl create -f deployments/frontend.yaml
$ kubectl create -f services/frontend.yaml
```

取得 `frontend` 的外部 `IP`，並用 `curl` 請求
```shell
$ kubectl get services frontend
$ curl -ks https://<EXTERNAL-IP>
```

也可以使用 `kubectl` 的輸出模板功能將 `curl` 作為一起使用。

```shell
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`
```

## Scale a Deployment
現在我們已經創建了一個 `Deployment`，我們可以對其進行*擴展*。透過更新 `spec.replicas` 字段來執行此操作。可以再次使用 `kubectl explain` 查看該字段的解釋。

```shell
$ kubectl explain deployment.spec.replicas
```

使用 `kubectl scale` 可以以輕鬆方式更新 `replicas` 字段
```shell
$ kubectl scale deployment hello --replicas=5
$ kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP            NODE                                      NOMINATED NODE   READINESS GATES
auth-7b97477d9f-7pd2h       1/1     Running   0          2m52s   10.112.1.3    gke-bootcamp-default-pool-3433fbf1-rlqp   <none>           <none>
frontend-7779fdcd57-9lm7m   1/1     Running   0          103s    10.112.3.4    gke-bootcamp-default-pool-3433fbf1-gkr6   <none>           <none>
hello-97b76db46-k9jmt       0/1     Running   0          7s      10.112.2.3    gke-bootcamp-default-pool-3433fbf1-zhvp   <none>           <none>
hello-97b76db46-kf9sn       1/1     Running   0          112s    10.112.4.3    gke-bootcamp-default-pool-3433fbf1-7vk1   <none>           <none>
hello-97b76db46-ks6jc       1/1     Running   0          112s    10.112.1.4    gke-bootcamp-default-pool-3433fbf1-rlqp   <none>           <none>
hello-97b76db46-l998r       1/1     Running   0          112s    10.112.3.3    gke-bootcamp-default-pool-3433fbf1-gkr6   <none>           <none>
hello-97b76db46-vdzcv       0/1     Running   0          7s      10.112.0.10   gke-bootcamp-default-pool-3433fbf1-7287   <none>           <none>
```

部署更新後，`Kubernetes` 將自動更新關聯的 `ReplicaSet` 並啟動新 `POD`，以使 `POD` 總數等於 5。

驗證數量
```shell
$ kubectl get pods | grep hello- | wc -l
```

縮小該應用程式，也是使用相同方式，只不過更改數量而已。

```shell
$ kubectl scale deployment hello --replicas=3
$ kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE                                      NOMINATED NODE   READINESS GATES
auth-7b97477d9f-7pd2h       1/1     Running   0          3m53s   10.112.1.3   gke-bootcamp-default-pool-3433fbf1-rlqp   <none>           <none>
frontend-7779fdcd57-9lm7m   1/1     Running   0          2m44s   10.112.3.4   gke-bootcamp-default-pool-3433fbf1-gkr6   <none>           <none>
hello-97b76db46-kf9sn       1/1     Running   0          2m53s   10.112.4.3   gke-bootcamp-default-pool-3433fbf1-7vk1   <none>           <none>
hello-97b76db46-ks6jc       1/1     Running   0          2m53s   10.112.1.4   gke-bootcamp-default-pool-3433fbf1-rlqp   <none>           <none>
hello-97b76db46-l998r       1/1     Running   0          2m53s   10.112.3.3   gke-bootcamp-default-pool-3433fbf1-gkr6   <none>           <none>
$ kubectl get pods | grep hello- | wc -l # 驗證
```

此章節了解了 `Kubernetes` 部署以及如何管理和擴展 `POD`。


## Rolling update
部署支持透過滾動更新機制將鏡像更新到新版本。當用新版本更新部署時，它將創建新的 `ReplicaSet`，並隨著減少舊 `ReplicaSet` 中的副本而緩慢增加新`ReplicaSet` 中的副本數。

![](https://cdn.qwiklabs.com/uc6D9jQ5Blkv8wf%2FccEcT35LyfKDHz7kFpsI4oHUmb0%3D)

### Trigger a rolling update

更新部署，運行

```shell
$ kubectl edit deployment hello
...
containers:
- name: hello
  image: kelseyhightower/hello:2.0.0 # 變更鏡像
...
```

一旦退出 `edit` 的模式，更新的 `Deployment` 將保存到集群中，`Kubernetes` 將開始滾動更新。

查看 `Kubernetes` 創建的新 ReplicaSet`
```shell
$ kubectl get replicaset
NAME                  DESIRED   CURRENT   READY   AGE
auth-7b97477d9f       1         1         1       4m37s
frontend-7779fdcd57   1         1         1       3m28s
hello-657bc68c5c      2         2         0       6s # this
hello-97b76db46       2         2         2       3m37s
```

還可以在 `rollout` 歷史記錄中看到一個新列表

```shell
$ kubectl rollout history deployment/hello
deployment.extensions/hello
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### Pause a rolling update
如果發現正在運行的 `rollout` 出現問題，請暫停它以停止更新。

```shell
$ kubectl rollout pause deployment/hello
```

驗證部署的當前狀態，也可以直接在 `POD` 上驗證此內容
```shell
$ kubectl rollout status deployment/hello
$ kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

### Resume a rolling update
`rollout` 已暫停，這表示有些 `POD` 使用的是新版本，有些 `POD` 使用的是舊版本。可以使用 `resume` 繼續進行 `rollout`。

```shell
$ kubectl rollout resume deployment/hello
```

`rollout` 完成後，運行 `status` 觀察，其內容如下
```shell
$ kubectl rollout status deployment/hello
deployment "hello" successfully rolled out
```

## Rollback an update
假設在新的版本中檢測到錯誤。由於假定新版本有問題，因此連接到新 `POD` 的所有用戶都將遇到這些問題。這將需要回滾到以前的版本，以便可以調查然後發布正確修復的版本。

使用 `rollout` 回滾到以前的版本
```shell
$ kubectl rollout undo deployment/hello
```

驗證歷史記錄中的回滾，同樣也可以使用 `POD` 的方式
```shell
$ kubectl rollout history deployment/hello
$ kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
auth-7b97477d9f-7pd2h           kelseyhightower/auth:1.0.0
frontend-7779fdcd57-9lm7m               nginx:1.9.14
hello-657bc68c5c-r64tl          kelseyhightower/hello:2.0.0 # edit 時變更為 2.0.0
hello-657bc68c5c-vtfdw          kelseyhightower/hello:2.0.0
hello-97b76db46-4s9qv           kelseyhightower/hello:1.0.0 # 回滾
hello-97b76db46-gb7lz           kelseyhightower/hello:1.0.0 # 回滾
hello-97b76db46-lc4w6           kelseyhightower/hello:1.0.0 # 回滾
```


這邊了解了 `Kubernetes` 部署的*滾動更新*以及如何*在不停機*的情況下更新應用程序。

## Canary deployments
當要使用*一部分用戶*測試生產中的新部署時，請使用 `Canary` 部署。透過 `Canary` 部署，可以將更改發布給一小部分用戶，以*減輕與新版本相關的風險*。

### Create a canary deployment
`Canary` 部署包括具有新版本的單獨部署和針對正常、穩定部署以及 `Canary` 部署的服務。

![](https://cdn.qwiklabs.com/qSrgIP5FyWKEbwOk3PMPAALJtQoJoEpgJMVwauZaZow%3D)

首先，為新版本創建一個新的 `canary` 部署：

```shell
$ cat deployments/hello-canary.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-canary
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: hello
        track: canary
        # Use ver 1.0.0 so it matches version on service selector
        version: 1.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
...
```

現在創建 `canary` 部署

```shell
$ kubectl create -f deployments/hello-canary.yaml
```
創建 `canary` 部署之後，應該有兩個部署，`hello` 和 `hello-canary`。使用 `kubectl` 進行驗證

```shell
$ kubectl get deployments
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
auth           1/1     1            1           10m
frontend       1/1     1            1           9m23s
hello          3/3     3            3           9m32s
hello-canary   1/1     1            1           36s
```

在 `hello` 服務上，`selector` 使用 `app:hello` 選擇器，該 `selector` 將與 `prod` 部署和 `canary` 部署中的 `POD` 匹配。但是，由於 `Canary` 部署的 `POD` 數量較少，因此對較少的用戶可見。

### Verify the canary deployment
可以驗證請求提供的 `Hello` 版本
```shell
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"2.0.0"}
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
```
運行幾次，應該看到 `hello 1.0.0` 為某些請求提供了服務，而 `2.0.0` 為一小部分約 `1/4` 提供了服務

### Canary deployments in production - session affinity
在本實驗中，發送給 `Nginx` 服務的每個請求都有機會被 `Canary` 部署服務。但是，如果想確保 `Canary` 部署不會為用戶服務，該怎麼辦？一個用例可能是應用程式的 `UI` 發生了更改，並且不想使客戶端感到困惑，在這種情況下，希望用戶持續某一個部署或另一個部署。可以透過創建具有 `session affinity` 的 `Service` 來做到這一點。這樣，同一客戶端將始終從同一版本獲得服務，在下面的範例中，`Service` 與以前相同，但是已添加了新的 `sessionAffinity` 字段，並將其設置為 `ClientIP`，具有相同 `IP` 地址的所有客戶端都將其請求發送到相同版本的 `hello` 應用程序。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  sessionAffinity: ClientIP
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

應用上，可能希望將 `sessionAffinity` 用於生產環境中的 `Canary` 部署。

## Blue-green deployments
滾動更新是理想的選擇，因為它們能夠以最小的開銷、最小的性能影響和最小的停機時間來緩慢的部署應用程序。在某些情況下，只有在完全部署新版本之後，修改負載均衡器以使其指向該新版本是有益的，在這種情況下，必須進行 `Blue-green` 部署。`Kubernetes` 透過創建兩個單獨的部署來實現這一目標。一種用於舊的 `blue` 版本，另一種用於新的 `green` 版本，將現有的 `hello` 部署用於 `blue` 版本。將透過充當路由器的 `Service` 來訪問部署，新的 `green` 版本啟動並運行後，將透過更新 `Service` 切換到使用該版本。

![](https://cdn.qwiklabs.com/POW8Q247ZKNY%2ByHIartCsoEu8MAih7k4u1twusCx6pw%3D)

>`Blue-green` 部署的主要缺點是，需要在群集中擁有至少兩倍於託管應用程序所需的資源。在一次部署兩個版本的應用程式之前，需確保集群中有足夠的資源。

### The service
使用現有的 `hello` 服務，對其進行更新，使其 `selector` 具有 `app:hello` 和 `version:1.0.0`。`selector` 將匹配現有的 `blue` 部署。但是它不匹配 `green` 部署，因為它將使用不同的版本。

```shell
$ cat services/hello-blue.yaml
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  selector:
    app: "hello"
    version: 1.0.0
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
$ kubectl apply -f services/hello-blue.yaml
```

### Updating using Blue-Green Deployment
為了支持 `Blue-green` 部署樣式，我們將為新版本創建一個新的 `green` 部署。`green` 部署會更新版本標籤和鏡像路徑。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: hello
        track: stable
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: 10Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1
```

建立 `green` 部署

```shell
$ kubectl create -f deployments/hello-green.yaml
```

`green` 部署後並正確啟動，驗證是否仍在使用當前版本 `1.0.0`

```shell
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
```
現在，更新 `Service` 以指向新版本

```shell
$ kubectl apply -f services/hello-green.yaml
```

`Service` 更新後，立即使用`green`部署。現在，可以驗證是否一直在使用新版本

```shell
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"2.0.0"}
```

### Blue-Green Rollback
如有必要，可以使用相同的方法回滾到舊版本。在 `blue` 部署仍在運行時，只需將 `Service` 更新回舊版本即可。

```shell
$ kubectl apply -f services/hello-blue.yaml
```

更新 `Service` 後，回滾將成功。再次，驗證是否正在使用正確的版本

```shell
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
{"version":"1.0.0"}
```

這邊了解了 `blue-green` 部署以及如何將更新部署到需要立即切換版本的應用程式。

在 `Kubernetes`中，`Ingress` 允許外部使用者和客戶端應用程式訪問 `HTTP` 服務。`Ingress` 包含兩個組件：`Ingress Resource ` 和 `Ingress Controller`

- Ingress Resource
    - 是 `inbound` 流量到達服務的規則的集合。這些是 L7 規則，允許將主機名和可選的路徑定向到 `Kubernetes` 中的特定服務
- Ingress Controller
    - 通常透過 `HTTP` 或 `L7` 負載均衡器，根據 `Ingress Resource` 設置的規則進行操作。重要的是，正確配置這兩部分，以便可以將流量從外部客戶端路由到 `Kubernetes` 服務。

`NGINX` 是一種高性能的 `Web` 服務器，由於其強大的功能因此成為 `Ingress Controller` 受歡迎選擇。`NGINX` 它支持：
- Websockets, which allows you to load balance Websocket applications.
- SSL Services, which allows you to load balance HTTPS applications.
- Rewrites, which allows you to rewrite the URI of a request before sending it to the application.
- Session Persistence (NGINX Plus only), which guarantees that all the requests from the same client are always passed to the same backend container.
- JWTs (NGINX Plus only), which allows NGINX Plus to authenticate requests by validating JSON Web Tokens (JWTs).

下圖說明了 `Google Cloud` 中 `Ingress Controller` 的基本流程，並大致了解了將要構建的內容：

![](https://cdn.qwiklabs.com/SZramEpFXLuw3ufMN28ZIZcS9iQij80jMKrbxh76ukg%3D)


### Understanding Regions and Zones
某些 `Compute Engine` 資源位於地域或區域中，區域是可以在其中運行資源的特定地理位置。每個區域都有一個或多個地域。例如，`us-central1` 區域表示美國中部的一個地區，其區域有 `us-central1-a`、`us-central1-b`、`us-central1-c` 和 `us-central1-f`。

![](https://cdn.qwiklabs.com/BErmNT8ZIzd5yqxO0lEJj8lAlKT3jKC%2BtI%2Byj3OSKDA%3D)

在區域中的資源稱為區域資源，虛擬機實例和持久性儲存位於區域中。要將持久性儲存連接到虛擬機實例，兩個*資源必須位於同一區域中*；如果要為 `VM` 分配靜態 `IP` 地址，也必須在同一區域。

## Set a zone
```shell=
$ gcloud compute zones list
$ gcloud config set compute/zone us-central1-a #  設置區域
```

## Create a Kubernetes cluster
部署一個 `Kubernetes Engine` 集群。
```shell
$ gcloud container clusters create nginx-tutorial --num-nodes 2 # 建立名為 nginx-tutorial 的集群，由兩個節點組成
NAME            LOCATION       MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
nginx-tutorial  us-central1-a  1.15.12-gke.2   35.222.33.125  n1-standard-1  1.15.12-gke.2  2          RUNNING
```

## Install Helm
`Helm` 是可簡化 `Kubernetes` 應用程式安裝和管理的工具。可將其視為 `Kubernetes` 的 `apt`、`yum` 或 `homebrew`。`Helm` 有兩部分：用戶端指的是 `Helm` 和 服務端指的是 `tiller`
- `Tiller` 在 `Kubernetes` 集群中運行，並管理 `Helm Charts` 的安裝
- `Helm` 可以在筆電、`CI/CD` 或 `Cloud Shell` 上運行

`Helm` 預先配置了一個安裝腳本，該腳本會自動獲取最新版本的 `Helm` 客戶端，並安裝在本地。

```shell
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > install-helm.sh
$ chmod u+x install-helm.sh
$ ./install-helm.sh --version v2.16.3
$ helm init # 初始
```

## Installing Tiller
從 `Kubernetes` v1.8+ 開始，默認情況下啟用 `RBAC`。在安裝 `tiller` 之前，需確認已為 `tiller` 服務配置了正確的 `ServiceAccount` `和ClusterRoleBinding`。這會讓 `tiller` 能夠在默認 `namespace` 中安裝服務。

將 tiller 設定到啟用了 `RBAC` 的 `Kubernetes` 集群中
```shell
$ kubectl create serviceaccount --namespace kube-system tiller 
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'  
```

使用新創建的 `ServiceAccount` 初始化 `Helm`
```shell
$ helm init --service-account tiller --upgrade
```

還可以檢查 `kube-system` 的 `namespace` 中 `tiller-deploy` 部署來確認 `tiller` 正在運行。

```shell
$ kubectl get deployments -n kube-system
NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
event-exporter-v0.3.0                      1/1     1            1           2m47s
fluentd-gcp-scaler                         1/1     1            1           2m42s
heapster-gke                               1/1     1            1           2m48s
kube-dns                                   2/2     2            2           2m48s
kube-dns-autoscaler                        1/1     1            1           2m48s
l7-default-backend                         1/1     1            1           2m48s
metrics-server-v0.3.3                      1/1     1            1           2m46s
stackdriver-metadata-agent-cluster-level   1/1     1            1           2m47s
tiller-deploy                              1/1     1            1           51s
```

## Deploy an application in Kubernetes Engine

前面配置了 `Helm`，我們從 `Google Cloud Repository` 部署一個基於 `Web` 的簡單應用程式，該應用程序將用作 `Ingress` 的後端。

```shell
$ kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:1.0
```

將 `hello-app` 部署作為服務暴露
```shell
$ kubectl expose deployment hello-app  --port=8080
```

## Deploying the NGINX Ingress Controller via Helm
`Kubernetes` 平台在結合 `Ingress Controller` 時為管理員提供了靈活性，可以整合自己的資料庫，而不必與提供商的內置產品一起使用。然而，必須暴露`NGINX` 控制器以供外部訪問，這由 `NGINX` 控制器服務上的 `Service` 資源類型為 `LoadBalancer` 完成。在 `Kubernetes Engine` 上，這將使用`NGINX` 控制器服務作為後端創建一個 `Google Cloud Network (TCP/IP) Load Balancer`。`Google Cloud` 還在 `Service` 的 `VPC` 創建適當的防火牆規則，以允許 Web HTTP(s) 流量到達負載均衡器前端 IP 地址。

### NGINX Ingress Controller on Kubernetes Engine
以下流程圖值覺得表示了 `NGINX` 控制器如何在 `Kubernetes Engine` 集群上運行

![](https://cdn.qwiklabs.com/E1bWzR59KZ8Yjlwzj9Jm5M138L1Jwn%2FydY9cZdD2A4U%3D)

### Deploy NGINX Ingress Controller


```shell
$ helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true # 部署 `NGINX Ingress Controller`       
```

注意 `nginx-ingress-default-backend`，預設後端是一個服務，它處理所有 `URL` 路徑並託管 `NGINX` 控制器。預設後端公開兩個 `URL`

- `/healthz` that returns 200
- `/` that returns 404

確認已部署 `nginx-ingress-controller` 服務，並且具有與該服的對外 `IP` 地址
```shell
$ kubectl get service nginx-ingress-controller
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.3.250.165   35.225.56.203   80:31673/TCP,443:31899/TCP   76s
```

## Configure Ingress Resource to use NGINX Ingress Controller
`Ingress Resource` 對像是 `L7` 規則的集合，用於將 `inbound` 流量路由到 `Kubernetes` 的 `Services` 資源。可以在一個 `Ingress Resource` 中定義多個規則，也可以將它們拆分為多個 `Ingress Resource` 清單，`Ingress Resource` 還確定使用哪個控制器來服務流量。我們在 `Ingress Resource` 的元數據部分中使用 `kubernetes.io/ingress.class` 的 `annotation` 進行設置，對於 `NGINX` 控制器，我們使用 `nginx` 值

```shell
$ annotations: kubernetes.io/ingress.class: nginx
```

在 `Kubernetes Engine` 上，如果未在元數據部分下定義 `annotation`，則 `Ingress resource` 將使用 `Google Cloud GCLB L7` 負載均衡器來提供流量，也可透過將註釋的值設置為 `gce` 來強制執行此方法。
```shell
$ annotations: kubernetes.io/ingress.class: gce
```

我們創建一個簡單的 `Ingress Resource` 的 `YAML`，該 `YAML` 使用 `NGINX Ingress Controller` 並輸入以下命令定義的路徑規則

```shell
$ touch ingress-resource.yaml
$ vi ingress-resource.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /hello
        backend:
          serviceName: hello-app
          servicePort: 8080
```

`kind:Ingress` 表示它是一個 `Ingress` 資源對象。此 `inbound` 資源將路徑 `/hello` 定義了入站 `L7` 規則，為端口 `8080` 上的 `hello-app` 服務。


```shell
$ kubectl apply -f ingress-resource.yaml
$ kubectl get ingress ingress-resource # 驗證
NAME               HOSTS   ADDRESS       PORTS   AGE
ingress-resource   *       34.71.15.66   80      37s
```

### Test Ingress and default backend
運行 `kubectl get service nginx-ingress-controller` 找到對外 IP 並訪問 `Web` 應用程式，打開新頁面輸入 `http://external-ip-of-ingress-controller/hello` 和 `http://external-ip-of-ingress-controller/test` 並觀察是否是要的結果

```shell
$ kubectl get svc nginx-ingress-controller
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.3.250.165   35.225.56.203   80:31673/TCP,443:31899/TCP   4m20s
```

![](https://i.imgur.com/AsENHAN.png)

![](https://i.imgur.com/W6yAyGa.png)



## Other
```shell
$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
hello-app-6cc8c46757-f4zp9                       1/1     Running   0          8m22s
nginx-ingress-controller-6b5498d8dc-h6b6r        1/1     Running   0          7m33s
nginx-ingress-default-backend-674d599c48-9274q   1/1     Running   0          7m33s
$ kubectl exec -it nginx-ingress-controller-6b5498d8dc-h6b6r -- /bin/bash
bash-5.0$ vi nginx.conf # 發現會自動配在 `yaml` 的需求
```

![](https://i.imgur.com/iCfk7lo.png)
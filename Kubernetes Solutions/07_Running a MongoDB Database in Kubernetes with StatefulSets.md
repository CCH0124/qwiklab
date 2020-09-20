Kubernetes 是一個開源代碼的容器編排工具，用於處理運行中的容器化應用程式的複雜性。本實驗中，將學習如何使用 `StatefulSet` 設置 `MongoDB` ，從而獲得一些 Kubernetes 實踐經驗。

## Set a Compute Zone

```shell
gcloud config set compute/zone us-central1-f
```

## Create a new Cluster
```shell
gcloud container clusters create hello-world
NAME         LOCATION       MASTER_VERSION  MASTER_IP   MACHINE_TYPE   NODE_VERSION    NUM_NODES  STATUS
hello-world  us-central1-f  1.15.12-gke.20  34.66.17.9  n1-standard-1  1.15.12-gke.20  3          RUNNING
```

## Setting up
叢集建立好後，接著是整合 MongoDB。我們使用一個副本，使我們的數據高度可用和冗餘，這是運行生產應用程式必需的。我們需要做一下的事情

- Download the MongoDB `replica` set/sidecar (or utility container in our cluster).
- Instantiate a `StorageClass`.
- Instantiate a `headless` service.
- Instantiate a `StatefulSet`.

下載 google 提供的專案，並切換至本實驗目錄
```shell
git clone https://github.com/thesandlord/mongo-k8s-sidecar.git
cd ./mongo-k8s-sidecar/example/StatefulSet/
```

## Create the StorageClass
`StorageClass` 告訴 Kubernetes 要用於數據庫節點的存儲類型。在 Google Cloud 上，有兩種選擇：SSD 和 HDD。如果查看 `StatefulSet` 目錄，將看到 Azure 和 Google Cloud 的 SSD 和 HDD 配置文件，內容如下。

```shell
cat googlecloud_ssd.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd # ssd 類型
```

布署 `StorageClass`
```shell
kubectl apply -f googlecloud_ssd.yaml
```

## Deploying the Headless Service and StatefulSet
### Find and inspect the files
```shell
vi mongo-statefulset.yaml
apiVersion: v1   <-----------   Headless Service configuration
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1beta1    <------- StatefulSet configuration
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo" # 對應定義的 Service 資源名稱
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: test
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--smallfiles" # 移除
            - "--noprealloc" # 移除
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage # 對應儲存的定義資源
              mountPath: /data/db
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=test"
  volumeClaimTemplates: # 定義儲存相關的資源
  - metadata:
      name: mongo-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast" # 對應定義的 StorageClass
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi
```


將以下兩行移除

```shell
            - "--smallfiles" # 移除
            - "--noprealloc" # 移除
# 移除後內容
...
containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--replSet"
            - rs0
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
...
```

##### Headless service: overview
上面 `mongo-statefulset.yaml` 的第一部分是無頭服務(headless service)。以 Kubernetes 來說，該 `Service` 描述了訪問特定 `POD` 的策略或規則。簡而言之，無頭服務是沒有負載均衡的一個服務。當與 `StatefulSets` 結合使用時，使我們可以使用 `DNS` 來訪問我們的 `POD`，這可以單獨連接到我們的所有 `MongoDB` 節點。在 `yaml` 中，可以透果設置 `clusterIP` 為 `None` 來確保該服務是無頭的。

##### StatefulSet: overview
`StatefulSet` 配置是 `mongo-statefulset.yaml` 的第二部分。這是運行 MongoDB 的工作負載，並且是協調 Kubernetes 資源的原因。參照 `yaml`，我們看到第一部分描述了 `StatefulSet` 對象，接著到 `Metadata` 部分，在其中指定 `labels` 和 `replicas` 數量。接下來是 `POD` 規格，當縮小副本數時，可以使用 `TerminationGracePeriodSeconds` 正常關閉 `POD`。然後顯示了兩個容器的配置，第一個運行帶有配置複製集名稱的命令選項的 MongoDB，它還會將持久性儲存卷關聯到 `/data/db`，也就是 MongoDB 保存數據的位置。第二個容器運行邊車(sidecar)，該 `sidecar` 容器將自動配置 MongoDB 副本，如前所述，`sidecar` 是一個輔助容器，可以幫助主容器運行其作業和任務。最後，還有 `volumeClaimTemplates`，是我們在配置 `volume` 之前創建的`StorageClass` 的內容，它為每個 MongoDB 副本提供 100GB 硬碟。

##### Deploy Headless Service and the StatefulSet
```shell
kubectl apply -f mongo-statefulset.yaml
```


## Connect to the MongoDB Replica Set
### Wait for the MongoDB replica set to be fully deployed
Kubernetes 的 `StatefulSets` 資源按順序部署每個 `POD`。它等待 MongoDB 副本集成員完全啟動並創建儲存，然後再啟動下一個成員。

```shell
kubectl get statefulset # 用來確認當前的布署
```

### Initiating and Viewing the MongoDB replica set
透過以下命令，應該可以看到三個運行的 `POD`
```shell
kubectl get pods
```

需要布署完成才能執行以下，交互第一個 POD
```shell
kubectl exec -ti mongo-0 mongo
rs.initiate() # 以預設配置實例化副本集
rs.conf() # 查看配置的配置檔
```
這邊只會看到一個，除非使用 nodePort 或 load balancer 方式才會取得所有副本資訊。
```shell
rs0:OTHER> rs.conf()
{
        "_id" : "rs0",
        "version" : 1,
        "protocolVersion" : NumberLong(1),
        "writeConcernMajorityJournalDefault" : true,
        "members" : [
                {
                        "_id" : 0,
                        "host" : "localhost:27017",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {
                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                }
        ],
        "settings" : {
                "chainingAllowed" : true,
                "heartbeatIntervalMillis" : 2000,
                "heartbeatTimeoutSecs" : 10,
                "electionTimeoutMillis" : 10000,
                "catchUpTimeoutMillis" : -1,
                "catchUpTakeoverDelayMillis" : 30000,
                "getLastErrorModes" : {
                },
                "getLastErrorDefaults" : {
                        "w" : 1,
                        "wtimeout" : 0
                },
                "replicaSetId" : ObjectId("5c526b6501fa2d29fc65c48c")
        }
}
```

## Scaling the MongoDB replica set
使用以下方式
```shell
kubectl scale --replicas=5 statefulset mongo
```

使用以下確認有無 5 個 `POD`
```shell
kubectl get pods
```

將其縮回 3 個
```shell
kubectl scale --replicas=3 statefulset mongo
kubectl get pods
```

### Using the MongoDB replica set
由無頭服務(headless service)支持 `StatefulSet` 中每個 `POD` 有一個穩定的 `DNS` 名稱，遵循以下格式：`<pod-name>.<service-name>`，表示 `DNS` 名稱為以下

```shell
mongo-0.mongo
mongo-1.mongo
mongo-2.mongo
```

連接方式可以如下
```shell
"mongodb://mongo-0.mongo,mongo-1.mongo,mongo-2.mongo:27017/dbname_?"
```

## Clean up
```shell
kubectl delete statefulset mongo
kubectl delete svc mongo
kubectl delete pvc -l role=mongo
gcloud container clusters delete "hello-world"
```
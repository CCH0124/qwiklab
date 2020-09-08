---
layout: post
title: Kubernetes Solutions - 02
date: 2020-09-08
excerpt: "Using Kubernetes Engine to Deploy Apps with Regional Persistent Disks"
tags: [qwiklab, ML, Kubernetes]
comments: true
---

在本實驗中將學習如何透過使用 `Kubernetes Engine` 上的區域持久性儲存部署 `WordPress` 來配置高可用性應用程序。區域持久性儲存在兩個區域之間提供同步複製，以防單個區域發生故障或故障時保持應用程序正常運行。透過區域持久性儲存部署 `Kubernetes Engine` 集群將使應用程序更加穩定、安全和可靠。

![](https://cdn.qwiklabs.com/I6ZSnXZbwDN4OsOo1hoVEkhu4DMvwxygEuEPamH9y8c%3D)

## Creating the Regional Kubernetes Engine Cluster

打開 `Cloud Shell` 終端機。首先，將創建一個區域性 `Kubernetes Engine` 集群，該集群跨越 `us-west1` 區域中的三個區域。透過運行以下獲取`us-west1` 區域的服務器配置並導出環境變量
```shell=
$ CLUSTER_VERSION=$(gcloud container get-server-config --region us-west1 --format='value(validMasterVersions[0])')
$ export CLOUDSDK_CONTAINER_USE_V1_API_CLIENT=false
```

創建一個標準的 `Kubernetes Engine` 集群
```shell=
$ gcloud container clusters create repd \
  --cluster-version=${CLUSTER_VERSION} \
  --machine-type=n1-standard-4 \
  --region=us-west1 \
  --num-nodes=1 \
  --node-locations=us-west1-a,us-west1-b,us-west1-c
...
NAME  LOCATION  MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
repd  us-west1  1.16.13-gke.1   35.185.246.18  n1-standard-4  1.16.13-gke.1  3          RUNNING
```


剛剛創建一個區域集群，每個區域中都有一個節點，實驗分別在 `us-west1-a`、`us-west1-`、`us-west1-c`。從左側選單到 `Compute Engine` 查看 `VM`。`gcloud` 指令還自動將 `kubectl` 命令配置，使得連接到集群。

![](https://i.imgur.com/BvjtSxG.png)

## Deploying the App with a Regional Disk

這邊已經運行了 `Kubernetes` 集群，將執行以下三件事：
- 安裝 `Helm`
- 創建區域永久性儲存使用 `Kubernetes StorageClass`
- 部署 `wordpress`


### Install and initialize Helm to install the chart package
隨 `Helm` 安裝的 `chart` 包含運行 `WordPress` 所需的資源。

1. `Cloud Shell` 中安裝 `Helm`

這一步似乎無法執行，我手動安裝 3.0 版本，或者使用 `wget` 代替 `curl`
```shell
curl https://raw.GitHubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```
2. 初始 `Helm`
```shell
$ kubectl create serviceaccount tiller --namespace kube-system
$ kubectl create clusterrolebinding tiller-cluster-rule \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:tiller
$ helm init --service-account=tiller
$ until (helm version --tiller-connection-timeout=1 >/dev/null 2>&1); do echo "Waiting for tiller install..."; sleep 2; done && echo "Helm install complete"
```

### Create the StorageClass
接下來，創建由 `chart` 使用的 `StorageClass` 來定義區域儲存區域。`StorageClass` 中列出的區域將與 `Kubernetes Engine` 集群的區域匹配。

運行以下為區域儲存創建 `StorageClass`
```shell
$ kubectl apply -f - <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: repd-west1-a-b-c
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
  zones: us-west1-a, us-west1-b, us-west1-c
EOF
storageclass.storage.k8s.io/repd-west1-a-b-c created
```

現在，有了一個能夠配置 `PersistentVolumes` 的 `StorageClass`，並在 `us-west1-a`、`us-west1-b` 和 `us-west1-c` 區域中。

查看可用的 `storageclass`
```shell
$ kubectl get storageclass
NAME                 PROVISIONER            AGE
repd-west1-a-b-c     kubernetes.io/gce-pd   31s
standard (default)   kubernetes.io/gce-pd   16m
```

## Create Persistent Volume Claims
為應用程式創建 `persistentvolumeclaims`。

使用標準 `StorageClass` 創建 `data-wp-repd-mariadb-0` 的 `PVC`。

```shell
kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: data-wp-repd-mariadb-0
  namespace: default
  labels:
    app: mariadb
    component: master
    release: wp-repd
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
EOF
persistentvolumeclaim/data-wp-repd-mariadb-0 created
```

使用 `repd-west1-a-b-c` `StorageClass` 創建 `wp-repd-wordpress` `PVC`。

```shell
kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wp-repd-wordpress
  namespace: default
  labels:
    app: wp-repd-wordpress
    chart: wordpress-5.7.1
    heritage: Tiller
    release: wp-repd
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: repd-west1-a-b-c
EOF
persistentvolumeclaim/wp-repd-wordpress created
```

列出可用 `persistentvolumeclaims`

```shell
$ kubectl get persistentvolumeclaims
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
data-wp-repd-mariadb-0   Bound    pvc-b3a160ba-7e80-4602-b214-78ecafae539c   8Gi        ROX            standard           31s
wp-repd-wordpress        Bound    pvc-f1b7c7bf-236c-4ee0-851f-7d668442af33   200Gi      ROX            repd-west1-a-b-c   14s
```

## Deploy WordPress
我們已經配置了 `StorageClass`，`Kubernetes` 會自動將持久儲存附加到可用區域之一中的適當節點上。

1. 部署配置為使用先前創建的 `StorageClass` 的 `WordPress` chart
```shell
helm install --name wp-repd \
  --set smtpHost= --set smtpPort= --set smtpUser= \
  --set smtpPassword= --set smtpUsername= --set smtpProtocol= \
  --set persistence.storageClass=repd-west1-a-b-c \
  --set persistence.existingClaim=wp-repd-wordpress \
  --set persistence.accessMode=ReadOnlyMany \
  stable/wordpress
```

```shell
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update
$ helm install wp-repd bitnami/wordpress --set smtpHost= --set smtpPort= --set smtpUser=   --set smtpPassword= --set smtpUsername= --set smtpProtocol=   --set persistence.storageClass=repd-west1-a-b-c   --set persistence.existingClaim=wp-repd-wordpress   --set persistence.accessMode=ReadOnlyMany
```
2. 列出 `POD`
```shell
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
wp-repd-mariadb-0                  1/1     Running   0          103s
wp-repd-wordpress-7498f958-vd4ld   1/1     Running   0          103s
```

3. 等待創建 `Service` 負載均衡器的外部 `IP` 地址
```shell
$ while [[ -z $SERVICE_IP ]]; do SERVICE_IP=$(kubectl get svc wp-repd-wordpress -o jsonpath='{.status.loadBalancer.ingress[].ip}'); echo "Waiting for service external IP..."; sleep 2; done; echo http://$SERVICE_IP/admin
Waiting for service external IP...
http://34.105.41.165/admin
```
4. 驗證是否已創建永久儲存
```shell
$ while [[ -z $PV ]]; do PV=$(kubectl get pvc wp-repd-wordpress -o jsonpath='{.spec.volumeName}'); echo "Waiting for PV..."; sleep 2; done

$ kubectl describe pv $PV
Name:              pvc-f1b7c7bf-236c-4ee0-851f-7d668442af33
Labels:            failure-domain.beta.kubernetes.io/region=us-west1
                   failure-domain.beta.kubernetes.io/zone=us-west1-a__us-west1-c
Annotations:       kubernetes.io/createdby: gce-pd-dynamic-provisioner
                   pv.kubernetes.io/bound-by-controller: yes
                   pv.kubernetes.io/provisioned-by: kubernetes.io/gce-pd
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      repd-west1-a-b-c
Status:            Bound
Claim:             default/wp-repd-wordpress
Reclaim Policy:    Delete
Access Modes:      ROX
VolumeMode:        Filesystem
Capacity:          200Gi
Node Affinity:
  Required Terms:
    Term 0:        failure-domain.beta.kubernetes.io/zone in [us-west1-c, us-west1-a]
                   failure-domain.beta.kubernetes.io/region in [us-west1]
Message:
Source:
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     gke-repd-fa61927f-dyna-pvc-f1b7c7bf-236c-4ee0-851f-7d668442af33
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
Events:         <none>
```

5. 取得 `WordPress` 管理頁面的 `URL`
```shell
$ echo http://$SERVICE_IP/admin
```

6. 點擊鏈接以在瀏覽器的新分頁中打開 `WordPress`
7. 至 `Cloud Shell` 取得用戶名和密碼，以便登錄到應用程序
```shell
cat - <<EOF
Username: user
Password: $(kubectl get secret --namespace default wp-repd-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
EOF
```
8. 至 `WordPress` 頁面，使用弟 7 步的用戶名和密碼登錄

到這邊將有一個工作正常的 `WordPress` 部署，該部署由三個區域中的區域持久性儲存所支援。

## Simulating a zone failure
接下來，模擬一個區域故障，並觀察 `Kubernetes` 將工作負載移至另一個區域並將區域儲存連接到新節點。

1. 獲取 `WordPress` `POD` 的當前節點
```shell
$ NODE=$(kubectl get pods -l app.kubernetes.io/instance=wp-repd  -o jsonpath='{.items..spec.nodeName}')

$ ZONE=$(kubectl get node $NODE -o jsonpath="{.metadata.labels['failure-domain\.beta\.kubernetes\.io/zone']}")
$ IG=$(gcloud compute instance-groups list --filter="name~gke-repd-default-pool zone:(${ZONE})" --format='value(name)')
$ echo "Pod is currently on node ${NODE}"
Pod is currently on node gke-repd-default-pool-1aa06166-x1sl
$ echo "Instance group to delete: ${IG} for zone: ${ZONE}"
Instance group to delete: gke-repd-default-pool-1aa06166-grp for zone: us-west1-a
```
```
$ kubectl get pods -l app.kubernetes.io/instance=wp-repd -o wide # 驗證
NAME                               READY   STATUS    RESTARTS   AGE    IP         NODE                                  NOMINATED NODE   READINESS GATES
wp-repd-wordpress-7498f958-vd4ld   1/1     Running   0          6m9s   10.0.1.7   gke-repd-default-pool-1aa06166-x1sl   <none>           <none>
```
記下節點將刪除此節點以模擬區域故障。

2. 現在運行以下以刪除運行 `WordPress` `POD` 的節點 `VM`
```shell
$ gcloud compute instance-groups managed delete ${IG} --zone ${ZONE} # 點擊 Y
```

![](https://i.imgur.com/Bo0jizR.png)

`Kubernetes` 現在正在檢測故障並將 `POD` 遷移到另一個區域中的節點。

3. 驗證 `WordPress` `POD` 和永久卷都(persistent volume)已遷移到另一個區域中的節點
```shell
$ kubectl get pods -l app.kubernetes.io/instance=wp-repd -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP         NODE                                  NOMINATED NODE   READINESS GATES
wp-repd-wordpress-7498f958-g5976   0/1     Running   0          60s   10.0.2.4   gke-repd-default-pool-931308c7-n263   <none>           <none>
```

4. 一旦新服務處於運行狀態在瀏覽器中打開 `WordPress` 管理頁面
```shell
$ echo http://$SERVICE_IP/admin
```

已將區域持久性儲存附加到其他區域中的節點。
在 Kubernetes Engine 中，私有集群使您的主服務器無法透過公有 Internet 存取集群。在私有群集中，節點沒有公有 IP 地址，只有私有地址，因此工作負載在隔離的環境中運行，節點和主節點使用 VPC 對等關係相互通訊。在 Kubernetes Engine API 中，地址範圍表示為 Classless Inter-Domain Routing(CIDR)。

## Set a zone

```shell
gcloud config set compute/zone us-central1-a
```

>顯示可用的區域 gcloud compute zones list

## Creating a private cluster
創建私有集群時，須為運行 Kubernetes 主組件的 VM 指定 `/28` CIDR 範圍，並且需要啟用 IP 別名。接下來，創建一個名為 private-cluster 的集群，並為主服務器指定 CIDR 範圍為 `172.16.0.16/28`。啟用 IP 別名後，可讓 Kubernetes Engine 自動創建一個子網。下面將使用`--private-cluster`、`--master-ipv4-cidr` 和 `--enable-ip-alias` 標誌創建私有集群。

```shell
gcloud beta container clusters create private-cluster \
    --enable-private-nodes \
    --master-ipv4-cidr 172.16.0.16/28 \
    --enable-ip-alias \
    --create-subnetwork ""
```

## Viewing your subnet and secondary address ranges

列出預設網路中的子網

```shell
gcloud compute networks subnets list --network default
```

透過以下，獲取有關自動創建的子網的信息，將 `[SUBNET_NAME]` 替換為創建的子網，輸出顯示主要地址範圍以及 GKE 私有集群的名稱和 CIDR，在輸出中，還可以看到一個次要範圍用於 `POD`，另一個範圍用於 `services`。

```shell
gcloud compute networks subnets describe [SUBNET_NAME] --region us-central1
```
輸出
```shell
creationTimestamp: '2020-10-08T20:52:18.094-07:00'
description: auto-created subnetwork for cluster "private-cluster"
fingerprint: roMIkPZh35c=
gatewayAddress: 10.33.168.1
id: '4387689689274015901'
ipCidrRange: 10.33.168.0/22
kind: compute#subnetwork
name: gke-private-cluster-subnet-bb96378c
network: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-f0e4939bcdb1/global/networks/default
privateIpGoogleAccess: true
privateIpv6GoogleAccess: DISABLE_GOOGLE_ACCESS
purpose: PRIVATE
region: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-f0e4939bcdb1/regions/us-central1
secondaryIpRanges:
- ipCidrRange: 10.36.0.0/14
  rangeName: gke-private-cluster-pods-bb96378c
- ipCidrRange: 10.33.176.0/20
  rangeName: gke-private-cluster-services-bb96378c
selfLink: https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-f0e4939bcdb1/regions/us-central1/subnetworks/gke-private-cluster-subnet-bb96378c
```

>privateIPGoogleAccess 設置為 true，這讓只有私有 IP 地址的群集主機可以與 Google API 和服務進行通訊


## Enabling master authorized networks
此時，唯一可以存取主服務器的 IP 地址是以下範圍內的地址
- 子網的主要範圍，這是用於節點的範圍
- 子網用於 `POD` 的次要範圍

要提供對主服務器的其他訪問，必須授權選定的地址範圍。

##### Create a VM instance
創建一個用於檢查與 Kubernetes 集群的連接性的 VM 實例

```shell
gcloud compute instances create source-instance --zone us-central1-a --scopes 'https://www.googleapis.com/auth/cloud-platform'
```

使用以下獲取 VM 實例的 `<External_IP>`
```shell
gcloud compute instances describe source-instance --zone us-central1-a | grep natIP
    natIP: 34.72.102.240
```

`[MY_EXTERNAL_RANGE]` 替換為上一步驟的輸出 `natIP`。
```shell
gcloud container clusters update private-cluster \
    --enable-master-authorized-networks \
    --master-authorized-networks [MY_EXTERNAL_RANGE]
```

接下來將安裝 `kubectl`，以便可以使用它來獲取有關集群的信息，例如可以使用 `kubectl` 來驗證節點沒有外部 IP 地址。

`SSH` 到 source-instance 
```shell
gcloud compute ssh source-instance --zone us-central1-a
```
安裝 `kubectl`
```shell
gcloud components install kubectl
```

使用以下命令從 SSH shell 配置對 Kubernetes 集群的訪問
```shell
gcloud container clusters get-credentials private-cluster --zone us-central1-a
```
驗證是否有公有 IP
```shell
kubectl get nodes --output yaml | grep -A4 addresses
    addresses:
    - address: 10.33.168.3
      type: InternalIP
    - address: ""
      type: ExternalIP
--
    addresses:
    - address: 10.33.168.2
      type: InternalIP
    - address: ""
      type: ExternalIP
--
    addresses:
    - address: 10.33.168.4
      type: InternalIP
    - address: ""
      type: ExternalIP
```
另一種驗證方式
```shell
kubectl get nodes --output wide
```

## Clean Up
```shell
gcloud container clusters delete private-cluster --zone us-central1-a
```

## Creating a private cluster that uses a custom subnetwork
建立自己的子網
```shell
gcloud compute networks subnets create my-subnet \
    --network default \
    --range 10.0.4.0/22 \
    --enable-private-ip-google-access \
    --region us-central1 \
    --secondary-range my-svc-range=10.0.32.0/20,my-pod-range=10.4.0.0/14
```
建立使用自己子網的集群

```shell
gcloud beta container clusters create private-cluster2 \
    --enable-private-nodes \
    --enable-ip-alias \
    --master-ipv4-cidr 172.16.0.32/28 \
    --subnetwork my-subnet \
    --services-secondary-range-name my-svc-range \
    --cluster-secondary-range-name my-pod-range
```

授權外部地址範圍，將 `[MY_EXTERNAL_RANGE]` 替換為來自先前輸出的外部地址的 CIDR 範圍
```shell
gcloud container clusters update private-cluster2 \
    --enable-master-authorized-networks \
    --master-authorized-networks [MY_EXTERNAL_RANGE]
```


```shell
gcloud compute ssh source-instance --zone us-central1-a
gcloud container clusters get-credentials private-cluster2 --zone us-central1-a
kubectl get nodes --output yaml | grep -A4 addresses
    addresses:
    - address: 10.0.4.4
      type: InternalIP
    - address: ""
      type: ExternalIP
--
    addresses:
    - address: 10.0.4.2
      type: InternalIP
    - address: ""
      type: ExternalIP
--
    addresses:
    - address: 10.0.4.3
      type: InternalIP
    - address: ""
      type: ExternalIP
```
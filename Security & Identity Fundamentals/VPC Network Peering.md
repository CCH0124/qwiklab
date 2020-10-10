Google Cloud Virtual Private Cloud(VPC)網路對等(Network Peering)允許跨兩個 VPC 網路的私有連接，無論它們是屬於同一項目還是屬於同一組織。`VPC` 網路對等可以在 Google Cloud 中構建 SaaS 生態系統，從而使服務可以在組織內部和組織之間的不同 VPC 網路之間私有使用，從而允許工作負載在私有空間中進行通訊。

VPC 網路對等可用於：
- 具有多個網路管理域的組織
- 想要與其他組織建立對等關係的組織

與使用外部 IP 或 VPN 連接網路相比，VPC 網路對等提供了多個優勢，包括：
- Network Latency
    - 專用網路比公有 IP 網路具有更低的延遲
- Network Security
    - 服務擁有者不需要將其服務公開到公共 Internet 並應對其相關風險
- Network Cost
    - 對等網路可以使用內部 IP 進行通訊並節省 Google Cloud 出口頻寬成本

## VPC Network Peering setup
在同一組織節點內，網路可能託管著需要從相同或不同項目中的其他 VPC 網路存取服務。或者，一個組織可能希望存取第三方服務產品。專案名稱在所有Google Cloud 中都是唯一的，因此在建立對等關係時無需指定組織，Google Cloud 根據專案名稱了解組織。

### Create a custom network in projects
實驗給兩個專案，用 Project-A 和 Project-B 作為替代名稱

切換專案
```shell
gcloud config set project <PROJECT_ID2>
```

##### Project-A
```shell
gcloud compute networks create network-a --subnet-mode custom
```

在此 VPC 內創建一個子網，並用以下命令指定區域和 IP 範圍
```shell
gcloud compute networks subnets create network-a-central --network network-a \
    --range 10.0.0.0/16 --region us-central1
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-6ad8b4aeff00/regions/us-central1/subnetworks/network-a-central].
NAME               REGION       NETWORK    RANGE
network-a-central  us-central1  network-a  10.0.0.0/16
```
建立 VM
```shell
gcloud compute instances create vm-a --zone us-central1-a --network network-a --subnet network-a-central
```

運行以下啟用 SSH 和 icmp 協定，因為在連接測試期間需要安全的 Shell 才能與 VM 通訊

```shell
gcloud compute firewall-rules create network-a-fw --network network-a --allow tcp:22,icmp
```

##### Project-B:
```shell
gcloud compute networks create network-b --subnet-mode custom
```

```shell
gcloud compute networks subnets create network-b-central --network network-b \
    --range 10.8.0.0/16 --region us-central1
```

```shell
gcloud compute instances create vm-b --zone us-central1-a --network network-b --subnet network-b-central
```

```shell
gcloud compute firewall-rules create network-b-fw --network network-b --allow tcp:22,icmp
```

## Setting up a VPC Network Peering session
考慮一個需要在項目 A 的網路 A 和項目 B 的網路 B 之間建立 VPC 網路對等的組織。為了成功建立 VPC 網路對等，網路 A 和網路 B 的管理員必須分別配置對等關聯。

### Peer network-a with network-b:

![](https://cdn.qwiklabs.com/i6fwOoVXTt4J7oToas%2BFV61tUgLuDiaw5y7zGEnr6lU%3D)

##### Project-A
藉由 navigating 點擊 `VPC Network > VPC network peering`

1. 點擊 Create connection.
2. 點擊 Continue.
3. 輸入 "peer-ab 作為連線名稱
4. 在"Your VPC network"下，選擇要對等的網路(網路-a)
5. 將"Peered VPC network" 單選按鈕設置為 "In another project"
6. 貼上第二個專案的 "Project ID"
7. 輸入另一個網路的 VPC 名稱(network-b)
8. 點擊 Create.

因為另一方上未設置因此狀態為未激活是正常。

![](https://i.imgur.com/y8FUu31.png)

### Peer network-b with network-a
![](https://cdn.qwiklabs.com/u1lfqiOZTxehsnL%2FYaYF7Xg0XA%2FFXsq8YubRiAYwx4w%3D)

##### Project-B
和 Project-A 作法類似。

![](https://i.imgur.com/bx8e7aM.png)



VPC 網路對等變為激活狀態並交換路由，一旦對等變成到激活狀態，就會建立流量：

- 在對等網路中的 VM 實例之間是全網狀連接(Full mesh connectivity)
- 從一個網路中的 VM 實例到對等網路中的 Internal Load Balancing 端點

![](https://cdn.qwiklabs.com/a3mCwnSLHBiYoGJjFqeP4kgj7HgaY87W9CevTJR0eK4%3D)


列出路由方式
```shell
gcloud compute routes list --project <FIRST_PROJECT_ID>
```

## Connectivity Test
### Project-A
點擊置 ` Navigation Menu > Compute Engine > VM instances`，複製 `vm_a` 的 `INTERNAL_IP`

### Project-B
點擊 `Product & services > Compute > Compute Engine > VM instances`，SSH 置 vm-b，執行以下 ping
```shell
ping -c 5 <INTERNAL_IP_OF_VM_A>
```

```shell
student_00_de8613958f36@vm-b:~$ ping -c 5 34.123.44.60
PING 34.123.44.60 (34.123.44.60) 56(84) bytes of data.
64 bytes from 34.123.44.60: icmp_seq=1 ttl=64 time=2.11 ms
64 bytes from 34.123.44.60: icmp_seq=2 ttl=64 time=2.01 ms
^C
--- 34.123.44.60 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 2.012/2.058/2.105/0.064 ms
```
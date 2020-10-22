在 Terraform 中，提供程序是上游 API 的邏輯抽象。本實驗將演示如何設置 Kubernetes 集群並在其上部署 Load Balancer 類型的NGINX 服務。

## Kubernetes Services
`Service` 是在集群上運行的一群 POD。`Service` 是廉價的，可以在集群中擁有許多 `Service`。Kubernetes `Service` 可以有效支持微服務架構。

`Service` 提供了跨集群標準化的重要功能：負載均衡，應用程序之間的服務發現以及支持零停機應用程式部署的功能。

每個 `Service` 都有一個 POD 標籤查詢，該查詢定義了將處理該服務數據的 POD。此標籤查詢經常匹配由一個或多個 `RS` 創建的 POD。透過使用 `Deployment` 經由 Kubernetes API 更新服務的標籤查詢，可以實現強大的路由方案。

## Why Terraform?
雖然可以使用映射到調用 API 的 `kubectl` 或類似基於 CLI 的工具來管理 `YAML` 中描述的所有 Kubernetes 資源，但是使用`Terraform` 進行編排會帶來一些好處。

- 使用相同的[配置語法](https://www.terraform.io/docs/configuration/syntax.html)來配置 Kubernetes 基礎架構並將應用程式部署到該基礎架構中
- drift detection
    - `terraform plan` 持續呈現給定時間的實際狀況與你打算應用的配置之間的差異
- full lifecycle management
    - `Terraform` 不僅最初會創建資源，且提供一個命令來創建、更新和刪除追蹤的資源，而無需檢查 API 來識別那些資源
- synchronous feedback 
    - 儘管異步行為通常有用處，但有時會有反效果，因為操作結果的識別工作留給了用戶，可能是失敗或所創建資源的詳細訊息。例如，在完成配置之前，沒有負載均衡器的 `IP/hostname`，因此無法創建指向它的 DNS 記錄。
- [graph of relationships](https://www.terraform.io/docs/internals/graph.html)
    - `Terraform` 了解資源之間的關係，這有助於調度。例如在集群存在之前，`Terraform` 不會嘗試在 `Kubernetes` 集群中創建服務。

## Clone the sample code

這邊是實驗準備好的資源

```shell
gsutil -m cp -r gs://spls/gsp233/* .
cd tf-gke-k8s-service-lb
```

## Understand the code
```shell
cat main.tf
variable "region" {
  default = "us-west1"
}

variable "location" {
  default = "us-west1-b"
}

variable "network_name" {
  default = "tf-gke-k8s"
}

provider "google" {
  region = var.region
}

resource "google_compute_network" "default" {
  name                    = var.network_name
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "default" {
  name                     = var.network_name
  ip_cidr_range            = "10.127.0.0/20"
  network                  = google_compute_network.default.self_link
  region                   = var.region
  private_ip_google_access = true
}

data "google_client_config" "current" {
}

data "google_container_engine_versions" "default" {
  location = var.location
}

resource "google_container_cluster" "default" {
  name               = var.network_name
  location           = var.location
  initial_node_count = 3
  min_master_version = data.google_container_engine_versions.default.latest_master_version
  network            = google_compute_subnetwork.default.name
  subnetwork         = google_compute_subnetwork.default.name

  // Use legacy ABAC until these issues are resolved:
  //   https://github.com/mcuadros/terraform-provider-helm/issues/56
  //   https://github.com/terraform-providers/terraform-provider-kubernetes/pull/73
  enable_legacy_abac = true

  // Wait for the GCE LB controller to cleanup the resources.
  // Wait for the GCE LB controller to cleanup the resources.
  provisioner "local-exec" {
    when    = destroy
    command = "sleep 90"
  }
}

output "network" {
  value = google_compute_subnetwork.default.network
}

output "subnetwork_name" {
  value = google_compute_subnetwork.default.name
}

output "cluster_name" {
  value = google_container_cluster.default.name
}

output "cluster_region" {
  value = var.region
}

output "cluster_location" {
  value = google_container_cluster.default.location
}
```


- 為 `region`、`zone` 和 `network_name` 定義變量。這些值將用於創建 Kubernetes 集群。
- Google Cloud 提供商將允許我們在此項目中創建資源
- 定義了幾種資源來創建適當的網路和群集
- 最後，在運行 `terraform` 後，將看到一些輸出

```shell
cat k8s.tf
provider "kubernetes" {
  version = "~> 1.10.0"
  host    = google_container_cluster.default.endpoint
  token   = data.google_client_config.current.access_token
  client_certificate = base64decode(
    google_container_cluster.default.master_auth[0].client_certificate,
  )
  client_key = base64decode(google_container_cluster.default.master_auth[0].client_key)
  cluster_ca_certificate = base64decode(
    google_container_cluster.default.master_auth[0].cluster_ca_certificate,
  )
}

resource "kubernetes_namespace" "staging" {
  metadata {
    name = "staging"
  }
}

resource "google_compute_address" "default" {
  name   = var.network_name
  region = var.region
}

resource "kubernetes_service" "nginx" {
  metadata {
    namespace = kubernetes_namespace.staging.metadata[0].name
    name      = "nginx"
  }

  spec {
    selector = {
      run = "nginx"
    }

    session_affinity = "ClientIP"

    port {
      protocol    = "TCP"
      port        = 80
            target_port = 80
    }

    type             = "LoadBalancer"
    load_balancer_ip = google_compute_address.default.address
  }
}

resource "kubernetes_replication_controller" "nginx" {
  metadata {
    name      = "nginx"
    namespace = kubernetes_namespace.staging.metadata[0].name

    labels = {
      run = "nginx"
    }
  }

  spec {
    selector = {
      run = "nginx"
    }

    template {
      metadata {
          name = "nginx"
          labels = {
              run = "nginx"
          }
      }

      spec {
        container {
            image = "nginx:latest"
            name  = "nginx"

            resources {
                limits {
                    cpu    = "0.5"
                    memory = "512Mi"
                }

                requests {
                    cpu    = "250m"
                    memory = "50Mi"
                }
            }
        }
      }
    }
  }
}

output "load-balancer-ip" {
  value = google_compute_address.default.address
}
```


- 該腳本使用 `Terraform` 配置 `Kubernetes` 程序，並創建 `Service`、`namespace` 和 `Replication_controller` 資源
- 該腳本回傳 nginx 服務 IP 作為輸出


## Initialize and install dependencies
`terraform init` 用於初始化包含 `Terraform` 配置檔案的工作目錄。

此指令執行幾個不同的初始化步驟，以準備要使用的工作目錄，並始終可以安全的運行多次，以使工作目錄與配置中的更改保持最新：

```shell
terraform init
```

`terraform apply` 命令用於運行更改以達到所需的配置狀態。

```shell
terraform apply
```

如果需要可以查看 `Terraform` 的操作並檢查要創建的資源。準備就緒後，鍵入 `yes` 開始 `Terraform` 操作。

輸出
```shell
...
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

cluster_location = us-west1-b
cluster_name = tf-gke-k8s
cluster_region = us-west1
load-balancer-ip = 34.105.4.9
network = https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-00-a744abf05b02/global/networks/tf-gke-k8s
subnetwork_name = tf-gke-k8s
```

### Verify resources created by Terraform
- 至  `Navigation menu > Kubernetes Engine`
- 點擊 tf-gke-k8s 集群並檢查其配置
- 在左側點擊 `Services & Ingress`，然後檢查 nginx 服務狀態
- 單擊 `Endpoints` IP 地址以打開 nginx 初始頁面

![](https://i.imgur.com/kv1G2R6.png)
![](https://i.imgur.com/Wlwerex.png)
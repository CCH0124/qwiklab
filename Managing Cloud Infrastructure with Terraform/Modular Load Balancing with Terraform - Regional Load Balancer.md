Google Cloud 上的負載均衡不同於其他雲提供商。Google Cloud 使用轉發規則而不是路由實例。這些轉發規則與後端服務、目標池、URL 映射和目標代理相結合，以構建跨多個區域和實例群組的功能性負載均衡器。Terraform 是一種開源基礎架構管理工具，可透過使用模塊簡化 Google Cloud 上負載均衡器的配置。

## Terraform Modules Overview
### terraform-google-lb (regional forwarding rule)
此模組創建一個 [TCP 網路負載均衡](https://cloud.google.com/compute/docs/load-balancing/network/example)，以實現跨區域管理實例群組的負載均衡。提供對託管實例群組的引用，模組將其添加到目標池。創建區域轉發規則以將流量轉發到目標池中的正常實例。

![](https://cdn.qwiklabs.com/vosb%2FYcbcF%2BIB%2FbUJoMKlsN87f%2BtZ%2FGU7fR60tdzHVM%3D)

##### Example Snippet:
```shell
module "gce-lb-fr" {
  source       = "github.com/GoogleCloudPlatform/terraform-google-lb"
  region       = "${var.region}"
  name         = "group1-lb"
  service_port = "${module.mig1.service_port}"
  target_tags  = ["${module.mig1.target_tags}"]
}
```

### terraform-google-lb-internal (regional internal forwarding rule)
此模組創建一個[內部負載均衡](https://cloud.google.com/compute/docs/load-balancing/internal/)，用於內部資源的區域負載均衡。提供對託管實例群組的引用，模組將其添加到區域的後端服務，創建內部轉發規則以將流量轉發到運行狀況良好的實例。

![](https://cdn.qwiklabs.com/%2BHUnHV2BgVhK0JbEFbqywqXaW2BBDG2KjVOrJCkqYX4%3D)

##### Example Snippet:

```shell
module "gce-ilb" {
  source         = "github.com/GoogleCloudPlatform/terraform-google-lb-internal"
  region         = "${var.region}"
  name           = "group2-ilb"
  ports          = ["${module.mig2.service_port}"]
  health_port    = "${module.mig2.service_port}"
  source_tags    = ["${module.mig1.target_tags}"]
  target_tags    = ["${module.mig2.target_tags}","${module.mig3.target_tags}"]
  backends       = [
    { group = "${module.mig2.instance_group}" },
    { group = "${module.mig3.instance_group}" },
  ]
}
```

### terraform-google-lb-http (global HTTP(S) forwarding rule)
此模組創建一個[全域 HTTP 負載均衡](https://cloud.google.com/compute/docs/load-balancing/http/)，用於基於區域內容的多區域負載平衡。提供對託管實例群組的引用，用於 SSL 終止的可選證書，並且該模組創建 [http 後端服務](https://cloud.google.com/compute/docs/load-balancing/http/backend-service)、[URL 映射](https://cloud.google.com/compute/docs/load-balancing/http/url-map)、[HTTP(S) 目標代理](https://cloud.google.com/compute/docs/load-balancing/http/target-proxies)和[全域 http 轉發規則](https://cloud.google.com/compute/docs/load-balancing/http/global-forwarding-rules)，以將基於 HTTP 路徑的流量路由到正常運行實例。

![](https://cdn.qwiklabs.com/SVuyORdqRZ1dPr1nApd8xit7EeMvLp3%2Fza0FBKG4ycw%3D)

```shell
module "gce-lb-http" {
  source            = "github.com/GoogleCloudPlatform/terraform-google-lb-http"
  name              = "group-http-lb"
  target_tags       = ["${module.mig1.target_tags}", "${module.mig2.target_tags}"]
  backends          = {
    "0" = [
      { group = "${module.mig1.instance_group}" },
      { group = "${module.mig2.instance_group}" }
    ],
  }
  backend_params    = [
    # health check path, port name, port number, timeout seconds.
    "/,http,80,10"
  ]
}
```

## Install Terraform
```shell
terraform version
```

安裝和配置 tfswitch
```shell
wget https://github.com/warrensbox/terraform-switcher/releases/download/0.7.737/terraform-switcher_0.7.737_linux_amd64.tar.gz
mkdir -p ${HOME}/bin
tar -xvf terraform-switcher_0.7.737_linux_amd64.tar.gz -C ${HOME}/bin
export PATH=$PATH:${HOME}/bin
tfswitch -b ${HOME}/bin/terraform 0.11.14
echo "0.11.14" >> .tfswitchrc
exit
```

會變成 0.11.14 版
```shell
terraform version
```

## Clone the examples repository
複製[此](https://github.com/GoogleCloudPlatform/terraform-google-lb)倉庫
```shell
git clone https://github.com/GoogleCloudPlatform/terraform-google-lb
cd ~/terraform-google-lb/
git checkout 028dd3dc09ae1bf47e107ab435310c0a57b1674c
cd ~/terraform-google-lb/examples/basic
```


更新 Terraform `Google Provider` 以獲取該版本，`vi main.tf`

```shell
version = "1.18.0"
```
會向下面一樣
```shell
provider google {
  region = "${var.region}"
  version = "1.18.0"
}
```


```shell
$ cat main.tf
/*
 * Copyright 2017 Google Inc.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

variable "region" {
  default = "us-central1"
}

variable "zone" {
  default = "us-central1-b"
}

variable "network_name" {
  default = "tf-lb-basic"
}

provider "google" {
  region = "${var.region}"
  version = "1.18.0"
}

resource "google_compute_network" "default" {
  name                    = "${var.network_name}"
  auto_create_subnetworks = "false"
  }

resource "google_compute_subnetwork" "default" {
  name                     = "${var.network_name}"
  ip_cidr_range            = "10.127.0.0/20"
  network                  = "${google_compute_network.default.self_link}"
  region                   = "${var.region}"
  private_ip_google_access = true
}

data "template_file" "group1-startup-script" {
  template = "${file("${format("%s/gceme.sh.tpl", path.module)}")}"

  vars {
    PROXY_PATH = ""
  }
}

module "mig1" {
  source            = "GoogleCloudPlatform/managed-instance-group/google"
  version           = "1.1.13"
  region            = "${var.region}"
  zone              = "${var.zone}"
  name              = "${var.network_name}-group1"
  size              = 2
  service_port      = 80
  service_port_name = "http"
  http_health_check = false
  target_pools      = ["${module.gce-lb-fr.target_pool}"]
  target_tags       = ["allow-service1"]
  startup_script    = "${data.template_file.group1-startup-script.rendered}"
  network           = "${google_compute_subnetwork.default.name}"
  subnetwork        = "${google_compute_subnetwork.default.name}"
}

module "gce-lb-fr" {
  source       = "../../"
  region       = "${var.region}"
  name         = "${var.network_name}"
  service_port = "${module.mig1.service_port}"
  target_tags  = ["${module.mig1.target_tags}"]
    network      = "${google_compute_subnetwork.default.name}"
}

output "load-balancer-ip" {
  value = "${module.gce-lb-fr.external_ip}"
}
```

## TCP load balancer with regional forwarding rule
本實驗創建一個託管實例群組，該實例群組在同一區域中具有兩個實例，以及一個網路 TCP 負載均衡器。

![](https://cdn.qwiklabs.com/PcsgtHDeWwbgBzYPLoxdchDhvkoJzuV45gEgKhOCKLg%3D)

運行 Terraform 部署架構：
```shell
export GOOGLE_PROJECT=$(gcloud config get-value project)
```

`terraform init` 用於初始化包含 `Terraform` 配置檔案的工作目錄。該命令執行幾個不同的初始化步驟，以準備要使用的工作目錄。多次運行該命令始終是安全的，它使工作目錄與配置中的更改保持同步。

```shell
terraform init
terraform plan
terraform apply
```


最後佈署完成後，可以透過至 `Navigation Menu > Networking Services > Load Balancing`，並在 Cloud Console 中檢查負載均衡器的狀態。

```shell
EXTERNAL_IP=$(terraform output | grep load-balancer-ip | cut -d = -f2 | xargs echo -n)
echo "http://${EXTERNAL_IP}"
```


![](![](https://i.imgur.com/Zn2MntV.png))
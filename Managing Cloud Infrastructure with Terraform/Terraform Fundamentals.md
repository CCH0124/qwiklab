`Terraform` 能夠安全、可預測創建、更改和改善基礎架構。它是一個開放源代碼工具，將 API 編碼為聲明式配置檔案，這些檔案可以在團隊成員之間共享、被視為代碼、進行編輯、查看和版本控制。

## What is Terraform?
`Terraform` 是用於安全有效構建、更改和版本控制基礎結構的工具。`Terraform` 可以管理現有的受歡迎服務提供商以及自定義的內部解決方案。

配置檔案向 `Terraform` 描述了運行單個應用程式或整個數據中心所需的組件。`Terraform` 產生執行計劃，以描述達到預期狀態所需執行的操作，然後執行該計劃以構建所描述的基礎結構。隨著配置的更改，`Terraform` 能夠確定更改的內容並創建可以應用的增量執行計劃。

`Terraform` 可以管理的基礎結構包括低層級組件，例如 compute instances、storage 和 networking，以及高層級組件，例如 DNS entries、SaaS 功能等。

### Key Features
##### Infrastructure as Code
使用高層級配置語法描述基礎架構。這樣就可以像對待任何其他程式碼一樣對數據中心的藍圖進行版本控制和處理。此外，基礎架構可以共享和重複使用。
##### Execution Plans
`Terraform` 有一個 `planning` 步驟，在其產生執行計劃。執行計劃將顯示 `Terraform` 在調用 apply 時將執行的操作。這樣可以避免 `Terraform` 操做基礎架構時出現任何意外。
##### Resource Graph
`Terraform` 構建所有資源的圖形，並並行化所有非依賴資源的創建和修改。因此，`Terraform` 盡可能高效構建基礎架構，並且操作員可以熟悉其基礎架構中的依賴性。

##### Change Automation
可以將復雜的變更集合應用於基礎架構，而無需進行過多的人工干預。使用前面提到的`Execution Plans`和`Resource Graph`，可以準確知道 `Terraform` 將要更改的內容和順序，從而避免了許多可能的人為錯誤。

## Build Infrastructure
### Configuration
用於描述 `Terraform` 中的基礎架構的檔案集合簡稱為 `Terraform configuration`。現在，我們編寫第一個配置以啟動單個 VM 實例。

配置檔案的格式在[此](https://www.terraform.io/docs/configuration/index.html)。建議使用 JSON 創建配置檔案。

```shell
vi instance.tf
```

```shell
resource "google_compute_instance" "default" {
  project      = "<PROJECT_ID>"
  name         = "terraform"
  machine_type = "n1-standard-1"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9"
    }
  }

  network_interface {
    network = "default"
    access_config {
    }
  }
}
```

這是 `Terraform` 準備應用的完整配置。整體結構應該直觀而直接。

`instance.tf` 檔案中的 `resource` 區塊定義了基礎架構中存在的資源。資源可能是實體組件，例如 VM 實例。

在打開 `resource` 之前，`resource` 區塊有兩個字符串：`resource type` 和 `resource name`。在本實驗中，`resource type` 為 `google_compute_instance`，`resource name` 為 `terraform`。類型的前綴映射到提供程式：`google_compute_instance` 自動告訴 `Terraform` 它由 Google 提供程式管理。

在 `resource` 區塊本身內是資源所需的配置。

### Initialization

這將初始化各種本地設置和後續命令將使用的數據。

`Terraform` 使用基於插件的體系結構來支持眾多可用的基礎架構和服務提供商。每個*提供商*`都是其自己的封裝二進製文件，與Terraform` 本身分別分發。`terraform init` 命令將自動下載並安裝任何提供程序二進製文件，提供程序在配置中使用，在這種情況下，該二進製檔案僅是 Google 提供程序。

```shell
terraform init
```

Google 提供插件將與其他各種簿記檔案一起下載並安裝在當前工作目錄的子目錄中。將看到*Initializing provider plugins*`訊息。Terraform` 知道你正在從 Google 項目中運行，並且正在獲取 Google 資源。

```shell
* provider.google: version = "~> 3.42"
```
訊息輸出指定要安裝的插件版本，並建議在以後的配置文件中指定該版本，以確保 `terraform init` 將安裝兼容版本。

`terraform plan` 命令用於建立執行計劃。除非明確禁用，否則 `Terraform` 會執行刷新，然後確定要實現配置檔案中指定的所需狀態需要執行哪些操作。

```shell
terraform plan
```

該命令是檢查一組更改的執行計劃(execution plan)是否符合期望而不對實際資源或狀態進行任何更改的便捷方法。例如，可以在對版本控制進行更改之前運行 `terraform plan`，以確定它會按預期運行。

>可使用 `-out` 參數將產生的計劃儲存到檔案中，以便之後使用 `terraform apply` 執行


### Apply Changes
與創建的 `instance.tf` 檔案相同的目錄中，運行 `terraform apply`。

```shell
terraform apply
```

此輸出顯示 `Execution Plan`，該 Plan 描述 `Terraform` 為更改實際基礎架構以匹配配置而將採取的操作。輸出格式類似於由 Git 之類的工具生成的 diff 格式。

`google_compute_instance.terraform` 旁邊有一個 `+`，表示 `Terraform` 將建立此資源。在下面，將看到將要設置的屬性。當顯示的值是 `<computed>` 時，表示在創建資源之前將不知道該值。

如果成功創建了 `Plan`，則 `Terraform` 現在將暫停並等待批准，然後再繼續。在生產環境中，如果 `Execution Plan` 中的任何內容看起來不正確或危險，可以在此處安全地中止，該基礎架構將未進行任何更改。

對於這種情況，該 `Plan` 看起來是可以接受的，因此在確認提示下鍵入 yes 以繼續。

點擊 `Compute Engine> VM` 實例以查看創建的 VM。`Terraform` 已將一些數據寫入 `terraform.tfstate` 檔案。這個狀態檔案非常重要。它追蹤已創建資源的 `ID`，以便 `Terraform` 知道它正在管理什麼。可以使用 `terraform show` 查看當前狀態
```shell
$ terraform show
# google_compute_instance.default:
resource "google_compute_instance" "default" {
    can_ip_forward       = false
    cpu_platform         = "Intel Haswell"
    current_status       = "RUNNING"
    deletion_protection  = false
    enable_display       = false
    guest_accelerator    = []
    id                   = "projects/qwiklabs-gcp-01-6b180ceb0db1/zones/us-central1-a/instances/terraform"
    instance_id          = "8256165311136164078"
    label_fingerprint    = "42WmSpB8rSM="
    machine_type         = "n1-standard-1"
    metadata_fingerprint = "qZMHW1nvfqs="
    name                 = "terraform"
    project              = "qwiklabs-gcp-01-6b180ceb0db1"
    self_link            = "https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-6b180ceb0db1/zones/us-central1-a/instances/terraform"
    tags_fingerprint     = "42WmSpB8rSM="
    zone                 = "us-central1-a"

    boot_disk {
        auto_delete = true
        device_name = "persistent-disk-0"
        mode        = "READ_WRITE"
        source      = "https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-6b180ceb0db1/zones/us-central1-a/disks/terraform"

        initialize_params {
            image  = "https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/debian-9-stretch-v20200910"
            labels = {}
            size   = 10
            type   = "pd-standard"
        }
    }

    network_interface {
        name               = "nic0"
        network            = "https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-6b180ceb0db1/global/networks/default"
        network_ip         = "10.128.0.2"
        subnetwork         = "https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-01-6b180ceb0db1/regions/us-central1/subnetworks/default"
        subnetwork_project = "qwiklabs-gcp-01-6b180ceb0db1"

        access_config {
            nat_ip       = "35.194.50.213"
            network_tier = "PREMIUM"
        }
    }

    scheduling {
        automatic_restart   = true
        on_host_maintenance = "MIGRATE"
        preemptible         = false
    }
}
```

可以看到透過創建此資源，還收集了很多有關它的資訊。如果要在 `execution plan` 執行後對其進行檢查，可使用 `terraform plan` 命令。
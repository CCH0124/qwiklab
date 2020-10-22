此實驗創建一個 HTTPS 負載均衡器，以將流量轉發到自定義 URL 映射。URL 映射使用來自 Cloud Storage 儲存桶的靜態資產將流量發送到距離最近的區域。TLS 密鑰和證書由 Terraform 使用 [TLS](https://www.terraform.io/docs/providers/tls/index.html) 程序生成。

## Content-Based Load Balancer

![](https://cdn.qwiklabs.com/8BAGcQptix4GVZGtF1UCJxH7rjKvaLjx5y5lgXBCwSk%3D)

## Verifying Terraform Installation

```shell
terraform version
```

## Clone the sample repository

```shell
git clone https://github.com/GoogleCloudPlatform/terraform-google-lb-http.git
cd ~/terraform-google-lb-http/examples/multi-backend-multi-mig-bucket-https-lb
```

## Run Terraform
### Initialize a working directory
`terraform init` 用於初始化包含 Terraform 配置檔案的工作目錄。該命令執行幾個不同的初始化步驟來準備要使用的工作目錄。多次運行該命令會是安全的，它使工作目錄與配置中的更改保持同步。

```shell
terraform init
```

### Create an execution plan
`terraform plan` 用於建立執行計劃。除非明確禁用，否則 Terraform 會執行刷新，然後確定要實現配置檔案中指定的所需狀態需要執行哪些操作。此命令是檢查更改的 execution plan 是否符合期望而不對實際資源或狀態進行任何更改的便捷方法。例如，可以在對版本控制進行更改之前運行 `terraform plan`，以確認它會按預期運行。

```shell
terraform plan -out=tfplan -var 'project=<PROJECT_ID>'
```
可選的 `-out` 參數可用於將產生的計劃保存到檔案中，以便之後使用 `terraform apply` 執行。


### Apply the changes
`terraform apply` 命令用於執行所需的更改，以達到所需的配置狀態，或由 `terraform plan` 執行計劃產生的預定集合。

```shell
terraform apply tfplan
...

Apply complete! Resources: 42 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

asset-url = https://35.201.118.12/assets/gcp-logo.svg
group1_region = us-west1
group2_region = us-central1
group3_region = us-east1
load-balancer-ip = 35.201.118.12
```

驗證 Terraform 創建的資源：
- 至 `Network services > Load Balancing`
- 點擊 `ml-bk-ml-mig-bkt-s-lb` 查看詳細訊息

![](https://i.imgur.com/U4ETxCy.png)


運行以下獲取對外 URL：
```shell
EXTERNAL_IP=$(terraform output | grep load-balancer-ip | cut -d = -f2 | xargs echo -n)
echo https://${EXTERNAL_IP}
```
點擊該輸出的 URL，同時對 `https://EXTERNAL_IP/{group1|group2|group3}` 做瀏覽。

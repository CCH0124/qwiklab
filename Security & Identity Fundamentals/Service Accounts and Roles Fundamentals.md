`Service account`(服務帳戶) 是一種特殊的 Google 帳戶，它面向虛擬機而不是最終用戶授予權限。`Service account` 主要用於確保與 API 和 Google Cloud 服務的安全或託管連接。它授予對可信連接的訪問權限並拒絕惡意連接，是任何 Google Cloud 項目的必備安全功能。

## What are Service Accounts?
`Service account` 是屬於應用程式或虛擬機而非單一最終用戶的特殊 Google 帳戶。應用程式使用 `Service account` 來調用服務的 Google API，這樣不會直接與用戶相關。

`Service account` 由其帳戶唯一的電子郵件地址標識。

## Types of Service Accounts

### User-managed service accounts
當使用 Cloud Console 建立新的 Cloud 項目時，如果項目啟用了 Compute Engine API，域設下下會創建一個 Compute Engine 的 `Service account`，使用電子郵件可以識別：
```shell
PROJECT_NUMBER-compute@developer.gserviceaccount.com
```
如果啟用 App Engine 應用則是
```shell
PROJECT_ID@appspot.gserviceaccount.com
```

### Google-managed service accounts
除了用戶管理的服務帳戶外，可能還會在項目的 IAM policy 或 Cloud Console 中看到一些其他服務帳戶，這些服務帳戶由 Google 創建並擁有。這些帳戶同時代表不同的 Google 服務，並且自動為每個帳戶授予 IAM 角色，以訪問我們的 Google Cloud 項目。

### Google APIs service account
Google 管理的服務帳戶的一個範例是可以使用電子郵件識別的 Google API 服務帳戶：

```shell
PROJECT_NUMBER@cloudservices.gserviceaccount.com
```

該服務帳戶專門用於代表運行內部 Google 流程，並且未在 Cloud Console 的 `Service Accounts` 部分中列出。預設下，該帳戶將自動被授予項目的項目編輯者角色，並在 Cloud Console 的 IAM 部分中列出。當刪除項目時，才會刪除此 `Service accounts`，Google 服務依賴於有權訪問項目的帳戶，因此不應刪除或更改服務帳戶在項目中的角色。


## Creating and Managing Service Accounts

### Creating a service account
```shell
gcloud iam service-accounts create my-sa-123 --display-name "my service account"
```
### Granting Roles to Service Accounts
在授予 IAM 角色時，可將服務帳戶視為資源或身份(identity)。應用程式使用服務帳戶作為身份來對 Google Cloud 服務進行身份驗證。同時，可能還想控制誰可以啟動 VM，可以藉由向用戶（身份）授予服務帳戶（資源）的 `serviceAccountUser` 角色來實現。

##### Granting roles to a service account for specific resources
將角色授予服務帳戶，以便該服務帳戶有權在 Cloud Platform 項目中完成對資源的特定操作。

```shell
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:my-sa-123@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/editor
```

## Understanding Roles
透過身份(identity)調用 Google Cloud API 時，Google Cloud 身份和存取管理要求該身份具有使用資源的適當權限。可以藉由向用戶、群組或服務帳戶授予角色權限。

### Types of Roles
Cloud IAM 中有三種角色：
- Primitive roles
    - 包括在引入 Cloud IAM 之前存在的 `Owner`、`Editor` 和 `Viewer` 角色
- Predefined roles
    - 它們提供對特定服務的細節訪問，並由 Google Cloud 管理
- Custom roles
    - 根據用戶指定的權限列表提供細節的訪問

細節可點此[鏈接](https://cloud.google.com/iam/docs/understanding-roles)

## Create a service account

`Navigation menu > IAM & Admin` 選擇 `Service accounts` 並點擊 `+ Create Service Account`
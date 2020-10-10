Cloud IAM 提供最小的麻煩和高度的自動化來管理資源權限，我們不會直接授予使用者權限。相反，授予他們的角色會綁定一個或多個權限。這可以將公司內的職務職能映射到群組和角色，使得使用者只能存取完成工作所需的權限，管理員可以輕鬆將默認權限授予整個使用者群組。

Cloud IAM 中有兩種角色

- Predefined Roles
    - 由 Google 建立和維護
    - 權限會根據需求自動更新，例如在將新功能或服務添加到 Google Cloud 時
- Custom Roles
    - 用戶定義，允許綁定一個或多個支援的權限來滿足特定需求
    - Google 不維護此。當新的權限、功能或服務添加到 Google Cloud 時，`Custom Roles` 不會自動更新
    - 可藉由組合一個或多個可用的 Cloud IAM 權限來創建 `Custom Roles`


## Understanding IAM Custom Roles
Cloud IAM 提供用於創建和管理自定義角色的 `UI` 和 `API`。`Custom Roles` 使得能夠實施最小特權原則，從而確保組織中的用戶和服務帳戶僅具有執行其預期功能所必需的權限。

我們可以透過組合一個或多個可用的 Cloud IAM 權限來創建 `Custom Roles`。

>可以在組織(organization)級別和專案(project)級別創建 Custom Roles。但是，不能在目錄(folder)級別創建 Custom Roles。

在 Cloud IAM 權限以以下規則表示：

```shell
<service>.<resource>.<verb>
```
範例：`compute.instances.list` 權限允許使用者列出他們擁有的 Compute Engine 實例，而 compute.instances.stop 允許使用者停止 VM。

每個 `Google Cloud` 服務對其擁有的每個 `REST` 方法都具有關聯的權限，要調用方法，調用者需要該權限，例如：`topic.publish()` 的調用者需要`pubsub.topics.publish` 權限。

`Custom Roles` 只能用於在擁有其下的角色或資源的同一項目或組織的策略中授予權限；反而不能在不同項目或組織擁有的資源上授予一個項目或組織的`Custom Roles`。

## Required permissions and roles
要創建 `Custom Roles`，調用者必須具有 `iam.roles.create` 權限。

## Viewing the available permissions for a resource
可以使用 gcloud 工具、Cloud Console 或 IAM API 獲得可應用於資源以及層次結構中資源之下的所有權限。如下

```shell
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
```

## Getting the role metadata
創建 `Custom Roles` 之前，可能需要獲取預定義角色和自定義角色(Custom Roles)的元數據。我們可以使用 Cloud Console 或 IAM API 查看元數據，如下

```shell
gcloud iam roles describe [ROLE_NAME]
```

## Viewing the grantable roles on resources
使用 `gcloud iam list-grantable-roles` 返回可應用於給定資源的所有角色的清單。

```shell
gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
```

## Creating a custom role
要創建自 `Custom role`，調用者須擁有 `iam.roles.create` 權限。預設下，專案或組織的所有者擁有此權限，且可以創建和管理 `Custom role`。創建方式可使用 `gcloud iam roles create`，但也可以使用 `YAML`。

## To create a custom role using a YAML file
建立一個 `YAML`，其中包含 `Custom role` 的定義，結構如下：

```yaml
title: [ROLE_TITLE] # 像是 Role Viewer
description: [ROLE_DESCRIPTION] # 像是 ALPHA、BETA 或 GA
stage: [LAUNCH_STAGE] # 
includedPermissions: # 權限列表，像是 iam.roles.get
- [PERMISSION_1]
- [PERMISSION_2]
```

```yaml
vi role-definition.yaml
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
```

定義好後執行

```shell
gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID \
--file role-definition.yaml
```

## Create a custom role using flags
使用 `flag` 方法創建一個新的 `Custom role`。如下

```shell
gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
--title "Role Viewer" --description "Custom role description." \
--permissions compute.instances.get,compute.instances.list --stage ALPHA
```

## Listing the custom roles
列出 `Custom role`，並指定專案或組織級別的 `Custom role`
```shell
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
```
列出以定義
```shell
gcloud iam roles list
```

## Editing an existing custom role
`etag` 屬性用於驗證自上次請求以來 `Custom role` 是否已更改。可藉由使用 `gcloud iam Roles update` 更新 `Custom role`，同樣的可用 `YAML` 或是 `flag` 方式。在更新 `Custom role` 時，須使用 `--organization [ORGANIZATION_ID]` 或 `--project [PROJECT_ID]` 指定它是應用於組織級別還是專案級別。


## To update a custom role using a YAML file

```shell
gcloud iam roles describe [ROLE_ID] --project $DEVSHELL_PROJECT_ID
```

將上面輸出新增以下
- storage.buckets.get
- storage.buckets.list

```shell
vi new-role-definition.yaml
description: Edit access for App Versions
etag: BwVxIAbRq_I=
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
- storage.buckets.get
- storage.buckets.list
name: projects/[PROJECT_ID]/roles/editor
stage: ALPHA
title: Role Editor
```


使用 `update` 指令
```shell
gcloud iam roles update [ROLE_ID] --project $DEVSHELL_PROJECT_ID \
--file new-role-definition.yaml
```

## To update a custom role using flags
關於更新詳細資訊可透過此[鏈結](https://cloud.google.com/sdk/gcloud/reference/iam/roles/update)。
- --add-permissions
    - 新增權限
- --remove-permissions
    - 移除權限

```shell
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--add-permissions storage.buckets.get,storage.buckets.list
```

## Disabling a custom role
禁用角色後，與該角色相關的所有策略綁定都將被停用。禁用現有 `Custom role` 的最簡單方法是使用 `--stage` 並設置為 `DISABLED`。
以下範例是禁用 `viewer`
```shell
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--stage DISABLED
```

## Undeleting a custom role

透過以下方式可取消刪除角色。
```shell
gcloud iam roles undelete viewer --project $DEVSHELL_PROJECT_ID
```

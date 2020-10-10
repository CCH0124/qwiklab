Google Cloud 的身份和存取管理(IAM)服務可創建和管理 Google Cloud 資源的權限。Cloud IAM 將對 Google Cloud 服務訪問控制統一到一個系統中，並提供一致的操作集合。

## The IAM console and project level roles
`Navigation menu > IAM & Admin > IAM` 進入 IAM 控制台。點擊 `+ADD`，並點擊 `Select a role` 來選擇角色。其中會看到 `Browser`、`Editor`、`Owner` 和 `Viewer` 的角色。這四個在 Google Cloud 為基本角色。基本角色設置專案級別的權限，除非另有說明，否則將控制對所有Google Cloud 服務的訪問和管理。相關角色文檔可參考此[鏈接](https://cloud.google.com/iam/docs/understanding-roles#primitive_roles)。

登入實驗兩個的帳號，其權限設置如下

![](https://i.imgur.com/oEKxHyY.png)

檔使用第二個帳號登入時只有 `views` 權限，因此進行新增動作會有以下警告。
![](https://i.imgur.com/mV8Cpew.png)

使用第一個帳號建立儲存如下

![](https://i.imgur.com/7SknhqL.png)

第二帳號可以以 `views` 權限進行瀏覽

![](https://i.imgur.com/C2r8wkL.png)

從第一個帳號移除帳號二的 `views` 權限

![](https://i.imgur.com/gcl7H2q.png)

移除後，發現帳號二無法做瀏覽
![](https://i.imgur.com/arccPMp.png)

重新賦予 `views` 權限

![](https://i.imgur.com/3sGQ1Vg.png)


使用帳號2查看是否有瀏覽權限，從結果來看是有
```shell
$ gsutil ls gs://example-cch
gs://example-cch/sample.txt
```
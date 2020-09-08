---
layout: post
title: Machine Learning APIs - 01
date: 2020-09-03
excerpt: "Introduction to APIs in Google"
tags: [qwiklab, ML, google]
comments: true
---

社群有活動，因此報名參加，我會將 qwiklab 的東西做一個紀錄。

## Overivew

`API（Application Programming Interfaces）`使開發人員能夠存取計算資源和數據的軟體程序。來自許多不同領域的公司提供公開可用的 `API`，以便開發人員可以將專用工具、服務或函式庫與自己的應用程序和程式庫整合在一起。

我們將學習以下

- Google APIs
- API architecture
- HTTP protocol and methods
- Endpoints
- REST (Representational State Transfer) and RESTful APIs
- JSON (JavaScript Object Notation)
- API authentication services

## Cloud APIs
`Google` 提供了可應用於許多不同領域和行業的 `API`。經驗豐富的用戶幾乎將在其本地環境中整合和使用 `Cloud API`，很少使用 `Cloud Console` 來運行工具和服務。

## API Library

選擇 `APIs & Services > Library`，對應中文是 `API 服務 > 資料庫`。`API` 資料庫提供了 200 多個 `Google API` 的快速訪問與文檔和配置選項。

搜訊該資料庫的 `Fitness` 進行練習，點擊啟用。

## API Dashboard

選擇 `APIs & Services > Dashboard`，中文為 `API 服務 > 資訊主頁`。`API` 儀表板詳細說明了專案對特定 `API` 的使用情況，包括 `traffic levels`、`error rates` 甚至 `latencies`，可幫助快速分類使用 `Google` 服務的應用程序中的問題。

![](https://i.imgur.com/8sbm49k.png)

滑到下面有 API 清單，點擊 `Fitness API`

![](https://i.imgur.com/F5Zesrh.png)


點擊後再從左邊選單選擇*配額(Quotas)*，可以查看和請求配額，控制對資源和數據的訪問以及查看指標。其中它會要求跳到 `IAM` 查看，如下圖顯示，為每一天能請求的量。

![](https://i.imgur.com/2dF3nFo.png)


## API Architecture

`API` 是一個允許程序相互通訊的方法，同時也需要遵守協定進行傳輸。

### Client-server model
網際網路是 `API` 用於在程序之間*傳輸請求*和*響應*的標准通訊，[client-server](https://en.wikipedia.org/wiki/Client%E2%80%93server_model)模型是基於 `Web` 的 `API` 用於交換訊息的基礎體系結構。

客戶端(client)是對某些計算資源或數據進行請求的設備，如：智慧型手機、筆電等），而客戶端的請求需要按照約定的協定進行格式化；服務器(server)上儲存數據或計算資源，其工作是解析並滿足客戶端的需求。下圖為一個較直覺的事例。

![](https://cdn.qwiklabs.com/NCUCvY6YCvhRw9iyhaZWY1XcoXU4X0F7iwhzSOyyo6E%3D)


## HTTP protocol and request methods

許多 `API` 都遵循 `HTTP` 協定，該協定指定了藉由網際網路在客戶端和服務器之間進行數據交換的*規則*和*方法*。利用 `HTTP` 協定的 `API` 使用 `HTTP` 請求方法，將客戶端請求傳輸到服務端，常用的 `HTTP` 請求方法是 `GET`、`POST`、`PUT` 和 `DELETE`。

儘管有很多種 `API`，它們都有各自獨特的用途和專業知識，但重要的是最終它們都使用相同的協定和底層方法進行客戶端-服務端通訊。


## Endpoints
`API` 利用稱為端點(endpoints)的通訊，讓客戶端可以訪問所需要的資源，而不會造成複雜性或不規則性。端點是服務端上託管的數據或計算資源的訪問點，其採用 [HTTP URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) 的形式。將端點添加到 `API` 的基本 `URL`，如：`http://example.com` 中，用以創建指向特定資源或資源容器的路徑。以下是端點的一些示例：

- `http://example.com/storelocations`
- `http://example.com/accounts`
- `http://example.com/employees`
- `http://example.com/storelocations/sanfrancisco`
- `http://example.com/storelocations/newdelhi`
- `http://example.com/storelocations/london`

可以將查詢字串添加到端點，如`http://example.com/endpoint/?id=1`，以完成傳遞 `API` 請求所需的變量。值觀來說，客戶端發送一個由 `HTTP` 方法（動詞）和端點（名詞）組成的請求，以接收特定數據或在服務端上執行特定操作。而服務端是藉由提供的*方法*和*端點*轉換並執行特定操作來滿足客戶請求的服務端。

## RESTful APIs
利用 `HTTP` 協定、請求方法和端點的 `API` 稱為 `RESTful API`。`REST（Representational State Transfer）`是一種架構樣式，規定了基於 `Web` 的通訊的標準。

除了查詢字符串，`RESTful API` 還可以在請求中使用以下字段：
- header：詳細說明 `HTTP` 請求本身的參數
- body：客戶端要發送到服務器的數據
    - 以 `JSON` 或 `XML` 數據格式語言編寫


- [google about-restful-apis](https://developers.google.com/photos/library/guides/about-restful-apis)

## API Data Formats (JSON)

`JSON` 在 `RESTful API` 中的使用超過 `XML`，這在很大程度上是因為 `JSON` 輕巧、易於閱讀且解析速度更快。

`JSON` 支持以下數據類型：
- Numbers：所有類型-整數和浮點值之間沒有區別
- Strings
- Booleans
- Arrays
- Null

`JSON` 數據由 `key-value` 組成。`key` 必須是*字串*類型，並且值可以是上面列出的任何數據類型。

簡單範例
```json
"Key1" : "Value 1"
"Key2" : 64
"Key3" : True
"Key4" : ["this", "is", "an", "array"]
```

`JSON` 物件使用大括號`{}`對按 `key-value` 排列的數據進行分組。以下是包含三個 `key-value` 的對象的示例：

```json
{
	"Name": "Julie",
	"Hometown": "Los Angeles, CA",
	"Age": 28
}
```

## Creating a JSON File in the Cloud Console
點擊 API 和服務，並從資料庫搜尋 `google cloud storage JSON API`。進入 `cloud shell` 並新增 `values.json` 檔案輸入以下。

```shell
{  "name": "my-test",
   "location": "us",
   "storageClass": "multi_regional"
}
```

上述創建了一個 `JSON` 檔案，其中包含一個物件，該物件具有三個 `key-value`，分別是 `name`、`location` 和 `storageClass`。這些是使用 `gsutil` 命令工具或控制台創建儲存桶時所需的值。


在使用 `Cloud Storage REST / JSON API` 創建儲存桶之前，需要獲取適當的*身份驗證*和*授權*策略。

## Authentication and Authorization
- Authentication 是指確定客戶端身份的過程
- Authorization 是指確定經過身份驗證的客戶端對一組資源具有哪些權限的過程

總之，`身份驗證(Authentication)` 確定你是誰，`授權(Authorization)` 確定你可以做什麼。

Google `API` 使用三種身份驗證/授權服務。這些是*API Keys*，*"Service accounts* 和 *OAuth*。`API` 將根據請求的資源以及從何處調用 `API` 來使用其中一種身份驗證服務。

### API Keys

`API Keys` 是 `令牌(secret tokens)`，通常以加密字串的形式出現。`API Keys` 可以快速生成和使用。使用公有數據或方法並希望使開發人員快速啟動並運行的 `API` 通常會使用 `API​​ Keys` 來驗證用戶身份。透過識別調用專案，`API Keys` 讓使用訊息可以與該專案相關聯，且它們可以拒絕來自未經 `API` 授予訪問權限或未啟用該項目的調用。

### OAuth

`OAuth tokens` 的格式類似於 `API Keys`，但它更加安全，可以鏈接到用戶帳號或身份。這些 `tikens` 主要在 `API` 為開發人員提供訪問客戶端數據的方式時使用。儘管 `API Keys` 使開發人員可以訪問 `API` 的所有功能，但 `OAuth` 客戶端 `ID` 皆基於作用域(scope)，*不同的特權將授予不同的身份*。

### Service Accounts

服務帳戶(Service Accounts)是屬於你的應用程式或虛擬機(VM)，而不是終端用戶的一種特殊類型的 `Google` 帳戶。當應用程式採用服務帳戶的身份來調用Google `API`，這樣用戶就不會直接參與其中。

可以透過向服務帳戶提供其*私鑰*來使用服務帳戶，也可以使用在 `Google Cloud Functions`、`Google App Engine`、`Google Compute Engine` 或`Google Kubernetes Engine` 上運行時可用的內置服務帳戶。


## Authenticate and authorize the Cloud Storage JSON/REST API

打開 [ OAuth 2.0 playground ](https://developers.google.com/oauthplayground/)，選擇 ` Cloud Storage JSON API V1`，再選擇 `https://www.googleapis.com/auth/devstorage.full_control`，如下圖。

![](https://i.imgur.com/nogn5mi.png)

點擊 `Authorize APIs`，第二步應已生成授權碼(authorization code)。再點擊 `Exchange authorization code for tokens`，會有下圖類似結果

![](https://cdn.qwiklabs.com/qR7J7L1r4Qc0N038G4COgd5sJAbMeuxbf5YfBZPjXVQ%3D)

複製 `Access token` 下面實驗會接著用。


## Create a bucket with the Cloud Storage JSON/REST API

開啟 `cloude shell`，接著操作以下

```shell
export OAUTH2_TOKEN=<YOUR_TOKEN>
export PROJECT_ID=<YOUR_PROJECT_ID>
$ curl -X POST --data-binary @values.json     -H "Authorization: Bearer $OAUTH2_TOKEN"     -H "Content-Type: application/json"    "https://www.googleapis.com/storage/v1/b?project=$PROJECT_ID"
{
  "kind": "storage#bucket",
  "selfLink": "https://www.googleapis.com/storage/v1/b/itachicchlab",
  "id": "itachicchlab",
  "name": "itachicchlab",
  "projectNumber": "462549267451",
  "metageneration": "1",
  "location": "US",
  "storageClass": "MULTI_REGIONAL",
  "etag": "CAE=",
  "timeCreated": "2020-09-02T15:33:53.788Z",
  "updated": "2020-09-02T15:33:53.788Z",
  "iamConfiguration": {
    "bucketPolicyOnly": {
      "enabled": false
    },
    "uniformBucketLevelAccess": {
      "enabled": false
    }
  },
  "locationType": "multi-region"
}
```


上面使用 `curl CLI` 工具發出了 `HTTP POST` 方法請求，將`values.json` 傳遞到請求 `body` 中。將 `OAuth tokens`和 `JSON` 規範作為請求*標頭*傳遞，該請求已路由到 `Cloud Storage` 端點，該端點包含設置為你的 `Google Cloud Project ID` 的查詢字符串參數。


## View your newly created Cloud Storage Bucket
點擊  `Storage > Browser`：

![](https://i.imgur.com/gRXnRW6.png)



## Upload a file using the Cloud Storage JSON/REST API

![](https://cdn.qwiklabs.com/E4%2BSx10I0HBeOFPB15BFPzf9%2F%2FOK%2Btf7S0Mbn6aQ8fw%3D)，保存此圖片並命名。


在 `Cloud Shell` 中，單擊右上角的三點圖標，然後單擊上載文件。選擇 `demo-image.png`。這將圖片添加到目錄。

```shell
$ export BUCKET_NAME=<YOUR_BUCKET>
$ curl -X POST --data-binary @$OBJECT \
>     -H "Authorization: Bearer $OAUTH2_TOKEN" \
>     -H "Content-Type: image/png" \
>     "https://www.googleapis.com/upload/storage/v1/b/$BUCKET_NAME/o?uploadType=media&name=demo-image"
Warning: Couldn't read data from file "", this makes an empty POST.
{
  "kind": "storage#object",
  "id": "itachicchlab/demo-image/1599061128047052",
  "selfLink": "https://www.googleapis.com/storage/v1/b/itachicchlab/o/demo-image",
  "mediaLink": "https://www.googleapis.com/download/storage/v1/b/itachicchlab/o/demo-image?generation=1599061128047052&alt=media",
  "name": "demo-image",
  "bucket": "itachicchlab",
  "generation": "1599061128047052",
  "metageneration": "1",
  "contentType": "image/png",
  "storageClass": "MULTI_REGIONAL",
  "size": "0",
  "md5Hash": "1B2M2Y8AsgTpgAmY7PhCfg==",
  "crc32c": "AAAAAA==",
  "etag": "CMzTi/TmyusCEAE=",
  "timeCreated": "2020-09-02T15:38:48.046Z",
  "updated": "2020-09-02T15:38:48.046Z",
  "timeStorageClassUpdated": "2020-09-02T15:38:48.046Z"
}
```
從 `gcp` 點擊創建的儲存名稱，之後會見到上傳的圖片，如下圖

![](https://i.imgur.com/1wqq1Jc.png)
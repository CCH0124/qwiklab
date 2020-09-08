---
layout: post
title: Baseline Data, ML, AI- 04
date: 2020-09-05
excerpt: "Video Intelligence"
tags: [qwiklab, ML, google]
comments: true
---

`Google Cloud Video Intelligence` 透過使用易於使用的 `REST API` 提取元數據，使視頻可搜索和發現。現在，可以搜索目錄中每個視頻文件的每時每刻。它可以快速註釋存儲在 `Cloud Storage` 中的視頻，並幫助您識別視頻中的關鍵實體（名詞）；以及它們出現在視頻中的時間。通過逐幀或逐幀檢索整個視頻中的相關信息，將訊號與聲噪分離。

## Enable the Video Intelligence API

`API and Service -> Database -> Cloud Video Intelligence API `

## Set up authorization

在 Cloud Shell 中，運行以下創建一個名為 `quickstart` 的新服務帳戶：
```shell
gcloud iam service-accounts create quickstart
```

創建一個服務帳戶 key 檔案，將`<your-project-123>` 替換為專案 ID：
```shell
gcloud iam service-accounts keys create key.json --iam-account quickstart@<your-project-123>.iam.gserviceaccount.com
```

透過傳遞服務帳戶 key 檔案的位置來驗證服務帳戶：
```shell
gcloud auth activate-service-account --key-file key.json
```

使用服務帳戶獲取授權 `token`：

```shell
gcloud auth print-access-token
```

token 在後面都會使用到。

## Make an annotate video request
編輯 `request.json`
```json
{
   "inputUri":"gs://spls/gsp154/video/train.mp4",
   "features": [
       "LABEL_DETECTION"
   ]
}
```

使用 `curl` 發出一個 `videos:annotate` 請求，以傳遞實體請求的檔案名：

```shell
curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/videos:annotate' \
    -d @request.json
```

回應結果，其 `name` 在後續會使用到
```shell
{
  "name": "us-west1.18358601230245040268"
}
```

使用此 shell 透過調用 `v1.operations` 端點來請求有關操作的訊息。將`OPERATION_NAME` 替換為剛回應的值：

```shell
curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/operations/OPERATION_NAME'
```

現在，將看到與操作有關的訊息。如果操作已完成，則包含完成字段，並將其設置為 `true`：

```json
{
  "name": "projects/627711253886/locations/asia-east1/operations/3101866506724155991",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.videointelligence.v1.AnnotateVideoProgress",
    "annotationProgress": [
      {
        "inputUri": "/spls/gsp154/video/train.mp4",
        "startTime": "2020-09-05T13:04:21.210079Z",
        "updateTime": "2020-09-05T13:04:21.210079Z"
      }
    ]
  }
}
```

給請求一段時間（通常大約一分鐘），重新運行該指令，同一請求返回帶註釋的結果：

```json
{
  "name": "OPERATION_NAME",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.videointelligence.v1.AnnotateVideoProgress",
    "annotationProgress": [
      {
        "inputUri": "/spls/gsp154/video/train.mp4",
        "progressPercent": 100,
        "startTime": "2020-09-05T13:04:21.210079Z",
        "updateTime": "2020-09-05T13:05:39.121394Z"
      }
    ]
  },
  "done": true,
  "response": {
    "@type": "type.googleapis.com/google.cloud.videointelligence.v1.AnnotateVideoResponse",
    "annotationResults": [
      {
        "inputUri": "/spls/gsp154/video/train.mp4",
        "segmentLabelAnnotations": [
          {
            "entity": {
              "entityId": "/m/01yrx",
              "languageCode": "en-US"
            },
            "segments": [
              {
                "segment": {
                  "startTimeOffset": "0s",
                  "endTimeOffset": "14.833664s"
                },
                "confidence": 0.98509187
              }
            ]
          },
         ...
```
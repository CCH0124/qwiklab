---
layout: post
title: Machine Learning APIs - 02
date: 2020-09-03
excerpt: "Extract, Analyze, and Translate Text from Images with the Cloud ML APIs"
tags: [qwiklab, ML, google]
comments: true
---


將從 `Cloud Vision API` 的文本檢測方法開始，利用光學字符識別（OCR）從圖像中提取文本。接著，我們將學習如何使用 `Translation API` 轉換文本並使用 `Natural Language API` 對其進行分析。


## Create an API Key

我們將使用 `curl` 將請求發送到 `Vision API`，因此需要生成 `API Keys` 以傳遞請求 `URL`。創建 `API Keys` 至 `APIs & services > Credentials`，如下圖

![](https://i.imgur.com/7afq5Sc.png)

最後複製生成的 `key`，將 `API Keys` 保存到環境變量中，以避免必須在每個請求中插入 `API Keys` 的值。

```shell
export API_KEY=<YOUR_API_KEY>
```

## Upload an image to a cloud storage bucket
### Creating a Cloud Storage bucket

將圖像發送到 `Vision API` 進行圖像檢測的方法有兩種：
- 通過向 `API` 發送 `base64` 編碼的圖像字串
- 向其傳遞儲存在 `Cloud Storage` 中的檔案的 `URL`

我們將創建一個 `Cloud Storage` 儲存桶來儲存圖像。先至 `Navigation menu > Storage` 並 `Create bucket`，將儲存桶指定一個唯一名稱，接著點擊 `create`。

![](https://i.imgur.com/ywpSQfe.png)

### Upload an image to your bucket

儲存並命名下面這張圖到本機端。在 storage browser 中創建的儲存桶點擊上傳檔案選擇下載至本機的圖片。

![](https://cdn.qwiklabs.com/cBoI5P4dZ6k%2FAr5Mv7eME%2F0fCb4G6nIGB0odCXzpEa4%3D)

![](https://i.imgur.com/0qz5dup.png)
![](https://i.imgur.com/D2CnJ8V.png)

接著，允許公開查閱檔案，同時保持對儲存桶的訪問權限為*私有*，點擊圖像檔案的 3 個點圖示，選擇*編輯權限(Edit Permissions)*，點擊*添加項目(Add Itme)*並設置以下內容：
- 為實體選擇 *Public*
- 確認 *allUsers* 是 *Name* 的值
- 選擇 *Reader* 作為訪問權限

最後*儲存(Save)*。


![](https://i.imgur.com/2iEPXRE.png)


現在已將檔案儲存在儲存桶中，可開始創建 `Vision API` 請求，並向其傳遞此圖片的 `URL`。


## Create your Vision API request
在 `Cloud Shell` 環境中，創建一個 `ocr-request.json` 檔案，將以下內容添加到該檔案中，將 `my-bucket-name` 替換為創建的儲存桶的名稱。

```json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/sign.jpg"
          }
        },
        "features": [
          {
            "type": "TEXT_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
```

我們將使用 `Vision API` 的 `TEXT_DETECTION` 功能。這將在圖像上運行光學字符識別（OCR）以提取文本。

## Call the Vision API's text detection method
在 `Cloud shell` 執行以下

```shll
curl -s -X POST -H "Content-Type: application/json" --data-binary @ocr-request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

會應如下

```shell
{
  "responses": [
    {
      "textAnnotations": [
        {
          "locale": "fr",
          "description": "A LE BIEN PUBLIC\nles dépêches\nPour Obama,\nla moutarde\nest\nde Dijon\n",
          "boundingPoly": {
            "vertices": [
              {
                "x": 176,
                "y": 41
              },
              {
                "x": 622,
                "y": 41
              },
              {
                "x": 622,
                "y": 794
              },
              {
                "x": 176,
                "y": 794
              }
            ]
          }
        },
...
```


`OCR` 方法能夠從我們的圖像中提取很多文本。從 `textAnnotations` 獲得的第一條數據是圖像中找到的 `API` 的整個文本區塊，這包括語言代碼，在這種情況下為 fr（法語），文本字串以及指示文本在圖像中所處位置的邊框。然後，會為文本中找到的每個單詞獲得一個對象，並帶有該特定單詞的邊界框。

>`Vision API` 還具有 [DOCUMENT_TEXT_DETECTION](https://cloud.google.com/vision/docs/reference/rest/v1/images/annotate#TextAnnotation) 功能，該功能針對帶有更多文本的圖像進行了優化。響應包括其它訊息，並將文本分為頁面、塊、段落和單詞。

下一步是翻譯，除非了解該語言...

```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @ocr-request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY} -o ocr-response.json
```

## Sending text from the image to the Translation API
[Translation API](https://cloud.google.com/translate/docs/reference/translate) 可以將文本翻譯成 100 多種語言。它還可以檢測輸入文本的語言。要將法語文本翻譯成英語，需要做的就是將文本和目標語言的*語言代碼*傳遞給 `Translation API`。

首先，創建一個 `translation-request.json` 文件，並添加以下內容：
```json
{
  "q": "your_text_here", // 傳遞字串進行翻譯的地方
  "target": "en"
}
```

提取圖像文本，並將其複製到 `translation-request.json`

```json
STR=$(jq .responses[0].textAnnotations[0].description ocr-response.json) && STR="${STR//\"}" && sed -i "s|your_text_here|$STR|g" translation-request.json
```

現在，可以調用 `Translation API` 。此命令把響應複製到 `translation-response.json` 文件中：
```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @translation-request.json https://translation.googleapis.com/language/translate/v2?key=${API_KEY} -o translation-response.json
```

使用 `cat` 查看內容
```shell
cat translation-response.json
```

結果

```shell
{
  "data": {
    "translations": [
      {
        "translatedText": "PUBLIC PROPERTY dispatches For Obama, mustard is from Dijon",
        "detectedSourceLanguage": "fr"
      }
    ]
  }
}
```

在響應中，`ranslationText` 包含翻譯結果，而 `detectedSourceLanguage` 為 `fr`，即法語的 ISO 語言代碼。`Translation API` 支持 100 多種語言，所有這些都在[此處](https://cloud.google.com/translate/docs/languages)列出。


## Analyzing the image's text with the Natural Language API
`Natural Language API` 透過提取實體，分析情感和語法以及將文本分類為類別來幫助我們理解文本。使用 `analyzerEntities` 方法查看 `Natural Language API API` 可以在圖像文本中找到哪些實體。

設置 `API` 請求，使用以下內容創建一個 `nl-request.json` 檔案

```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"your_text_here"
  },
  "encodingType":"UTF8"
}
```

在請求中，要告訴 `Natural Language API` 要發送的文本：


- type
    - 支持的類型值為 `PLAIN_TEXT` 或 `HTML`
- content
    - 將文本傳遞給 `Natural Language API` 進行分析。`Natural Language API` 還支援發送儲存在 `Cloud Storage` 中的檔案以進行文本處理
    - 要從 `Cloud Storage` 發送檔案，可以將 `content` 替換為 `gcsContentUri`，並在 `Cloud Storage` 中使用文本檔案 `URI` 的值
- encodingType
    - 告訴 `API` 處理文本時使用哪種類型的文本編碼，`API` 使用它來計算特定實體在文本中出現的位置

將翻譯後的文本複製到 `Natural Language API` 請求的 `content` 中：

```shell
STR=$(jq .data.translations[0].translatedText  translation-response.json) && STR="${STR//\"}" && sed -i "s|your_text_here|$STR|g" nl-request.json
```

請求調用 `Natural Language API` 的 `analyticsEntities` 端點
```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @nl-request.json
```

回應結果

```shell
{
  "entities": [
    {
      "name": "dispatches",
      "type": "OTHER",
      "metadata": {},
      "salience": 0.3560996,
      "mentions": [
        {
          "text": {
            "content": "dispatches",
            "beginOffset": 19
          },
          "type": "COMMON"
        }
      ]
    },
    {
      "name": "PUBLIC GOOD",
      "type": "OTHER",
      "metadata": {
        "wikipedia_url": "https://en.wikipedia.org/wiki/Public_good_(economics)",
        "mid": "/m/017bkk"
      },
      "salience": 0.26664853,
      "mentions": [
        {
          "text": {
            "content": "PUBLIC GOOD",
            "beginOffset": 3
          },
          "type": "PROPER"
                }
      ]
    },
    {
      "name": "Obama",
      "type": "PERSON",
      "metadata": {
        "wikipedia_url": "https://en.wikipedia.org/wiki/Barack_Obama",
        "mid": "/m/02mjmr"
      },
      "salience": 0.18647619,
      "mentions": [
        {
          "text": {
            "content": "Obama",
            "beginOffset": 34
          },
          "type": "PROPER"
        }
      ]
    },
    {
      "name": "mustard",
      "type": "OTHER",
      "metadata": {},
      "salience": 0.098094895,
      "mentions": [
        {
          "text": {
            "content": "mustard",
            "beginOffset": 45
          },
          "type": "COMMON"
        }
      ]
    },
    {
              "name": "Dijon",
      "type": "LOCATION",
      "metadata": {
        "wikipedia_url": "https://en.wikipedia.org/wiki/Dijon",
        "mid": "/m/0pbhz"
      },
      "salience": 0.09268077,
      "mentions": [
        {
          "text": {
            "content": "Dijon",
            "beginOffset": 61
          },
          "type": "PROPER"
        }
      ]
    }
  ],
  "language": "en"
}
```

對於具有維基百科頁面的實體，`API` 提供的元數據包括該頁面的 `URL` 以及實體的 `mid` 部分。`mid` 是一個 `ID`，該 `ID` 映射到 `Google` 的知識圖譜中的該實體，要獲取更多信息，可以調用 [Knowledge Graph API (知識圖譜 API)](https://developers.google.com/knowledge-graph/) ，並為其傳遞此 `ID`。對於所有實體，`Natural Language API` 會告訴它在文本中所處的位置（`mentions`），實體的 `type` 和 `salience`（[0,1] 範圍，指示實體對整個文本的重要性）。除了英語之外，`Natural Language API` 還支持[此處](https://cloud.google.com/natural-language/docs/languages)列出的語言。



## Other

使用成中文方式去嘗試調用 `API`。


```shell
$ vi translation-request.json
{
  "q": "A LE BIEN PUBLIC
les dépêches
Pour Obama,
la moutarde
est
de Dijon
",
  "target": "zh-Hant"
}
$ curl -s -X POST -H "Content-Type: application/json" --data-binary @translation-request.json https://translation.googleapis.com/language/translate/v2?key=${API_KEY} -o translation-response.json
$ cat translation-response.json
{
  "data": {
    "translations": [
      {
        "translatedText": "對公眾的好感對於奧巴馬，芥末醬來自第戎",
        "detectedSourceLanguage": "fr"
      }
    ]
  }
}
```


```shell
STR=$(jq .data.translations[0].translatedText  translation-response.json) && sed -i "s|\"TO PUBLIC GOOD the dispatches For Obama, the mustard is from Dijon\"|$STR|g" nl-request.json
student_03_f2ab3fc5dcd3@cloudshell:~ (qwiklabs-gcp-03-128b4171c964)$ cat nl-request.json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"對公眾的好感對於奧巴馬，芥末醬來自第戎"
  },
  "encodingType":"UTF8"
}
$ curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
>   -s -X POST -H "Content-Type: application/json" --data-binary @nl-request.json
{
  "entities": [
    {
      "name": "公眾",
      "type": "PERSON",
      "metadata": {},
      "salience": 0.3971286,
      "mentions": [
        {
          "text": {
            "content": "公眾",
            "beginOffset": 3
          },
          "type": "COMMON"
        }
      ]
    },
    {
      "name": "芥末醬",
      "type": "PERSON",
      "metadata": {
        "wikipedia_url": "https://en.wikipedia.org/wiki/Mustard_(condiment)",
        "mid": "/m/07_0xk"
      },
      "salience": 0.20135868,
      "mentions": [
        {
          "text": {
            "content": "芥末醬",
            "beginOffset": 36
          },
          "type": "PROPER"
        }
              ]
    },
    {
      "name": "奧巴馬",
      "type": "OTHER",
      "metadata": {
        "wikipedia_url": "https://en.wikipedia.org/wiki/Barack_Obama",
        "mid": "/m/02mjmr"
      },
      "salience": 0.14879385,
      "mentions": [
        {
          "text": {
            "content": "奧巴馬",
            "beginOffset": 24
          },
          "type": "PROPER"
        }
      ]
    },
    {
      "name": "好感",
      "type": "OTHER",
      "metadata": {},
      "salience": 0.14599532,
      "mentions": [
        {
          "text": {
            "content": "好感",
            "beginOffset": 12
          },
          "type": "COMMON"
        }
      ]
    },
        {
      "name": "第戎",
      "type": "LOCATION",
      "metadata": {
        "mid": "/m/0pbhz",
        "wikipedia_url": "https://en.wikipedia.org/wiki/Dijon"
      },
      "salience": 0.10672355,
      "mentions": [
        {
          "text": {
            "content": "第戎",
            "beginOffset": 51
          },
          "type": "PROPER"
        }
      ]
    }
  ],
  "language": "zh-Hant"
}
```
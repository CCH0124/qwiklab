---
layout: post
title: Machine Learning APIs - 04
date: 2020-09-03
excerpt: "Detect Labels, Faces, and Landmarks in Images with the Cloud Vision API"
tags: [qwiklab, ML, google]
comments: true
---

- Create an API Key
- Upload an Image to a Cloud Storage bucket
    - Edit Permissions

![](https://cdn.qwiklabs.com/V4PmEUI7yXdKpytLNRqwV%2ByGHqym%2BfhdktVi8nj4pPs%3D)

上面步驟跟前面系列文章的建立方式一樣，不再贅述。

![](https://i.imgur.com/0lbjDju.png)

## Create your Vision API request

在 `cloud shell` 中建立 `request.json` 檔案，內容如下

```json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/donuts.png" // my-bucket-name 換成建立的名稱
          }
        },
        "features": [
          {
            "type": "LABEL_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
```

## Label Detection

試用的第一個 `Cloud Vision API` 功能是*標籤檢測(label detection)*。此方法將返回圖像內容的標籤（單詞）列表。

```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

回應

```shell
{
  "responses": [
    {
      "labelAnnotations": [
        {
          "mid": "/m/01dk8s",
          "description": "Powdered sugar",
          "score": 0.9861496,
          "topicality": 0.9861496
        },
        {
          "mid": "/m/01wydv",
          "description": "Beignet",
          "score": 0.9565117,
          "topicality": 0.9565117
        },
        {
          "mid": "/m/02wbm",
          "description": "Food",
          "score": 0.9424965,
          "topicality": 0.9424965
        },
        {
          "mid": "/m/0hnyx",
          "description": "Pastry",
          "score": 0.8173416,
          "topicality": 0.8173416
        },
        {
          "mid": "/m/02q08p0",
          "description": "Dish",
          "score": 0.8076026,
          "topicality": 0.8076026
        },
        {
          "mid": "/m/01ykh",
          "description": "Cuisine",
          "score": 0.79036003,
          "topicality": 0.79036003
        },
        {
          "mid": "/m/03nsjgy",
          "description": "Kourabiedes",
          "score": 0.77726763,
          "topicality": 0.77726763
        },
        {
          "mid": "/m/06gd3r",
          "description": "Angel wings",
          "score": 0.73792106,
          "topicality": 0.73792106
        },
        {
          "mid": "/m/06x4c",
          "description": "Sugar",
          "score": 0.71921736,
          "topicality": 0.71921736
        },
        {
          "mid": "/m/01zl9v",
          "description": "Zeppole",
          "score": 0.7111677,
          "topicality": 0.7111677
        }
      ]
    }
  ]
}
```


`API` 能夠識別糖粉的特殊類型的甜甜圈。對於視覺 `API` 找到的每個標籤，它返回：

- 帶有商品名稱的*描述(description)*
- *分數(score)*，從 0 到 1 的數字表示說明與圖片中的內容相匹配的置信度
- *mid*，該值對應於 Google [知識圖表](https://www.google.com/intl/bn/insidesearch/features/search/knowledge.html) 中項目的 *mid*。可以在調用 [Knowledge Graph API](https://developers.google.com/knowledge-graph/) 時使用 *mid*，以獲取有關該項目的更多訊息

## Web Detection

除了獲得圖像上的標籤以外，`Vision API` 還可以在網際網路上搜索圖像的其它詳細訊息。透過 `API` 的 [webDetection](https://cloud.google.com/vision/docs/reference/rest/v1/images/annotate#WebDetection) 方法，可以獲得很多有趣的數據：

- 根據圖像相似頁面的內容，在圖像中找到的實體列表
- 在網絡上找到的完全匹配的圖像和部分匹配的圖像的 `URL`，以及這些頁面的 `URL`
- 相似圖片的網址，例如進行反向圖片搜索

使用相同的 `beignets` 圖像並在 `request.json` 中更改一行（也可以嘗試使用完全不同的圖像）。

1. 在`features`列表下，將類型從 `LABEL_DETECTION` 更改為 `WEB_DETECTION`。 `request.json` 應如下所示：

```json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/donuts.png"
          }
        },
        "features": [
          {
            "type": "WEB_DETECTION",
            "maxResults": 10
          }
        ]
      }
  ]
}
```

2. 要將其發送到 `Vision API`，使用與先前相同的 `curl`

```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```
從 `webEntities` 開始深入研究回應。以下是此圖像返回的一些實體：

```shell=
{
  "responses": [
    {
      "webDetection": {
        "webEntities": [
          {
            "entityId": "/m/01hyh_",
            "score": 0.8826,
            "description": "Machine learning"
          },
          {
            "entityId": "/m/01dk8s",
            "score": 0.51894134,
            "description": "Powdered sugar"
          },
          {
            "entityId": "/m/01wydv",
            "score": 0.5052478,
            "description": "Beignet"
          },
          {
            "entityId": "/g/11bwp1s2k3",
            "score": 0.4328,
            "description": "TensorFlow"
          },
          {
            "entityId": "/m/0z5n",
            "score": 0.4258,
            "description": "API"
          },
          {
            "entityId": "/m/01xzx",
            "score": 0.4076,
            "description": "Computer vision"
          },
          {
            "entityId": "/m/02y_9m3",
            "score": 0.376,
            "description": "Cloud computing"
          },
          {
            "entityId": "/m/0105pbj4",
            "score": 0.3277,
            "description": "Google Cloud Platform"
          },
          {
            "entityId": "/m/06j0j4",
            "score": 0.3006,
            "description": "OpenCV"
          },
          {
            "entityId": "/m/0h1fn8h",
            "score": 0.2965,
            "description": "Deep learning"
                      }
        ],
        "fullMatchingImages": [
          {
            "url": "https://codelabs.developers.google.com/codelabs/cloud-vision-intro-ja/img/b05d9534e547f5ee.png"
          },
          {
            "url": "https://miro.medium.com/max/3200/1*JSX2yNORAAOxdGoU17c4_A.png"
          },
          {
            "url": "https://storage.googleapis.com/aju-dev-demos-codelabs/images/vision_donuts_sm.jpeg"
          }
        ],
        "partialMatchingImages": [
          {
            "url": "https://i1.wp.com/www.fxfrog.com/wp-content/uploads/2020/05/donuts.jpg?resize=730%2C410&ssl=1"
          },
          {
            "url": "https://iq.opengenus.org/content/images/2019/11/imagetweet.PNG"
          },
          {
            "url": "https://photo.isu.pub/3845434/photo_large.jpg"
          }
        ],
        "pagesWithMatchingImages": [
          {
            "url": "https://developers.google.com/codelabs/cloud-vision-intro",
            "pageTitle": "Detect labels, faces, and landmarks in images with the Cloud Vision ...",
            "fullMatchingImages": [
              {
                "url": "https://developers.google.com/codelabs/cloud-vision-intro/img/b05d9534e547f5ee.jpeg"
              }
            ]
          },
          {
            "url": "https://google.dev/codelabs/cloud-vision-intro",
            "pageTitle": "Detect labels, faces, and landmarks in images with the Cloud Vision ...",
            "fullMatchingImages": [
              {
                "url": "https://google.dev/codelabs/cloud-vision-intro/img/b05d9534e547f5ee.jpeg"
              }
            ]
          },
          {
            "url": "https://codelabs.developers.google.com/codelabs/cloud-vision-intro/index.html?index=..%2F..cloudai",
            "pageTitle": "Detect labels, faces, and landmarks in images with the Cloud Vision ...",
            "fullMatchingImages": [
              {
                "url": "https://codelabs.developers.google.com/codelabs/cloud-vision-intro/img/b05d9534e547f5ee.jpeg"
              }
            ]
          },
          {
                     "url": "https://codelabs.developers.google.com/codelabs/cloud-vision-intro/index.html?index=..%2F..cloudai",
            "pageTitle": "Detect labels, faces, and landmarks in images with the Cloud Vision ...",
            "fullMatchingImages": [
              {
                "url": "https://codelabs.developers.google.com/codelabs/cloud-vision-intro/img/b05d9534e547f5ee.jpeg"
              }
            ]
          },
          {
            "url": "https://kiosk-dot-codelabs-site.appspot.com/codelabs/cloud-vision-intro/index.html?index=..%2F..index",
            "pageTitle": "Detect labels, faces, and landmarks in images with the Cloud Vision ...",
            "fullMatchingImages": [
              {
                "url": "https://kiosk-dot-codelabs-site.appspot.com/codelabs/cloud-vision-intro/img/b05d9534e547f5ee.jpeg"
              }
            ]
          },
          {
            "url": "https://github.com/GoogleCloudPlatform/cloud-shell-tutorials/pull/8/files",
            "pageTitle": "Text-to-speech API intro tutorial · Issue #8 · GoogleCloudPlatform ...",
            "fullMatchingImages": [
              {
                "url": "https://storage.googleapis.com/aju-dev-demos-codelabs/images/vision_donuts_sm.jpeg"
              }
            ]
          },
          {
            "url": "https://iq.opengenus.org/post-image-twitter-api/",
            "pageTitle": "Post a tweet with image using Twitter API - OpenGenus IQ",
            "partialMatchingImages": [
              {
                "url": "https://iq.opengenus.org/content/images/2019/11/imagetweet.PNG"
              }
            ]
          },
          {
            "url": "https://ai-create.net/magazine/2019/05/22/ml-study-jams-detect-labels-faces-landmarks-images-cloud-vision/",
            "pageTitle": "ML Study Jams 機械学習 中級者向けコース 「Detect Labels, Faces ...",
            "fullMatchingImages": [
              {
                "url": "https://cdn.qwiklabs.com/V4PmEUI7yXdKpytLNRqwV%2ByGHqym%2BfhdktVi8nj4pPs%3D"
              }
            ]
          },
          {
            "url": "https://issuu.com/taedp",
            "pageTitle": "TAEDP - Issuu",
            "partialMatchingImages": [
              {
                "url": "https://photo.isu.pub/3845434/photo_large.jpg"
              }
            ]
          },
          {
            "url": "https://issuu.com/3845434",
            "pageTitle": "張佑菖 - Issuu",
                        "partialMatchingImages": [
              {
                "url": "https://photo.isu.pub/3845434/photo_large.jpg"
              }
            ]
          },
          {
            "url": "https://medium.com/@kcchien/google-ml-study-jam-%E6%A9%9F%E5%99%A8%E5%AD%B8%E7%BF%92%E7%AD%86%E8%A8%98-02-%E4%BD%BF%E7%94%A8-cloud-vision-api-%E4%BE%86%E5%81%B5%E6%B8%AC%E5%9C%96%E7%89%87%E4%B8%AD%E7%9A%84%E6%A8%99%E7%B1%A4-%E8%87%89%E5%AD%94%E4%BB%A5%E5%8F%8A%E5%9C%B0%E6%A8%99-4316b30d62fe",
            "pageTitle": "Google ML Study Jam 機器學習筆記(02) — 使用Cloud Vision API 來 ...",
            "fullMatchingImages": [
              {
                "url": "https://miro.medium.com/max/3200/1*JSX2yNORAAOxdGoU17c4_A.png"
              }
            ]
          }
        ],
        "visuallySimilarImages": [
          {
            "url": "https://i.pinimg.com/600x315/16/2d/75/162d75cb0bff0952070cc24fe3dce88c.jpg"
          },
          {
            "url": "https://www.trendideas.net/wp-content/uploads/2018/07/4a8c89048c0381c4f9ffdfd0876fa400.jpeg"
          },
          {
            "url": "https://images.food52.com/bSO5OY3pUpk6obZx-uZYuiT4fCY=/965x643/66b63dd8-b48e-4504-af4f-4cf71f16d535--IMG_3593.jpeg"
          },
          {
            "url": "http://www.lisbetheats.com/wp-content/uploads/2014/07/beignets.jpg"
          },
          {
            "url": "https://i.pinimg.com/originals/77/84/dc/7784dcc40c26013a72329a4b65d8443f.jpg"
          },
          {
            "url": "https://developers.google.com/codelabs/cloud-vision-intro/img/487150a1f044cc04.jpeg"
          },
          {
            "url": "https://www.ticonsigliounposticino.it/wp-content/uploads/2015/03/IMG_0835.jpg"
          },
          {
            "url": "https://i1.wp.com/www.backpackingdiplomacy.com/wp-content/uploads/2013/01/lafayettebreakfast3.jpg?resize=360%2C270"
          }
        ],
        "bestGuessLabels": [
          {
            "label": "powdered sugar"
          }
        ]
      }
    }
  ]
}
```

此圖片已在 `Cloud ML API` 的許多場景中使用，這就是 `API` 為何找到實體 `Machine learning` 和 `Google Cloud Platform` 的原因。如果在 `fullMatchingImages`、`partialMatchingImages` 和 `pagesWithMatchingImages` 下檢查 `URL`，會注意到許多 `URL` 指向此實驗站點。

假設想查找其它關於該圖片訊息，但沒有找到完全相同的圖片。這就是 `API` 響應的`visuallySimilarImages` 部分派上用場的地方。這是它發現的一些視覺上相似的圖像：

```shell
        "visuallySimilarImages": [
          {
            "url": "https://media.istockphoto.com/photos/cafe-du-monde-picture-id1063530570?k=6&m=1063530570&s=612x612&w=0&h=b74EYAjlfxMw8G-G_6BW-6ltP9Y2UFQ3TjZopN-pigI="
          },
          {
            "url": "https://s3-media2.fl.yelpcdn.com/bphoto/oid0KchdCqlSqZzpznCEoA/o.jpg"
          },
          {
            "url": "https://s3-media1.fl.yelpcdn.com/bphoto/mgAhrlLFvXe0IkT5UMOUlw/348s.jpg"
          },

          ...
]
```

可以瀏覽這些 `URL` 以查看類似的圖像。

借助 `Cloud Vision`，可以使用易於使用的 `REST API` 來存取此功能，並將其整合到應用程式中。

## Face Detection

探索 `Vision API` 的臉部檢測(Face Detection)方法。

- 臉部檢測方法返回圖像中發現的臉部數據，包括臉部表情及其在圖像中的位置

### Upload a new image

要使用此方法，需要將帶有面孔的新圖像上傳到 `Cloud Storage` 儲存桶。

以下是實驗的圖像，須將其傳至 `Cloud Stirage`，並設定權限
![](https://cdn.qwiklabs.com/5%2FxwpTRxehGuIRhCz3exglbWOzueKIPikyYj0Rx82L0%3D)

### Updating request file

接下來，使用以下命令更新 `request.json` ，其中包括新圖像的 `URL`，並使用臉部和地標檢測(landmark detection)而不是標籤檢測。

```json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/selfie.png" // 名稱需要注意
          }
        },
        "features": [
          {
            "type": "FACE_DETECTION"
          },
          {
            "type": "LANDMARK_DETECTION"
          }
        ]
      }
  ]
}
```

## Calling the Vision API and parsing the response

```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```

看一下回應中的 `faceAnnotations` 對象。會注意到，`API` 為圖像中找到的每個面孔返回一個對象。在這種情況下為三個。

```shell
{
      "faceAnnotations": [
        {
          "boundingPoly": {
            "vertices": [
              {
                "x": 669,
                "y": 324
              },
              ...
            ]
          },
          "fdBoundingPoly": {
            ...
          },
          "landmarks": [
            {
              "type": "LEFT_EYE",
              "position": {
                "x": 692.05646,
                "y": 372.95868,
                "z": -0.00025268539
              }
            },
            ...
          ],
          "rollAngle": 0.21619819,
          "panAngle": -23.027969,
          "tiltAngle": -1.5531756,
          "detectionConfidence": 0.72354823,
          "landmarkingConfidence": 0.20047489,
          "joyLikelihood": "POSSIBLE",
          "sorrowLikelihood": "VERY_UNLIKELY",
          "angerLikelihood": "VERY_UNLIKELY",
          "surpriseLikelihood": "VERY_UNLIKELY",
          "underExposedLikelihood": "VERY_UNLIKELY",
          "blurredLikelihood": "VERY_UNLIKELY",
          "headwearLikelihood": "VERY_LIKELY"
        }
        ...
     }
}
```

- `boundingPoly` 提供圖像中人臉周圍的 `x`、`y` 坐標
- `fdBoundingPoly` 是一個比 `boundingPoly` 小的資源，專注於臉部的皮膚部分
- `landmarks` 是每個臉部特徵的對象陣列，有些甚至可能都不知道。這告訴我們地標的類型，以及該要素的 `3D` 位置（`x`，`y`，`z`坐標），其中`z`坐標是深度。剩餘的值可為提供更多臉部*表情*，包括*喜悅*、*悲傷*、*憤怒*和*驚奇*的可能性。


## Landmark Annotation

- 地標檢測(Landmark detection)可以識別常見或含糊地標。它返回地標的名稱，其緯度和經度坐標以及在圖像中識別出地標的位置。

### Upload a new image

將下面圖片傳送到 `Cloud Storage`，並設定權限

![](https://cdn.qwiklabs.com/%2Fv47QS0KOC28%2F03bZx0R%2FO0iLLvtYQUOZyvnjIfz%2BIE%3D)

### Updating request file

接下來，使用以下內容更新 `request.json` ，其中包括新圖像的 `URL`，並使用地標檢測(landmark detection)。

```json
{
  "requests": [
      {
        "image": {
          "source": {
              "gcsImageUri": "gs://my-bucket-name/city.png"
          }
        },
        "features": [
          {
            "type": "LANDMARK_DETECTION",
            "maxResults": 10,
          }
        ]
      }
  ]
}
```

## Calling the Vision API and parsing the response

```shell
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json  https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}
```
查看回應的 `landmarkAnnotations` 部分

```shell
      "landmarkAnnotations": [
        {
          "mid": "/m/041cp3",
          "description": "Boston",
          "score": 0.788803,
          "boundingPoly": {
            "vertices": [
              {
                "y": 576
              },
              {
                "x": 1942,
                "y": 576
              },
              {
                "x": 1942,
                "y": 1224
              },
              {
                "y": 1224
              }
            ]
          },

...

```

`Vision API` 能夠說明這張照片是在波士頓拍攝的，並提供了確切位置的地圖。此回應中的值應類似於上面的 `labelAnnotations` 回應。


## Explore other Vision API methods

已經查看了 `Vision API` 的 `label`、`face` 和 `landmark` 偵測方法，但是還沒有探索其他三種方法。深入研究[文檔](https://cloud.google.com/vision/reference/rest/v1/images/annotate#Feature)以了解其他三個：

- Logo detection
    - 識別常見徽標及其在圖像中的位置
- Safe search detection
    - 確定圖像是否包含顯式內容。這對於具有用戶生成內容的任何應用程序很有用。可以根據四個因素過濾圖像：成人、醫療、暴力和欺騙內容。
- Text detection
    - 運行 `OCR` 從圖像中提取文本。該方法甚至可以識別圖像中存在的文本的語言
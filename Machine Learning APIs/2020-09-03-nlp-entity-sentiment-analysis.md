---
layout: post
title: Machine Learning APIs - 05
date: 2020-09-03
excerpt: "Entity and Sentiment Analysis with the Natural Language API"
tags: [qwiklab, ML, google]
comments: true
---

## Make an Entity Analysis Request

這次使用的第一個自然語言 `API` 方法是 `analyticsEntities`。使用此方法，`API` 可以從文本中提取實體，如：人物、地點和事件。要使用 `API` 的實體分析，請使用以下語句：

```
Joanne Rowling, who writes under the pen names J. K. Rowling and Robert Galbraith, is a British novelist and screenwriter who wrote the Harry Potter fantasy series.
```

建立一個請求的 `request.json` 

```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Joanne Rowling, who writes under the pen names J. K. Rowling and Robert Galbraith, is a British novelist and screenwriter who wrote the Harry Potter fantasy series."
  },
  "encodingType":"UTF8"
}
```

`type` 支援 `PLAIN_TEXT` 或 `HTML`。在 `content` 中，傳遞文本以發送到 `Natural Language API` 進行分析，`Natural Language API` 還支援發送儲存在 `Cloud Storage` 中的檔案以進行文本處理，如果想從 `Cloud Storage` 發送檔案，則可以用 `gcsContentUri` 替換 `content `，並在 `Cloud Storage` 中為其提供文本檔案 `uri` 的值。`encodingType` 告訴 `API` 在處理文本時要使用哪種文本編碼類型。 `API` 將使用它來計算特定實體在文本中出現的位置。

## Call the Natural Language API

發送請求

```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json > result.json
```

察看結果

```shell
cat result.json
...
            "content": "Joanne Rowling",
            "beginOffset": 0
          },
          "type": "PROPER"
        },
        {
          "text": {
            "content": "Rowling",
            "beginOffset": 53
          },
          "type": "PROPER"
        },
        {
          "text": {
            "content": "novelist",
            "beginOffset": 96
          },
          "type": "COMMON"
        },
        {
          "text": {
            "content": "Robert Galbraith",
            "beginOffset": 65
          },
          "type": "PROPER"
        }
      ]
    },

    ...
  ]
}
```

這邊結果和此系列第二篇的內容可呼應

## Sentiment analysis with the Natural Language API

除了提取實體之外，`Natural Language API` 還允許對文本塊執行情感分析。該 `JSON` 請求將包含與上述請求相同的參數，但是這次將文本更改為包含一些具有更強情感的東西。

更改 `request.json` 中 `content`。

```json
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"Harry Potter is the best book. I think everyone should read it."
  },
  "encodingType": "UTF8"
}
```

將請求發送到 `API` 的 `analyticsSentiment` 端點

```shell
curl "https://language.googleapis.com/v1/documents:analyzeSentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

回應

```shell
{
  "documentSentiment": {
    "magnitude": 0.8,
    "score": 0.4
  },
  "language": "en",
  "sentences": [
    {
      "text": {
        "content": "Harry Potter is the best book.",
        "beginOffset": 0
      },
      "sentiment": {
        "magnitude": 0.7,
        "score": 0.7
      }
    },
    {
      "text": {
        "content": "I think everyone should read it.",
        "beginOffset": 31
      },
      "sentiment": {
        "magnitude": 0.1,
        "score": 0.1
      }
    }
  ]
}
```

結果會得到兩種類型的情感值：整個文檔的情感和按句子細分的情感。情感方法返回兩個值：

- score 
    - 是從`-1.0`到`1.0`的數字，指示語句的正面或負面
- magnitude 
    - 是一個介於 0 到無窮大之間的數字，代表語句中表達的情緒權重，無論是正數還是負數

帶有權重較大的語句的較長文本區塊具有較高的 `magnitude `。第一句的分數為正（0.7），第二句的分數為中性（0.1）。

使用中文

```shell
$ curl "https://language.googleapis.com/v1/documents:analyzeSentiment?key=${API_KEY}" \
>   -s -X POST -H "Content-Type: application/json" --data-binary @request.json
{
  "documentSentiment": {
    "magnitude": 1.9,
    "score": 0.9
  },
  "language": "zh-Hant",
  "sentences": [
    {
      "text": {
        "content": "每次都點海鮮河粉，湯頭非常的清爽好喝，常常來買。",
        "beginOffset": -1
      },
      "sentiment": {
        "magnitude": 0.9,
        "score": 0.9
      }
    },
    {
      "text": {
        "content": "還有很多熱炒的菜，以後再試試，適合一家人或票朋友一同點熱炒來吃。",
        "beginOffset": -1
      },
      "sentiment": {
        "magnitude": 0.9,
        "score": 0.9
      }
    }
  ]
}
```
## Analyzing entity sentiment
除了提供有關整個文本文檔的情感詳細信息之外，`Natural Language API` 還可以按文本中的實體分解情感。以這句話為例：

```
I liked the sushi but the service was terrible.
```

在這種情況下，像上面那樣獲得整個句子的情感分數可能沒有什麼用處。如果這是一家餐廳評論，並且同一家餐廳有數百條評論，那麼想確切知道人們在其評論中喜歡和不喜歡的東西。`Natural Language API` 具有一種方法，可獲取文本中每個實體的情感，即`analyticEntitySentiment`。

更改 `request.json`

```json
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"I liked the sushi but the service was terrible."
  },
  "encodingType": "UTF8"
}
```

請求 `analyzeEntitySentiment` 端點

```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntitySentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

回應

```json
{
  "entities": [
    {
      "name": "sushi",
      "type": "CONSUMER_GOOD",
      "metadata": {},
      "salience": 0.52716845,
      "mentions": [
        {
          "text": {
            "content": "sushi",
            "beginOffset": 12
          },
          "type": "COMMON",
          "sentiment": {
            "magnitude": 0.9,
            "score": 0.9
          }
        }
      ],
      "sentiment": {
        "magnitude": 0.9,
        "score": 0.9
      }
    },
    {
      "name": "service",
      "type": "OTHER",
      "metadata": {},
      "salience": 0.47283158,
      "mentions": [
        {
          "text": {
            "content": "service",
            "beginOffset": 26
          },
          "type": "COMMON",
          "sentiment": {
            "magnitude": 0.9,
            "score": -0.9
          }
        }
      ],
      "sentiment": {
        "magnitude": 0.9,
        "score": -0.9
      }
    }
  ],
  "language": "en"
}
```

可以看到，"sushi" 的得分為0.9，而 "service" 的得分為-0.9。且，每個實體都返回了兩個情感對象。如果多次提及這些術語中的任何一個，則 `API` 將針對每個提及返回不同的情感 `score` 和`magnitude`，以及該實體的總體情感。

中文好像不支援...

## Analyzing syntax and parts of speech

使用*語法分析(syntactic analysis)*，這是自然語言 `API` 的另一種方法，可以更深入研究文本的語言細節。`analyticsSyntax` 提取語言信息，將給定的文本分成一系列句子和標記，通常是單詞邊界(word boundaries)，以提供對這些標記的進一步分析。對於文本中的每個單詞，`API` 都會說出單詞的詞性，像是名詞、動詞或形容詞等。以及它與句子中其他單詞的關係，是 root verb？modifier？

編輯 `request.json` 檔案

```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content": "Joanne Rowling is a British novelist, screenwriter and film producer."
  },
  "encodingType": "UTF8"
}
```

請求 `analyzeSyntax` API

```shell
curl "https://language.googleapis.com/v1/documents:analyzeSyntax?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

回應

```json
{
      "text": {
        "content": "is",
        "beginOffset": 15
      },
      "partOfSpeech": {
        "tag": "VERB",
        "aspect": "ASPECT_UNKNOWN",
        "case": "CASE_UNKNOWN",
        "form": "FORM_UNKNOWN",
        "gender": "GENDER_UNKNOWN",
        "mood": "INDICATIVE",
        "number": "SINGULAR",
        "person": "THIRD",
        "proper": "PROPER_UNKNOWN",
        "reciprocity": "RECIPROCITY_UNKNOWN",
        "tense": "PRESENT",
        "voice": "VOICE_UNKNOWN"
      },
      "dependencyEdge": {
        "headTokenIndex": 2,
        "label": "ROOT"
      },
      "lemma": "be"
    },
```

- `partOfSpeech` 告訴我們"Joanne"是一個名詞
- `dependencyEdge` 包含可用於創建文本的[依賴關係分析樹](https://en.wikipedia.org/wiki/Parse_tree#Dependency-based_parse_trees)的數據。本質上，這是一個圖，顯示句子中的單詞如何相互關聯。上面句子的依賴關係分析樹如下所示：

![](https://cdn.qwiklabs.com/Xh7b1inigZEjOOGzkM2OtVQFK9PLn0SdY3wM9tdZLUU%3D)

- `headTokenIndex` 是 token 的索引，該令牌的弧線指向"Joanne"。將句子中的每個標記視為數組中的單詞。
- "Joanne"的 `headTokenIndex` 為 1 表示單詞 "Rowling"，它在樹中連接到該單詞。標籤 `NN`（short for noun compound modifier）描述單詞在句子中的作用。" Joanne"修改了句子的主題"Rowling"。
- `lemma` 是該詞的規範形式。例如，run、runs、ran 和 running 一詞都有運行的意思。`lemma` 值對於隨著時間的推移追蹤大文本中單詞的出現很有用

##### Other

以中文方式做嘗試。

```shell
$ cat request.json 
 {
  "document":{
    "type":"PLAIN_TEXT",
    "content":"我是一個玩CNCF的肥宅"
  }
}
 curl "https://language.googleapis.com/v1/documents:analyzeSyntax?key=${API_KEY}" \
>   -s -X POST -H "Content-Type: application/json" --data-binary @request.json
{
  "sentences": [
    {
      "text": {
        "content": "我是一個玩CNCF的肥宅",
        "beginOffset": -1
      }
    }
  ],
  "tokens": [
    {
      "text": {
        "content": "我",
        "beginOffset": -1
      },
      "partOfSpeech": {
        "tag": "PRON",
        "aspect": "ASPECT_UNKNOWN",
        "case": "CASE_UNKNOWN",
        "form": "FORM_UNKNOWN",
        "gender": "GENDER_UNKNOWN",
        "mood": "MOOD_UNKNOWN",
        "number": "NUMBER_UNKNOWN",
        "person": "FIRST",
        "proper": "NOT_PROPER",
        "reciprocity": "RECIPROCITY_UNKNOWN",
        "tense": "TENSE_UNKNOWN",
        "voice": "VOICE_UNKNOWN"
      },
      "dependencyEdge": {
        "headTokenIndex": 1,
        "label": "NSUBJ"
      },
      "lemma": "我"
    },
    {
          "text": {
        "content": "是",
        "beginOffset": -1
      },
      "partOfSpeech": {
        "tag": "VERB",
        "aspect": "ASPECT_UNKNOWN",
        "case": "CASE_UNKNOWN",
        "form": "FORM_UNKNOWN",
        "gender": "GENDER_UNKNOWN",
        "mood": "MOOD_UNKNOWN",
        "number": "NUMBER_UNKNOWN",
        "person": "PERSON_UNKNOWN",
        "proper": "NOT_PROPER",
        "reciprocity": "RECIPROCITY_UNKNOWN",
        "tense": "TENSE_UNKNOWN",
        "voice": "VOICE_UNKNOWN"
      },
      "dependencyEdge": {
        "headTokenIndex": 1,
        "label": "ROOT"
      },
      "lemma": "是"
    },
    {
      "text": {
        "content": "一個",
        "beginOffset": -1
      },
      "partOfSpeech": {
        "tag": "NOUN",
        "aspect": "ASPECT_UNKNOWN",
        "case": "CASE_UNKNOWN",
        "form": "FORM_UNKNOWN",
        "gender": "GENDER_UNKNOWN",
        "mood": "MOOD_UNKNOWN",
        "number": "NUMBER_UNKNOWN",
        "person": "PERSON_UNKNOWN",
        "proper": "NOT_PROPER",
        "reciprocity": "RECIPROCITY_UNKNOWN",
                "tense": "TENSE_UNKNOWN",
        "voice": "VOICE_UNKNOWN"
      },
      "dependencyEdge": {
        "headTokenIndex": 3,
        "label": "NSUBJ"
      },
      "lemma": "一個"
    },
    ...
```
##  Multilingual natural language processing

`Natural Language API` 支援[多個語言](https://cloud.google.com/natural-language/docs/languages)。修改 `request.json` 進行實驗

```json
{
  "document":{
    "type":"PLAIN_TEXT",
    "content":"日本のグーグルのオフィスは、東京の六本木ヒルズにあります"
  }
}
```
請求至 `analyzeEntities` API

```shell
curl "https://language.googleapis.com/v1/documents:analyzeEntities?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" --data-binary @request.json
```

回應

```json
{
  "entities": [
    {
      "name": "日本",
      "type": "LOCATION",
      "metadata": {
        "mid": "/m/03_3d",
        "wikipedia_url": "https://en.wikipedia.org/wiki/Japan"
      },
      "salience": 0.23854347,
      "mentions": [
        {
          "text": {
            "content": "日本",
            "beginOffset": 0
          },
          "type": "PROPER"
        }
      ]
    },
    {
      "name": "グーグル",
      "type": "ORGANIZATION",
      "metadata": {
        "mid": "/m/045c7b",
        "wikipedia_url": "https://en.wikipedia.org/wiki/Google"
      },
      "salience": 0.21155767,
      "mentions": [
        {
          "text": {
            "content": "グーグル",
            "beginOffset": 9
          },
          "type": "PROPER"
        }
      ]
    },
    ...
  ]
  "language": "ja"
}
```
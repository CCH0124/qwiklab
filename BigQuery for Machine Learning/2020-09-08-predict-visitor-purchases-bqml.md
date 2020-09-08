---
layout: post
title: BigQuery for Machine Learning-02
date: 2020-09-08
excerpt: "Predict Visitor Purchases with a Classification Model in BQML"
tags: [qwiklab, ML, google, BigQuery]
comments: true
---

`BigQuery` 是 `Google` 完全管理的 `NoOps`、低成本分析數據庫。藉由 `BigQuery`，可以查詢 `TB` 或 `TB` 以上的數據，且無需管理任何基礎架構或資料庫管理員，`BigQuery` 可專注於分析數據以找到有意義的見解。`BigQuery Machine Learning`(BQML) 是 `BigQuery` 的一項新功能，數據分析員可以使用最少的代碼完成機器學習模型的創建、訓練，評估和預測。

本實驗與 `BigQuery for Machine Learning-01` 的數據集是相同的。




### Access the course dataset

至 `BigQuery` 面板後，點擊  `data-to-insights`，資料集 `data-to-insights` 的字段定義在[此處](https://support.google.com/analytics/answer/3437719?hl=en)，此鏈接可在新頁面中保持開啟以供參考。

![](https://cdn.qwiklabs.com/wAOpgsMVrpanjH%2F8OqKMFHWaIUfTMjG6C1O8HBYRlzM%3D)

## Explore ecommerce data

Scenario：你的數據分析團隊將電子商務網站的 `Google Analytics` 日誌導出到 `BigQuery` 中，並創建了一個包含所有原始電子商務訪問者連線數據的新表，以供瀏覽。

Question：在訪問我們網站的總訪客中，有百分之幾進行了購買動作？


在 `BigQuery`，執行
```sql
#standardSQL
WITH visitors AS(
SELECT
COUNT(DISTINCT fullVisitorId) AS total_visitors
FROM `data-to-insights.ecommerce.web_analytics`
),

purchasers AS(
SELECT
COUNT(DISTINCT fullVisitorId) AS total_purchasers
FROM `data-to-insights.ecommerce.web_analytics`
WHERE totals.transactions IS NOT NULL
)

SELECT
  total_visitors,
  total_purchasers,
  total_purchasers / total_visitors AS conversion_rate
FROM visitors, purchasers
```

運行，結果為 2.6984540008979117%



Question：銷量最高的 5 種產品是什麼？
```sql
SELECT
  p.v2ProductName,
  p.v2ProductCategory,
  SUM(p.productQuantity) AS units_sold,
  ROUND(SUM(p.localProductRevenue/1000000),2) AS revenue
FROM `data-to-insights.ecommerce.web_analytics`,
UNNEST(hits) AS h,
UNNEST(h.product) AS p
GROUP BY 1, 2
ORDER BY revenue DESC
LIMIT 5;
```

結果
![](https://i.imgur.com/5nEDaDC.png)

Question：在之後訪問該網站時有多少訪問者購買了？

```sql
# visitors who bought on a return visit (could have bought on first as well
WITH all_visitor_stats AS (
SELECT
  fullvisitorid, # 741,721 unique visitors
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid
)

SELECT
  COUNT(DISTINCT fullvisitorid) AS total_visitors,
  will_buy_on_return_visit
FROM all_visitor_stats
GROUP BY will_buy_on_return_visit
```

![](https://i.imgur.com/keOrjnU.png)

從結果可以看到`(11873/729848)=1.6%` 的總訪問者將返回該網站購買。其中包括訪問者的子集，這些訪問者在第一次存取網站就購買了，然後又回來再次購買。


Question：典型的電子商務客戶會瀏覽，則不瀏覽直到下次訪問才購買的原因有哪些？

Answer：這沒有一個正確的答案，但一個普遍的原因是，在最終做出購買決定之前，在不同的電子商務網站之間進行比較購物。對於奢侈品這很常見，因為在決定考慮購買汽車之前，客戶需要進行大量的研究和比較，但對於此站點上的商品，如：T-shirts、配飾等，這種要求在較便宜的單價也是如此。

在線上營銷的世界中，根據首次訪問者的特徵來識別和營銷對這些未來的客戶將提高轉變並減少外流到競爭對手網站。


## Identify an objective
現在，將在 `BigQuery` 中創建一個機器學習模型，以預測新使用者未來是否有購買可能。確定這些高價值用戶可以幫助營銷團隊針對他們進行特殊促銷和廣告活動，以確保他們在比較電子商務網站之間的購物時產生變化。


## Select features and create your training dataset
`Google Analytics` 捕獲了有關此電子商務網站上使用者訪問的各種維度和度量。[這邊](https://support.google.com/analytics/answer/3437719?hl=en)可瀏覽完整的字段列表，接著預覽實驗數據集以找到有用的特徵，這些特徵將幫助機器學習模型了解有關訪問者首次訪問網站的數據與它們是否會返回並進行購買之間的關係。

團隊決定測試以下兩個字段是否是分類模型的良好輸入
- totals.bounces
    - 訪客是否立即離開網站
- totals.timeOnSite
    - 訪客在網站上停留了多長時間

Question：僅使用以上兩個字段會有什麼風險？

Answer：機器學習會與輸入的訓練數據一樣好。如果沒有足夠的訊息提供模型確定和了解輸入特徵和標籤之間的關係。在這種情況下，則說明訪客將來是否購買，那麼將沒有一個準確的模型。

```sql
SELECT
  * EXCEPT(fullVisitorId)
FROM

  # features
  (SELECT
    fullVisitorId,
    IFNULL(totals.bounces, 0) AS bounces,
    IFNULL(totals.timeOnSite, 0) AS time_on_site
  FROM
    `data-to-insights.ecommerce.web_analytics`
  WHERE
    totals.newVisits = 1)
  JOIN
  (SELECT
    fullvisitorid,
    IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM
      `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid)
  USING (fullVisitorId)
ORDER BY time_on_site DESC
LIMIT 10;
```

![](https://i.imgur.com/11VPzYg.png)


Question：輸入特徵和標籤是哪些字段？
Answer：首次訪問後，`will_buy_on_return_visit` 標籤是不知道。同樣，將預測回傳到網站並購買了用戶的子集。由於在預測時不了解未來，因此無法確定說新訪客是否會回來購買。建立 `ML` 模型的價值在於根據有關他們的第一次訪問的數據來取得未來回購的可能性。

Question：查看初始數據結果，認為 `time_on_site` 和 `bounces` 是否可以很好指示用戶是否會返回並購買？
Answer：在訓練和評估模型之前先說出來還為時過早，但是乍看在前 10 個 `time_on_site` 中，只有 1 個客戶返回購買，這並不是很理想


## Create a BigQuery dataset to store models

建立一個新的 `BigQuery` 數據集，該數據集將儲存 `ML` 模型。

1. 點擊專案名稱，之後再點擊 `Create Dataset`
![](https://cdn.qwiklabs.com/QZ8JT8gSgMzUxN%2B7ZxxLmlnMDDAc8WARu5H8m2JhSbQ%3D)

2. 在彈跳視窗中輸入
- `Dataset ID`，`ecommerce`
- 其它使用預設

![](https://cdn.qwiklabs.com/iLtK62bwChP7Z9ECizLQ1y%2FSe4QIld%2FKQKekZ69V%2FLg%3D)

3. 最後 `Create dataset`

## Select a BQML model type and specify options
前面已經選擇了初始特徵，那麼現在就可在 `BigQuery` 中創建第一個 `ML` 模型。

有兩種模型類型可供選擇：

|Model|Model Type|Label Data type|Example|
|---|---|---|---|
|Forecasting|linear_reg|Numeric value (typically an integer or floating point)|Forecast sales figures for next year given historical sales data.|
|Classification|logistic_reg|0 or 1 for binary classification|Classify an email as spam or not spam given the context.|


>機器學習中還使用了許多其他模型類型，如：神經網路和決策樹等，並可使用 `TensorFlow` 等使用。

### Which model type should you choose?
由於正在將訪客分類為*將來會購買*或*將來不會購買*，因此請在分類模型中使用 `logistic_reg`。

以下查詢將創建模型並指定模型選項，當運行此查詢時會觸發訓練模型
```sql
CREATE OR REPLACE MODEL `ecommerce.classification_model`
OPTIONS
(
model_type='logistic_reg',
labels = ['will_buy_on_return_visit']
)
AS

#standardSQL
SELECT
  * EXCEPT(fullVisitorId)
FROM

  # features
  (SELECT
    fullVisitorId,
    IFNULL(totals.bounces, 0) AS bounces,
    IFNULL(totals.timeOnSite, 0) AS time_on_site
  FROM
    `data-to-insights.ecommerce.web_analytics`
  WHERE
    totals.newVisits = 1
    AND date BETWEEN '20160801' AND '20170430') # train on first 9 months
  JOIN
  (SELECT
    fullvisitorid,
    IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM
      `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid)
  USING (fullVisitorId)
;
```

當執行完後，會出現 `This statement created a new model named qwiklabs-gcp-xxxxxxxxx:ecommerce.classification_model` 的訊息，也點擊 `Go to model`。

查看電子商務數據集內部，並確認現在出現了 `classification_model`。接下來要對模型進行評估。

![](https://cdn.qwiklabs.com/4J4Zcu1NA70NBc07evKZsQ4xw9skVdQt90DiTgpviHk%3D)


## Evaluate classification model performance
### Select your performance criteria
對於 `ML` 中的分類問題，希望最小化`False Positive Rate`，並最大化`True Positive Rate`。這種關係透過 `ROC`(Receiver Operating Characteristic) 可視化，如此處所示，可以在其中嘗試讓曲線或 `AUC` 下的面積最大化：

![](https://cdn.qwiklabs.com/GNW5Bw%2B8bviep9OK201QGPzaAEnKKyoIkDChUHeVdFw%3D)

在 `BQML` 中，`roc_auc` 只是評估訓練 `ML` 模型時的一個可查詢字段。透過前面訓練完之後，運行以下查詢以評估模型的效果如何，這部分使用 `ML.EVALUATE` 關鍵字

```sql
SELECT
  roc_auc,
  CASE
    WHEN roc_auc > .9 THEN 'good'
    WHEN roc_auc > .8 THEN 'fair'
    WHEN roc_auc > .7 THEN 'decent'
    WHEN roc_auc > .6 THEN 'not great'
  ELSE 'poor' END AS model_quality
FROM
  ML.EVALUATE(MODEL ecommerce.classification_model,  (

SELECT
  * EXCEPT(fullVisitorId)
FROM

  # features
  (SELECT
    fullVisitorId,
    IFNULL(totals.bounces, 0) AS bounces,
    IFNULL(totals.timeOnSite, 0) AS time_on_site
  FROM
    `data-to-insights.ecommerce.web_analytics`
  WHERE
    totals.newVisits = 1
    AND date BETWEEN '20170501' AND '20170630') # eval on 2 months
  JOIN
  (SELECT
    fullvisitorid,
    IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM
      `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid)
  USING (fullVisitorId)

));
```

結果

|Row|roc_auc|model_quality|
|---|---|---|
|1|0.7238601398601399|decent|

評估模型後，`roc_auc` 為 0.72，這模型具有不錯的預測能力，但預測能力效果卻不高。由於目標是使曲線下的面積盡可能接近 1.0，因此存在改進的空間。


## Improve model performance with Feature Engineering
如前所述，數據集中還有許多特徵可幫助模型更好了解訪客的首次訪問與他們下次訪問時購買的可能性之間的關係。添加一些新特徵，並建立第二個機器學習模型 `category_model_2`。

- How far the visitor got in the checkout process on their first visit
- Where the visitor came from (traffic source: organic search, referring site etc..)
- Device category (mobile, tablet, desktop)
- Geographic information (country)


```sql
CREATE OR REPLACE MODEL `ecommerce.classification_model_2`
OPTIONS
  (model_type='logistic_reg', labels = ['will_buy_on_return_visit']) AS

WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid
)

# add in new features
SELECT * EXCEPT(unique_session_id) FROM (

  SELECT
      CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,

      # labels
      will_buy_on_return_visit,

      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,

      # behavior on the site
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      IFNULL(totals.pageviews, 0) AS pageviews,

      # where the visitor came from
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,

      # mobile or desktop
      device.deviceCategory,

      # geographic
      IFNULL(geoNetwork.country, "") AS country

  FROM `data-to-insights.ecommerce.web_analytics`,
     UNNEST(hits) AS h

    JOIN all_visitor_stats USING(fullvisitorid)

  WHERE 1=1
    # only predict for new visits
    AND totals.newVisits = 1
    AND date BETWEEN '20160801' AND '20170430' # train 9 months

  GROUP BY
  unique_session_id,
  will_buy_on_return_visit,
  bounces,
  time_on_site,
  totals.pageviews,
  trafficSource.source,
  trafficSource.medium,
  channelGrouping,
  device.deviceCategory,
  country
);
```

訓練數據集查詢中添加的一項關鍵新特徵是每個訪客在其訪問中達到的最大結帳進度，該進度記錄在 `hits.eCommerceAction.action_type` 字段中。

>Web 分析數據集具有嵌套和重複的字段，例如 `ARRAYS`，需要將其拆分為數據集中的其中一行，可透過使用 `UNNEST()` 函數來完成，可以在上面的查詢中看到該函數。


評估模型
```sql
#standardSQL
SELECT
  roc_auc,
  CASE
    WHEN roc_auc > .9 THEN 'good'
    WHEN roc_auc > .8 THEN 'fair'
    WHEN roc_auc > .7 THEN 'decent'
    WHEN roc_auc > .6 THEN 'not great'
  ELSE 'poor' END AS model_quality
FROM
  ML.EVALUATE(MODEL ecommerce.classification_model_2,  (

WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid
)

# add in new features
SELECT * EXCEPT(unique_session_id) FROM (

  SELECT
      CONCAT(fullvisitorid, CAST(visitId AS STRING)) AS unique_session_id,

      # labels
      will_buy_on_return_visit,

      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,

      # behavior on the site
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      totals.pageviews,

      # where the visitor came from
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,

      # mobile or desktop
      device.deviceCategory,

      # geographic
      IFNULL(geoNetwork.country, "") AS country

  FROM `data-to-insights.ecommerce.web_analytics`,
     UNNEST(hits) AS h

    JOIN all_visitor_stats USING(fullvisitorid)

  WHERE 1=1
    # only predict for new visits
    AND totals.newVisits = 1
    AND date BETWEEN '20170501' AND '20170630' # eval 2 months

  GROUP BY
  unique_session_id,
  will_buy_on_return_visit,
  bounces,
  time_on_site,
  totals.pageviews,
  trafficSource.source,
  trafficSource.medium,
  channelGrouping,
  device.deviceCategory,
  country
)
));
```
|Row|roc_auc|model_quality|
|---|---|---|
|1|0.9094905094905095|good|

## Predict which new visitors will come back and purchase

編寫查詢以預測哪些新訪客會回來並進行購買。下面的預測查詢使用改進的分類模型來預測首次訪問 `Google` 商品商店的訪客在以後的訪問中進行購買的可能性
```sql
SELECT
*
FROM
  ml.PREDICT(MODEL `ecommerce.classification_model_2`,
   (

WITH all_visitor_stats AS (
SELECT
  fullvisitorid,
  IF(COUNTIF(totals.transactions > 0 AND totals.newVisits IS NULL) > 0, 1, 0) AS will_buy_on_return_visit
  FROM `data-to-insights.ecommerce.web_analytics`
  GROUP BY fullvisitorid
)

  SELECT
      CONCAT(fullvisitorid, '-',CAST(visitId AS STRING)) AS unique_session_id,

      # labels
      will_buy_on_return_visit,

      MAX(CAST(h.eCommerceAction.action_type AS INT64)) AS latest_ecommerce_progress,

      # behavior on the site
      IFNULL(totals.bounces, 0) AS bounces,
      IFNULL(totals.timeOnSite, 0) AS time_on_site,
      totals.pageviews,

      # where the visitor came from
      trafficSource.source,
      trafficSource.medium,
      channelGrouping,

      # mobile or desktop
      device.deviceCategory,

      # geographic
      IFNULL(geoNetwork.country, "") AS country

  FROM `data-to-insights.ecommerce.web_analytics`,
     UNNEST(hits) AS h

    JOIN all_visitor_stats USING(fullvisitorid)

  WHERE
    # only predict for new visits
    totals.newVisits = 1
    AND date BETWEEN '20170701' AND '20170801' # test 1 month

  GROUP BY
  unique_session_id,
  will_buy_on_return_visit,
  bounces,
  time_on_site,
  totals.pageviews,
  trafficSource.source,
  trafficSource.medium,
  channelGrouping,
  device.deviceCategory,
  country
)

)

ORDER BY
  predicted_will_buy_on_return_visit DESC;
```

模型將輸出對 2017 年 7 月那些電子商務訪問的預測，可以看到三個新添加的字段：

- predicted_will_buy_on_return_visit: whether the model thinks the visitor will buy later (1 = yes)
- predicted_will_buy_on_return_visit_probs.label: the binary classifier for yes / no
- predicted_will_buy_on_return_visit.prob: the confidence the model has in it's prediction (1 = 100%)

下圖針對結果指截取部分

![](https://i.imgur.com/ZbaX7OU.png)


## Additional information
Tip：如果要在現有模型上重新訓練新數據以加快訓練時間，請在模型選項中添加 `warm_start = true`。其不能更改特徵列，更改表示將訓練一個新模型。

`roc_auc` 只是模型評估期間可用的性能指標之一。還可以提供 `accuracy`、`precision` 和 `recall`。知道要依賴哪個評估指標高度取決於總體目標。

下圖使用第二個模型的評估 `SQL` 語法中進行設置。修改如下

```sql
SELECT
  recall,	precision
FROM
  ML.EVALUATE(MODEL ecommerce.classification_model_2
  ...
```
![](https://i.imgur.com/rSmV0aV.png)
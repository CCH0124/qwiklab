---
layout: post
title: BigQuery for Machine Learning-03
date: 2020-09-08
excerpt: "Predict Taxi Fare with a BigQuery ML Forecasting Model"
tags: [qwiklab, ML, google, BigQuery]
comments: true
---

此實驗將在 `BigQuery Public Dataset` 中探索數以百萬計的紐約市黃色出租車行車路線。將在 `BigQuery` 中創建一個機器學習模型，以根據模型輸入來預測出租車的票價，當中也會評估模型的性能並進行預測。



## Explore NYC taxi cab data
Question：在 2015 年，黃色出租車每月旅遊幾次？

```sql
#standardSQL
SELECT
  TIMESTAMP_TRUNC(pickup_datetime,
    MONTH) month,
  COUNT(*) trips
FROM
  `bigquery-public-data.new_york.tlc_yellow_trips_2015`
GROUP BY
  1
ORDER BY
  1
```

![](https://cdn.qwiklabs.com/i3PmHY7jPV1XuAIql%2FDlkUHWFWuPJLcW1VECFP9P%2BuI%3D)

如結果所示，2015 年每個月紐約出租車都有超過 1000 萬的行程

Question：2015 年黃色出租車的平均行駛速度是多少？

```sql
#standardSQL
SELECT
  EXTRACT(HOUR
  FROM
    pickup_datetime) hour,
  ROUND(AVG(trip_distance / TIMESTAMP_DIFF(dropoff_datetime,
        pickup_datetime,
        SECOND))*3600, 1) speed
FROM
  `bigquery-public-data.new_york.tlc_yellow_trips_2015`
WHERE
  trip_distance > 0
  AND fare_amount/trip_distance BETWEEN 2
  AND 10
  AND dropoff_datetime > pickup_datetime
GROUP BY
  1
ORDER BY
  1
```

![](https://cdn.qwiklabs.com/%2BF8Mj8HqPM9b8%2Fna1JYPy66xTKDcUu%2BQs1oh5Gy07A4%3D)

結果來看，白天平均速度約為 11-12 MPH，在 5:00 AM，平均速度幾乎翻了一倍，達到 21 MPH。從直覺上講，因為在凌晨 5:00，道路上的交通流量可能會減少。



## Identify an objective
現在，將在 `BigQuery` 中創建一個機器學習模型，以根據 `trip` 的歷史數據集來預測紐約市出租車的價格。在乘車之前預測票價對於騎和出租車公司的 `trip` 計劃可能非常有用。

## Select features and create your training dataset
紐約市 `Yellow Cab` 數據集是該城市提供的[公開數據集](https://cloud.google.com/bigquery/public-data/nyc-tlc-trips)，已加載到 `BigQuery` 中以供探索。[此處](https://bigquery.cloud.google.com/table/nyc-tlc:yellow.trips)可瀏覽完整的字段列表，然後[預覽數據集](https://bigquery.cloud.google.com/table/nyc-tlc:yellow.trips?tab=preview)以找到有用的特徵，這些特徵將幫助機器學習模型了解有關歷史出租車的數據和票價之間的關係。

團隊決定測試以下這些字段是否是票價預測模型的良好輸入

- Tolls Amount
- Fare Amount
- Hour of Day
- Pick up address
- Drop off address
- Number of passengers

```sql
#standardSQL
WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
  )

  SELECT *
  FROM taxitrips
```

1. 查詢的主要部分在底部，`SELECT * from taxitrips`
2. `taxitrips` 會為 `NYC` 數據集進行大部分提取，而 `SELECT` 包含訓練特徵和標籤
3. `WHERE` 會過濾不想繼續訓練的數據
4. `WHERE` 還包括一個採樣語句，僅可提取 `1/1000` 的數據
5. 定義一個名為 `TRAIN` 的變數，以便可以快速構建一個獨立的 `EVAL` 驗證集

運行結果如下

![](https://i.imgur.com/8qkFQWp.png)

`total_fare` 是標籤是要預測的內容。我們使用 `tolls_amount` 和 `fare_amount` 創建此字段的，因此可以忽略客戶提示，因為它們是可自行決定的，這屬於模型的一部分。

## Create a BigQuery dataset to store models
使用 `BigQuery` 建立 `ML` 模型。

1. 點擊專案名稱，之後再點擊 `Create Dataset`
2. 在彈跳視窗中輸入
- `Dataset ID`，`taxi`
- 其它使用預設

![](https://cdn.qwiklabs.com/QGOFCQMb3UNnOf2dByXcmcH7%2BX6xwnoQFX0Fdo7fRLU%3D)

3. 最後 `Create dataset`


## Select a BQML model type and specify options
前面已經選擇了初始特徵，現在就可在 `BigQuery` 中創建第一個 `ML` 模型了。

有幾種模型類型可供選擇：
- **Forecasting** numeric values like next month's sales with Linear Regression (linear_reg).
- Binary or Multiclass **Classification** like spam or not spam email by using Logistic Regression (logistic_reg).
- k-Means **Clustering** for when you want unsupervised learning for exploration (kmeans).

透過以下查詢建立模型和定義選項
```sql
CREATE or REPLACE MODEL taxi.taxifare_model
OPTIONS
  (model_type='linear_reg', labels=['total_fare']) AS

WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
  )

  SELECT *
  FROM taxitrips
```

查看數據集，並確認現在出現了 `taxifare_model`

![](https://i.imgur.com/ubA660r.png)

## Evaluate classification model performance
### Select your performance criteria
對於線性回歸模型，使用像是*均方根誤差(RMSE)*之類的度量，其訓練結果要具有最低的 `RMSE`。在 `BQML` 中，評估 `ML` 模型時，`mean_squared_error` 是一個可查詢的字段，添加一個 `SQRT()` 以實現 `RMSE`。

現在訓練已經完成，可以使用 `ML.EVALUATE` 評估模型在此查詢中的性能

```sql
#standardSQL
SELECT
  SQRT(mean_squared_error) AS rmse
FROM
  ML.EVALUATE(MODEL taxi.taxifare_model,
  (

  WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.EVAL
  )

  SELECT *
  FROM taxitrips

  ))
```

現在可使用 `params.EVAL` 過濾器，針對不同的出租車 `trip` 對模型進行評估。模型結果如下

|Row|rmse|
|---|---|
|1|9.476637019770257|

評估模型後，`RMSE` 為 `9.47`。我們採用 `RMSE`，因此可以使用與 `total_fare` 相同的單位來估算 `9.47` 錯誤，因此它的誤差為 `+-$ 9.47`。

## Predict taxi fare amount
預測
```sql
#standardSQL
SELECT
*
FROM
  ml.PREDICT(MODEL `taxi.taxifare_model`,
   (

 WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    pickup_longitude AS pickuplon,
    pickup_latitude AS pickuplat,
    dropoff_longitude AS dropofflon,
    dropoff_latitude AS dropofflat,
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
  WHERE
    trip_distance > 0 AND fare_amount > 0
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.EVAL
  )

  SELECT *
  FROM taxitrips

));

```

結果將看到模型對出租車費率的預測以及這些出租車的實際費率和其他功能。結果如下

![](https://i.imgur.com/9BQ6TI2.png)


## Improving the model with Feature Engineering
建立機器學習模型是一個反覆的過程。評估了初始模型的性能後，我們通常會回過頭來修正特徵和欄位，以查看是否可以獲得更好的模型。

### Filtering the training dataset

查看出租車的常用票價統計信息
```sql
SELECT
  COUNT(fare_amount) AS num_fares,
  MIN(fare_amount) AS low_fare,
  MAX(fare_amount) AS high_fare,
  AVG(fare_amount) AS avg_fare,
  STDDEV(fare_amount) AS stddev
FROM
`nyc-tlc.yellow.trips`
# 1,108,779,463 fares
```

結果如下

![](https://i.imgur.com/NehhDOY.png)


結果上，數據集中存在一些奇怪的異常值，如負價或超過 $50,000 的價格。我們將數據限制為僅在 6 美元到 200 美元之間的票價。
```sql
SELECT
  COUNT(fare_amount) AS num_fares,
  MIN(fare_amount) AS low_fare,
  MAX(fare_amount) AS high_fare,
  AVG(fare_amount) AS avg_fare,
  STDDEV(fare_amount) AS stddev
FROM
`nyc-tlc.yellow.trips`
WHERE trip_distance > 0 AND fare_amount BETWEEN 6 and 200
# 843,834,902 fares
```

結果如下

![](https://i.imgur.com/wwm7FM5.png)

結果好一點了，我們限制行進的距離，讓其專注於紐約市。

```sql
SELECT
  COUNT(fare_amount) AS num_fares,
  MIN(fare_amount) AS low_fare,
  MAX(fare_amount) AS high_fare,
  AVG(fare_amount) AS avg_fare,
  STDDEV(fare_amount) AS stddev
FROM
`nyc-tlc.yellow.trips`
WHERE trip_distance > 0 AND fare_amount BETWEEN 6 and 200
    AND pickup_longitude > -75 #limiting of the distance the taxis travel out
    AND pickup_longitude < -73
    AND dropoff_longitude > -75
    AND dropoff_longitude < -73
    AND pickup_latitude > 40
    AND pickup_latitude < 42
    AND dropoff_latitude > 40
    AND dropoff_latitude < 42
    # 827,365,869 fares
```

結果如下

![](https://i.imgur.com/ZqNKC8C.png)

仍然擁有龐大的訓練數據集，其中有超過 8 億次乘車可供我們的新模型學習。我們使用這些新過濾的數據重新訓練模型，並查看其性能如何。

### Retraining the model
我們將新模型稱為 `taxi.taxifare_model_2` 重新訓練線性回歸模型以預測總票價。會注意到，還為`pick up`和 `drop off`之間的歐幾里得距離（直線）添加了一些計算出的功能。

```sql
CREATE OR REPLACE MODEL taxi.taxifare_model_2
OPTIONS
  (model_type='linear_reg', labels=['total_fare']) AS


WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    SQRT(POW((pickup_longitude - dropoff_longitude),2) + POW(( pickup_latitude - dropoff_latitude), 2)) as dist, #Euclidean distance between pickup and drop off
    SQRT(POW((pickup_longitude - dropoff_longitude),2)) as longitude, #Euclidean distance between pickup and drop off in longitude
    SQRT(POW((pickup_latitude - dropoff_latitude), 2)) as latitude, #Euclidean distance between pickup and drop off in latitude
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
WHERE trip_distance > 0 AND fare_amount BETWEEN 6 and 200
    AND pickup_longitude > -75 #limiting of the distance the taxis travel out
    AND pickup_longitude < -73
    AND dropoff_longitude > -75
    AND dropoff_longitude < -73
    AND pickup_latitude > 40
    AND pickup_latitude < 42
    AND dropoff_latitude > 40
    AND dropoff_latitude < 42
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.TRAIN
  )

  SELECT *
  FROM taxitrips
```

完成後有類似下圖的訊息

![](https://cdn.qwiklabs.com/WsL7xhxMxQFpf5HkKpC0P3bFAlLxeStYzA3uLFCPa6E%3D)

### Evaluate the new model
```sql
SELECT
  SQRT(mean_squared_error) AS rmse
FROM
  ML.EVALUATE(MODEL taxi.taxifare_model_2,
  (

  WITH params AS (
    SELECT
    1 AS TRAIN,
    2 AS EVAL
    ),

  daynames AS
    (SELECT ['Sun', 'Mon', 'Tues', 'Wed', 'Thurs', 'Fri', 'Sat'] AS daysofweek),

  taxitrips AS (
  SELECT
    (tolls_amount + fare_amount) AS total_fare,
    daysofweek[ORDINAL(EXTRACT(DAYOFWEEK FROM pickup_datetime))] AS dayofweek,
    EXTRACT(HOUR FROM pickup_datetime) AS hourofday,
    SQRT(POW((pickup_longitude - dropoff_longitude),2) + POW(( pickup_latitude - dropoff_latitude), 2)) as dist, #Euclidean distance between pickup and drop off
    SQRT(POW((pickup_longitude - dropoff_longitude),2)) as longitude, #Euclidean distance between pickup and drop off in longitude
    SQRT(POW((pickup_latitude - dropoff_latitude), 2)) as latitude, #Euclidean distance between pickup and drop off in latitude
    passenger_count AS passengers
  FROM
    `nyc-tlc.yellow.trips`, daynames, params
WHERE trip_distance > 0 AND fare_amount BETWEEN 6 and 200
    AND pickup_longitude > -75 #limiting of the distance the taxis travel out
    AND pickup_longitude < -73
    AND dropoff_longitude > -75
    AND dropoff_longitude < -73
    AND pickup_latitude > 40
    AND pickup_latitude < 42
    AND dropoff_latitude > 40
    AND dropoff_latitude < 42
    AND MOD(ABS(FARM_FINGERPRINT(CAST(pickup_datetime AS STRING))),1000) = params.EVAL
  )

  SELECT *
  FROM taxitrips

  ))
```

結果

|Row|rmse|
|---|---|
|1|5.1246527777688025|

現在 `RMSE` 降至 `+-$ 5.12`，明顯優於第一個模型的`+-$ 9.47`。
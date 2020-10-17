在此實驗室中，您擁有紐約市出租車隊，並希望實時監控您的業務狀況。將建立一個流數據管道來捕獲出租車收入，乘客人數，乘車狀態等，並在管理儀表板上可視化結果。

## Task 1. Create a Pub/Sub topic and BigQuery dataset
[Pub/Sub](https://cloud.google.com/pubsub/)(發布/訂閱) 是異步的全局消息服務。透過解偶發送者和接收者，它允許獨立編寫的應用程序之間進行安全且高度可用的通信。發布/訂閱(Pub/Sub )提供低延遲、持久的消息傳遞。

在發布/訂閱中，發布者應用程式和訂閱者應用程式透過使用稱為主題(topic)的字串共享相互連接。發布者應用程式創建並向主題發送消息。訂閱者應用程式創建對主題的訂閱，以從中接收消息。

BigQuery 是無服務器數據倉庫。BigQuery 中的表格被組織到數據集中。在本實驗中，發佈到發布/訂閱中的消息將被匯總並存儲在 BigQuery 中。

要創建 BigQuery 數據集，請執行以下操作：

### Option 1: The command-line tool
```shell
bq mk taxirides
bq mk \
--time_partitioning_field timestamp \
--schema ride_id:string,point_idx:integer,latitude:float,longitude:float,\
timestamp:timestamp,meter_reading:float,meter_increment:float,ride_status:string,\
passenger_count:integer -t taxirides.realtime
```

### Option 2: The BigQuery Console UI
1. Navigation menu > BigQuery
2. 從左側點擊 project ID
3. 在 Cloud Console 右側的查詢編輯器下方，點擊 `Create dataset`
4. 為新數據集命名為 `taxirides`，將所有其他字段保留原樣，點擊 `Create dataset`
5. 如果查看左側 menu，則應該看到新創建的數據集
6. 點擊 `taxirides` 數據集
7. 點擊 `create table`
8. 名稱叫 `realtime`
9. 對於 `schema`，點擊 `edit as text` 並複製以下進行張貼
```sql
ride_id:string,
point_idx:integer,
latitude:float,
longitude:float,
timestamp:timestamp,
meter_reading:float,
meter_increment:float,
ride_status:string,
passenger_count:integer
```
10. 在 `Partition and cluster settings`，為 `Partitioning` 字段選擇 `timestamp` 選項
11. 確認以下
![](https://cdn.qwiklabs.com/gFVQnlq%2Fu40OWdjZFaOVXjiMr5QuEcHDgZPItdIFY6k%3D)

12. 點擊 `Create table` 建立

## Task 2. Create a Cloud Storage bucket
Cloud Storage 允許隨時在全世界內存儲和檢索任意數量的數據。可以將 Cloud Storage 用於多種情況，包括提供網站內容、儲存用於存檔和災難恢復的數據，或通過直接下載將大數據對象分發給用戶。在本實驗中，將使用 Cloud Storage 為 Dataflow 管道提供工作空間。

- Navigation menu > Storage.
- 點擊 Create bucket.
- 輸入名稱可以使用 Project ID
- 對於 `Default storage class` 點擊 `Multi-regional` 如果尚未被選擇
- 對於 `Location` 選擇離最近的
- 點擊 Create


## Task 3. Set up a Dataflow Pipeline
[Dataflow](https://cloud.google.com/dataflow/)是一種無服務器的數據分析方法。在本實驗中，將建立一個流數據管道，以從 `Pub/Sub` 中讀取傳感器數據，計算一個時間窗口內的最高溫度，並將其寫到 BigQuery 中。

1. Navigation menu > Dataflow.
2. 點擊 Create job from template
3. Job 名稱輸入 streaming-taxi-pipeline 
4. 在 Dataflow template 下，選擇  Pub/Sub Topic to BigQuery 模板。
5. 在 "Input Pub/Sub topic" 下輸入"projects/pubsub-public-data/topics/taxirides-realtime"
6. 在 BigQuery output table，輸入 `<myprojectid>:taxirides.realtime`。project 和數據集名稱之間有一個冒號`：`和一個點 `.` 在數據集和表名稱之間
7. 在 Temporary location 下，輸入 `gs://<mybucket>/tmp/`

![](https://cdn.qwiklabs.com/oasq%2BeZva7Nnh%2FcEdbl%2Bq3gJasU5C8ptBhpvTnxka6M%3D)

8. 點擊 `Run Job` 按鈕

## Task 4. Analyze the taxi data using BigQuery
1. 在 Cloud Console 中，打開 Navigation，然後選擇 BigQuery
2. 輸入以下查詢並點擊 Run
```shell
SELECT * FROM taxirides.realtime LIMIT 10
```
3. 如果未返回任何記錄，需等待。再重新運行以上查詢（Dataflow 需要 3-5 分鐘來設置流）。類似輸出如下：
![](https://cdn.qwiklabs.com/QUoiZJGPV5Wwhr5UpZI6VYhU37Tr77uNaFxNAykHRf4%3D)

## Task 5. Perform aggregations on the stream for reporting
1. 運行以下
```sql
WITH streaming_data AS (

SELECT
  timestamp,
  TIMESTAMP_TRUNC(timestamp, HOUR, 'UTC') AS hour,
  TIMESTAMP_TRUNC(timestamp, MINUTE, 'UTC') AS minute,
  TIMESTAMP_TRUNC(timestamp, SECOND, 'UTC') AS second,
  ride_id,
  latitude,
  longitude,
  meter_reading,
  ride_status,
  passenger_count
FROM
  taxirides.realtime
WHERE ride_status = 'dropoff'
ORDER BY timestamp DESC
LIMIT 100000

)

# calculate aggregations on stream for reporting:
SELECT
 ROW_NUMBER() OVER() AS dashboard_sort,
 minute,
 COUNT(DISTINCT ride_id) AS total_rides,
 SUM(meter_reading) AS total_revenue,
 SUM(passenger_count) AS total_passengers
FROM streaming_data
GROUP BY minute, timestamp
```

結果按分鐘顯示每個下車的關鍵指標。

## Task 6. Create a real-time dashboard
1. 打開 [Google Data Studio 鏈結](https://datastudio.google.com/) 在另一個網頁上
2. 在 Reports 頁面上的 Start with a Template 部分中，點擊  `[+] Blank Report`
3. 如果出現 `Welcome to Google Studio` 窗口的提示，點擊 Get started
4. 點選復選框以確認 Google Data Studio 附加條款，然後點擊 Accept
5. 對四個問題都選擇 No thanks，然後點擊 Done
6. 切換回 BigQuery 控制台
7. 在 BigQuery 上點擊 Explore Data > Explore with Data Studio
8. 點擊 Get Started 再點擊 Authorize
9. 設定以下設置

- Chart type: Combo chart
- Date range Dimension: dashboard_sort
- Dimension: dashboard_sort
- Drill Down: dashboard_sort (Make sure that Drill down option is turned ON)
- Metric: SUM() total_rides, SUM() total_passengers, SUM() total_revenue
- Sort: dashboard_sort, Ascending (latest rides first)

![](https://cdn.qwiklabs.com/5Dg6ORZo6JY2wq%2BTsujwhkVVxJHNS3HnVPVlz6YPWn8%3D)

10. 當對 dashboard 感到滿意時，可點擊 `Save` 以保存此數據源
11. 每當有人訪問該儀表板時，它都會是最新訊息。可以透過點擊 `Save` 按鈕附近的 `Refresh` 按鈕來自己嘗試

## Task 7. Create a time series dashboard
1. 打開 [Google Data Studio 鏈結](https://datastudio.google.com/) 在另一個網頁上
2. 在 Reports 頁面上的 Start with a Template 部分中，點擊  `[+] Blank Report`
![](https://cdn.qwiklabs.com/syqv9AI1AOr9ilCnkUU4YrtAfdO%2FrwIXyHLYxdVrLpI%3D)
3. 會打開一個新的空報告，其中包含 Add data to report
![](https://cdn.qwiklabs.com/syqv9AI1AOr9ilCnkUU4YrtAfdO%2FrwIXyHLYxdVrLpI%3D)
4. 從 Google Connectors 列表中，選擇 BigQuery
5. 在 Custom query 下，點擊 qwiklabs-gcp-xxxxxxx > Enter Custom Query,，添加以下查詢
```sql
SELECT
  *
FROM
  taxirides.realtime
WHERE
  ride_status='dropoff'
```

![](https://cdn.qwiklabs.com/bcoApYeKgC2jpRFe91GCMYe1Gry%2BcP7kQqTwO%2FpQuTk%3D)

6. 點擊 Add > Add to Report.

### Create a time series chart
1. 在 Data panel 中，向下滾動到右下角，然後點擊 Add a Field 部分。點擊左上角的 All Fields
2. 將字段 timestamp 類型更改為 Date & Time > Date Hour Minute (YYYYMMDDhhmm)
3. 點擊 Done
4. 點擊 Add a chart
5. 選擇 Time series chart
6. 將圖表放在左下角-空白處。
7. 在右側的 Data panel 中，更改以下內容：
- Dimension: timestamp
- Metric: meter_reading(SUM)

結果類似以下
![](https://cdn.qwiklabs.com/vVTV%2FQiUABSyp7zlqUtBr%2Bs9fh%2BP%2B8kJF79iH48nJmI%3D)


## Task 8. Stop the Dataflow job
1. 點選 Dataflow
2. 點擊 streaming-taxi-pipeline
3. 點擊 Stop 和選擇 Cancel > Stop Job

在本實驗中，將在 Cloud SQL 中放入租賃數據以供租賃推薦引擎使用。推薦引擎本身將使用 Spark ML 會在 Dataproc 上運行。

## Task 1. Create a Cloud SQL instance
1. Navigation menu > SQL 
2. Create instance
3. Choose MySQL
4. instance ID 輸入 rentals
5. 向下滾動並指定 Root 密碼。需記下root密碼。
6. Create

## Task 2. Create tables

在 Cloud SQL 中，點擊 rentals 可以查看實例訊息。

```sql
CREATE DATABASE IF NOT EXISTS recommendation_spark;

USE recommendation_spark;

DROP TABLE IF EXISTS Recommendation;
DROP TABLE IF EXISTS Rating;
DROP TABLE IF EXISTS Accommodation;

CREATE TABLE IF NOT EXISTS Accommodation
(
  id varchar(255),
  title varchar(255),
  location varchar(255),
  price int,
  rooms int,
  rating float,
  type varchar(255),
  PRIMARY KEY (ID)
);

CREATE TABLE  IF NOT EXISTS Rating
(
  userId varchar(255),
  accoId varchar(255),
  rating int,
  PRIMARY KEY(accoId, userId),
  FOREIGN KEY (accoId)
    REFERENCES Accommodation(id)
);

CREATE TABLE  IF NOT EXISTS Recommendation
(
  userId varchar(255),
  accoId varchar(255),
  prediction float,
  PRIMARY KEY(userId, accoId),
  FOREIGN KEY (accoId)
    REFERENCES Accommodation(id)
);

SHOW DATABASES;
```

### Connect to the database
1. 找到 `Connect to this instance` 並點擊 `connect using Cloud Shell`
![](https://i.imgur.com/8yhX3Rd.png)
2. 點擊 Cloud Shell 並等待加載，加載後輸入 `gcloud sql connect rentals --user=root --quiet`
3. 按下 ENTER
4. 等待 IP 被列入白名單
5. 輸入密碼並按 ENTER 鍵
6. 輸入 SHOW DATABASES;
```sql
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
7. 將之前分析的以下 SQL 複製到 CLI 中
```sql
CREATE DATABASE IF NOT EXISTS recommendation_spark;

USE recommendation_spark;

DROP TABLE IF EXISTS Recommendation;
DROP TABLE IF EXISTS Rating;
DROP TABLE IF EXISTS Accommodation;

CREATE TABLE IF NOT EXISTS Accommodation
(
  id varchar(255),
  title varchar(255),
  location varchar(255),
  price int,
  rooms int,
  rating float,
  type varchar(255),
  PRIMARY KEY (ID)
);

CREATE TABLE  IF NOT EXISTS Rating
(
  userId varchar(255),
  accoId varchar(255),
  rating int,
  PRIMARY KEY(accoId, userId),
  FOREIGN KEY (accoId)
    REFERENCES Accommodation(id)
);

CREATE TABLE  IF NOT EXISTS Recommendation
(
  userId varchar(255),
  accoId varchar(255),
  prediction float,
  PRIMARY KEY(userId, accoId),
  FOREIGN KEY (accoId)
    REFERENCES Accommodation(id)
);

SHOW DATABASES;
```

8. 點擊 ENTER

9. 確認數據庫有 `commissionation_spark`
```sql
+----------------------+
| Database             |
+----------------------+
| information_schema   |
| mysql                |
| performance_schema   |
| recommendation_spark |
| sys                  |
+----------------------+
```
10. 運行以下

```sql
USE recommendation_spark;

SHOW TABLES;
```

11. 按壓 ENTER

12. 確認表
```shell
+--------------------------------+
| Tables_in_recommendation_spark |
+--------------------------------+
| Accommodation                  |
| Rating                         |
| Recommendation                 |
+--------------------------------+
```

13. 運行以下

```sql
SELECT * FROM Accommodation;
```

## Task 3. Stage data in Cloud Storage
### Option 1: Use the command line
1. 在新增一個 `Cloud Shell`
2. 輸入以下
```shell
echo "Creating bucket: gs://$DEVSHELL_PROJECT_ID"
gsutil mb gs://$DEVSHELL_PROJECT_ID

echo "Copying data to our storage from public dataset"
gsutil cp gs://cloud-training/bdml/v2.0/data/accommodation.csv gs://$DEVSHELL_PROJECT_ID
gsutil cp gs://cloud-training/bdml/v2.0/data/rating.csv gs://$DEVSHELL_PROJECT_ID

echo "Show the files in our bucket"
gsutil ls gs://$DEVSHELL_PROJECT_ID

echo "View some sample data"
gsutil cat gs://$DEVSHELL_PROJECT_ID/accommodation.csv
```
3. 按壓 ENTER

### Option 2: Use the Cloud Console UI
1. Storage > Browser
2. Create Bucket
3. 將項專案名稱指定為存儲桶名稱
4. Create
5. 下載以下並上傳

- [accommodation.csv](https://storage.googleapis.com/cloud-training/bdml/v2.0/data/accommodation.csv)
- [rating.csv](https://storage.googleapis.com/cloud-training/bdml/v2.0/data/rating.csv)

## Task 4. Load data from Cloud Storage into Cloud SQL tables
1. 回到 SQL 面板
2. 點擊 rentals

### Import accommodation data
1. 點擊 Import
2. 指定以下
- Source: 點 Browse > [Your-Bucket-Name] > accommodation.csv 再點 Select
- Format of import: CSV
- Database: 從下拉列選擇 Recommendation_spark
- Table: copy and paste: Accommodation

![](https://i.imgur.com/ETAhGBQ.png)
![](https://i.imgur.com/Nb9kTT8.png)

3. 點擊 Import

### Import user rating data
將上面第二步中  accommodation.csv  替換 rating.csv。Table 則是 Accommodation 替換 Rating。

## Task 5. Explore Cloud SQL data
```sql
USE recommendation_spark;

SELECT * FROM Rating
LIMIT 15;
```

使用 SQL 聚合函數計算表中的行數
```sql
SELECT COUNT(*) AS num_ratings
FROM Rating;
```

住宿的平均評價等級是多少？

```sql
SELECT
    COUNT(userId) AS num_ratings,
    COUNT(DISTINCT userId) AS distinct_user_ratings,
    MIN(rating) AS worst_rating,
    MAX(rating) AS best_rating,
    AVG(rating) AS avg_rating
FROM Rating;
```

在機器學習中，將需要豐富的用戶偏好歷史來學習模型。運行以下查詢以查看哪些用戶提供了最高的評分。

```sql
SELECT
    userId,
    COUNT(rating) AS num_ratings
FROM Rating
GROUP BY userId
ORDER BY num_ratings DESC;
```

離開 MySQL 輸入 exit。

## Task 6. Launch Dataproc
可以使用 Dataproc 來根據用戶以前的評分來訓練推薦的機器學習模型。接著，可以應用該模型為數據庫中的每個用戶創建建議列表。

要啟動 Dataproc 並對其進行配置，以使集群中的每台計算機都可以訪問 Cloud SQL

1. 在 Navigation menu  點擊 SQL，並記下 Cloud SQL 的區域
2. 在 Navigation menu 點擊 Dataproc 接著點擊 Enable API 如果有提示視窗
3. 啟用後，點擊 Create cluster 並命名集群為 rentals
4. 將 Region 保持不變，即 us-central1 並將 Zone 更改為us-central1-a (與 Cloud SQL 位於同一區域)。這將最小化群集和數據庫之間的網絡延遲。
5. 對於 Master node 和 Machine type,，選擇 n1-standard-2(2 個 vCPU，7.5 GB 記憶體)
6. 對於 Worker nodes 和 Machine type,，選擇 n1-standard-2(2 個 vCPU，7.5 GB 記憶體)
7. 將所有其他值保留為預設值，然後點擊 Create。等待建立
8. 注意群集中的 Name、Zone 和 Total worker nodes 節點
9. 將以下 bash 複製到 Cloud Shell 中(在運行前，如有必要，可以選擇更改 CLUSTER、ZONE 或 NWORKERS)
10. 點擊 ENTER。出現提示時，輸入 Y，然後再次按 ENTER 以繼續
11. 在 Cloud SQL 主頁上的  Connect to this instance 下，將 Public IP Address 複製或是記下它

![](https://i.imgur.com/kMxyfFk.png)

## Task 7. Run the ML model
接下來，將創建一個訓練模型並將其應用於系統中的所有用戶。數據科學團隊已使用 Apache Spark 創建了推薦模型，並使用 Python 編寫。將其複製到暫存桶中。

1. Cloud Shell 執行以下
```shell
gsutil cp gs://cloud-training/bdml/v2.0/model/train_and_apply.py train_and_apply.py
cloudshell edit train_and_apply.py
```
2. 當提示跳出，選擇 Open in New Window
3. 等待 Editor UI 加載
4. 打開 train_and_apply.py，找到第 30 行：CLOUDSQL_INSTANCE_IP，然後貼上先前複製的 Cloud SQL IP 地址
```python
# MAKE EDITS HERE
CLOUDSQL_INSTANCE_IP = '<paste-your-cloud-sql-ip-here>'   # <---- CHANGE (database server IP)
CLOUDSQL_DB_NAME = 'recommendation_spark' # <--- leave as-is
CLOUDSQL_USER = 'root'  # <--- leave as-is
CLOUDSQL_PWD  = '<type-your-cloud-sql-password-here>'  # <---- CHANGE
```
5. CLOUDSQL_PWD 也是
6. 儲存檔案，File > Save
7. 點擊 Open Terminal

## Task 8. Run your ML job on Dataproc
1. 在 Dataproc console 點擊 rentals 集群
2. 點擊 Submit job
3. 對於 Job type，選擇 PySpark；Main python file，指定上傳到存儲桶中的 Python 檔案的位置，`gs://<bucket-name>/train_and_apply.py`
4. 點擊 Submit
![](https://i.imgur.com/T8Rdkts.png)
5. 選擇 Navigation menu > Dataproc > Job 查看作業狀態

![](https://i.imgur.com/gizn7NF.png)

## Task 9. Explore inserted rows with SQL
1. 打開 SQL(在 Storage 部分)
2. 單擊 rentals 以查看與 Cloud SQL 實例相關的詳細信息
3. 在 Connect to this instance 部分下，點擊 Connect using Cloud Shell。這將啟動一個新的 Cloud Shell。在 Cloud Shell 中，按 Enter。
4. 出現提示時，輸入配置的 root 密碼，然後按 Enter
5. 在 MySQL CLI 中輸入以下
```sql
USE recommendation_spark;

SELECT COUNT(*) AS count FROM Recommendation;
```

如果要獲取 Empty Set (0)，請等待 Dataproc 作業完成。如果超過 5 分鐘，則您的工作可能失敗了，需要進行故障排除。

6. 查找給用戶的建議：
```sql
SELECT
    r.userid,
    r.accoid,
    r.prediction,
    a.title,
    a.location,
    a.price,
    a.rooms,
    a.rating,
    a.type
FROM Recommendation as r
JOIN Accommodation as a
ON r.accoid = a.id
WHERE r.userid = 10;
```
7. 結果應類似於以下結果：
```sql
+--------+--------+------------+-----------------------------+----------+-------+-------+--------+---------+
| userid | accoid | prediction | title                       | location | price | rooms | rating | type    |
+--------+--------+------------+-----------------------------+----------+-------+-------+--------+---------+
| 10     | 21     |  2.1067479 | Big Peaceful Cabin          | Seattle  |    80 |     2 |    4.9 | cottage |
| 10     | 40     |  1.8135235 | Colossal Private Castle     | Seattle  |  2900 |    24 |    1.5 | castle  |
| 10     | 77     |  1.6707717 | Great Private Country House | Dublin   |  1150 |    10 |    2.4 | mansion |
| 10     | 7      |  1.6594443 | Vast Peaceful Fortress      | Seattle  |  3200 |    24 |    1.9 | castle  |
| 10     | 64     |  1.6276065 | Enormous Peaceful Fort      | Berlin   |  3500 |    13 |    1.8 | castle  |
+--------+--------+------------+-----------------------------+----------+-------+-------+--------+---------+
5 rows in set (0.15 sec)
```

這些是您推薦的五種住宿。請注意，由於數據集太小（因此預測的評分不是很高），因此建議的質量不是很高。儘管如此，本實驗仍說明您創建產品推薦所要經過的過程。
SQL(Structured Query Language) 是用於操作數據的標準語言，可提出問題並從結構化數據集中獲得見解。它通常用於資料庫管理，並允許執行像是將交易記錄寫入關資料聯數據庫和 PB 級數據分析之類的任務。本實驗是 `SQL` 的簡介，準備 `Qwiklabs` 中有關數據科學主題的許多實驗和任務。實驗分為兩個部分：在上半部中，將學習基本的 `SQL` 查詢關鍵字，這些關鍵字將在 `BigQuery` 控制台中的操作公開數據集，該數據集包含有關`London bikeshares`的訊息。

在下半部中，將學習如何將 `London bikeshare` 數據集的子集導出為 `CSV` ，將其上傳到 `Cloud SQL`。

## The Basics of SQL
### Databases and Tables
如前面所述，`SQL` 允許從`structured datasets`中獲取訊息。結構化的數據集具有清晰的規則和格式，通常將時間組織成表格或以行和列格式化的數據。非結構化數據的一個示例是圖片檔案，非結構化數據無法使用 `SQL` 進行操作，並且不能存儲在 `BigQuery` 數據集或表中。要使用影像數據，可能會使用`Cloud Vision` 之類的服務，並通過其 `API` 操作。

以下是結構化數據集的示例，一個簡單的表：

|User|Price|Shipped
|---|---|---|
|Sean|$35|Yes|
|Rocky|$50|No|

該表具有`User`、`Price` 和 `Shipped` 欄位以及填充值的行組成的兩行。資料庫本質上是*一個或多個表的集合*。`SQL` 是一種結構化的資料庫管理工具，經常將連接在一起的一個或多個表上而不是整個資料庫上運行查詢。

## SELECT and FROM
SQL本質上是拼湊的，因此在運行查詢之前，需清楚要問數據的問題。`SQL` 有預定義的關鍵字，可以使用這些關鍵字將問題轉換為偽英語 `SQL` 語法，以便獲取資料庫引擎返回所需的結果。

最重要的關鍵字是 `SELECT` 和 `FROM`
- SELECT
    - 指定要從資料表中提取哪些字段
- FROM
- 指定要提取數據的表

假設我們有下表 `example_table`，其中有 `USER`、`PRICE` 和 `SHIPPED` 欄位

![](https://cdn.qwiklabs.com/XOxiZPGbXJ2GBjG164aXcFmKI35BtpMO0%2FiWcbIYLfQ%3D)

假設想提取在 `USER` 欄位的數據。可以運行以下使用 `SELECT` 和 `FROM` 的查詢

```sql=
SELECT USER FROM example_table #  從 example_table 中 USER 欄位找所有名稱
```

也可以使用 `SELECT` 關鍵字選擇多個欄位。假設要提取 `USER` 和 `SHIPPED` 欄位中的數據

```sql=
SELECT USER, SHIPPED FROM example_table
```

![](https://cdn.qwiklabs.com/u4knQcxxNRrBjThYXtSccEq9%2BroxOz0FL0Jg6zCzIAE%3D)


## WHERE
`WHERE` 關鍵字是另一個 `SQL` 語法，用於過濾表以獲取特定的欄位值。假設要從 example_table 中提取包裹已發貨的使用者，可以使用 `WHERE` 補充查詢
```sql=
SELECT USER FROM example_table WHERE SHIPPED='YES'
```

返回結果如下圖紅框的使用者

![](https://cdn.qwiklabs.com/wst%2BjF1gd%2FvqJaN8OPxTgoWdUbT%2BcWKMVipwc2wlYWw%3D)


## Exploring the BigQuery Console
### Open BigQuery Console

`Navigation menu > BigQuery`，會進入到 `BigQuery` 控制台

![](https://cdn.qwiklabs.com/vd3QNHGB4BAyHjAvOHw9qD0iqCPNaHAzY657z%2FGWLtY%3D)

控制台的右側有 `Query editor`，它可以編寫和運行 `SQL`。下方是 `Query history`，它是之前運行的查詢的列表。控制台的左窗格是 `Navigation panel`。除了 `query history`、`saved queries` 和 `job history` 之外，還有 `Resources` 欄位。該欄位在 `BigQuery` 中，專案包含數據集，而數據集包含表。

### Uploading queryable data

練習在 `BigQuery` 中運行 `SQL`

點擊 `+ ADD DATA`鏈接，然後選擇 `Explore public datasets`，接著在搜索欄中，輸入 `london` 選擇 `London Bicycle Hires` ，然後選擇 `View Dataset`。

![](https://cdn.qwiklabs.com/F9lhtxyIiwttzF%2FyortRrX8dOAJ60sTN%2BKCNyuXHcfU%3D)

在 `Resources` 那邊一個新的選項打開，現在將在 `Resources` 面板中添加一個名為 `bigquery-public-data` 的新項目

![](https://cdn.qwiklabs.com/f1yJh%2FCYs%2FKZ1D9Ta%2FU1mWAQh3VsOzu0Z%2Becb5CYimY%3D)

點擊 `bigquery-public-data> london_bicycles> cycle_hire`。此時，擁有遵循 `BigQuery` 規範的數據如下規範

- GCP Project → bigquery-public-data
- Dataset → london_bicycles
- Table → cycle_hire

點擊 `cycle_hire` 表，在控制台中點擊 `Preview`。頁面應類似於以下內容

![](https://i.imgur.com/X1BtkzO.png)

## Running SELECT, FROM, and WHERE in BigQuery

如果看控制台的右下角，會發現在 `2015` 年至 `2017` 年之間有 `24,369,201` 筆數據，或在倫敦進行的個人自行車旅遊。其第 7 列關鍵字為 `end_station_name`，它指定了 `Bikeshare` 騎乘的最終目的地。使用以下進行查詢

```sql=
SELECT end_station_name FROM `bigquery-public-data.london_bicycles.cycle_hire`; # 回傳 24369201 筆
```

點擊 `COMPOSE NEW QUERY` 以清除 `Query editor`，然後使用 `WHERE` 關鍵字查詢 20 分鐘或更長時間的自行車旅行次數

```sql=
SELECT * FROM `bigquery-public-data.london_bicycles.cycle_hire` WHERE duration>=1200;
```

`SELECT *` 返回表中的所有行值。`duration` 以秒為單位，因此 `1200 = 60 * 20`。該結果有 7,334,890 筆，以總數計算 7334890/24369201 這表示約30% 的倫敦自行車騎乘持續了 20 分鐘或更長的時間。

## More SQL Keywords: GROUP BY, COUNT, AS, and ORDER BY
### GROUP BY
`GROUP BY` 關鍵字將聚合共享相同條件，像是欄位值的結果集行，並回傳為此類條件找到的所有唯一項目。這是找出表中類別訊息的有用關鍵字。範例如下

```sql=
SELECT start_station_name FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name;
```
結果如下圖

![](https://i.imgur.com/HXh82IY.png)

如果沒有 `GROUP BY`，查詢將回傳全部的 `24,369,201` 筆。`GROUP BY` 將輸出在表中找到的唯一(非重複)列值，可透過查看右下角的內容來查看，將看到 `880` 行，表示有 880 個不同的倫敦 `Bikeshare` 起點。


### COUNT

`COUNT` 函數將返回共享相同條件的行數。與 `GROUP BY` 一起使用時非常有用。將 `COUNT` 函數添加到先前查詢，計算每個起點開始多少次騎乘。點擊`COMPOSE NEW QUERY`，運行以下
```sql
SELECT start_station_name, COUNT(*) FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name;
```

結果如下，顯示了在每個起始位置開始有多少輛共享自行車。

![](https://i.imgur.com/nKbZXub.png)


### AS
`SQL` 還具有 `AS` 關鍵字，該關鍵字創建表或欄位的*別名*。別名是回傳欄位或表賦予的新名稱，無論 `AS` 指定什麼名稱。將 `AS` 關鍵字添加到我們運行的最後一個查詢中，觀察其結果。

```sql
SELECT start_station_name, COUNT(*) AS num_starts FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name;
```
結果如下，將返回表中的 `COUNT(*)` 列設置為別名 `num_starts`。這是一個方便使用的關鍵字，對於在處理大量數據時。

![](https://i.imgur.com/U4kGv7v.png)

### ORDER BY
`ORDER BY` 關鍵字根據指定的條件或欄位值以*升序*或*降序*對查詢返回的數據進行排序。這邊將執行以下操作，
- 回傳一個表，該表包含從每個起始站開始的自行車共享騎乘次數，並按起始站的字母順序進行組織

```sql=
SELECT start_station_name, COUNT(*) AS num FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name ORDER BY start_station_name;
```

- 回傳一個表，該表包含從每個起始站開始的自行車共享騎乘次數，並按從低到高的順序進行編號

```sql=
SELECT start_station_name, COUNT(*) AS num FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name ORDER BY num;
```

- 回傳一個表，該表包含從每個起始站開始的自行車共享騎乘次數，並按從高到低的順序進行編號。

```sql=
SELECT start_station_name, COUNT(*) AS num FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name ORDER BY num DESC;
```

我們看到 `Belgrove Street, King's Cross` 的起始次數最多。但以總數來說是一小部分，我們看到不到1%(234458/24369201)的騎乘是從該車站開始的。

## Working with Cloud SQL
### Exporting queries as CSV files
`Cloud SQL` 是一個完全託管的資料庫服務，可輕鬆在雲中設置、維護、管理和管理相關  `PostgreSQL` 和 `MySQL` 資料庫。`Cloud SQL` 接受兩種數據格式：`dump files (.sql)` 或 `CSV files (.csv)`。將學習如何將 `cycle_hire` 表的子集導出到 `CSV` 並將其作為中間位置上傳到 `Cloud Storage`。

在 `BigQuery` 最後運行是

```sql=
SELECT start_station_name, COUNT(*) AS num FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY start_station_name ORDER BY num DESC;
```

在查詢結果中，點擊 `SAVE RESULTS > CSV(local file)`，會觸發下載，此下載會將查詢存為 `CSV`，須注意下載的名稱等。


再執行以下，這將回傳一個表，該表包含在每個終點站完成的自行車共享騎乘次數，並按從高到低的數字進行排序。
```shell
SELECT end_station_name, COUNT(*) AS num FROM `bigquery-public-data.london_bicycles.cycle_hire` GROUP BY end_station_name ORDER BY num DESC;
```
輸出結果，也將其儲存成 `CSV`
![](https://cdn.qwiklabs.com/mAesoVBfXplwi1b5bh1nIRILSAi7%2FBbebkAhpzb2CZ4%3D)


### Upload CSV files to Cloud Storage
到 `Cloud Console`，將創建一個儲存桶並上傳剛剛下載的檔案。選擇 `Navigation menu > Storage > Browser`，接著點擊 `Create bucket`。為儲存桶輸入唯一的名稱，其他設置讓它為預設，然後點擊 `Create`。

![](https://cdn.qwiklabs.com/MFrZ8g98FGK%2FgBtWhPVHBAvbMqkeIZK%2FC%2FtKy6r0a4w%3D)


此時應該在 `GCP` 控制台中查看創建的 `Cloud Storage Bucket`。點擊 `Upload files`，然後選擇包含 `start_station_name` 的 `CSV`，然後單擊 `Open`，`end_station_name` 數據也重複此操作。透過點擊檔案遠端旁邊的三個點來重新命名 `start_station_name`，點擊 `rename`，重命名檔案為 `start_station_data.csv`，而 `end_station_name` 同樣的操作但名稱是 `end_station_data.csv`。

![](https://cdn.qwiklabs.com/O0gGDUAw3%2BKFgvwpeQvYtmRFgfAlChH09mZMXpztL%2FM%3D)

### Create a Cloud SQL instance

選擇 `Navigation menu > SQL`，點擊 `Create Instance`，資料庫引擎選擇 `MySQL`，現在輸入該 `Instance` 的名稱，接著在 `root password` 字段中輸入密碼，最後點擊 `Create`

![](https://cdn.qwiklabs.com/WUXtEyQAsB3cC2zATXIR1P8kq2BvnrT88egc0y%2F0sFo%3D)

創建成功後，應該類似於下圖

![](https://i.imgur.com/8OCXp9K.png)


## New Queries in Cloud SQL

### CREATE keyword (databases and tables)
剛已經啟動並運行了 `Cloud SQL` 實例，接著使用 `Cloud Shell` 在其創建資料庫。這邊先啟動 `cloud shell`

### Activate Cloud Shell
```shell=
gcloud sql connect  qwiklabs-demo --user=root #  qwiklabs-demo 為 SQL 實例名稱
```
之後輸入剛建立 SQL 實例的密碼，完成後如下
```shell
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 494
Server version: 5.7.14-google-log (Google)

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]>
```

建立資料庫
```sql=
CREATE DATABASE bike;
```

建立兩張表
```sql=
USE bike;
CREATE TABLE london1 (start_station_name VARCHAR(255), num INT);
CREATE TABLE london2 (end_station_name VARCHAR(255), num INT);
```

使用了 `CREATE` 關鍵字，但是這一次它使用 `TABLE` 子句指定要*構建表*而不是資料庫。`USE` 關鍵字指定要連接的資料庫。現在，您有了一個名稱是 ` london1`的表，該表包含兩列：`start_station_name` 和 `num`。 `VARCHAR(255)` 指定可變長度字串欄位，最多可輸入 255 個字符，而 `INT` 是整數類型的欄位。

嘗試查詢，結果應為空，因為尚未導入數據
```sql=
SELECT * FROM london1;
SELECT * FROM london2;
```

### Upload CSV files to tables

至 `Cloud SQL` 控制台。將 `start_station_name` 和 `end_station_name` 的 `CSV` 上傳到創建的 `london1` 和 `london2` 表中。

1. 點擊 `IMPORT`
2. 在 `Cloud Storage` 字段中，點擊 `Browse`，然後點擊存儲桶名稱旁邊的箭頭，再單擊 `start_station_data.csv`，最後點擊 `Select`。
3. 選擇 `bike` 資料庫，然後在表格中輸入 `london1`
4. 點擊 `Import`

兩個檔案使用以上方式上傳

![](https://cdn.qwiklabs.com/AmE%2BAIYHcE48Xt0k7wmRrB%2FJYPoe8dS%2BAmNFP2eZm5g%3D)


上傳後對兩個表(london1、london2)執行查詢，
```sql=
SELECT * FROM london1;
```
會有 881 筆數據

![](https://cdn.qwiklabs.com/gHiOlNC7sj3nJvU6dgHAVtU4bzDaVWvK6MBEaKRwLH8%3D)

### DELETE keyword
刪除 `london1` 和 `london2` 的第一行

```sql=
DELETE FROM london1 WHERE num=0;
DELETE FROM london2 WHERE num=0;
```


刪除的行是 `CSV` 中的表頭列。`DELETE` 關鍵字本身不會刪除檔案的第一行，但會刪除表的所有行，其中欄位名(num)包含指定的值(0)。如果運行 `SELECT * FROM london1;` 和 `SELECT * FROM london2;` 查詢並滾動到表格頂部，這些行將不存在。

### INSERT INTO keyword
使用 `INSERT INTO` 關鍵字將數據插入表中。`INSERT INTO` 關鍵字需要一張表，並用括號指定該表的欄位，而接續使用 `VALUES` 插入要插入至表中的值

```sql=
INSERT INTO london1 (start_station_name, num) VALUES ("test destination", 1); # 在 london1 插入，該行將 start_station_name 設為 test destination，num 設為 1 
```

使用 `SELECT * FROM london1` 再最後一列將看到插入的數據

![](https://cdn.qwiklabs.com/eYhqa3ycQ83rA1PDALG5rffOmKQ5OkLYuduwGhjejK4%3D)


### UNION keyword
此關鍵字將兩個或多個 `SELECT` 查詢的輸出組合到一個結果集中。可以使用 `UNION` 組合 `london1` 和 `london2` 表的子集。

```sql=
SELECT start_station_name AS top_stations, num FROM london1 WHERE num>100000
UNION
SELECT end_station_name, num FROM london2 WHERE num>100000
ORDER BY top_stations DESC;
```

第一個 `SELECT` 查詢從 `london1` 表中選擇兩欄位，並為 `start_station_name` 創建別名 `top_stations`。使用 `WHERE` 拉出 100,000 輛自行車開始騎乘的共享自行車站名，第二個 `SELECT` 類似。兩者之間的 `UNION` 關鍵字將 `london2` 數據與 `london1` 組合這些查詢的輸出。`RDER BY` 按照 ` top_stations` 欄位值的字母降序對結果進行排序。結果如下

![](https://cdn.qwiklabs.com/z9UUN0K%2FyW1UWaCn0M%2BU1b0srEzOO7EO%2BZ1xHk5aNq0%3D)
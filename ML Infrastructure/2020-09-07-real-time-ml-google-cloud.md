---
layout: post
title: ML Infrastructure-03
date: 2020-09-07
excerpt: "Real Time Machine Learning with Google Cloud ML"
tags: [qwiklab, ML, google]
comments: true
---

此實驗中用於預測服務的基礎數據集提供了有關美國內部航班的歷史訊息，並且已從美國運輸統計局網站上找的到。該數據集可用於實驗廣泛的數據科學概念和技術，並且可用於 `Google Cloud Platform` 上的 `Data Science` 和 `Google Cloud Platform` 上的 `Data Science:Machine Learning` 任務的所有其他實驗室。即時航班延誤預測服務將使用 `Google Cloud Machine Learning` 模型根據出發時的可用數據預測即時航班是否準時到達。預測服務將使用串流`Google Cloud Dataflow` 作業來處理輸入到 `Google Cloud PubSub` 中的模擬實時航班事件數據。實時飛行事件模擬數據的程式碼用 `Python` 編寫，即時機器學習預測代碼用 `Java` 編寫。

## Get started
同樣的環境會事先準備好，有準備 VM，使用 SSH 連接後執行以下
```shell
sudo apt -y update
sudo apt -y upgrade
sudo apt-get install git -y
git clone  https://github.com/GoogleCloudPlatform/data-science-on-gcp/
sudo apt-get install virtualenv -y
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export BUCKET=${PROJECT_ID}
export REGION=us-central1
```

使用此數據集訓練的 `TensorFlow 2.x` 機器學習模型已復製到 `Google Cloud` 儲存桶中，供本實驗使用。`TensorFlow 2.x` 模型是使用 `Google Cloud Platform` 上的 `Data Science: Machine Learning` 任務中先前實驗中涵蓋的技術進行訓練的，一旦訓練並導出了 `TensorFlow 2.x` 模型，任何`TensorFlow 2.x` 相關引擎都可以重用它。本實驗中，將直接從 `Google Cloud Storage` 將模型加載到 `Google Cloud ML` 中。

設置一些環境變量，指向數據的輸出位置和模型的來源位置的 `Cloud Storage` 存儲桶
```shell
export OUTPUT_DIR=gs://${BUCKET}/flights/chapter9/output/tf2
export MODEL_LOCATION=$(gsutil ls $OUTPUT_DIR/export/exporter | tail -1) # 來源數據將復製到本機的位置
```
創建一個模型和該模型的初始版本
```shell
gcloud ai-platform models create flights --regions $REGION
gcloud ai-platform versions create tf2 --model flights \
                                    --origin ${MODEL_LOCATION} \
                                    --runtime-version 2.1 \
                                    --python-version 3.7
```

### Configure Java components
使用 `SSH` 連線 `VM`，並執行以下
```shell
sudo apt-get install -yq openjdk-8-jdk
sudo update-alternatives --set java /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java
sudo apt-get install -yq maven
cd ~/data-science-on-gcp/10_realtime/chapter10
vi src/main/java/com/google/cloud/training/flights/FlightsMLService.java
private static final String PROJECT = "cloud-training-demos"; # 將這行 cloud-training-demos 替換專案 ID
```

使用 `Maven` 進行 `clean` 重新編譯專案，修復 `pom.xml` 中的所有依賴項
```shell
mvn clean compile
```

## Start the real-time simulation script
切換回原始 `SSH Shell`

### Configure Python Components

```shell
cd ~/data-science-on-gcp/04_streaming/simulate # 移至此位置
virtualenv -p python3 env # 建立虛擬環境
source env/bin/activate # 啟動虛擬環境
pip install google-cloud-pubsub # 安裝套件
pip install google-cloud-bigquery # 安裝套件
pip install google_compute_engine # 安裝套件
pip install google-cloud-storage # 安裝套件
gcloud auth application-default login 
```

`gcloud auth application-default login` 執行後點擊此實驗 ID，然後點擊 `Allow`。複製顯示的驗證代碼。將驗證碼貼到有`Enter verification code:` 提示的 `SSH` 窗口中，然後按 `Enter`。

運行，飛行模擬，此模擬使用 2015 年以來的實際飛行數據生成飛行數據事件
```shell
cd ~/data-science-on-gcp/04_streaming/simulate
export PROJECT_ID=$(gcloud info --format='value(config.project)')
python ./simulate.py --project $PROJECT_ID --startTime '2015-01-01 06:00:00 UTC' --endTime '2015-01-03 00:00:00 UTC' --speedFactor=60
```

### Start the real-time prediction service
在安裝 `Java` 的第二個 `SSH`，確保仍在正確的 `chapter10` 目錄中
```shell
cd ~/data-science-on-gcp/10_realtime/chapter10
```

創建預測代碼所需的臨時 `Cloud Pub/Sub` 主題

```shell
gcloud pubsub topics create dataflow_temp
```

更新 `pom.xml`
```shell
mvn versions:use-latest-versions
```

設置環境變數
```shell
export BUCKET=<YOUR_BUCKET>
export PROJECT_ID=<YOUR_PROJECT>
```

將預測代碼編譯並部署到 `Cloud Dataflow`

```shell
mvn compile exec:java \
-Dexec.mainClass=com.google.cloud.training.flights.AddRealtimePrediction \
 -Dexec.args="--realtime --speedupFactor=60 --maxNumWorkers=10 --autoscalingAlgorithm=THROUGHPUT_BASED --bucket=$BUCKET --project=$PROJECT_ID --region=us-central1"
```

在 GCP 主頁選單點擊 `Dataflow`，然後單擊 `job` 名稱以監視正在運行的管道。一旦看到資源被寫入 `Bigquery`，跳至 `BigQuery` 頁面，執行以下查詢

```sql
SELECT * from flights.predictions ORDER by notify_time DESC LIMIT 5
```

一旦開始看到數據正在寫入 `BigQuery` 表中，運行在 `Cloud Dataflow` 上的即時預測服務就會使用 `Simulate.py` 生成的飛行數據即時生成預測。
---
layout: post
title: ML Infrastructure-02
date: 2020-09-07
excerpt: "Distributed Machine Learning with Google Cloud ML"
tags: [qwiklab, ML, google]
comments: true
---

此實驗，使用 `Google Cloud ML` 建立和配置深度神經網路模型，然後使用 `Google Cloud ML Engine` 透過模型進行預測。實驗使用的基本數據集提供了有關美國內部航班的歷史訊息，並從美國運輸統計局網站檢索的到。此數據集可用於實驗各種數據科學概念和技術，並可用於 `Google Cloud Platform` 上的 `Data Science` 和 `Google Cloud Platform` 上的 `Data Science: Machine Learning` 任務的所有其它實驗。此實驗中使用的特定數據檔案提供單獨的線練和評估數據集。生成這些檔案的是使用 `Apache Beam` 和 `Cloud Dataflow(Java)` 處理時間窗數據。

`Cloud Datalab` 是一款功能強大的交互式工具，在探索、分析、轉換和可視化數據並在 `Google Cloud Platform` 上構建機器學習模型。它運行在 `Google Compute Engine` 上，並連接到 `Google BigQuery`、`Cloud SQL` 或儲存在 `Google Cloud Storage` 上的簡單文本數據等多種雲服務，因此可以專注於數據科學任務。`Google BigQuery` 是 `RESTful` 網路服務，可與 `Google Cloud Storage` 一起對大型數據集進行交互式分析。 


## Get started
此實驗已經開好了虛擬機和 `Cloud Storage`。指需至 `Compute Engine` 使用 `SSH` 方式連線虛擬機，連線後執行以下

```shell
$ sudo apt-get update
$ sudo apt-get install virtualenv
$ virtualenv -p python3 venv
$ source venv/bin/activate
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y install python3-pip
$ pip install --upgrade pip
$ export PROJECT_ID=$(gcloud info --format='value(config.project)')
$ export BUCKET=${PROJECT_ID}
```

## Create a deep neural network machine learning model
`Google Cloud ML Engine` 可以使用模型線性分類器開發機器學習模型，以及如何修改該模型以使用各種功能。實驗使用相同的 `Python` 框架來實現基本的深度神經網路模型，然後將其擴展到廣泛的深度神經網路模型，該模型結合了 `features` 以評估它們如何改善模型的性能。

重 `Cloud Storage` 複製到虛擬機
```shell
$ gsutil cp gs://${BUCKET}/flights/chapter9/linear-model.tar.gz ~
$ cd ~
$ tar -zxvf linear-model.tar.gz
$ cd ~/tensorflow
$ vi ~/tensorflow/flights/trainer/model.py # 配置程式碼以使用深度神經網路機器學習模型
```

將會在 `model.py` 添加一個將降維功能應用於分類字段的函數。這減少了使用線性分類器進行實驗時使用的 `1000` 個離散值中的 `one-hot encoded` 變量的數量。嵌入會轉換一個稀疏的數據字段，將其映射到一個較小的數目，從而顯著減少了必須處理的變量維數，當進行一次 `one-hot encoded` 時，該字段可能由 `1000` 個獨立的列表示。

將以下程式碼插入 `linear_model` 函數下方，`serving_input_fn` 的定義上方

```python
def create_embed(sparse_col):
    dim = 10 # default
    if hasattr(sparse_col, 'bucket_size'):
       nbins = sparse_col.bucket_size
       if nbins is not None:
          dim = 1 + int(round(np.log2(nbins)))
    return tflayers.embedding_column(sparse_col, dimension=dim)
```

在 `create_embed` 下方添加以下
```python
def dnn_model(output_dir):
    real, sparse = get_features()
    all = {}
    all.update(real)

    # create embeddings of the sparse columns
    embed = {
       colname : create_embed(col) \
          for colname, col in sparse.items()
    }
    all.update(embed)

    estimator = tflearn.DNNClassifier(
         model_dir=output_dir,
         feature_columns=all.values(),
         hidden_units=[64, 16, 4])
    estimator = tf.contrib.estimator.add_metrics(estimator, my_rmse)
    return estimator
```

找到 `run_experiment` 函數。重新配置實驗以調用深度神經網路 `estimator` 函數，而不是線性分類器函數，如下做替換。

```shell
#estimator = linear_model(output_dir)
estimator = dnn_model(output_dir)
```
```shell
$ export REGION=us-central1
$ export OUTPUT_DIR=gs://${BUCKET}/flights/chapter9/output
$ export DATA_DIR=gs://${BUCKET}/flights/chapter8/output
```

現在，可以使用 `Python` 模型將作業提交到 `Cloud ML Engine`，以使用 `TensorFlow` 的分散式雲資源，處理更大的數據集，也可透過 `Google Cloud ML` 使用。

建立一個 `jobname` 環境變數

```shell
$ export JOBNAME=dnn_flights_$(date -u +%y%m%d_%H%M%S)
$ cd ~/tensorflow
```

提交 `Cloud-ML` 任務提供區域，用於階段和工作數據的 `cloud storage` 桶、訓練集目錄、訓練模組名稱和其他參數。
```shell
$ gcloud ai-platform jobs submit training $JOBNAME \
  --module-name=trainer.task \
  --package-path=$(pwd)/flights/trainer \
  --job-dir=$OUTPUT_DIR \
  --staging-bucket=gs://$BUCKET \
  --region=$REGION \
  --scale-tier=STANDARD_1 \
  --runtime-version=1.15 \
  -- \
  --output_dir=$OUTPUT_DIR \
  --traindata $DATA_DIR/train* --evaldata $DATA_DIR/test*
```

回到 `GCP` 主頁，並在選單中點擊 `AI Platform > Jobs`，再點擊剛定義的  `JOBNAME`。在訓練過程中監視該工作，需要不斷刷新瀏覽器以保證正在查看最新訊息。作業完成後，點擊 `View Logs` 查看 `Log`。再點擊 `event` 以打開詳細訊息，將看到列出的分析指標的完整列表，如下所示

![](https://i.imgur.com/cXnPUOo.png)

![](https://i.imgur.com/xDDW3lV.png)

## Add a wide and deep neural network model
現在，將透過創建功能元件來擴展模型以包括其他功能元件，這些功能元件可以將機場與廣闊的地理區域相關聯，並從中獲得簡化的空中流量走道。首先，為覆蓋美國的 `n*n` 網格創建位置儲存桶，然後將每個出發和到達機場分配給它們的特定網格位置。還可以使用一種稱為`feature crossing`的技術來創建其他特徵，該技術創建特徵組合，在組合時可以提供有用的見解。在這種情況下，將出發和到達網格位置組合分組在一起以創建空中交通走道的近似值，並且還對出發地和目的地機場 `ID` 組合進行分組以將每個這樣的對用作模型中的特徵。

再次編輯 `model.py`，在 `linear_model` 函數上方添加以下兩個函數，以實現深度神經網路模型

```python
def parse_hidden_units(s):
    return [int(item) for item in s.split(',')]

def wide_and_deep_model(output_dir,nbuckets=5,
                        hidden_units='64,32', learning_rate=0.01):
    real, sparse = get_features()
    # lat/lon cols can be discretized to "air traffic corridors"
    latbuckets = np.linspace(20.0, 50.0, nbuckets).tolist()
    lonbuckets = np.linspace(-120.0, -70.0, nbuckets).tolist()
    disc = {}
    disc.update({
       'd_{}'.format(key) : \
           tflayers.bucketized_column(real[key], latbuckets) \
           for key in ['dep_lat', 'arr_lat']
    })
    disc.update({
       'd_{}'.format(key) : \
           tflayers.bucketized_column(real[key], lonbuckets) \
           for key in ['dep_lon', 'arr_lon']
    })
    # cross columns that make sense in combination
    sparse['dep_loc'] = tflayers.crossed_column( \
           [disc['d_dep_lat'], disc['d_dep_lon']],\
           nbuckets*nbuckets)
    sparse['arr_loc'] = tflayers.crossed_column( \
           [disc['d_arr_lat'], disc['d_arr_lon']],\
           nbuckets*nbuckets)
    sparse['dep_arr'] = tflayers.crossed_column( \
           [sparse['dep_loc'], sparse['arr_loc']],\
           nbuckets ** 4)
    sparse['ori_dest'] = tflayers.crossed_column( \
           [sparse['origin'], sparse['dest']], \
           hash_bucket_size=1000)

    # create embeddings of all the sparse columns
    embed = {
       colname : create_embed(col) \
          for colname, col in sparse.items()
    }
    real.update(embed)

    #lin_opt=tf.train.FtrlOptimizer(learning_rate=learning_rate)
    #l_rate=learning_rate*0.25
    #dnn_opt=tf.train.AdagradOptimizer(learning_rate=l_rate)
    estimator = tflearn.DNNLinearCombinedClassifier(
         model_dir=output_dir,
         linear_feature_columns=sparse.values(),
         dnn_feature_columns=real.values(),
         dnn_hidden_units=parse_hidden_units(hidden_units))
         #linear_optimizer=lin_opt,
         #dnn_optimizer=dnn_opt)
    estimator = tf.contrib.estimator.add_metrics(estimator, my_rmse)
    return estimator
```

在檔案找到 `run_experiment` 函數，需要重新配置實驗以調用深度神經網路 `estimator` 函數，而不是線性分類器函數。
```python
  #estimator = linear_model(output_dir)
  # estimator = dnn_model(output_dir)
  estimator =  wide_and_deep_model(output_dir, 5, '64,32', 0.01)
```

修改 `OUTPUT` 環境變數以指向該模型運行的位置，同樣的建立一個可識別的 `job` 名稱
```shell
$ export OUTPUT_DIR=gs://${BUCKET}/flights/chapter9/output2
$ export JOBNAME=wide_and_deep_flights_$(date -u +%y%m%d_%H%M%S)
```

再次提交以建立新模型

```shell
$ gcloud ai-platform jobs submit training $JOBNAME \
  --module-name=trainer.task \
  --package-path=$(pwd)/flights/trainer \
  --job-dir=$OUTPUT_DIR \
  --staging-bucket=gs://$BUCKET \
  --region=$REGION \
  --scale-tier=STANDARD_1 \
  --runtime-version=1.15 \
  -- \
  --output_dir=$OUTPUT_DIR \
  --traindata $DATA_DIR/train* --evaldata $DATA_DIR/test*
```

跟上一章一樣，`AI Platform > Jobs` 查看和檢視輸出過程

![](https://cdn.qwiklabs.com/0grFJSsk2%2BorRmnmrJKfW7ZK%2FOwEOu8WSJ3kLQTGfDQ%3D)



## Changing the learning rate

打開 `model.py` 進行編輯，找到 `wide_and_deep_model` 函數後，將配置線性優化器學習率和 dnn 優化器學習率註解拿掉。

![](https://cdn.qwiklabs.com/uya8EEmmVd%2FakVKH5XdLnTnxCQQJKiTBmqAGWyAl6Gw%3D)

將優化器的默認學習率從 `FtrlOptimizer` 的默認學習率更改為 `0.2` 或 `1/sqrt(N)`，其中 `N` 是線性欄位的數量。這種情況下，表示該優化器的學習率為 `0.2`，對於 `DNN` 部分使用的 `AdagradOptimizer`，分類器使用默認學習率 `0.05`。再程式碼中將 `FtrlOptimizer` 的學習率更改為 `0.01`，將 `AdagradOptimizer` 的學習率更改為 `0.0025`。

配置輸出位置，並建立一個可識別的 `job` 名稱
```shell
$ export OUTPUT_DIR=gs://${BUCKET}/flights/chapter9/output3
$ export JOBNAME=learn_rate_flights_$(date -u +%y%m%d_%H%M%S)
```
再運行模型
```shell
$ gcloud ai-platform jobs submit training $JOBNAME \
  --module-name=trainer.task \
  --package-path=$(pwd)/flights/trainer \
  --job-dir=$OUTPUT_DIR \
  --staging-bucket=gs://$BUCKET \
  --region=$REGION \
  --scale-tier=STANDARD_1 \
  --runtime-version=1.15 \
  -- \
  --output_dir=$OUTPUT_DIR \
  --traindata $DATA_DIR/train* --evaldata $DATA_DIR/test*
```

和上述的作法一樣

![](https://cdn.qwiklabs.com/9cG8D2QE%2FXn6fyYUa1ZivzVft0JC7333ICwXr20Zzw4%3D)


## Deploying and using the Model
代碼中針對實驗模型的評估規範定義了一個服務輸入函數，一旦將其加載到 `ML Engine` 中，它將處理對模型的 `REST API` 調用
```shell
$ MODEL_LOCATION=$(gsutil ls $OUTPUT_DIR/export/exporter | tail -1)
```

創建一個模型和該模型的初始版本：
```shell
$ gcloud ai-platform models create flights --regions us-central1
$ gcloud ai-platform versions create v1 --model flights \
                                    --origin ${MODEL_LOCATION} \
                                    --runtime-version 1.15
```

點選 `AI platform -> model` 
![](https://i.imgur.com/mtP1Zn4.png)

現在以交互方式使用 `Python` 對模型執行查詢。對其先安裝 `Google API client Python` 套件
```shell
$ pip install --upgrade google-api-python-client
$ pip install --upgrade oauth2client
$ python # 進入交互式 Python
```

輸入以下
```python
from googleapiclient import discovery
from oauth2client.client import GoogleCredentials
import os
import json
# Create the credential object to allow you to access the API
credentials = GoogleCredentials.get_application_default()
# Create the API call
api = discovery.build('ml', 'v1', credentials=credentials,
      discoveryServiceUrl=
     'https://storage.googleapis.com/cloud-ml/discovery/ml_v1_discovery.json')
PROJECT = os.environ['PROJECT_ID']
parent = 'projects/%s/models/%s/versions/%s' % (PROJECT, 'flights', 'v1')
# Create a request object with some sample flight details
request_data = {'instances':
  [
      {
        'dep_delay': 16.0,
        'taxiout': 13.0,
        'distance': 160.0,
        'avg_dep_delay': 13.34,
        'avg_arr_delay': 67.0,
        'carrier': 'AS',
        'dep_lat': 61.17,
        'dep_lon': -150.00,
        'arr_lat': 60.49,
        'arr_lon': -145.48,
        'origin': 'ANC',
        'dest': 'CDV'
      }
  ]
}
# Query the API
response = api.projects().predict(body=request_data, name=parent).execute()
print ("response={0}".format(response))
```

回應

```python
response={
    'predictions': [
        {'logistic': [0.4227225184440613], 
        'probabilities': [0.5772774815559387, 0.42272254824638367], 
        'all_classes': ['0', '1'], 
        'classes': ['0'], 
        'logits': [-0.3116070330142975], 
        'all_class_ids': [0, 1], 
        'class_ids': [0]
        }]}
```

結果顯示使用此模型，航班延誤的概率為 `0.42`。由於模型是在非常有限的演示數據集上進行訓練的，因此這種預測並不可靠，但是可以看到使用 `Google Cloud ML Engine` 在應用程序中將經過訓練的模型轉換為操作功能是多麼容易。
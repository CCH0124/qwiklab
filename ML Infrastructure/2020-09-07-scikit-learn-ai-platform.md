---
layout: post
title: ML Infrastructure-01
date: 2020-09-07
excerpt: "Scikit-learn Model Serving with Online Prediction Using AI Platform"
tags: [qwiklab, ML, google]
comments: true
---
如果已經使用 `scikit-learn` 構建了機器學習模型，並且希望為應用程式即時提供模型，那麼管理所產生的基礎架構可能聽起來像一場噩夢。在 [AI Platform](https://cloud.google.com/ai-platform/) 上為 `scikit-learn` 模型提供服務。現在可以將已訓練的模型上傳到 `Cloud Storage` 上，並使用 [AI Platform Prediction](https://cloud.google.com/ml-engine/docs/prediction) 支援針對訓練後的模型的可擴展預測請求。




### How to bring a scikit-learn model to AI Platform
準備好模型以進行預測可以透過 5 個步驟完成：

1. Create and save a model to a file
2. Upload the saved model to Cloud Storage
3. Create a model resource in AI Platform
4. Create a model version (linking your scikit-learn model)
5. Make an online prediction

### What you will build

![](https://cdn.qwiklabs.com/7A36PQ8uDmm0YGhZZF4cVa6AhdmJmkb%2Ffz7q%2F%2BSvtjM%3D)

## Create a storage bucket

選擇 `Navigation menu > Storage > Browser`，然後點擊 `Create Bucket`，給它指定一個唯一的名稱，並將`Default storage class`設置為`regional`，並確保將位置設置為`us-central1`，最後單擊 `Create`。

![](https://i.imgur.com/dlhAOG9.png)

## Create Virtual Machine

```shell
$ gcloud compute instances create scikit-vm \
 --image-project=debian-cloud \
 --image-family=debian-9 \
 --service-account=$(gcloud config get-value project)@$(gcloud config get-value project).iam.gserviceaccount.com \
 --scopes=cloud-platform,default,storage-full \
 --zone=us-central1-a \
 --tags http-server,https-server
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-03-5483a3500be3/zones/us-central1-a/instances/scikit-vm].
NAME       ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
scikit-vm  us-central1-a  n1-standard-1               10.128.0.2   34.121.182.45  RUNNING
```

>`service account`，`scope`和 `tags` 選項用於使我們的虛擬機可以存取儲存桶和 `AI Platform API`。

虛擬機創建完成後，使用 `SSH` 連接
```shell
$ gcloud compute ssh --zone=us-central1-a scikit-vm
```

連上後，在虛擬機上更新並安裝 `pip` 和 `virtualenv`
```shell
$ sudo apt-get update
$ sudo apt-get install -y python3-pip
$ sudo apt-get install -y virtualenv
```

## Install scikit-learn

創建一個隔離的 `Python` 虛擬環境，並啟動
```shell
$ virtualenv ml-env -p python3.5
$ source ml-env/bin/activate # 啟動
```

以下是在 `virtualenv` 中安裝所需的軟體
```shell
$ pip install google-api-python-client==1.6.2
$ pip install scikit-learn==0.19.1
$ pip install pandas==0.22.0
$ pip install scipy==1.0.0
$ pip install numpy==1.17
```

## Set up environment variables

設置環境變數，以方便後續步驟的使用。

- PROJECT_ID
    - 使用與 `Google Cloud` 專案匹配的 `PROJECT_ID`
- MODEL_PATH
    - 在 `Cloud Storage` 上建立模型的路徑
- MODEL_NAME
    - 使用的模型名稱
- VERSION_NAME
    - 使用自己定義的版本
- REGION
    - 在此處選擇一個區域，或使用預設的 `us-central1`，這是部署模型的區域
- INPUT_DATA_FILE
    - 包括一個 `JSON` 檔案，其中包含用作模型的預測方法輸入的數據

```shell
$ export PROJECT_ID=your-project-id
$ export MODEL_PATH=gs://your-created-bucket-id
$ export MODEL_NAME=census
$ export VERSION_NAME=v1
$ export REGION=us-central1
```

## The data for this lab
該樣本用於訓練 `Census Income`[數據](https://archive.ics.uci.edu/ml/datasets/Census+Income)由 [UC Irvine 機器學習倉庫](https://archive.ics.uci.edu/ml/datasets/)託管。Citation: Dua, D. and Karra Taniskidou, E. (2017). UCI Machine Learning Repository. Irvine, CA: University of California, School of Information and Computer Science.

- 訓練檔案是 `adult.data`
- 驗證檔案是 `adult.test`

```shell
$ mkdir census_data # 用於存放資料集
$ curl https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.data --output census_data/adult.data
$ curl https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.test --output census_data/adult.test
```

>Disclaimer: This dataset is provided by a third party. Google provides no representation, warranty, or other guarantees about the validity or any other aspects of this dataset.


## Train and save your model

![](https://cdn.qwiklabs.com/IyDMld2TanjMXFr1%2FzOWRGoREy7IKt4aaC2uh08IE6I%3D)

將數據加載到 `pandas` 的 `DataFrame` 中，將與 `XGBoost` 一起使用。在 `XGBoost` 中訓練一個簡單的模型，將模型儲存到可以上傳到 `AI Platform` 的檔案中。首先，我們需要創建一個 `scikit` 學習模型並對其進行訓練。完成後，可以儲存模型並將其放入 `AI Platform` 可用的格式。此時，需要一個代碼編輯器來*創建*和*更新*檔案，可以使用虛擬機上安裝的 `shell` 編輯器，如 `nano` 或 `vim`。

建立一個 `train.py`
```shell
$ vim train.py
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from sklearn.externals import joblib
from sklearn.feature_selection import SelectKBest
from sklearn.pipeline import FeatureUnion
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import LabelBinarizer
# 定義輸入數據的格式，包括未使用的列
# Define the format of your input data including unused columns (These are the columns from the census data files)
COLUMNS = (
    'age',
    'workclass',
    'fnlwgt',
    'education',
    'education-num',
    'marital-status',
    'occupation',
    'relationship',
    'race',
    'sex',
    'capital-gain',
    'capital-loss',
    'hours-per-week',
    'native-country',
    'income-level'
)

# Categorical columns are columns that need to be turned into a numerical value to be used by scikit-learn
CATEGORICAL_COLUMNS = (
    'workclass',
    'education',
    'marital-status',
    'occupation',
    'relationship',
    'race',
    'sex',
    'native-country'
)
# 添加以加載訓練 census 數據集
# Load the training census dataset
with open('./census_data/adult.data', 'r') as train_data:
    raw_training_data = pd.read_csv(train_data, header=None, names=COLUMNS)

# Remove the column we are trying to predict ('income-level') from our features list
# Convert the Dataframe to a lists of lists
train_features = raw_training_data.drop('income-level', axis=1).as_matrix().tolist()
# Create our training labels list, convert the Dataframe to a lists of lists
train_labels = (raw_training_data['income-level'] == ' >50K').as_matrix().tolist()


# Load the test census dataset
with open('./census_data/adult.test', 'r') as test_data:
    raw_testing_data = pd.read_csv(test_data, names=COLUMNS, skiprows=1)
# Remove the column we are trying to predict ('income-level') from our features list
# Convert the Dataframe to a lists of lists
test_features = raw_testing_data.drop('income-level', axis=1).as_matrix().tolist()
# Create our training labels list, convert the Dataframe to a lists of lists
test_labels = (raw_testing_data['income-level'] == ' >50K.').as_matrix().tolist()

# 以下內容將分類列轉換為訓練數據集中的數值
# Since the census data set has categorical features, we need to convert
# them to numerical values. We'll use a list of pipelines to convert each
# categorical column and then use FeatureUnion to combine them before calling
# the RandomForestClassifier.
categorical_pipelines = []

# Each categorical column needs to be extracted individually and converted to a numerical value.
# To do this, each categorical column will use a pipeline that extracts one feature column via
# SelectKBest(k=1) and a LabelBinarizer() to convert the categorical value to a numerical one.
# A scores array (created below) will select and extract the feature column. The scores array is
# created by iterating over the COLUMNS and checking if it is a CATEGORICAL_COLUMN.
for i, col in enumerate(COLUMNS[:-1]):
    if col in CATEGORICAL_COLUMNS:
        # Create a scores array to get the individual categorical column.
        # Example:
        #  data = [39, 'State-gov', 77516, 'Bachelors', 13, 'Never-married', 'Adm-clerical',
        #         'Not-in-family', 'White', 'Male', 2174, 0, 40, 'United-States']
        #  scores = [0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
        #
        # Returns: [['Sate-gov']]
        scores = []
        # Build the scores array
        for j in range(len(COLUMNS[:-1])):
            if i == j: # This column is the categorical column we want to extract.
                scores.append(1) # Set to 1 to select this column
            else: # Every other column should be ignored.
                scores.append(0)
        skb = SelectKBest(k=1)
        skb.scores_ = scores
        # Convert the categorical column to a numerical value
        lbn = LabelBinarizer()
        r = skb.transform(train_features)
        lbn.fit(r)
        # Create the pipeline to extract the categorical feature
        categorical_pipelines.append(
            ('categorical-{}'.format(i), Pipeline([
                ('SKB-{}'.format(i), skb),
                ('LBN-{}'.format(i), lbn)])))

# Create pipeline to extract the numerical features
skb = SelectKBest(k=6)
# From COLUMNS use the features that are numerical
skb.scores_ = [1, 0, 1, 0, 1, 0, 0, 0, 0, 0, 1, 1, 1, 0]
categorical_pipelines.append(('numerical', skb))

# Combine all the features using FeatureUnion
preprocess = FeatureUnion(categorical_pipelines)
```


在數據被模型使用之前進行預處理時需要權衡：

- 優點，模型非常簡單
- 缺點，必須先將數據轉換為數值，然後再傳遞給模型

會後，創建、訓練和分類器儲存到檔案，接續上面代碼

```python
# Create the classifier
classifier = RandomForestClassifier()

# Transform the features and fit them to the classifier
classifier.fit(preprocess.transform(train_features), train_labels)

# Create the overall model as a single pipeline
pipeline = Pipeline([
    ('union', preprocess),
    ('classifier', classifier)
])

# Export the model to a file
joblib.dump(pipeline, 'model.joblib')

print('Model trained and saved')
```

### Completed File

下以下指另訓練 
```shell
$ python train.py
Model trained and saved
```

## Upload the saved model
![](https://cdn.qwiklabs.com/20nb4Z%2F7OUtHWRmbCXjgFqqlhlNZj2USm7RqmxH83uE%3D)

如果模型與 `AI Platform`一起使用，需要將其上傳到 `Cloud Storage`。此動作將獲取本地 `model.joblib` 檔案，同時使用 `gsutil` 透過 `Cloud SDK` 將其上傳到 `Cloud Storage`。

```shell
$ gsutil cp ./model.joblib $MODEL_PATH/model.joblib
```
![](https://i.imgur.com/fpuEBes.png)

## Create a model resource
![](https://cdn.qwiklabs.com/2tqrj3DYgCNGYRYqdrteXxOOYlkskQjV5NDkpcDNUFs%3D)

`AI Platform` 使用*模型*和*版本資源*來組織訓練的模型，`AI Platform` 模型是機器學習模型版本的容器。可以在 `scikit-learn` [文件](https://cloud.google.com/ml-engine/docs/deploying-models#create_a_model_version)中找到有關模型資源和模型版本的更多信息。

創建一個可以容納實際模型的多個不同版本的容器

```shell
$ gcloud ai-platform models create $MODEL_NAME --regions $REGION
```

![](https://i.imgur.com/NLF5DiV.png)

## Create a model version
![](https://cdn.qwiklabs.com/J%2Fsr7Dtv8NAvYLOhV5jK87IKMqwVmEoL1crUY%2BKUBS0%3D)

現在是時候將模型的版本上傳到容器中，這將可以進行線上預測。模型版本需要一些元件，在[此](https://cloud.google.com/ml-engine/reference/rest/v1/projects.models.versions#resource-version)指定。

- name - The name specified for the version when it was created. This will be the `VERSION_NAME` variable you declared at the beginning.
- deploymentUri - The Cloud Storage location of the trained model used to create the version. This is the bucket where you uploaded the model with your `MODEL_PATH`.
- runtimeVersion - The AI Platform runtime version to use for this deployment. This is set to 1.14.
- framework - This specifies whether you're using `SCIKIT_LEARN` or XGBOOST. This is set to `SCIKIT_LEARN`.
- pythonVersion - This specifies whether you're using Python 2.7 or Python 3.5. The default value is set to "2.7", if you are using - Python 3.5, set the value to "3.5"

```shell
# 參數，使用前面定義好的環境變數帶入
$ gcloud beta ai-platform versions create $VERSION_NAME \
    --model $MODEL_NAME \
    --origin $MODEL_PATH \
    --runtime-version="1.14" \
    --framework="SCIKIT_LEARN" \
    --python-version="3.5"
```

用以下方式確認模型的部署

```shell
$ gcloud ai-platform versions list --model $MODEL_NAME
NAME  DEPLOYMENT_URI  STATE
v1    gs://ml-cch     READY
```

## Make an online prediction

![](https://cdn.qwiklabs.com/L3lOLduu0m8GfautoHuB1DSnQm4umJFiHZNsbMhLqV0%3D)

現在該使用部署的模型在 `Python` 中進行線上預測。首先，創建 `test.py` 檔案。
```shell
$ vi test.py
```
現在將內容添加到該文件。第一部分與您對訓練數據所做的非常相似，因此我們將略過這一部分。

- 定義列的格式
- 加載測試數據集
- 將測試數據轉換為數值

```shell
import googleapiclient.discovery
import os
import pandas as pd

PROJECT_ID = os.environ['PROJECT_ID']
VERSION_NAME = os.environ['VERSION_NAME']
MODEL_NAME = os.environ['MODEL_NAME']

# Define the format of your input data including unused columns (These are the columns from the census data files)
COLUMNS = (
    'age',
    'workclass',
    'fnlwgt',
    'education',
    'education-num',
    'marital-status',
    'occupation',
    'relationship',
    'race',
    'sex',
    'capital-gain',
    'capital-loss',
    'hours-per-week',
    'native-country',
    'income-level'
)

# Categorical columns are columns that need to be turned into a numerical value to be used by scikit-learn
CATEGORICAL_COLUMNS = (
    'workclass',
    'education',
    'marital-status',
    'occupation',
    'relationship',
    'race',
    'sex',
    'native-country'
)


# Load the training census dataset
with open('./census_data/adult.data', 'r') as train_data:
    raw_training_data = pd.read_csv(train_data, header=None, names=COLUMNS)

# Remove the column we are trying to predict ('income-level') from our features list
# Convert the Dataframe to a lists of lists
train_features = raw_training_data.drop('income-level', axis=1).as_matrix().tolist()
# Create our training labels list, convert the Dataframe to a lists of lists
train_labels = (raw_training_data['income-level'] == ' >50K').as_matrix().tolist()


# Load the test census dataset
with open('./census_data/adult.test', 'r') as test_data:
    raw_testing_data = pd.read_csv(test_data, names=COLUMNS, skiprows=1)
# Remove the column we are trying to predict ('income-level') from our features list
# Convert the Dataframe to a lists of lists
test_features = raw_testing_data.drop('income-level', axis=1).as_matrix().tolist()
# Create our training labels list, convert the Dataframe to a lists of lists
test_labels = (raw_testing_data['income-level'] == ' >50K.').as_matrix().tolist()

# 設置 `Google API` 客戶端以對測試數據進行預測請求
service = googleapiclient.discovery.build('ml', 'v1')
name = 'projects/{}/models/{}'.format(PROJECT_ID, MODEL_NAME)
name += '/versions/{}'.format(VERSION_NAME)

# Due to the size of the data, it needs to be split in 2
first_half = test_features[:int(len(test_features)/2)]
second_half = test_features[int(len(test_features)/2):]

complete_results = []
for data in [first_half, second_half]:
    responses = service.projects().predict(
        name=name,
        body={'instances': data}
    ).execute()

    if 'error' in responses:
        print(responses['error'])
    else:
        complete_results.extend(responses['predictions'])

# Print the first 10 responses
for i, response in enumerate(complete_results[:10]):
    print('Prediction: {}\tLabel: {}'.format(response, test_labels[i]))
```

運行 `test.py`，預測和標籤盡可能是相同，這表示訓練的模型表現良好

```shell
$ python test.py
Prediction: False       Label: False
Prediction: False       Label: False
Prediction: True        Label: True
Prediction: True        Label: True
Prediction: False       Label: False
Prediction: False       Label: False
Prediction: False       Label: False
Prediction: False       Label: True
Prediction: False       Label: False
Prediction: False       Label: False
```
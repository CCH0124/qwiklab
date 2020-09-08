---
layout: post
title: Baseline Data, ML, AI- 01
date: 2020-09-05
excerpt: "AI Platform: Qwik Start"
tags: [qwiklab, ML, google]
comments: true
---

本實驗將提供本地和 `AI` 平台上的 `TensorFlow` 2.x 模型培訓的動手實踐。訓練後，將學習如何將模型部署到 `AI` 平台進行服務也就是預測。將使用美國人口普查收入數據集訓練模型來預測某人的收入類別。

## Launch AI Platform Notebooks
1. 點擊 `Navigation` 清單並點擊`AI Platform`然後啟動`Notebooks`
![](https://i.imgur.com/qPG2GOK.png)

2. 在`Notebooks`頁面上點擊`New Instance`(新增執行個體)。選擇沒有 `GPU` 的 `TensorFlow 2.x` 的最新版本

![](https://i.imgur.com/k5LjUpN.png)


最後在彈出窗口中，確認深度學習 `VM` 的名稱，移至底部然後點擊 `create`

![](https://i.imgur.com/daWLqsO.png)


3. 點擊打開 `JupyterLab`，`JupyterLab` 窗口將在新視窗中打開

![](https://i.imgur.com/ZdjWl7j.png)


## Clone the example repo within your AI Platform Notebooks instance

要在 `JupyterLab` 實例中複製訓練數據分析筆記本：

1. 在 `JupyterLab` 中，單擊`Terminal`圖標以打開一個新終端
2. 輸入 `git clone https://github.com/GoogleCloudPlatform/training-data-analyst`

![](https://i.imgur.com/5UFEe1v.png)


3. 透過雙擊 `training-data-analyst` 目錄並確保可以看到其內容來確認已複製儲存庫

![](https://i.imgur.com/b25xoCy.png)


### Navigate to the example notebook
在 `AI Platform Notebooks` 中，導航到 `training-data-analyst/self-paced-labs/ai-platform-qwikstart` 並打開 `ai_platform_qwik_start.ipynb`。


## Get your training data

```shell=
gsutil -m cp gs://cloud-samples-data/ml-engine/census/data/* data/
```

## Other

實驗部分直接從該 `Notebooks` 運行因此就沒顯示。
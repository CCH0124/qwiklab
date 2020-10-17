在本實驗中，將圖片上傳到 `Cloud Storage`，並使用它們來訓練自定義模型以識別不同類型的雲(cumulus, cumulonimbus, etc.)。

## Task 1. Set up AutoML Vision
AutoML Vision 為訓練圖像分類模型並在其上生成預測的所有步驟提供了界面。首先啟用 AutoML API。

再左側選單中點選 `APIs & Services > Library`，搜索欄中輸入 `Cloud AutoML API`。點擊 Cloud AutoML API 結果，然後點擊 Enable。

### Create a Cloud Storage bucket for your training data

在 Cloud Shell 中，貼上以下以創建一個新儲存桶來保存訓練資訊。我們使用變量 `$DEVSHELL_PROJECT_ID` 來知道當前的專案，並在末尾添加 `-vcm` 即可。
```shell
gsutil mb -p $DEVSHELL_PROJECT_ID \
    -c regional    \
    -l us-central1 \
    gs://$DEVSHELL_PROJECT_ID-vcm/
```

打開一個新的瀏覽器，然後到 AutoML UI，驗證 API 後，將進入 AutoML Vision 數據集頁面。
![](https://cdn.qwiklabs.com/98VPTpEJjEeJ20gdGWgx1sAolKY8Lz1X7Mz6jsiimko%3D)

## Task 2. Upload training images to Cloud Storage
為了訓練模型對雲的圖像進行分類，需要提供有標籤的訓練數據，以便模型可以加深對與不同類型的雲相關聯的圖像特徵的理解。在此範例中，模型將學習對三種不同類型的雲進行分類：cirrus、cumulus 和 cumulonimbus。要使用 AutoML Vision，需要將訓練圖像放置在 Cloud Storage 中。

選擇 Storage > Browser，執行以下命令用來將訓練的圖片複製到儲存桶中
```shell
gsutil -m cp -r gs://automl-codelab-clouds/* gs://$DEVSHELL_PROJECT_ID-vcm/
gsutil ls gs://$DEVSHELL_PROJECT_ID-vcm/ # 查看三種不同的雲
```

### Optional: View the images using the Cloud Storage Console UI
圖片完成複製後，點擊 `Cloud Storage` 頁面頂部的 Refresh 按鈕。然後點擊儲存桶名稱，對於應該分類的 3 種不同的雲類型，應該會看到 3 種目錄：
![](https://cdn.qwiklabs.com/iTWcjepUqKcvjRXi04RAZl%2BFBGVXg1DyCXLlyZT7oYY%3D)

## Task 3. Create an AutoML Vision training dataset
訓練數據在 Cloud Storage 中，需要一種 AutoML Vision 訪問它的方法。創建一個 CSV ，其中每一行都包含一個訓練圖像的 URL 和該圖像的相關標籤。該 CSV已在此範例中創建；只需要使用儲存桶名稱進行更新即可。

- Copy the template file to your Cloud Shell instance
- Update the CSV with the files in your project
- Upload this file to your Cloud Storage bucket
- Show the bucket to confirm the data.csv file is present

```shell
gsutil cp gs://automl-codelab-metadata/data.csv .
head --lines=10 data.csv
sed -i -e "s/placeholder/$DEVSHELL_PROJECT_ID-vcm/g" ./data.csv
head --lines=10 data.csv
gsutil cp ./data.csv gs://$DEVSHELL_PROJECT_ID-vcm/
gsutil ls gs://$DEVSHELL_PROJECT_ID-vcm/
```

```shell
gsutil ls gs://$DEVSHELL_PROJECT_ID-vcm/* # 查看目錄，Storeage 面板下也能查看
```

瀏覽回到 [AutoML Vision](https://console.cloud.google.com/vision/datasets) 數據集頁面。

![](https://cdn.qwiklabs.com/98VPTpEJjEeJ20gdGWgx1sAolKY8Lz1X7Mz6jsiimko%3D)

點擊 `+ New dataset`，輸入 `clouds` 作為名稱，選擇 `Single-label Classification`，最後點擊 `Create dataset`。
![](https://cdn.qwiklabs.com/kcW88jGJf8MImoDhmT94CogyHkcWM9WC%2BFVx3%2F%2Bt4Cg%3D)

接著將選擇訓練圖片的位置，選擇`Select a CSV file on Cloud Storage`，然後將檔案名添加到上一個剪貼板中文件的 URL 中。也可以使用瀏覽功能查找 csv 檔案，看到綠色的白色複選框後，可以選擇 `Continue` 以接續後續。

![](https://cdn.qwiklabs.com/kos32CKiB4D2YGyLvTlFy8AvENlU4f7fAHsd%2BegCkWY%3D)

導入完成後，點擊 Images 選項以查看數據集中的圖像。

## Task 4. Inspect the images

接下來，對圖片進行簡單檢查。

![](https://cdn.qwiklabs.com/HQI8YiKTkZbC5AtkVWLK%2FlMIIGtGtn0lyWXC8CitmGc%3D)

按左側 menu 中的不同標籤進行過濾以查看訓練圖像：
![](https://cdn.qwiklabs.com/JCLK1LJX2s8Ys6F1h%2BU8sk%2FfzbM7MHd6GqGvk3AWL9U%3D)


如果任何圖片標籤有錯誤，則可以點擊它們以切換標籤或從訓練集中刪除圖像：
![](https://cdn.qwiklabs.com/kHAeXRUT6rmAE9L40spHuXF65dCze2qExoLglLFGCpU%3D)

要查看每個標籤有多少張圖片的摘要紀錄，可點擊`Label stats`。應該看到瀏覽器右側顯示以下彈出框，查看列表後，按 Done。

![](https://cdn.qwiklabs.com/4O66Gd4AF5J4yXkYG4%2FeuMAnrNDqMotUDvjVxlHy0nw%3D)

## Task 5. Train your model
AutoML Vision 會自動處理一切，而無需編寫任何模型代碼。

要訓練雲模型，需到 `Train` 選項，然後點擊 Start training。

![](https://cdn.qwiklabs.com/yylK8%2Bx6T%2FDeodFCYkGvnmisu%2Beg6dWCtR%2BaC7U7HdU%3D)

輸入模型的名稱，或使用默認的自動生成名稱。

保持 Cloud hosted 為選中狀態，然後點擊 Continue。
![](https://cdn.qwiklabs.com/un1uIWJoWbt%2BkAK7oa9TMsuayRJgZdzXYJLf%2Bv%2FHXAE%3D)

對於下一步，在 `Set your budget *` 框中輸入值 `8`，然後選中 `Deploy model to 1 node after training`。測試完成後，此過程(自動部署)將使模型可立即用於預測。

![](https://cdn.qwiklabs.com/oXtpuux6dk8eTh6uFixd2XAlB15iDN8Suic7UV6uv%2BY%3D)


![](https://i.imgur.com/4KzWA0G.png)
## Task 6. Evaluate your model

訓練完成後，選擇 `Evaluate` 選項。在這裡，將看到有關模型的 Precision 和 Recall 訊息。它應類似於以下內容：

![](https://i.imgur.com/L87Jf6i.png)

也可以調整 `Confidence threshold` 以查看其影響。最後，向下滾動以查看 `Confusion matrix`。

![](https://i.imgur.com/Yu8B3N1.png)

此選項提供了一些常用的機器學習指標，以評估模型的準確性並查看可以在何處改進訓練數據。

## Task 7. Generate predictions
使用以前從未見過的數據在經過訓練的模型上生成預測。

有幾種生成預測的方法。在本實驗中，將使用 UI 上傳圖像。將看到模型如何對這兩個圖像進行分類，分別是 cirrus 和 cumulonimbus，如下圖

![](https://cdn.qwiklabs.com/N2psyplM3kFK9NEjjpak3CPIhh8IurY7Tn9vqzi4r8M%3D)
![](https://cdn.qwiklabs.com/GZlBRmAKGzsDoT8yNRCh6VmxflxLEQEkiKPohYwja94%3D)

在 AutoML UI 中至 Test & Use，在此頁面上，會看到 Model 列表中列出剛剛訓練和部署的模型。點擊 Upload images ，然後將上面提供的兩張圖的雲樣本圖片上傳到本地硬碟。

![](https://cdn.qwiklabs.com/Ir1KSf2Q7BRsmj6Hg4AcgyUE2PwNldl9HQ%2FStBqwJr4%3D)

預測請求完成後，會看到以下內容：

![](https://cdn.qwiklabs.com/btq3FuO13W1gt3MdHgzlJO1mr11X5VzDQKj%2FdDosvoQ%3D)

![](https://cdn.qwiklabs.com/HH5%2FheO%2BD08L8SAdzpHSumH9yDdAeYAv9dMk%2FHu4EtA%3D)



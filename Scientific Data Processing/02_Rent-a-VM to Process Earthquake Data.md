## Create Compute Engine instance with the necessary API access

建立 `Compute Engine` 實例，從 `Navigation menu` 點擊 `Compute Engine > VM instances`，使用預設區域，對 `Compute Engine` 預設 `service account` 的 `Identify API access` 改為 `Allow full access to all Cloud APIs`，最後單擊 `Create`

![](https://cdn.qwiklabs.com/%2BMV9Q1n1BHGkhXQUsQnmjMNRbt7Ol4ZYlpBhqaHtbig%3D)
![](https://cdn.qwiklabs.com/00OsxRQ2rsnhNG8ViAeLRmPkoMRcfuiYj0fAWvIY78M%3D)
![](https://cdn.qwiklabs.com/SzmD3Iuxsu2OVsSRjA0KgrOk9rwKr5FiiOy5FcvS8KE%3D)

## SSH into the instance


![](https://cdn.qwiklabs.com/nxsdG7HoU9zPMXQU%2FMfvyqQykjxkw3XxN9f0qrOb9lA%3D)


查看 `Compute Engine` 建立的虛擬機訊息
```shell
cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 63
model name      : Intel(R) Xeon(R) CPU @ 2.30GHz
stepping        : 0
microcode       : 0x1
cpu MHz         : 2300.000
cache size      : 46080 KB
physical id     : 0
siblings        : 2
core id         : 0
cpu cores       : 1
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
```

## Install software

```shell
sudo apt-get update
sudo apt-get -y -qq install git
sudo apt-get install python-mpltoolkits.basemap
```

## Ingest USGS data
```shell
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
cd training-data-analyst/CPB100/lab2b
bash ingest.sh
```

`ingest.sh` 從美國地質調查局下載過去 7 天的地震數據集。

## Transform the data
使用 `Python` 將數據轉換為地震活動圖，可參考[notebook](https://github.com/GoogleCloudPlatform/datalab-samples/blob/master/basemap/earthquakes.ipynb)

```shell
bash install_missing.sh # 安裝套件
python3 transform.py
```
完成後輸入 `ls -l`，會看到 `earthquakes.png`

```shell
total 764
-rw-r--r-- 1 student-03-6a38d5614f73 google-sudoers    637 Sep 10 03:44 commands.sh
-rw-r--r-- 1 student-03-6a38d5614f73 google-sudoers 427921 Sep 10 03:44 earthquakes.csv
-rw-r--r-- 1 student-03-6a38d5614f73 google-sudoers    751 Sep 10 03:44 earthquakes.htm
-rw-r--r-- 1 student-03-6a38d5614f73 google-sudoers 324089 Sep 10 03:46 earthquakes.png
-rwxr-xr-x 1 student-03-6a38d5614f73 google-sudoers    786 Sep 10 03:44 ingest.sh
-rwxr-xr-x 1 student-03-6a38d5614f73 google-sudoers    707 Sep 10 03:44 install_missing.sh
drwxr-xr-x 2 student-03-6a38d5614f73 google-sudoers   4096 Sep 10 03:44 scheduled
-rwxr-xr-x 1 student-03-6a38d5614f73 google-sudoers   3058 Sep 10 03:44 transform.py
```

## Create a Cloud Storage bucket
從 `Navigation menu` 點選 `Storage`，點擊 `Create Bucket`，為建立的桶輸入為一名稱，`Regional` 可以和 `VM` 同一個以提高傳輸速率也可以預設。


## Store data
學習如何在 `Cloud Storage` 中儲存原始數據和轉換後的數據。在 `Compute Engine` 的 `VM` `SSH` 畫面中輸入
```shell
gsutil cp earthquakes.* gs://<YOUR-BUCKET>/earthquakes/ # <YOUR-BUCKET> 剛輸入的唯一名稱
Copying file://earthquakes.csv [Content-Type=text/csv]...
Copying file://earthquakes.htm [Content-Type=text/html]...                      
Copying file://earthquakes.png [Content-Type=image/png]...                      
- [3 files][735.1 KiB/735.1 KiB]                                                
Operation completed over 3 objects/735.1 KiB.  
```

完成後，可以瀏覽 `Storage -> Browser` 的頁面，會有一下資訊

![](https://i.imgur.com/5p00ueo.png)

## Publish Cloud Storage files to web
將儲存桶中的檔案發佈到 `Web`。點擊 `earthquakes.htm` 檔案，再點擊三個點中的 `Edit Permissions`，從中再點擊 `+ Add entry`，輸入以下內容為所有用戶添加權限，同時 `earthquakes.png` 也要如上設定。

- `Entity` 欄位選 `Public`
- `Name` 欄位輸入 `allUsers`
- Access 欄位選 `Reader` 
- 最後 `Save`

![](https://i.imgur.com/zJQoTM9.png)

![](https://cdn.qwiklabs.com/HNVCDhrj7iYd%2Fb4X3UCof0wL4EQ205FEEfXKWifsWSQ%3D)\

最後再連覽器上輸入 `https://storage.googleapis.com/YOUR-BUCKET-NAME/earthquakes/earthquakes.htm` 就可以顯示該地震活動圖
## Create a Cloud Storage bucket
```shell
BUCKET_NAME=<YOUR_NAME>_enron_corpus
gsutil mb gs://${BUCKET_NAME}
```

## Check out the data
Enron Corpus 是一個很大的數據，包含由 Enron Corporation 公司 158 名員工生成的 600,000 封電子郵件。此數據已復製到 Cloud Storage 儲存`gs://enron_emails/`。

將其複製
```shell
gsutil cp gs://enron_emails/allen-p/inbox/1. .
```

## Enable Cloud KMS
[Cloud KMS](https://cloud.google.com/kms/) 是 Google Cloud 上的加密密鑰管理服務。使用 KMS 之前，需要啟用它。

```shell
gcloud services enable cloudkms.googleapis.com
```

## Create a Keyring and Cryptokey
為了加密數據，需要創建一個 `KeyRing` 和一個 `CryptoKey`。`KeyRings` 可用於對密鑰進行分組。密鑰可以按環境，例如 test、staging 或 prod等或是其他一些概念上的分組。

```shell
KEYRING_NAME=test CRYPTOKEY_NAME=qwiklab
```
建立 KMS

```shell
gcloud kms keyrings create $KEYRING_NAME --location global
```
使用新的 `KeyRing` 創建一個名為 `qwiklab` 的 `CryptoKey`。
```shell
gcloud kms keys create $CRYPTOKEY_NAME --location global \
      --keyring $KEYRING_NAME \
      --purpose encryption
```

![](https://i.imgur.com/hLAa3Bm.png)

>無法在 Cloud KMS 中刪除 CryptoKeys 和 KeyRings！

調制`Navigation menu > Security > Cryptographic keys`，透過控制台打開 [Cryptographic Keys](https://console.cloud.google.com/iam-admin/kms):

![](https://cdn.qwiklabs.com/UOh1NZtUw6oJdrAc1OJavTKPMoxIshshCVYaGzJaepE%3D)

`Key Management` UI 允許查看和管理 CryptoKey 和 KeyRing。
![](https://cdn.qwiklabs.com/FFwZ7wLrco%2Fdgzcl53oieGgsnFBADYEecE24Roan%2FSU%3D)


## Encrypt Your Data
```shell
PLAINTEXT=$(cat 1. | base64 -w0)
```

使用加密端點，可將要加密的 base64 編碼文本發送到指定的密鑰。

```shell
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
  -d "{\"plaintext\":\"$PLAINTEXT\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type: application/json"
```

>即使使用相同的文本和密鑰，每次加密操作也會返回不同的結果。


```shell
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
  -d "{\"plaintext\":\"$PLAINTEXT\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type:application/json" \
| jq .ciphertext -r > 1.encrypted
```

要驗證加密的數據是否可以解密，請調用解密端點以驗證解密的文本與原始電子郵件匹配。加密的數據包含有關使用哪個 `CryptoKey` 版本對其進行加密的訊息，因此不會將特定版本提供給解密端點。

```shell
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:decrypt" \
  -d "{\"ciphertext\":\"$(cat 1.encrypted)\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type:application/json" \
| jq .plaintext -r | base64 -d
```

>更多範例可以查看此鏈結[Cloud KMS Quickstart](https://cloud.google.com/kms/docs/quickstart)


將加密的文件上傳到 Cloud Storage。

```shell
gsutil cp 1.encrypted gs://${BUCKET_NAME}
```

## Configure IAM Permissions
在KMS中，有兩個主要權限需要關注。一種權限允許用戶或服務帳戶**管理 KMS 資源**，另一種權限允許用戶或服務帳戶使用密鑰來**加密和解密**數據。

管理密鑰的權限是 `cloudkms.admin`，它允許任何有權創建密鑰環以及創建、修改、禁用和銷毀 `CryptoKey` 的人。加密和解密的權限為 `cloudkms.cryptoKeyEncrypterDecrypter`，用於調用加密和解密 API 端點。

獲取當前分配的 IAM 權限
```shell
USER_EMAIL=$(gcloud auth list --limit=1 2>/dev/null | grep '@' | awk '{print $2}')
```
分配管理 KMS 資源的能力。運行以下 gcloud 命令以分配 IAM 權限來管理創建的 `KeyRing`：
```shell
gcloud kms keyrings add-iam-policy-binding $KEYRING_NAME \
    --location global \
    --member user:$USER_EMAIL \
    --role roles/cloudkms.admin
```

由於 `CryptoKeys` 屬於 `KeyRing`，而 `KeyRings` 屬於 `Project`，因此在該層次結構中具有特定角色或更高級別權限的用戶將繼承子資源的相同權限。例如，在項目中具有所有者角色的用戶同時也是該項目中所有 `KeyRings` 和 `CryptoKeys` 的所有者。同樣，如果為用戶授予了 `KeyRing` 上的`cloudkms.admin` 角色，則他們對該 `KeyRing` 中的 `CryptoKeys` 具有關聯的權限。

沒有 `cloudkms.cryptoKeyEncrypterDecrypter` 權限，將無法授權用戶使用密鑰來加密或解密數據。運行以下 `gcloud` 命令，為創建的 `KeyRing` 下的任何 `CryptoKey` 分配 `IAM` 權限以加密和解密數據：

```shell
gcloud kms keyrings add-iam-policy-binding $KEYRING_NAME \
    --location global \
    --member user:$USER_EMAIL \
    --role roles/cloudkms.cryptoKeyEncrypterDecrypter
```

現在可以在 `Key Management` 的 `Cryptographic Keys` 部分中查看分配的權限。點擊 `Permissions`，將可以看到分配的權限

![](https://i.imgur.com/1NNWLrz.png)


## Back up data on the Command Line
將 `allen-p` 的所有電子郵件複製到當前工作目錄中
```shell
gsutil -m cp -r gs://enron_emails/allen-p .
```

將所有電郵進行加密並上傳 `Cloud Storage`
```shell
MYDIR=allen-p
FILES=$(find $MYDIR -type f -not -name "*.encrypted")
for file in $FILES; do
  PLAINTEXT=$(cat $file | base64 -w0)
  curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
    -d "{\"plaintext\":\"$PLAINTEXT\"}" \
    -H "Authorization:Bearer $(gcloud auth application-default print-access-token)" \
    -H "Content-Type:application/json" \
  | jq .ciphertext -r > $file.encrypted
done
gsutil -m cp allen-p/inbox/*.encrypted gs://${BUCKET_NAME}/allen-p/inbox
```

完成後可至 ` Storage > Browser > YOUR_BUCKET > allen-p > inbox`  查看

![](https://i.imgur.com/qlNfcQM.png)

## View Cloud Audit Logs
Google Cloud Audit Logging 由兩個日誌流(管理員活動和數據訪問)組成，它們由 Google Cloud 服務生成。

要查看 KMS 中任何資源的活動，請返回 `Cryptographic keys` 頁面(Navigation menu > Security > Cryptographic keys)，點選 key ring 旁邊的框，然後單擊右側選單中的 `Activity` 選項。它將帶到 Cloud Activity UI，在其應該看到對 KeyRing 的創建和所有修改。

![](https://i.imgur.com/UdENHcY.png)
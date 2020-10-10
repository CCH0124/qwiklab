## Introduction
通常需要對 Web 應用程式的用戶進行身份驗證，且通常需要對應用程式進行特殊編程。對於 Google Cloud Platform 應用程式，可將這些職責移交給[Identity-Aware Proxy](https://cloud.google.com/iap/)服務。如果只需要限制對選定用戶的訪問，則無需對應用程式進行任何更改，如果需要知道用戶的身份，例如在服務器端保留用戶偏好，Identity-Aware Proxy 可為用戶提供最少的應用程序代碼。

### What is Identity-Aware Proxy?
Identity-Aware Proxy(IAP) 是 Google Cloud Platform 服務，它攔截發送到應用程式的 Web 請求，使用 Google Identity Service 對發出請求的用戶進行身份驗證，並僅在請求來自授權的用戶時才允許存取。另外，它可以修改請求標頭以包括有關已認證用戶的訊息。


## Restrict Access with IAP
點擊 `Navigation menu > Security > Identity-Aware Proxy`，再點擊 `Enable API`，再點擊 `Go to Identity-Aware Proxy`，再點擊 ` CONFIGURE CONSENT SCREEN`，接著在`User Type`下選擇`Internal`，然後單擊`Create`。

返回到 `Identity-Aware Proxy`，應該會看到可保護的資源列表。在 App Engine 應用行的 IAP 列中觸發切換按鈕，以打開 IAP。該域將受到 IAP 保護，點擊 `TURN ON`。

![](https://i.imgur.com/S384LXh.png)
```shell
export my_zone=us-central1-a
export my_cluster=standard-cluster-1
gcloud container clusters create $my_cluster --num-nodes 3 --zone $my_zone --enable-ip-alias
```

##  Modify GKE clusters
修改節點
```shell
gcloud container clusters resize $my_cluster --zone $my_zone --num-nodes=4
```

## Connect to a GKE cluster
使用當前用戶的憑證創建 kubeconfig 檔案，如果該命令尚不存在，它將在主目錄中創建一個 `.kube` 目錄。在 `.kube` 目錄中，該命令將創建一個 `config` 的檔案（如果不存在），該檔案用於儲存身份驗證和配置訊息。該配置檔案通常稱為 `kubeconfig` 檔案。
```shell
gcloud container clusters get-credentials $my_cluster --zone $my_zone
Fetching cluster endpoint and auth data.
kubeconfig entry generated for standard-cluster-1.
```

```shell
vi ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLakNDQWhLZ0F3SUJBZ0lRWVNEUy9ZR2h3SEVvQnpjdjBWemM4REFOQmdrcWhraUc5dzBCQVFzRkFEQXYKTVMwd0t3WURWUVFERXlRNVpHRTFNR0ZsTWkxbFpHRTRMVFEwWlRndFlqaG1ZaTB6TlRkak5qUmlPR1l4WkdRdwpIaGNOTWpBeE1ESXpNRFkxTURRM1doY05NalV4TURJeU1EYzFNRFEzV2pBdk1TMHdLd1lEVlFRREV5UTVaR0UxCk1HRmxNaTFsWkdFNExUUTBaVGd0WWpobVlpMHpOVGRqTmpSaU9HWXhaR1F3Z2dFaU1BMEdDU3FHU0liM0RRRUIKQVFVQUE0SUJEd0F3Z2dFS0FvSUJBUURLRTFvSTIzYXpTRlAzcGZ0Zk5xNnZYblRBTHFNUGJaNkVBaEp1TFJwcQpoTkttTU4xUGxDWlNvQXpDSmNKekpWQ2lybVhETExGZjVhNmY4ZFBtYjVsVzNJZHpRVTQzSEtNank5M1prU2RhClpYNGd3aXdmTUNycWlUNmIyY1FKSWtDOC8zRitFZi9ibG5PSE4yNnd5V1JZN2NxZ2RnRllSZ3dOdXdaYjJpangKajJua0NmWHZ4R3ZtU2RHZE91bXFQY1hDbWN6a1FIQXBqV1lkbVBNRndGZzJRclh4QzJUL0dGSEs0aVcxOXB5bApWd2NuQWlSZlRlbWhVcENQdU15dGJzbklUNmtGb0daU3VkQkp6d1FqTWtCWmlpZ01RblN0dDh0djBHaTdOTVQyCjhPUWtTQVRpMmhjL2xJeE5UaHpDSFltWWxrSStnUUkrTDZPbVZjY2JMRGVuQWdNQkFBR2pRakJBTUE0R0ExVWQKRHdFQi93UUVBd0lDQkRBUEJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJSL3BXbkhTblZzdEdxMwpvckJmMWphMFVpdERuVEFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBbHZlYVNnajkxV05PZHJjTFpEcDc3bDlwClVhQUNDV1k4OTBxdVJBQ3dVM2pCeTZGVG9uVS9NZjY4emdiK2l2MU93dXJZM2lSWGZrQ0MyOS83Sng5WEFZUmMKRmxhRXFtN1dLWTY1WGxTdTlXMXpoZEN0djFXYkVkc00yMFVxNG5DOVIydTRnSGF5V2ROQ0l6c3d3a2hMWlV3dwppZ2N6dmdxUzQvckZkWE5yT0NRZXJPYjRSdWN3dTZrcEpXNnRMSXBpODB2NGZPbFBaaFNhM3FpT1BPaVJ3b3NCCkdNTXdsNE1DenFHL2p1dDRlTWgrUGRFb2RoOXRvNEVEZk9ydVlpOEh0RndzWU5nK042WUdIOG45aWVCZmlrQ20KZVRVczc4d1laZWtKRzJwWkRZWldWYjdITG9FSHdKUU40RnpoaytJeTl1U2RFOWM3TlFINFZERHBBSEZQc1E9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://34.121.200.236
  name: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
contexts:
- context:
    cluster: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
    user: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
  name: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
current-context: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
kind: Config
preferences: {}
users:
- name: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
  user:
    auth-provider:
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /usr/lib/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

>Note: The kubeconfig file can contain information for many clusters. The currently active context (the cluster that kubectl commands manipulate) is indicated by the current-context property.
You don't have to run the gcloud container clusters get-credentials command to populate the kubeconfig file for clusters that you created in the same context (the same user in the same environment), because those clusters already have their details populated when the cluster is created. But you have to run the command to connect to a cluster created by another user or in another environment. The command is also an easy way to switch the active context to a different cluster.

## Use kubectl to inspect a GKE cluster
```shell
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://34.121.200.236
  name: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
contexts:
- context:
    cluster: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
    user: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
  name: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
current-context: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
kind: Config
preferences: {}
users:
- name: gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
  user:
    auth-provider:
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /usr/lib/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```
敏感的證書數據替換為 `DATA + OMITTED`。

```shell
$ kubectl cluster-info
Kubernetes master is running at https://34.121.200.236
GLBCDefaultBackend is running at https://34.121.200.236/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
KubeDNS is running at https://34.121.200.236/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://34.121.200.236/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```shell
$ kubectl config current-context
gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
```

```shell
$ kubectl config get-contexts
CURRENT   NAME                                                                CLUSTER                                                             AUTHINFO                                                            NAMESPACE
*         gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1   gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1   gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
```

```shell
$ kubectl config use-context gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1
Switched to context "gke_qwiklabs-gcp-03-8cbda212e4bf_us-central1-a_standard-cluster-1".
student_03_9b15aaf4e005@cloudshell:~ (qwiklabs-gcp-03-8cbda212e4bf)$
```
當有多個集群時，可使用此方式切換


```shell
$ kubectl top nodes
NAME                                                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
gke-standard-cluster-1-default-pool-e3946400-4jxl   50m          5%     670Mi           25%
gke-standard-cluster-1-default-pool-e3946400-kqts   83m          8%     715Mi           27%
gke-standard-cluster-1-default-pool-e3946400-lpjz   41m          4%     680Mi           25%
gke-standard-cluster-1-default-pool-e3946400-w5v0   38m          4%     562Mi           21%
```

自動補全
```shell
source <(kubectl completion bash)
```


## Deploy Pods to GKE clusters

```shell
kubectl create deployment --image nginx nginx-1
```
```shell
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
nginx-1-d5c9f69b7-8z2sf   1/1     Running   0          15s
```

```shell
$ kubectl describe pod nginx-1-d5c9f69b7-8z2sf
Name:         nginx-1-d5c9f69b7-8z2sf
Namespace:    default
Priority:     0
Node:         gke-standard-cluster-1-default-pool-e3946400-w5v0/10.128.0.5
Start Time:   Fri, 23 Oct 2020 16:08:46 +0800
Labels:       app=nginx-1
              pod-template-hash=d5c9f69b7
Annotations:  kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container nginx
Status:       Running
IP:           10.8.3.2
IPs:
  IP:           10.8.3.2
...
```

### Push a file into a container
```shell
$ vi ~/test.html
<html> <header><title>This is title</title></header>
<body> Hello world </body>
</html>
```

```shell
kubectl cp ~/test.html $my_nginx_pod:/usr/share/nginx/html/test.html
```

### Expose the Pod for testing
```shell
kubectl expose pod $my_nginx_pod --port 80 --type LoadBalancer
```

```shell
kubectl get services
$ kubectl get services
NAME                      TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes                ClusterIP      10.12.0.1     <none>        443/TCP        18m
nginx-1-d5c9f69b7-8z2sf   LoadBalancer   10.12.15.49   <pending>     80:32514/TCP   6s
```


```shell
$ kubectl top pods
NAME                      CPU(cores)   MEMORY(bytes)
nginx-1-d5c9f69b7-8z2sf   0m           2Mi
```

## Introspect GKE Pods

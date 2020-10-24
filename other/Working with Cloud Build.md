## Building Containers with DockerFile and Cloud Build
```shell
vi quickstart.sh
#!/bin/sh
echo "Hello, world! The time is $(date)."
vi Dockerfile
FROM alpine
COPY quickstart.sh /
CMD ["/quickstart.sh"]
chmod +x quickstart.sh
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/quickstart-image .
```
到 Container Registry > Images
![](https://i.imgur.com/RDj7C6b.png)

![](https://i.imgur.com/aKF4xbs.png)

## Building Containers with a build configuration file and Cloud Build

```shell
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
ln -s ~/training-data-analyst/courses/ak8s/v1.1 ~/ak8s
cd ~/ak8s/Cloud_Build/a
cat cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-t', 'gcr.io/$PROJECT_ID/quickstart-image', '.' ]
images:
- 'gcr.io/$PROJECT_ID/quickstart-image'
gcloud builds submit --config cloudbuild.yaml .
```

![](https://i.imgur.com/hpzSCgb.png)

![](https://i.imgur.com/gVdU0hD.png)


## Building and Testing Containers with a build configuration file and Cloud Build
```shell
cd ~/ak8s/Cloud_Build/b
cat cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: [ 'build', '-t', 'gcr.io/$PROJECT_ID/quickstart-image', '.' ]
- name: 'gcr.io/$PROJECT_ID/quickstart-image'
  args: ['fail']
images:
- 'gcr.io/$PROJECT_ID/quickstart-image
```
```shell
gcloud builds submit --config cloudbuild.yaml .

echo $?
```
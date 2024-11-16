# Домашнее задание к занятию «Helm»

### Цель задания

В тестовой среде Kubernetes необходимо установить и обновить приложения с помощью Helm.

------


### Задание 1. Подготовить Helm-чарт для приложения

Упаковал приложение в чарт для деплоя в разные окружения:
```bash 
user@microk8s:~/kuber-homeworks-2.5$ helm create nginx-chart
Creating nginx-chart
user@microk8s:~/kuber-homeworks-2.5$ cd nginx-chart/
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ ls
Chart.yaml  charts  templates  values.yaml
```

Очистил директорию templates и создал шаблонизированный манифест _deployment.yaml_ :
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: {{ .Values.image.name }}:{{ .Values.image.tag }}
        ports:
         - containerPort: {{ .Values.cp }}

```
Так же очистил файл _values.yaml_ и наполнил его своим содержимым:
```yml
image:
  name: nginx
  tag: latest
replicas: 3
cp: 80
```
Проверяю шаблон:
```bash
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm template nginx-chart .
---
# Source: nginx-chart/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
         - containerPort: 80
```
Запускаю шаблон:
```bash
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm install nginx .
NAME: nginx
LAST DEPLOYED: Sat Nov 16 05:49:47 2024
NAMESPACE: homework
STATUS: deployed
REVISION: 1
TEST SUITE: None
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ kubectl get po
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54b9c68f67-88p6d   1/1     Running   0          38s
nginx-deployment-54b9c68f67-np6jq   1/1     Running   0          38s
nginx-deployment-54b9c68f67-s95k5   1/1     Running   0          38s
```

------
### Задание 2. Запустить две версии в разных неймспейсах
Создаю namespace:
```bash
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ kubectl create namespace app1
namespace/app1 created
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ kubectl create namespace app2
namespace/app2 created
```

Запускаю одну версию в namespace=app1, вторую версию в том же неймспейсе, третью версию в namespace=app2.
```bash
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm install nginx-v1 . --namespace app1 --set image.tag=1.21.0
NAME: nginx-v1
LAST DEPLOYED: Sat Nov 16 06:38:18 2024
NAMESPACE: app1
STATUS: deployed
REVISION: 1
TEST SUITE: None
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm install nginx-v2 . --namespace app1 --set image.tag=1.21.1
NAME: nginx-v2
LAST DEPLOYED: Sat Nov 16 06:38:30 2024
NAMESPACE: app1
STATUS: deployed
REVISION: 1
TEST SUITE: None
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm install nginx-v3 . --namespace app2 --set image.tag=1.21.2
NAME: nginx-v3
LAST DEPLOYED: Sat Nov 16 06:39:39 2024
NAMESPACE: app2
STATUS: deployed
REVISION: 1
TEST SUITE: None
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm list -n app1
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
nginx-v1        app1            1               2024-11-16 06:38:18.997164833 +0000 UTC deployed        nginx-chart-0.1.0       1.16.0     
nginx-v2        app1            1               2024-11-16 06:38:30.322145651 +0000 UTC deployed        nginx-chart-0.1.0       1.16.0     
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm list -n app2
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
nginx-v3        app2            1               2024-11-16 06:39:39.18888957 +0000 UTC  deployed        nginx-chart-0.1.0       1.16.0     
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ 
```

------

### PS 
Во время выполнения 2-й части задания, при использовании деплоймента из 1 части задания, я столкнулся с ошибкой когда пытался запустить вторую копию приложения в том же namespace:
```bash
ser@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm install nginx-v2 . --namespace app1 --set image.tag=1.21.1 
Error: INSTALLATION FAILED: Unable to continue with install: Deployment "nginx-deployment" in namespace "app1" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-name" must equal "nginx-v2": current value is "nginx-v1"
user@microk8s:~/kuber-homeworks-2.5/nginx-chart$ helm delete nginx-v1 -n app1
```
Решил её модифицировав деплоймент следующим образом:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx-deployment
  labels:
    app: {{ .Release.Name }}-nginx
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-nginx
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-nginx
    spec:
      containers:
      - name: nginx
        image: {{ .Values.image.name }}:{{ .Values.image.tag }}
        ports:
         - containerPort: {{ .Values.cp }}
```


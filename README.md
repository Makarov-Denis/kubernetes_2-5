# Домашнее задание к занятию "`Helm`" - `Макаров Денис`

---

### Задание 1. Подготовить Helm-чарт для приложения

1. Необходимо упаковать приложение в чарт для деплоя в разные окружения. 
2. Каждый компонент приложения деплоится отдельным deployment’ом или statefulset’ом.
3. В переменных чарта измените образ приложения для изменения версии.

#### Решение

Проверяем установку ```helm```:

```
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm version
version.BuildInfo{Version:"v3.16.3", GitCommit:"cfd07493f46efc9debd9cc1b02a0961186df7fdf", GitTreeState:"clean", GoVersion:"go1.22.7"}
```

Подготовим ```deployment``` ```service``` и разместим их в директории ```templates/```:

- [templates/deployment.yaml](kubernetes_2-5/src/netology-chart/templates/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main
  labels:
    app: main
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: main
  template:
    metadata:
      labels:
        app: main
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: {{ .Values.appPort }}
              protocol: TCP
```
- [templates/service.yaml](kubernetes_2-5/src/netology-chart/templates/service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: main
  labels:
    app: main
spec:
  ports:
    - port: {{ .Values.appPort }}
      name: http
  selector:
    app: main
```
Подготовим ```Chart```: 
- [Chart.yaml](kubernetes_2-5/src/netology-chart/Chart.yaml)
```yaml
  apiVersion: v2
  name: netology

  type: application

  version: 0.1.2
  appVersion: "1.16.0"
```
Подготовим ```values```:
- [values.yaml](kubernetes_2-5/src/netology-chart/values.yaml)
```yaml
replicaCount: 1

image:
  repository: nginx
  tag: ""

appPort: 80
```
Создадим шаблон на основе этих файлов:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm template netology-chart
---
# Source: netology/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: main
  labels:
    app: main
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: main
---
# Source: netology/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main
  labels:
    app: main
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main
  template:
    metadata:
      labels:
        app: main
    spec:
      containers:
        - name: netology
          image: "nginx:1.16.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP

Поменяем номер версии приложения в файле ```Chart.yaml```:
```yaml
  apiVersion: v2
  name: netology

  type: application

  version: 0.1.2
  appVersion: "1.19.0"
```
Смотрим шаблон:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm template netology-chart
---
# Source: netology/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: main
  labels:
    app: main
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: main
---
# Source: netology/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main
  labels:
    app: main
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main
  template:
    metadata:
      labels:
        app: main
    spec:
      containers:
        - name: netology
          image: "nginx:1.19.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```
Видим, что версия образа поменялась на ```1.19.0```
Внесем версию в файл ```values.yaml```:

```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.20.0"

appPort: 80
```
Проверим шаблон теперь:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm template netology-chart
---
# Source: netology/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: main
  labels:
    app: main
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: main
---
# Source: netology/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main
  labels:
    app: main
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main
  template:
    metadata:
      labels:
        app: main
    spec:
      containers:
        - name: netology
          image: "nginx:1.20.0"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```
Видим, что версия образа встала ```1.20.0```, это произошло, потому что при задании условий подстановки переменных было указано, что в первую очередь данные брать из файла с переменными, если в них не будет указано данных, тогда подставлять данные из файла ```Chart.yaml```


------
### Задание 2. Запустить две версии в разных неймспейсах

1. Подготовив чарт, необходимо его проверить. Запуститe несколько копий приложения.
2. Одну версию в namespace=app1, вторую версию в том же неймспейсе, третью версию в namespace=app2.
3. Продемонстрируйте результат.

#### Решение

Проверим установку:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm install netology1 netology-chart
NAME: netology1
LAST DEPLOYED: Mon Dec  2 00:04:29 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
admden@admden-kuber-node:~/kubernetes_2-5/src$ kubectl get all
NAME                       READY   STATUS              RESTARTS       AGE
pod/main-9cc98f77c-ws7v6   0/1     ContainerCreating   0              19s
pod/test-multitool         1/1     Running             1 (109m ago)   5d11h

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP   25d
service/main         ClusterIP   10.152.183.116   <none>        80/TCP    19s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/main   0/1     1            0           19s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/main-9cc98f77c   1         1         0       19s
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                    STATUS          CHART           APP VERSION
netology1       default         1               2024-12-02 00:04:29.581048861 +0300 MSK    deployed        netology-0.1.2  1.19.0
```
Обновим установку с изменением параметра ```replicaCount=2```:
```bash
 admden@admden-kuber-node:~/kubernetes_2-5/src$ helm upgrade --install netology1 --set replicaCount=2 netology-chart
Release "netology1" has been upgraded. Happy Helming!
NAME: netology1
LAST DEPLOYED: Mon Dec  2 00:08:21 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None

admden@admden-kuber-node:~/kubernetes_2-5/src$ kubectl get all
NAME                       READY   STATUS    RESTARTS       AGE
pod/main-9cc98f77c-c4x54   1/1     Running   0              10m
pod/main-9cc98f77c-ws7v6   1/1     Running   0              14m
pod/test-multitool         1/1     Running   1 (123m ago)   5d12h

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.152.183.1     <none>        443/TCP   25d
service/main         ClusterIP   10.152.183.116   <none>        80/TCP    14m

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/main   2/2     2            2           14m

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/main-9cc98f77c   2         2         2       14m
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                    STATUS          CHART           APP VERSION
netology1       default         2               2024-12-02 00:08:21.956612502 +0300 MSK    deployed        netology-0.1.2  1.19.0     
```
Удалим данные для продолжения задания:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm uninstall netology1
release "netology1" uninstalled
admden@admden-kuber-node:~/kubernetes_2-5/src$ kubectl get all
NAME                       READY   STATUS    RESTARTS        AGE
pod/main-9cc98f77c-c4x54   1/1     Running   1 (2m40s ago)   15m
pod/main-9cc98f77c-ws7v6   1/1     Running   1 (2m40s ago)   19m
pod/test-multitool         0/1     Unknown   1               5d12h

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.152.183.1   <none>        443/TCP   25d

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/main-9cc98f77c   2         2         2       19m
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm install netology1 --namespace app1 --create-namespace netology-chart
NAME: netology1
LAST DEPLOYED: Mon Dec  2 00:24:46 2024
NAMESPACE: app1
STATUS: deployed
REVISION: 1
TEST SUITE: None
admden@admden-kuber-node:~/kubernetes_2-5/src$ kubectl get all -n app1 
NAME                       READY   STATUS    RESTARTS   AGE
pod/main-9cc98f77c-6vqjr   1/1     Running   0          12s

NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/main   ClusterIP   10.152.183.86   <none>        80/TCP    12s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/main   1/1     1            1           12s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/main-9cc98f77c   1         1         1       12s
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm list --namespace=app1
NAME            NAMESPACE       REVISION        UPDATED                                    STATUS          CHART           APP VERSION
netology1       app1            1               2024-12-02 00:24:46.26547533 +0300 MSK     deployed        netology-0.1.2  1.19.0     
```
Создаем вторую версию в ```namespace=app1```:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm upgrade --install netology2 --namespace app1 --create-namespace netology-chart
Release "netology2" does not exist. Installing it now.
Error: Unable to continue with install: Service "main" in namespace "app1" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-name" must equal "netology2": current value is "netology1"
```
Ошибка, т.к. мы пытаемся запустить несколько приложений в одном пространстве имен, с разными параметрами, но одинаковыми именами. Так нельзя. Для этого необходимо настраивать переменные таким образом, чтобы имена ресурсов приложения так же менялись, тогда возможен запуск дополнительного приложения в этом пространстве имен. Для решения задачи проведем манипуляции вручную перед запуском:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main2
  labels:
    app: main2
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: main2
  template:
    metadata:
      labels:
        app: main2
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: {{ .Values.appPort }}
              protocol: TCP
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: main2
  labels:
    app: main2
spec:
  ports:
    - port: {{ .Values.appPort }}
      name: http
  selector:
    app: main2
```

Запустим повторно:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm upgrade --install netology2 --namespace app1 --create-namespace netology-chart
Release "netology2" does not exist. Installing it now.
NAME: netology2
LAST DEPLOYED: Mon Dec  2 00:32:05 2024
NAMESPACE: app1
STATUS: deployed
REVISION: 1
TEST SUITE: None
admden@admden-kuber-node:~/kubernetes_2-5/src$ kubectl get all -n app1 
NAME                        READY   STATUS    RESTARTS   AGE
pod/main-9cc98f77c-6vqjr    1/1     Running   0          7m29s
pod/main2-66dfb89f7-fvc6g   1/1     Running   0          9s

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/main    ClusterIP   10.152.183.86    <none>        80/TCP    7m29s
service/main2   ClusterIP   10.152.183.234   <none>        80/TCP    9s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/main    1/1     1            1           7m29s
deployment.apps/main2   1/1     1            1           9s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/main-9cc98f77c    1         1         1       7m29s
replicaset.apps/main2-66dfb89f7   1         1         1       9s
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm list --namespace=app1
NAME            NAMESPACE       REVISION        UPDATED                                    STATUS          CHART           APP VERSION
netology1       app1            1               2024-12-02 00:24:46.26547533 +0300 MSK     deployed        netology-0.1.2  1.19.0     
netology2       app1            1               2024-12-02 00:32:05.849119678 +0300 MSK    deployed        netology-0.1.2  1.19.0     
```

Запускаем третью версию в ```namespace=app2```:
```bash
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm upgrade --install netology3 --namespace app2 --create-namespace netology-chart
Release "netology3" does not exist. Installing it now.
NAME: netology3
LAST DEPLOYED: Mon Dec  2 00:34:24 2024
NAMESPACE: app2
STATUS: deployed
REVISION: 1
TEST SUITE: None
admden@admden-kuber-node:~/kubernetes_2-5/src$ helm list --namespace=app2
NAME            NAMESPACE       REVISION        UPDATED                                    STATUS          CHART           APP VERSION
netology3       app2            1               2024-12-02 00:34:24.000131137 +0300 MSK    deployed        netology-0.1.2  1.19.0     
admden@admden-kuber-node:~/kubernetes_2-5/src$ kubectl get all -n app2
NAME                        READY   STATUS    RESTARTS   AGE
pod/main2-66dfb89f7-68krq   1/1     Running   0          41s

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/main2   ClusterIP   10.152.183.223   <none>        80/TCP    41s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/main2   1/1     1            1           41s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/main2-66dfb89f7   1         1         1       41s

```

---
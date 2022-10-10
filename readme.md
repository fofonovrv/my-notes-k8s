# Шпаргалка по Kubernetes

---

[Официальная документация](https://kubernetes.io/ru/docs/home/)

---

## minikube и cubectl

Установим minikube, cubectl и драйвер для использование виртуализации
Миникуб - тренировочный кластер k8s, запускается в одной виртуальной машине

```
wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
minikube version

curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl  /usr/local/bin/
kubectl version -o json
```

#### Если установлен пакет виртуализации libvirt (KVM), то ставим соответствующий драйвер

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-kvm2
chmod +x docker-machine-driver-kvm2
sudo mv docker-machine-driver-kvm2 /usr/local/bin/
docker-machine-driver-kvm2 version
```

#### Пользователь должен быть в группе libvirt

```
sudo usermod -aG libvirt $USER
newgrp libvirt
```

#### Задаём драйвер виртуализации и стартуем миникуб кластер

```
minikube config set vm-driver kvm2
```

### Команды minikub

```
# запустить кластер миникуб
minikube start

# список дополнений
minikube addons list

# Остановить или удалить миникуб машину:
minikube stop
minikube delete

# Полностью удалить и очистить:
minikube delete --all --purge

# запустить дополнение дашборда с WEB мордой
minikube dashboard --url
```

### Команды kubectl

```
# посмотреть конфиг
kubectl config view

# информация о нодах
kubectl get nodes

# информация о подах, включая системные
kubectl get nodes --all-namespaces
```

## Создание кластера на AWS

На AWS необходимо создать пользователя с полномочиями на создание кластера (либо AdministratorAccess).
После создания будут получены AccessKey и SecretKey

### [eksctl](https://eksctl.io/) запуска и управления k8s на AWS

#### [Инструкция AWS на русском языке](https://aws.amazon.com/ru/getting-started/hands-on/amazon-eks-with-spot-instances/)

#### Пример файла конфигурации кластера cluster.yaml

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: basic-cluster
  region: eu-north-1

nodeGroups:
  - name: ng-1
    instanceType: m5.large
    desiredCapacity: 10
	ssh:
	  allow: true  # проброс ключа на ноды группы ~/.ssh/id_rsa.pub
  - name: ng-2
    instanceType: m5.xlarge
    desiredCapacity: 2
	ssh:
	  allow: true
```

#### Создание кластера через eksctl

```
eksctl create cluster -f cluster.yaml
```

#### Удаление кластера через eksctl

```
eksctl delete cluster -f cluster.yaml
```

## Кластер на AWS с помощью [Terrafosm](https://www.terraform.io/downloads)

#### [Настройка Terraform для разных провайдеров](https://registry.terraform.io/browse/providers)
ссылка может не работать без vpn

#### [Настройка для AWS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

#### [Код для создания учебного кластера k8s на AWS с помощью Terraform (GitHub)](https://github.com/hashicorp/learn-terraform-provision-eks-cluster)

#### [Terraform AWS EKS module Git](https://github.com/terraform-aws-modules/terraform-aws-eks)

## Конфиг kubectl

При создании кластера миникуб, он автоматически настраивает конфиг kubectl
По умолчанию это путь <b>~/.kube/config</b>
Чтобы задать другой путь, необходимо переопределить переменную окружения <b>KUBECONFIG</b>:

```
export KUBECONFIG=/home/user/bube/config01
```
Можно задать одновременно несколько конфиг файлов:

```
export KUBECONFIG=/home/user/bube/config01:/home/user/bube/config02
```
Файл конфига содержит названия и адреса кластеров, пользователей, путей к сертификатам, пространства имен, контексты и текущий контекст (один)

[Шпаргалка по конфигу и командам для конфига](https://gist.github.com/bakavets/f2c508d7b7561c2ae80b2c17a59e0574)

```
# посмотреть конфиг
kubectl config view
```

Посмотреть текущий контекст

```
kubectl config current-context
```
Посмотреть все контексты

```
kubectl config get-contexts
```
Изменить текущий контекст:

```
kubectl config use-context <cpntext-name>
```

## [Дашборд](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) k8s

Дашборт kubernetes может быть создан для любого кластера.

#### Создаение дашборда для текущего кластера

yaml по ссылке создает все необходимые для дашборда объекты с рекомендуемыми параметрами

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml
```
Если процесс закончится ошибкой, смотреть [сюда](https://www.anycodings.com/1questions/1639367/the-clusterrolebinding-kubernetes-dashboard-is-invalid-roleref-invalid-value-when-deploying-web-ui)

##### Далее [создаем](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md) пользователя и роль, получаем токен для дашборда:

для этого сохраним в sa-dash.yml:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard

```

И применим:

```
kubectl apply -f sa-dash.yml
```
Создадим токен созданного сервис аккаунта:

```
kubectl -n kubernetes-dashboard create token admin-user
```

В будущем, созданный ранее токен можно получить так:

```
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

Создаем прокси:

```
kubectl proxy
```

Заходим по [ссылке](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/) локальный ресурс из документации, вводим токен и попадаем в дашборд

## Создание объекта pod, запуск контейнеров

Создать под без манифеста yml можно командой, используя образ на докерхабе:

```
kubectl run app-echoserver --image=fofonovrv/echoserver:alpine --port=8000

# или 
kubectl create -f new-pod.yml
# или 
kubectl apply -f new-pod.yml

```

Удаление пода:

```
kubectl delete pod <name>
```
Подробная информация о поде:
Здесь содержится инфа о лейблах, адресе, образах, статсе, контейнерах, нода, на которых запущен, а также, логи.

```
kubectl describe pods <pod_name>

# или 
kubectl delete -f new-pod.yml

# или на основе селектора меток
kubectl delete pods -l app=web-server
```

Подключиться к поду в контейнере:

>имя контейнера можно не указывать, если он один

```
kubectl exec -it <pod_name> --container <container_name> -- /bin/sh
```

Порт форвардинг напрямую в под для отладки:

```
kubectl port-forward <pod_name>  11111:8000
```

Логи пода:

>имя контейнера можно не указывать, если он один


```
kubectl logs <pod_name> --container <container_name>
```

## Лейблы (метки)

#### Лейбл это ключ=значение

Можно добавлять к новым или уже запущенным объектам.

Добавм лейблы к метадате объекта перед созданием:

```
metadata:
  name: app-echo-with-labels
  labels:
    environment: dev
	app: http-server
	
```
Вывести поды вместе с леблами:

```
kubectl get pods --show-labels
```

При создании пода командой kubectl run автоматически навешивается лейбл <b>run=pod_name</b>

Вывести поды с определенными лейблами:

```
kubectl get pods -L app,run
```

Можно выбирать поды с определенными значениями лейблов

Селекторы меток на основе сравнения поддерживают три оператора: <b>=</b>, <b>==</b>, <b>!=</b>	. Первые два - синонимы

```
kubectl get pods -l app=http-server
```

Селекторы меток на основе набора: in, notin, пример:

```
kubectl get pods -l 'app in (http-server,web-server)'
```

Если нужно при деплое пода выбрать только те ноды, где есть GPU, можем воспользоваться меткой. Поставим на ноде лейбл <b>gpu=true</b>

манифест для пода в этом случае должен содержать соответствующий nodeSelector:

```
apiVersion: v1
kind: Pod
metadata:
  name: app-gpu-server
  labels:
    app: gpu-server
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - name: app-echo-container
    image: fofonovrv/echoserver:alpine
    ports:
    - containerPort: 8000
```

### Аннотации

Похожи на лейблы, но без селеторов. Используются для добавления описания. Можно увидеть в pod describe

Добавить аннотацию к работающему поду:

```
kubectl annotate pod app-01 creator_email='ilon_mask@gmail.com'
```

## Пространства имен (ns)

Вывести список неймспейсов:
```
kubectl get ns
```

Манифест для создания ns:

```
apiVersion: v1
kind: Namespace
metadata:
    name: dev
```

Создание командой:

```
kubectl create namespace dev
```

При создании пода можно указать неймспейс в метадате:

```
apiVersion: v1
kind: Pod
metadata:
  name: app-01
  namespace: dev
```

Информация о всех подах во всей неймспейсах:

```
kubectl get pods --all-namespaces
```

При удалении неймспейса, его поды так же удаляются:

```
kubectl delete ns dev
```

## ReplicationController (rc)

ReplicationController постоянно следить, что все необходимые поды запущены. Если какая то нода падает, то ReplicationController перезапускает поды на другой ноде.

Создание ReplicationController с тремя репликами пода, выбранного селектором лейбла:

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kuber-rc
spec:
  replicas: 3
  selector:
    app: web-server
  template:
    metadata:
      name: web-echo
      labels:
        app: web-server
    spec:
      containers:
      - name: web-echo-image
        image: fofonovrv/echoserver:alpine
        ports:
        - containerPort: 8000
```

При применении измененного манифеста rc, будут созданы новые реплики, а старые перестанут контролироваться

Изменения можно вносить прямо через дашборд. Но если изменить контейнер, то необходимо удалять старые поды, чтобы rc развернул новые с новым контейнером

Операции с ReplicationController:

```
# посмотреть
kubectl get rc

# удалить
kubectl delete rc
```

## ReplicaSet (rs)

Является новым поколением, заменяющим ReplicationController.

Обчыно, создается автоматически при создании ресурса более высокого уровня - Deployment

Манифест для создания rs:

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kuber-rs-1
  labels:
    app: kuber-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      env: dev
  template:
    metadata:
      labels:
        env: dev
    spec:
      containers:
      - name: web-echo-image
        image: fofonovrv/echoserver:alpine
```

В селекторе вместо matchLabels использовать matchExpressions, в котором можно использовать четыре оператора: In, NotIn, Exists, DoesNotExist:

```
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
          - kuber
          - http-server
      - key: env
```

## Deployment (развёртывания)

Деплойменты служат для развертывания и обновления приложения с нулевым простоем.

Создать деплоймент командой:

```
kubectl create deployment kube-ctl-app --image=fofonovrv/echoserver:alpine --port=8000 --replicas=3
```

Если мы обновили образ, то можно командой заменить образ в деплоймент:

```
kubectl set image deployment/kuber-ctl-app echoserver=fofonovrv/echoserver:1.1 --record
```

<b>--record</b> необходимо для записи истории ревизий, однако этот флаг устарел, лучше сразу запускать с этим флагом apply

После этой команды, деплоймент создаст новый RS, с контейнерами новой версии, развернет их, а затем, изменит желаемое количесво реплик в старом RS на 0 (и удалит старые поды).

### Манифест создание деплоймента

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuber
  labels:
    app: kuber
spec:
  replicas: 5
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: http-server
  template:
    metadata:
      labels:
        app: http-server
    spec:
      containers:
        - name: app-echoserver1-0
          image: fofonovrv/echoserver:1.0
          ports:
          - containerPort: 8000

```

Новое в манифесте

+ <b>strategy</b> - стратегия обновления: RollingUpdate или Recreate
+ <b>RollingUpdate</b> - обновлять поды постепенно, исходя из параметров maxSurge и maxUnavailable
+ <b>Recreate</b> - удалить все поды, затем создать заново
+ <b>maxSurge</b> - максимальный всплеск / сколько подов будут добавляться единовременно
+ <b>maxUnavailable</b> - максимальное количество недоступных подов / по сколько подов будут убиваться
+ <b>minReadySeconds</b> - количество секунд для нового пода, после чего он считается доступным

### История обновления и откат

Посмотреть историю ревизий (попадают только --record)

```
kubectl rollout history deployment <deployment_name>
```

Откатить версию назад:

```
kubectl rollout undo deployment kuber
```

Откатиться к определенной версс ревизии (номер в history)

```
kubectl rollout undo deployment kuber --to-revision=1
```

Откат происходит на основании RS, каждая ревизии сответствует определенному RS. Поэтому их не стоит удалять

## Сервисы

Сервис - объект для предоставления единой точки входа к группе подов одного приложения

Есть 4 типа сервисов k8s:
+ <b>ClusterIP (default)</b> - для обращения внутри кластера, проксирует каждый отдельный запрос к рандомному поду
+ <b>NodePort</b> - робрасывает снаружи в кластер, открывает указанный порт на каждой ноде кластера. Проксирует на рандомный под.
+ <b>LoadBalancer</b> - только на cloud провайдере. 
+ <b>ExternalName</b> - реализется на уровне DNS, создает указанный cname, подключаться по нему можно напрямую, минуя прокси. (не подходит для https (TLS), т.к. в заголовке не оригинальное имя сервера)

### Манифест создания сервиса ClusterIP

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: http-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
```

Если после создания сервиса пересоздать поды, то мы увидим в них переменные окружения, образованные от нашего названия "my-service" и с нашими параметрами.
По этим переменным можно обращаться к сервису из любого пода. Поэтому желательно сначала создавать сервис, а затем поды, чтобы в каждый под переменные окружения были добавлены сразу

```
MY_SERVICE_PORT_80_TCP=tcp://10.107.114.235:80
MY_SERVICE_SERVICE_HOST=10.107.114.235
MY_SERVICE_SERVICE_PORT=80
MY_SERVICE_PORT=tcp://10.107.114.235:80
MY_SERVICE_PORT_80_TCP_ADDR=10.107.114.235
MY_SERVICE_PORT_80_TCP_PORT=80
MY_SERVICE_PORT_80_TCP_PROTO=tcp
```

Можно обращаться к службам из подов по имени сервиса в формате:

```
http://my-service.default.svc.cluster.local

# http://<service_name>.<namespace>.svc.cluster.local

# в пределах одного немспейса можно:
# http://<service_name>
```

### Headless сервисы

Такой сервис позволит не использовать прокси, а резолвится во все адреса подов одновременно.

Для создания Headless в манифесте укажем <b>clusterIP: None</b>

```
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None
  selector:
    app: http-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: ClusterIP
```

Имя такой службы внутри кластера резолвится всеми адресами подов:

```
/ # nslookup headless-service
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   headless-service.default.svc.cluster.local
Address: 172.17.0.13
Name:   headless-service.default.svc.cluster.local
Address: 172.17.0.6
Name:   headless-service.default.svc.cluster.local
Address: 172.17.0.10
Name:   headless-service.default.svc.cluster.local
Address: 172.17.0.12
Name:   headless-service.default.svc.cluster.local
Address: 172.17.0.11
```

Если делать curl службы:порт, будем попадать на рандомный адрес.
Может потребоваться в приложении, которое будет резолвить имя службы, брать адреса и подключаться сразу ко всем.

### Сервис ExternalName

Просто создает DNS CNAME

```
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: database01
```

### Сервис NodePort

Пробрасывает снаружи в кластер, открывает указанный порт на каждой ноде кластера. Если создавать в облаке, сразу будет external ip. Проксирует на рандомный под.

nodePort: 30000  - задавать не обязательно, кубер выберет любой свободный в диапазоне 30000-32767

```
apiVersion: v1
kind: Service
metadata:
  name: node-service
spec:
  # externalTrafficPolicy: Local
  # sessionAffinity: ClientIP
  selector:
    app: http-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
      nodePort: 30000   #port range 30000-32767
  type: NodePort
```

Чтобы подключиться к сервису на миникубе:

```
# узнаем ip кластера:
minikube ip

# curl <minicube_ip>:<nodeport>
```

<b>sessionAffinity: ClientIP</b> - перенаправлять каждого клиента только к одному поду (рандомно первый раз, потом запоминает под и клиента)

### Сервис LoadBalancer

Улучшеная версия NodePort. Проксирует не сама служба, а балансировщик нагрузки на стороне cloud провайдера

Чтобы распределить трафик в равной степени по нодам, а дальше использовать внутренний балансировщик:

```
spec:
  externalTrafficPolicy: Local
```

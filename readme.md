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
Получим токен созданного сервис аккаунта:

```
kubectl -n kubernetes-dashboard create token admin-user
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

```
Подробная информация о поде:
Здесь содержится инфа о лейблах, адресе, образах, статсе, контейнерах, нода, на которых запущен, а также, логи.

```
kubectl describe pod <pod_name>
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
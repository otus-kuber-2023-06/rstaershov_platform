# rstaershov_platform
rstaershov Platform repository
# Знакомство с Kubernetes, основные понятия и архитектура // ДЗ №1
## _1. Подготовка к выполнению ДЗ_
В репозитории создана новая ветка kubernetes-prepare и добавлены следующие файлы:
 - [.travis.yml](https://raw.githubusercontent.com/express42/otus-platform-tests/2020-04/.travis.yml) - для автоматизированной проверки ДЗ.
 - [.github/PULL_REQUEST_TEMPLATE.md](https://raw.githubusercontent.com/express42/otus-platform-tests/2020-04/.github/PULL_REQUEST_TEMPLATE.md) - шаблон для описания PR.
 - [.github/auto_assign.yml](https://raw.githubusercontent.com/express42/otus-platform-tests/2020-04/.github/auto_assign.yml) - файл конфигурации для плагина Auto Assign.

Выполнен merge kubernetes-prepare в main.
## _2. Настройка локального окружения. Запуск первого контейнера. Работа с kubectl_
- Установлен пакетный менеджер [chocolatey](https://chocolatey.org/install)
- Установлен и запущен [minikube](https://minikube.sigs.k8s.io/docs/start/) :
  ```
  choco install minikube
  minikube start --driver=hyperv
  ```
- Проверка подключения к кластеру:
  ```
  $ kubectl cluster-info
  Kubernetes control plane is running at https://172.27.13.188:8443
  CoreDNS is running at https://172.27.13.188:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
  ```
### 2.1  Проверка отказоустойчивости кластера:
  ```
  minikube ssh
  $ docker ps
  $ docker rm -f $(docker ps -a -q)
  $ docker ps
  ```
  Контейнеры автоматически запустились, после их удаления:
  - k8s_etcd_etcd-minikube_kube-system
  - k8s_kube-proxy_kube-proxy
  - k8s_kube-scheduler_kube-scheduler-minikube_kube-system
  - k8s_kube-apiserver_kube-apiserver-minikube_kube-system
  - k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system
  - k8s_coredns_coredns
  ```
  kubectl get pods -n kube-system
  kubectl delete pod --all -n kube-system
  ```
  Поды автоматически запустились, после их удаления:
  - coredns
  - etcd-minikube
  - kube-apiserver-minikube
  - kube-controller-manager-minikube
  - kube-proxy
  - kube-scheduler-minikube
  
  Статус кластера: "health":"true"
  ```
  PS C:\Windows\system32> kubectl get cs
  Warning: v1 ComponentStatus is deprecated in v1.19+
  NAME                 STATUS    MESSAGE                         ERROR
  controller-manager   Healthy   ok
  scheduler            Healthy   ok
  etcd-0               Healthy   {"health":"true","reason":""}
  ```

### Задание
#### _Разберитесь почему все pod в namespace kube-system восстановились после удаления._
### Ответ:
 - _coredns_ является объектом Deployment, абстракцией управляющей ReplicaSet, соответственно при удалении пода, ReplicaSet восстанавлиет его.
   ```
   PS C:\Windows\system32> kubectl get deploy -n kube-system
    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    coredns   1/1     1            1           160m
   ```
  - _kube-proxy_ является объектом DaemonSet, соответственно всегда будет запущен на каждой ноде кластера.
    ```
    PS C:\Windows\system32> kubectl get ds -n kube-system
    NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   176m
    ```
  - _etcd-minikube, kube-apiserver-minikube, kube-controller-manager-minikube, kube-scheduler-minikube_ являются статическими подами, управляемыми демоном kubelet, который отслеживает состояние статических подов.

#### 2.2 Dockerfile
- Создан докерфайл, описывающий образ веб-сервера, прослушивающего порт 8000 и позволяющего просматривать файлы из директории app.
- Собран образ iscander61/nginx:otus, на основе alpine и запушен в Dockerhub.
#### 2.3 Манифест pod
- Создан манифест web-pod.yaml
- В манифест добавлен initContainer, генерирующий index.html.
- Добален Volumes для доступности генерируемых в initContainers файлов в контейнере web.
- Под запущен, выполнена переадресация портов пода, веб-сервер работает, страница открывается.
  ```
  kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
  ```
#### 2.4 Hipster ShopHipster Shop
- Склонирован [репозиторий](https://github.com/GoogleCloudPlatform/microservices-demo).
- На основе имеющегося в репозитории dockerfile собран и запушен в dockerhub образ iscander61/hipster-shop:otus .
- Сгенерирован манифест frontend-pod.yaml.
- Выполнен запуск пода с манифкста frontend-pod.yaml, под находится в статусе Error.
  ```
  kubectl run frontend --image iscander61/hipster-shop:otus --restart=Never --dry-run -o yaml > frontend-pod.yaml
  ```
#### 2.5 Hipster Shop | Задание со ⭐
- Под frontend находится в статусе Error, из-за отсутствия env:
  ```
  panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
  panic: environment variable "CURRENCY_SERVICE_ADDR" not set
  panic: environment variable "CART_SERVICE_ADDR" not set
  panic: environment variable "RECOMMENDATION_SERVICE_ADDR" not set
  panic: environment variable "CHECKOUT_SERVICE_ADDR" not set
  panic: environment variable "SHIPPING_SERVICE_ADDR" not set
  panic: environment variable "AD_SERVICE_ADDR" not set
  ```
- Создан новый манифест frontend-pod-healthy.yaml, с указанием отсутствующих выше env.
- Запущенный с манифеста frontend-pod-healthy.yaml, под frontend находится в статусе Running.

# Механика запуска и взаимодействия контейнеров в Kubernetes // ДЗ №2
## _1. Подготовка к выполнению ДЗ_
Выполнена установка [kind](https://kind.sigs.k8s.io/docs/user/quick-start/), с помощью менеджера пакетов [choco](https://community.chocolatey.org/packages/kind):
```
choco install kind
```
Выполнена установка [Docker Engine](https://docs.docker.com/engine/).
С помощью указанного в домашнем задании манифеста запущен кластер kind из 3х рабочих нод и 3х март нод:
```
kind create cluster --config kind-config.yaml
```
## _2. Основная часть домашнего задания_
### - Работа с replicaSet:
- Создан манифест микросервиса frontend: frontend-replicaset.yaml. 
- Выполнена сборка image микросервиса paymentService, полученный image запушен в dockerhub. Создан манифест микросервиса paymentService: paymentservice-replicaset.yaml
- Выполнены задания по применению, скалированию, обновлению реплик.
### - Работа с Deployment:
- На основе paymentservice-replicaset.yaml создан paymentservice-deployment.yaml. 
- Выполнены задания по применению, обновлению, последовательности обновления под, просмотру истории обновления, откату обновления.
- Релизованы два сценария развертывания сервиса: paymentservice-deployment-bg.yaml и paymentservice-deployment-reverse.yaml.
- На примере микросервиса frontend рассмотрели как probes влияют на процесс развертывания и состояния под. Для этого создали манифест на примере приложения frontend: frontend-deployment.yaml.
## _3. DaemonSet | Задание со  ⭐_ 
- Создание и применение манифеста сервиса NodeExporter NodeExporter-daemonset.yaml.
- Проброс порта к сервису на лубой рабочей ноде:
    ```
    kubectl port-forward <имя pod в DaemonSet> 9100:9100
    ```
- Проверка доступности метрик:
    ```
    curl localhost:9100/metrics
    ```
## _4. DaemonSet | Задание со  ⭐ ⭐_
- Используя секцию tolerations дали доступ к запуску под NodeExporter на мастер нодах. После обновления Daemonset, под развернулся на всех 6 нодах.
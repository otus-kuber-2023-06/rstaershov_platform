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

# Хранение данных в Kubernetes: Volumes, Storages, Statefull-приложения // ДЗ №4
## _Подготовка к выполнению домашнего задания._
```
kind create cluster
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```
## _1. Запуск minio-statefulset_
- Применили манифест kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-statefulset.yaml
- Применили headless service: kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-headless-service.yaml
- Выполнена проверка работы minio:
    ```
    kubectl get statefulsets
    kubectl get pods
    kubectl get pvc
    kubectl get pv
    kubectl describe pod/minio-0
    ```
## _2. Запуск minio-statefulset-with-secrets | Задание со  ⭐_ 
- Создали secrets.yaml для пользователя и пароля minio.
- Создали и применили манифест с использованием созданного ранее secrets: minio-statefulset-with-secrets.yaml.
- Создали PersistentVolume "my-pv" 1Gi.
- Создали PersistentVolumeClaim "my-pvc" 500Mi.
- Создали Pod "my-pod", использующий "my-pvc" в качестве тома данных. Внутри Pod том примонтирован в директорию "/app/data" в которой создан файл "data.txt".
- Удалили ранее созданный Pod и создали новый Pod с тем же PVC "my-pvc" и проверили наличие файла "data.txt".

# Безопасность и управление доступом // ДЗ №5

 В рамках ДЗ изучено создание: namespace, ServiceAccount, Role, ClusterRole, RoleBinding, ClusterRoleBinding. И биндинг созданных ролей с разными правами к аккаунтам.

# Шаблонизация манифестов. Helm и его аналоги (Jsonnet, Kustomize) // ДЗ №6
 
 ## _Подготовка к выполнению ДЗ_
 - В [YC](https://cloud.yandex.ru/docs/managed-kubernetes/) выполнен запуск кластера kubernetes.
 - Выполнена установка [nginx-ingress](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/) по документации.
 - Выполнена установка [cert-manager](https://artifacthub.io/packages/helm/cert-manager/cert-manager).
   - создан ClusterIssuer.
 ## _1. Установка chartmuseum_
 Выполнена кастомная установка [chartmuseum](https://artifacthub.io/packages/helm/chartmuseum/chartmuseum).
 ```
 helm upgrade --install chartmuseum stable/chartmuseum --wait \
 --namespace=chartmuseum \
 --version=3.10.1 \
 -f kubernetes-templating/chartmuseum/values.yaml
 ```
 Создан ingress с автоматической генерацией сертификата LE, https://chartmuseum.158-160-42-144.nip.io/.
 ## chartmuseum | Задание со ⭐
Работа с chartmuseum выполняется через взаимодействие с его API.
Для работы с helm-push, необходимо добавить репозиторий: helm repo add chartmuseum http://<host репозитория>:8080
- Для загрузки helm package в репозиторий, необходимо выполнить POST запрос к эндпоинту /api/charts. Или установить и использовать [helm-push](https://github.com/chartmuseum/helm-push)
  -  curl --data-binary "@mychart-0.1.0.tgz" http://localhost:8080/api/charts
  -  helm cm-push mychart-0.3.2.tgz chartmuseum
- Для удаление helm package из репозитория, DELETE на эндпоинт /api/charts/<name>/<version>.
- Для получения списка helm package в репозитории, GET на эндпоинт /api/charts.
Более подробная информация доступна [git репозитории](https://github.com/helm/chartmuseum).
## _2. Установка Harbor_
Выполнена кастомная установка [harbor](https://artifacthub.io/packages/helm/harbor/harbor) с автоматической генерацией сетификата LE, https://harbor.158-160-42-144.nip.io/.
```
helm install harbor -f values.yaml harbor/harbor -n harbor --create-namespace
```
В репозиторий добавлены пакеты:
- helm push helm/hipster-shop oci://harbor.158-160-42-144.nip.io/helm
- helm push helm/frontend oci://harbor.158-160-42-144.nip.io/helm
## _3. Создаем свой helm chart_
Выполнена установка приложения из helm chart hipster-shop с зависимостями: front.
## Создаем свой helm chart | Задание со ⭐
Выполнена установка redis, зависимостью hipster-shop.

# Custom Resource Definitions. Operators // ДЗ №7
- Написали CustomResource и CustomResourceDefinition для mysql оператора;
- Написали mysql оператора с помощью python KOPF;
- Создали в mysql таблицу test, и добавили в нее 2 записи, после чего выполнили удаление инстанста mysql;
- Проверили выполнение backup-mysql-instance-job;
- Создали CustomResource и проверили, что содержимое таблицы восстановилось с помощью restore-mysql-instance-job.
- Выполнили build образа iscander61/mysql-operator:v002 и сделали push в registry dockerhub;
- В директории kubernetes-operator/deploy/ создали ServiceAccount, ClusterRole, ClusterRoleBinding и Deployment для mysql-operator;
- Применили созданные в директории kubernetes-operator/deploy/ манифесты, и проверили аналогично описанным выше шагам проверили работоспособность mysql-operator;
- ```
  $ kubectl get jobs
    NAME                         COMPLETIONS   DURATION   AGE
    backup-mysql-instance-job    1/1           5s         14m
    restore-mysql-instance-job   1/1           3m22s      16m
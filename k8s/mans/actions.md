# Полезные команды Kubernetes

## Minikube

Правильный и **самый безопасный** способ останавливать и запускать minikube,
чтобы **не потерять данные** (поды, deployments, PersistentVolumes, ConfigMaps,
Secrets, локальные образы и т.д.), зависит от того, что именно ты считаешь
«данными» и какой драйвер используешь (docker, qemu и т.п.). Но в большинстве
случаев на macOS в 2026 году схема такая:

## Запуск minikube + базовые команды для подов

```shell
# Старт minicube
minikube start
minikube start --memory 8192 --cpus 4

# Эта команда перенаправит Docker-клиент внутрь виртуальной машины Minikube:
eval $(minikube docker-env)

# Собираем Docker-образ приложения внутри Minikube - В ТОМ ЖЕ ТЕРМИНАЛЕ, ГДЕ ЗАПУЩЕН MINIKUBE!:Теперь собираем образ прямо там (находясь в корне проекта):
docker build -t tasks-app:latest .

# Применяем манифесты Kubernetes: 
kubectl apply -f k8s/mongodb.yaml
kubectl apply -f k8s/tasks-app.yaml

# Проверяем, что все Поды поднялись (статус Running):
kubectl get pods # или svc

# Подключение к подам, если в service.yaml spec.type - ClusterIP
kubectl port-forward service/tasks-service 8091:8091

# Можно подключиться к поду:
kubectl exec -it имя_пода -- bash

# Удалить некорректные конфииги для подов - удалтся поды
kubectl delete deployment mongodb
kubectl delete service mongodb

# Если обновил код приложения:
# 1) В терминале с запщеным minikube запускаем:
eval $(minikube docker-env)
# 2) Переходим в этом же терминале в папку приложения 
# и в корне собираем образ с тем же именем что и было:
docker build -t tasks-app:latest .
# 3) Перезагружаем деплоймент нужного приложения:
kubectl rollout restart deployment tasks-app

# Логи в реальном времени
kubectl logs deploy/users-app -c users-app -f
```


# Основные команды и что они реально делают с данными

| Команда                                                   | Что происходит с VM/контейнером                               | Сохраняются ли данные кластера?                        | Когда использовать                                  | Влияние на PersistentVolumes (PV) и образы        |
|-----------------------------------------------------------|---------------------------------------------------------------|--------------------------------------------------------|-----------------------------------------------------|---------------------------------------------------|
| `minikube pause`                                          | Замораживает (suspend) все процессы Kubernetes внутри         | Да, полностью (состояние в RAM)                        | Хочешь быстро освободить CPU, но часто возвращаться | Полностью сохраняются                             |
| `minikube unpause`                                        | Возобновляет замороженное состояние                           | —                                                      | После pause                                         | —                                                 |
| `minikube stop`                                           | Полностью останавливает VM или контейнер minikube (power off) | **Да**, почти всё сохраняется (диск, etcd, образы, PV) | Нормальный способ "выключить" на ночь/выходные      | Сохраняются (если PV типа hostPath или локальные) |
| `minikube start`                                          | Запускает заново (восстанавливает из диска)                   | —                                                      | После stop или после перезагрузки Mac               | —                                                 |
| `minikube delete`                                         | Полностью удаляет кластер (VM/контейнер + все данные)         | **Нет** — всё стирается                                | Хочешь начать с чистого листа                       | Всё теряется                                      |
| `minikube start --delete` или просто `start` после delete | Создаёт новый кластер                                         | —                                                      | Только если нужно сбросить                          | —                                                 |

### Рекомендуемый правильный workflow (чтобы ничего не терять)

1. **Обычный день / короткий перерыв**  
   → `minikube pause`  
   → потом `minikube unpause`  
   → самый быстрый вариант, потребляет минимум CPU, состояние в памяти
   сохраняется идеально

2. **Долгий перерыв (ночь, выходные, перезагрузка Mac)**  
   → `minikube stop`  
   → потом просто `minikube start` (без флагов, или с теми же
   `--cpus --memory --driver`, если менял)  
   → minikube восстанавливает **точно то же состояние**, что было до stop  
   → etcd, все объекты Kubernetes, локально загруженные образы (
   `minikube image load`), hostPath-тома — всё на месте

3. **Хочешь полностью очистить и начать заново**  
   → `minikube delete` (или `minikube delete --all --purge` для параноидальной
   очистки)  
   → потом `minikube start` с нужными параметрами

### Важные нюансы (чтобы действительно не потерять данные)

- **PersistentVolumes** (если используешь `hostPath`, `local` или встроенный
  storage-provisioner)  
  → При `stop → start` сохраняются почти всегда (данные на диске minikube).  
  → При `delete` — теряются полностью.  
  → Если используешь **внешние** PV (NFS, Longhorn, local-path-provisioner с
  хранением на Mac) → данные останутся даже после delete.

- **Локальные образы** (`docker build` или `minikube image load`)  
  → Сохраняются при `stop → start`  
  → Теряются при `delete`

- **После перезагрузки Mac или Docker Desktop**  
  → Часто kubeconfig "ломается" (порт меняется) → просто выполни
  `minikube update-context` или `minikube start` (оно само поправит)

- **Самый надёжный способ "выключить на долго" без потери**
  ```bash
  minikube stop
  # теперь можно закрыть Docker Desktop, перезагрузить Mac и т.д.
  # когда вернёшься:
  minikube start
  minikube update-context   # на всякий случай
  ```

### Краткая шпаргалка

- Хочу быстро освободить CPU → `pause` / `unpause`
- Хочу выключить на ночь/день → `stop` → потом `start`
- Всё сломалось или нужен чистый старт → `delete` → `start`
- После любой перезагрузки/остановки Docker → `minikube start` +
  `minikube update-context`

```shell
# Удалить кластер
minikube delete -all --purge

# Подключиться к ноде:
minikube ip
minikube ssh

# Ip адрес пода не доступен из локальной сети, поэтому:

# смотрим все сервисы
minikube service list

# и запускаем нужный
minikube service SERVICENAME

# или проброс портов
kubectl port-forward service/<имя_сервиса> 31200:31200


# Загрузить образ в minikube для запуска в нем и создания пода
minikube image load nginx:latest
```

## Kubernetes

### Pods


```sh
# Создание одиночного пода
kubectl run time-service --image {ваша_учетная_запись_Docker}/time-service:0.1.0 

# Создание управляемого пода
kubectl create deployment time-service --image {ваша_учетная_запись_Docker}/time-service:0.1.0 

# Запуск пода
kubectl run my-nginx-pod --image=nginx

# Посмотреть список подов, Посмотреть ip пода
kubectl get pods -o wide

# Удалить под
kubectl delete pod time-service

# Подробная информация о поде
kubectl describe pod time-service-6959bb4bbd-7qpjm

# Открыть порт
kubectl expose deployment time-service --port=8080 --type=NodePort

# Доступ к сервису
minikube service --url time-service | minikube service  time-service

# или так - проброс порта нуружу от нужного сервиса
kubectl port-forward service/time-service 8080 |  kubectl port-forward service/time-service 9999:8080

# Масштабирование
kubectl scale --replicas=3 deployment/time-service
```


### Проверка работоспособности пода

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot
spec:
  replicas: 3
  selector:
    matchLabels:
      pod: springboot-pod
  template:
    metadata:
      labels:
        pod: springboot-pod
    spec:
      containers:
        - name: springboot-container
          image: dmitriyf7498/springboot-deckerdemo:2.0.0
          readinessProbe:
            initialDelaySeconds: 15 # Величина начальной задержки
            periodSeconds: 30 # Выполняется с интервалом 30 секунд
            httpGet:
              path: / # Выполняется этот HTTP-запрос
              port: 8080
              scheme: HTTP
            timeoutSeconds: 2 # Тайм-аут ответа на запрос составляет 2 секунды
            failureThreshold: 1 # Признаком неготовности контейнера считается получение хотя бы одного ответа с кодом ошибки
            successThreshold: 1 # Признаком возвращения контейнера в состояние готовности считается получение хотя бы одного ответа с кодом успешного завершения

```



### Deployment


### Разворачивание

#### Манифест:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timeserver
spec:
  replicas: 3
  selector:
    matchLabels:
      pod: timeserver-pod
  template:
    metadata:
      labels:
        pod: timeserver-pod
    spec:
      containers:
        - name: timeserver-container
          image: dmitriyf7498/dmitriyf7498/time-service:0.1.0

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-kube-single-app # Имя деплоймента
  labels:
    app: my-app
spec:
  replicas: 4 # Количество копий (подов)
  selector:
    matchLabels:
      app: my-app # Должно совпадать с labels в шаблоне ниже
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: kube-single-app # Имя контейнера для команды kubectl set image
          image: dmitriyf7498/kube-single-app:latest
          imagePullPolicy: Always # ВСЕГДА скачивать образ (важно для тега :dev)
          ports:
            - containerPort: 80 # Порт, который слушает приложение внутри

```

#### Далее создаем сервис:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-kube-service
spec:
  type: NodePort
  selector:
    app: dmitriyf7498-kubesingleapp # ДОЛЖНО совпадать с labels в Deployment
  ports:
    - protocol: TCP
      port: 8092        # Порт самого сервиса внутри кластера
      targetPort: 8092   # Порт, который слушает приложение в контейнере
      nodePort: 30001  # Внешний порт (можно не указывать, K8s выберет сам)
```

```shell
kubectl apply -f kubesingleapp-service.yaml
kubectl get svc
minikube service SERVICENAME
```


```sh
# Создание deployment из файла
kubectl create -f deploy.yaml

# Создание из образа
kubectl create deployment my-nginx-deploy --image=nginx

# После обновления файла
kubectl apply -f deploy.yaml

# Информация о deployment
kubectl describe deployment/springboot

# Посмотреть список deployment
kubectl get deploy

# Пробросить порт для deployment или пода
kubectl port-forward deploy/springboot 8080:8080
kubectl port-forward pod/$POD_NAME $FROM_PORT:$TO_PORT

# Логи deployment
kubectl logs -f deploy/springboot 

# Чтобы разобраться в причинах сбоя, будет полезно отобразить
# записи журнала, созданные для предыдущего экземпляра контейнера (который
# работал до аварийного перезапуска), и посмотреть, какие сообщения об ошибках
# выводились в момент сбоя. Для этого добавьте в команду для извлечения записей
# журнала параметр --previous (или просто -p):
kubectl logs -p $POD_NAME

# Увеличение количества подов
kubectl scale deploy my-nginx-deploy --replicas=3

# Удаление deployment
kubectl delete deploy timeserver


# Expose адреса пода:
kubectl expose deployment DEPLOYMENTNAME --type=NodePort --port=8092

# Создать ссылку на под
minikube service SERVICENAME

# Обновить image для deployment
docker build . --tag=kube-single-app:dev-4
docker tag kube-single-app:dev-4 dmitriyf7498/kube-single-app:dev-4
docker push dmitriyf7498/kube-single-app:dev-4
kubectl set image deployment/my-kube-single-app kube-single-app=dmitriyf7498/kube-single-app:dev-4
kubectl rollout restart deployment/my-kube-single-app
```

---

## Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot
spec:
  selector:
    pod: springboot-pod
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  type: LoadBalancer

```

```sh
# Создание сервиса
kubectl create -f service.yaml
# Проброс порта для сервиса
kubectl port-forward service/$SERVICE_NAME $FROM_PORT:$TO_PORT

# Посмотреть ip подов
kubectl get endpoints order-service

# Список сервисов
kubectl get service
```



## Exec
```sh
kubectl exec -it deploy/timeserver -- sh # echo "Testing exec":Testing exec
kubectl exec -it deploy/timeserver -- echo "Testing exec" # Testing exec
```

## Последовательность команд императивно

```sh
# 1 Создаем развертывание:
kubectl create deployment timeserver --image=docker.io/wdenniss/timeserver:1
# 2 Создаем сервис типа LoadBalancer и делаем его доступным на порте 80:
kubectl expose deployment timeserver --type=LoadBalancer --port 80
# 3 Проверяем результат:
kubectl get deploy,svc
# 4 Обновляем контейнер в развертывании до новой версии:
kubectl set image deployment timeserver timeserver=wdenniss/timeserver:2
```

# Запуск приложения в Minikube. 

# Настройка приложения для запуска в Kubernetes

Отличный шаг! Minikube идеально подходит для локальной разработки и тестирования манифестов перед деплоем в облако.
Отвечая на ваш вопрос: да, файлы конфигурации Kubernetes обычно хранят прямо в корневой папке проекта (рядом с Dockerfile и pom.xml). Это стандартная практика, так как манифесты инфраструктуры версиируются в Git вместе с кодом приложения.
Для удобства рекомендуется создать в корне проекта отдельную папку k8s/ и положить манифесты туда.



## Шаг 1. Готовим структуру проекта
Создайте следующую структуру:

ваше-приложение/
├── k8s/
│   ├── mongodb.yaml      # База данных: Deployment, Service + PVC
│   └── tasks-app.yaml    # Приложение: Deployment + Service
├── src/
├── Dockerfile
└── pom.xml


## Шаг 2. Манифест для MongoDB (k8s/mongodb.yaml)
В Kubernetes база данных требует постоянного диска (PersistentVolumeClaim), чтобы данные не пропадали при перезапуске Пода.
Создайте файл k8s/mongodb.yaml:
```yaml
apiVersion: v1kind: PersistentVolumeClaimmetadata:
  name: mongo-pvcspec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---apiVersion: apps/v1kind: Deploymentmetadata:
  name: mongodbspec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0
          ports:
            - containerPort: 27017
          environment:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: rootadmin
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: supersecretpassword
            - name: MONGO_INITDB_DATABASE
              value: tasks_db
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-pvc
---apiVersion: v1kind: Servicemetadata:
  name: mongodb # ⚠️ Важно: имя сервиса станет DNS-именем для Spring Bootspec:
  ports:
    - port: 27017
  selector:
    app: mongodb

```



## Шаг 3. Манифест для Spring Boot приложения (k8s/tasks-app.yaml)
Обратите внимание, что URL подключения к базе остается точно таким же, как в Docker Compose (mongodb://...), потому что K8s внутри себя свяжет имя сервиса mongodb с базой данных.
Создайте файл k8s/tasks-app.yaml:

```yaml
apiVersion: apps/v1kind: Deploymentmetadata:
  name: tasks-appspec:
  replicas: 1 # Количество запущенных экземпляров вашего приложения
  selector:
    matchLabels:
      app: tasks-app
  template:
    metadata:
      labels:
        app: tasks-app
    spec:
      containers:
        - name: tasks-app
          # ⚠️ Локальный образ, который мы соберем внутри окружения Minikube
          image: tasks-app:latest 
          imagePullPolicy: Never # Говорим K8s не искать образ в Docker Hub
          ports:
            - containerPort: 8091
          env:
            - name: SERVER_PORT
              value: "8091"
            - name: MONGO_URI
              value: "mongodb://rootadmin:supersecretpassword@mongodb:27017/tasks_db?authSource=admin"
---apiVersion: v1kind: Servicemetadata:
  name: tasks-app-servicespec:
  type: NodePort # Позволяет открыть порт приложения наружу для вашего ПК
  ports:
    - port: 8091
      targetPort: 8091
      nodePort: 30091 # Приложение будет доступно по этому порту на IP-адресе Minikube
  selector:
    app: tasks-app


```


------------------------------
## Шаг 4. Пошаговый запуск в Minikube
Чтобы всё заработало, нужно выполнить команды в строгом порядке:
1. Запускаем Minikube:

minikube start

2. Переключаем терминал на Docker-окружение Minikube:
Это критически важный шаг! По умолчанию ваш Docker собирает образы на вашем ПК, и K8s их не увидит. Эта команда перенаправит ваш Docker-клиент внутрь виртуальной машины Minikube:

eval $(minikube docker-env)

3. Собираем Docker-образ приложения внутри Minikube:
Теперь собираем образ прямо там (находясь в корне проекта):

docker build -t tasks-app:latest .

4. Применяем манифесты Kubernetes:

kubectl apply -f k8s/mongodb.yaml
kubectl apply -f k8s/tasks-app.yaml

5. Проверяем, что все Поды поднялись (статус Running):

kubectl get pods

------------------------------
## Шаг 5. Как теперь отправить запрос к приложению?
Так как Minikube работает внутри своей виртуальной сети, localhost:8091 работать не будет. Чтобы узнать адрес, по которому доступно ваше приложение, выполните команду:

minikube service tasks-app-service --url

Она вернет вам адрес вида http://192.168.49.2:30091. Используйте его в Postman или браузере для отправки POST и GET запросов!
Получилось ли у вас переключить окружение Docker и собрать образ внутри Minikube? Если возникнут ошибки на этапе запуска подов, дайте знать, я помогу их отладить.

# Запуск приложений

## Запуск minikube

```shell
# Старт minicube
minikube start

# Эта команда перенаправит Docker-клиент внутрь виртуальной машины Minikube:
eval $(minikube docker-env)

# Собираем Docker-образ приложения внутри Minikube:Теперь собираем образ прямо там (находясь в корне проекта):
docker build -t tasks-app:latest .

# Применяем манифесты Kubernetes: 
kubectl apply -f k8s/mongodb.yaml
kubectl apply -f k8s/tasks-app.yaml

# Проверяем, что все Поды поднялись (статус Running):
kubectl get pods # или svc

# Подключение к подам, если в service.yaml spec.type - ClusterIP
kubectl port-forward service/tasks-service 8091:8091

# Можно подключиться к поду:
kubectl exec -it имя_пода -- bash

# Удалить некорректные конфииги для подов - удалтся поды
kubectl delete deployment mongodb
kubectl delete service mongodb

# Примените два новых раздельных манифеста:
kubectl apply -f k8s/tasks-mongodb.yaml
kubectl apply -f k8s/users-mongodb.yaml


```

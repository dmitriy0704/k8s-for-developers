> мне пока просто для понимания хватает, хотя бы просто поверхностно. можешь теперь рассказать про ConfigMap и Secret? Это какие то поды создаются в Kubernetes?

Нет, ConfigMap и Secret — это **не поды**. Это **объекты Kubernetes** (как Deployment, Service, Namespace), которые хранят данные и передают их подам. Сами по себе они не запускают никаких процессов.

## 🎯 Что это такое

| Ресурс | Что хранит | Для чего |
|---|---|---|
| **ConfigMap** | Несекретные данные: настройки, URL, флаги, конфиги приложений | Передать приложению конфигурацию |
| **Secret** | Секретные данные: пароли, токены, ключи, сертификаты | Передать приложению секреты |

Оба — просто **хранилища пар «ключ-значение»** внутри кластера. Под их **потребляет**: монтирует как файл, прокидывает как переменную окружения или читает через API.

## 📦 Аналогия

- **ConfigMap** — как файл `application.properties`, лежащий рядом с приложением.
- **Secret** — как тот же файл, но с паролями, и хранится в сейфе.
- **Под** — как процесс, который этот файл читает.

## 🛠️ Как они выглядят

**ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tasks-service-config
data:
  LOG_LEVEL: "DEBUG"
  KAFKA_BROKERS: "kafka:9092"
  application.yaml: |
    server:
      port: 8080
```

**Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials
type: Opaque
stringData:
  MONGO_URI: "mongodb://rootadmin:password@mongodb:27017/tasksdb"
```

## 🔗 Как под их использует — три способа

### 1. Как переменные окружения

```yaml
spec:
  containers:
    - name: tasks-service
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: tasks-service-config
              key: LOG_LEVEL
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: mongodb-credentials
              key: MONGO_URI
```

Приложение видит `LOG_LEVEL` и `MONGO_URI` как обычные переменные окружения.

### 2. Как файлы (volume)

```yaml
spec:
  containers:
    - name: tasks-service
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: tasks-service-config
```

Все ключи ConfigMap появятся как файлы в `/etc/config/`. Например, `/etc/config/LOG_LEVEL` и `/etc/config/application.yaml`.

### 3. Через `envFrom` (все ключи разом)

```yaml
envFrom:
  - configMapRef:
      name: tasks-service-config
  - secretRef:
      name: mongodb-credentials
```

Все ключи станут переменными окружения.

## 🔐 Чем Secret отличается от ConfigMap

**Технически** — почти ничем: те же ключи-значения. **Практически** — тремя вещами:

| Аспект | ConfigMap | Secret |
|---|---|---|
| **Хранение в etcd** | Открытым текстом | Base64 (не шифрование, но кодирование) |
| **RBAC** | Обычные права | Отдельные права, часто строже |
| **Отображение** | `kubectl get cm -o yaml` показывает значения | `kubectl get secret -o yaml` показывает base64, `kubectl describe` — скрывает |
| **Передача** | Обычно через env или volume | Часто через volume (не env — env видно в `/proc`) |

**Base64 — это не шифрование.** `echo "cGFzc3dvcmQ=" | base64 -d` вернёт `password`. Secret защищён **правами доступа** (RBAC), а не кодированием. Для настоящего шифрования включают **Encryption at Rest** в etcd или используют внешние хранилища (Vault, AWS Secrets Manager).

## 🆚 ConfigMap/Secret vs Spring Cloud Config vs Vault

| Инструмент | Где живёт | Что хранит | Когда использовать |
|---|---|---|---|
| **ConfigMap** | В кластере K8s | Несекретные настройки | Простые конфиги для подов |
| **Secret** | В кластере K8s | Секреты | Пароли, токены |
| **Spring Cloud Config** | Git-репозиторий | Всю конфигурацию | Централизованное управление, версионирование |
| **Vault** | Отдельный сервис | Только секреты, динамические credentials | Высокие требования к безопасности |

**ConfigMap/Secret** — «базовый уровень» в Kubernetes. Они не версионируются, не имеют UI, не генерируют динамические credentials. Но они **встроены** в K8s и не требуют внешних сервисов.

**Spring Cloud Config** и **Vault** — «продвинутый уровень», решают те же задачи, но с версионированием, аудитом, ротацией.

## 💡 В вашем проекте

У вас уже есть примеры:

- **`ratelimit-config`** — это ConfigMap. В нём лежит `config.yaml` с правилами rate limiting. Ratelimit-сервис монтирует его как файл в `/data/ratelimit/config/`.
- **`mongodb-credentials`** (если создадите) — Secret с `MONGO_URI`.
- **`tasks-app-gateway`, `VirtualService`** — это **не** ConfigMap/Secret, а CRD (Custom Resource Definitions) от Istio.

## 🎯 Ключевое отличие от «подов»

| | Под | ConfigMap / Secret |
|---|---|---|
| Что это | Запущенный процесс | Хранилище данных |
| Кто создаёт | Deployment / StatefulSet | `kubectl apply` напрямую |
| Потребляет ресурсы | Да (CPU, RAM) | Нет |
| Может «упасть» | Да | Нет (это просто данные) |
| Кто использует | Другие поды через Service | Поды через env / volume |

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| ConfigMap и Secret — это поды? | **Нет.** Это объекты Kubernetes для хранения данных |
| Они что-то запускают? | Нет, просто хранят ключи-значения |
| Как под их получает? | Через `env`, `envFrom` или `volumeMount` |
| Чем Secret отличается от ConfigMap? | Base64 вместо plaintext + более строгий RBAC |
| Base64 — это шифрование? | Нет, это кодирование. Для шифрования нужно Encryption at Rest или Vault |
| Когда использовать? | Для простых конфигов и секретов, когда не нужны версионирование и аудит |

ConfigMap и Secret — это «файлы настроек» для Kubernetes. Они не работают сами по себе, а служат «складом», откуда поды берут данные при старте или во время работы.

> я бы хотел уточнить, не совсем понял. мы с тобой тут создавали в deployment для tasks-service secret с логином и паролем подключения в БД:

>   containers:
        - name: tasks-service # ⚠️ Локальный образ, который мы соберем внутри окружения Minikube
          image: myorg/tasks-service:1.0
          imagePullPolicy: Never # Говорим K8s не искать образ в Docker Hub
          ports:
            - containerPort: 8091
          env:
            - name: SERVER_PORT
              value: "8091"
            - name: MONGO_URI
              valueFrom:
                # создается секрет:  kubectl create secret generic mongodb-credentials \
                #  --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin'
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_URI

> получается в кластере создался какой-то файл и логином и паролем? как-то вообще можно взаимодействовать с secret?

Да, всё верно — в кластере создался объект `Secret` с вашими данными. Разберу, что это за «файл», где он лежит, и как с ним взаимодействовать.

## 🗂️ Что именно создалось в кластере

Когда вы выполнили:

```bash
kubectl create secret generic mongodb-credentials \
  --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin'
```

Kubernetes создал **объект Secret** в etcd (база данных кластера). Это **не файл на диске** в привычном смысле, а запись в распределённом key-value хранилище.

Внутри он выглядит примерно так:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials
  namespace: default
type: Opaque
data:
  MONGO_URI: bW9uZ29kYjovL3Jvb3RhZG1pbjpzdXBlcnNlY3JldHBhc3N3b3JkQHRhc2tzLW1vbmdvZGItc2VydmljZToyNzAxNy90YXNrc2RiP2F1dGhTb3VyY2U9YWRtaW4=
```

Обратите внимание: значение закодировано в **base64**. Это не шифрование, а кодирование — любой может раскодировать.

## 🔍 Как посмотреть Secret

### 1. Список всех секретов в namespace

```bash
kubectl get secrets
```

Увидите что-то вроде:
```
NAME                  TYPE     DATA   AGE
mongodb-credentials   Opaque   1      2d
```

### 2. Посмотреть структуру (без значений)

```bash
kubectl describe secret mongodb-credentials
```

Вывод:
```
Name:         mongodb-credentials
Namespace:    default
Type:         Opaque

Data
====
MONGO_URI:  127 bytes
```

`describe` **не показывает значения** — только размер. Это защита от случайного попадания секрета в логи или скриншоты.

### 3. Посмотреть значение (base64)

```bash
kubectl get secret mongodb-credentials -o yaml
```

Вывод:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials
type: Opaque
data:
  MONGO_URI: bW9uZ29kYjovL3Jvb3RhZG1pbjpzdXBlcnNlY3JldHBhc3N3b3JkQHRhc2tzLW1vbmdvZGItc2VydmljZToyNzAxNy90YXNrc2RiP2F1dGhTb3VyY2U9YWRtaW4=
```

### 4. Декодировать значение

```bash
kubectl get secret mongodb-credentials -o jsonpath='{.data.MONGO_URI}' | base64 -d
echo  # для переноса строки
```

Вывод:
```
mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin
```

Есть короткий способ — через `-o go-template`:

```bash
kubectl get secret mongodb-credentials -o go-template='{{.data.MONGO_URI | base64decode}}'
```

## 🛠️ Как с ним взаимодействовать

### Через kubectl

```bash
# Посмотреть все ключи в секрете
kubectl get secret mongodb-credentials -o jsonpath='{.data}' | jq 'keys'

# Обновить значение (пересоздать)
kubectl create secret generic mongodb-credentials \
  --from-literal=MONGO_URI='новое_значение' \
  --dry-run=client -o yaml | kubectl apply -f -

# Удалить
kubectl delete secret mongodb-credentials
```

### Изнутри пода

Если секрет прокинут как env (как у вас), приложение видит его как обычную переменную окружения:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- env | grep MONGO
```

Если секрет смонтирован как volume, значение лежит в файле:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- cat /etc/secrets/MONGO_URI
```

### Через API Kubernetes

Любой под с нужными правами (RBAC) может прочитать секрет через API:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- \
  sh -c 'cat /var/run/secrets/kubernetes.io/serviceaccount/token' # токен SA
```

Но для этого ServiceAccount пода нужны права на `get secret`. По умолчанию их нет.

## 🔐 Про безопасность

**1. Base64 — это НЕ шифрование.** Любой с доступом к API Kubernetes или к etcd может прочитать значение.

**2. Защита — через RBAC.** По умолчанию поды **не могут** читать секреты других приложений. Но если ServiceAccount пода имеет права на `get secret`, он может прочитать **любой** секрет в namespace.

**3. Encryption at Rest.** В production включают шифрование etcd, чтобы секреты лежали зашифрованными даже в базе. В Minikube это обычно не настроено.

**4. `env` vs `volume`.** Переменные окружения видны в `/proc/<pid>/environ` и в `kubectl describe pod`. Volume-монтирование безопаснее — но всё равно доступно процессу.

**5. Секреты в Git — плохо.** Никогда не коммитьте YAML с секретами. Только `kubectl create secret` вручную или через внешние системы (Vault, Sealed Secrets).

## 🎯 Ваш случай

У вас:
```yaml
env:
  - name: MONGO_URI
    valueFrom:
      secretKeyRef:
        name: mongodb-credentials
        key: MONGO_URI
```

Что происходит **при старте пода**:
1. Kubelet обращается к API Kubernetes: «дай мне значение ключа `MONGO_URI` из секрета `mongodb-credentials`».
2. API проверяет, есть ли у ServiceAccount пода право на чтение этого секрета.
3. Если да — значение подставляется в переменную окружения `MONGO_URI` процесса.
4. Приложение видит `MONGO_URI` как обычную env-переменную.

**Важно:** это происходит **в момент старта пода**. Если секрет изменится, **уже запущенный под не увидит** нового значения. Нужно перезапустить под:

```bash
kubectl rollout restart deploy/tasks-service
```

## 📊 Сравнение с ConfigMap

| Аспект | ConfigMap | Secret |
|---|---|---|
| Хранение в etcd | Plaintext | Base64 |
| `kubectl describe` | Показывает значения | Скрывает |
| RBAC | Обычно не ограничен | Часто строгий |
| Обновление в поде | Нужен перезапуск | Нужен перезапуск |
| Encryption at Rest | Не применяется | Опционально |

## 💡 Практические команды

```bash
# Создать
kubectl create secret generic my-secret --from-literal=KEY=value

# Создать из файла
kubectl create secret generic my-secret --from-file=./password.txt

# Посмотреть
kubectl get secret my-secret -o yaml

# Декодировать
kubectl get secret my-secret -o jsonpath='{.data.KEY}' | base64 -d

# Обновить
kubectl create secret generic my-secret --from-literal=KEY=newvalue \
  --dry-run=client -o yaml | kubectl apply -f -

# Удалить
kubectl delete secret my-secret

# Посмотреть, какой под использует секрет
kubectl get pods -o json | jq '.items[] | select(.spec.containers[].env[]?.valueFrom.secretKeyRef.name == "my-secret") | .metadata.name'
```

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| Secret — это файл в кластере? | Нет, это объект в etcd (база данных K8s) |
| Где лежит значение? | В etcd, в base64-кодировке |
| Base64 — это шифрование? | Нет, это кодирование. Защита — через RBAC и Encryption at Rest |
| Как посмотреть? | `kubectl get secret -o yaml` (base64) или `base64 -d` |
| Как взаимодействовать? | kubectl, env в поде, volume, API Kubernetes |
| Обновляется в поде автоматически? | Нет, нужен перезапуск пода |
| Кто может прочитать? | Тот, у кого есть RBAC-права на `get secret` |

Ваш `MONGO_URI` сейчас лежит в etcd в base64. Под читает его при старте через API Kubernetes и получает как env-переменную. Всё работает по стандартной схеме K8s.


# Helm

Хороший вопрос — Helm встречается в 90% туториалов по Kubernetes, и без него дальше будет сложно. Разберу по порядку: что это, зачем нужно, и как им пользоваться в вашем конкретном случае.

## 📦 Что такое Helm

**Helm — это пакетный менеджер для Kubernetes.** Аналогия из мира Linux:
- `apt` / `yum` — устанавливают пакеты в ОС
- `npm` / `pip` / `maven` — управляют зависимостями в языках
- **`helm` — устанавливает «пакеты» в Kubernetes**

«Пакет» в Helm называется **chart** (чарт). Это набор YAML-манифестов (Deployment, Service, ConfigMap, RBAC и т.д.), упакованных в один архив с шаблонами и значениями по умолчанию.

## 🎯 Зачем Helm, если можно просто `kubectl apply`

Можно, но представьте, что для Redis в Kubernetes нужно написать **не один** YAML, а:

- `StatefulSet` (Redis — это stateful-приложение, ему нужен стабильный identity и volume)
- `Service` (headless для кластера + обычный для клиента)
- `ConfigMap` (конфигурация redis.conf)
- `Secret` (пароль, если включён)
- `PersistentVolumeClaim` (хранилище)
- `ServiceAccount` + `Role` + `RoleBinding` (права)
- `NetworkPolicy` (изоляция)
- `PodDisruptionBudget` (для HA)

Это 8–10 файлов, ~500 строк YAML, в которых легко ошибиться. **Helm делает то же самое одной командой** — вы указываете только те параметры, которые хотите переопределить.

```
helm install redis bitnami/redis --set auth.enabled=false
```

Эта команда:
1. Скачивает chart `redis` из репозитория `bitnami`.
2. Подставляет `auth.enabled=false` в шаблоны.
3. Рендерит YAML-манифесты.
4. Отправляет их в Kubernetes (`kubectl apply` под капотом).
5. Запоминает, что «релиз `redis` установлен» — чтобы потом можно было обновлять/удалять.

## 🧩 Терминология Helm

| Термин | Что означает |
|---|---|
| **Chart** | Пакет: набор шаблонов YAML + метаданные |
| **Repository** | Хранилище чартов (как Maven Central, но для Helm) |
| **Release** | Установленный экземпляр чарта. Один и тот же чарт можно установить несколько раз под разными именами |
| **Values** | Параметры, которые вы переопределяете (`--set` или `values.yaml`) |
| **Revision** | Версия релиза (для откатов) |

## 🚀 Установка Helm

**macOS:**
```bash
brew install helm
```

**Linux (Ubuntu/Debian):**
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

**Windows:** через `choco install kubernetes-helm` или скачать бинарник с [helm.sh](https://helm.sh).

Проверка:
```bash
helm version
```

## 📚 Разбор ваших команд

### Команда 1: `helm repo add bitnami https://charts.bitnami.com/bitnami`

Добавляет **репозиторий чартов** Bitnami в локальный список.

- **Bitnami** — компания, которая публикует поддерживаемые чарты для популярных приложений: Redis, PostgreSQL, Kafka, MongoDB и т.д. Это «де-факто стандарт» для быстрого развёртывания.
- `bitnami` — имя, под которым репозиторий будет храниться у вас локально. Могли бы назвать `my-redis-repo`.
- URL — адрес, где лежат чарты.

После этой команды Helm знает: «если я встречу `bitnami/redis`, ищи его по такому-то URL».

Проверить, что репозиторий добавлен:
```bash
helm repo list
```

Обновить список чартов (как `apt update`):
```bash
helm repo update
```

### Команда 2: `helm install redis bitnami/redis --set auth.enabled=false`

- `install redis` — создаёт **релиз** с именем `redis`.
- `bitnami/redis` — какой чарт использовать (репозиторий/имя).
- `--set auth.enabled=false` — переопределяет значение в `values.yaml` чарта. По умолчанию Bitnami-чарт Redis создаёт пароль; `false` отключает аутентификацию (для учебного проекта это нормально, для production — **нет**).

## ⚠️ Важный момент: `bitnami/redis` обновился и теперь не работает как раньше

В августе 2025 года **Bitnami изменил политику** — публичные образы и чарты переехали в «Bitnami Secure Images», а старые версии в `charts.bitnami.com/bitnami` помечены как deprecated. Многие туториалы устарели.

**Что делать, если команда `helm install redis bitnami/redis` упадёт:**

Попробуйте современный репозиторий:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/redis  # проверить, что чарт виден
helm install redis bitnami/redis --set auth.enabled=false --set architecture=standalone
```

Я добавил `--set architecture=standalone` — это **одноподовый Redis без реплик**, что для учебного проекта идеально: проще, легче, меньше ресурсов.

Если чарт всё равно не ставится — есть альтернатива **без Helm вообще**, см. ниже.

## 🔍 Что произойдёт после `helm install`

```bash
# Посмотреть релизы
helm list

# Посмотреть, какие ресурсы создал Helm
kubectl get all -l app.kubernetes.io/instance=redis

# Посмотреть статус пода
kubectl get pods -l app.kubernetes.io/name=redis
```

Вы должны увидеть под `redis-master-0` в статусе `Running`. Redis будет доступен внутри кластера по DNS:

```
redis-master.default.svc.cluster.local:6379
```
или, если архитектура standalone:
```
redis.default.svc.cluster.local:6379
```

## 🛠️ Полезные команды Helm

```bash
# Обновить релиз (например, поменять values)
helm upgrade redis bitnami/redis --set auth.enabled=false

# Откатить на предыдущую ревизию
helm rollback redis

# Посмотреть историю ревизий
helm history redis

# Удалить релиз (все созданные ресурсы тоже удалятся)
helm uninstall redis

# Отрендерить YAML без применения (посмотреть, что будет создано)
helm template redis bitnami/redis --set auth.enabled=false

# Посмотреть все значения чарта по умолчанию
helm show values bitnami/redis
```

## 🎓 Альтернатива: Redis без Helm

Если не хотите возиться с Helm, вот минимальный Redis для учебного проекта (одна команда, один под, без persistence — данные пропадут при рестарте, но для rate limiting это не критично):

```bash
kubectl run redis --image=redis:7-alpine --port=6379
kubectl expose pod redis --port=6379 --name=redis
```

Или через манифест `redis.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
```

```bash
kubectl apply -f redis.yaml
```

**Плюс:** полный контроль, никаких неожиданностей.  
**Минус:** нет persistence, нет HA, нет тонкой настройки — но для учебной задачи rate limiting этого достаточно.

## 💎 Что дальше по вашему учебнику

После Redis нужно будет:

1. Развернуть **ratelimit-сервис** — компонент Envoy, который хранит счётчики в Redis и решает «пропустить / заблокировать».
2. Настроить **EnvoyFilter** или **`rateLimit` action** в `VirtualService`, чтобы Istio Ingress Gateway обращался к ratelimit-сервису перед маршрутизацией.
3. Задать **rate limit descriptors** — правила вида «не более 10 запросов в минуту с одного IP на путь `/tasks/create`».

Это следующий большой пласт. Как разберётесь с Redis через Helm (или без него) — можно двигаться дальше.

## 📌 Что я рекомендую для вашего учебного проекта

1. Установите Helm — это навык, который точно пригодится.
2. Попробуйте `helm install redis bitnami/redis --set auth.enabled=false --set architecture=standalone`.
3. Если чарт не ставится из-за изменений Bitnami — не тратьте время, поставьте Redis через `kubectl apply` из манифеста выше. Учебник писался до изменений Bitnami, и цель главы — разобраться с rate limiting, а не с Helm.
4. Дальше по учебнику всё будет работать одинаково, независимо от способа установки Redis.

Если после установки Redis что-то пойдёт не так — пришлите вывод `kubectl get pods` и `helm list`, разберёмся.


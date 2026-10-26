> с этим вроде разобрались, дальше по учебнику идет тема Spring Cloud Config Server. В учебнике вкратце написано, что нужно создать spring boot приложение с зависимостью spring-cloud-config-server, далее создать рядом папку config-repo с конфигами для микросервисов. папку запушить на github и в Spring Cloud Config Server приложении указать ссылку на этот репозиторий. но что делать дальше? приложение Spring Cloud Config Server надо запустить как сервис? нужны базовые deployment/service? а virtualservice нужен? в учебнике у конфиг-сервера прописан server.port: 8888 и application.name:config-server. в ответ в микросервисе tasks-service прописано spring.config.import:optional:configserver:http://config-server:8888. как вообще это все должно работать? какие конфиги можно вынести в папку с конфигами и как вообще взаимодействовать с сервером конфигураций?

Разберём Spring Cloud Config Server по шагам — от развёртывания в Kubernetes до того, как им пользоваться.

## 🧩 Что такое Config Server и зачем он нужен

**Spring Cloud Config Server** — это отдельное приложение, которое **отдаёт конфигурации** другим микросервисам через HTTP. Конфигурации хранятся в **Git-репозитории** (папка `config-repo`), который вы пушите на GitHub.

Схема работы:

```
GitHub (config-repo) ◀── клонирует ─── Config Server (порт 8888)
                                              │
                                              │ HTTP GET /tasks-service/default
                                              ▼
                                       tasks-service (при старте)
```

Когда `tasks-service` стартует, он **сам** обращается к Config Server, забирает свои настройки и применяет их. Config Server **не толкает** конфиги — только отвечает на запросы.

## 📦 Развёртывание Config Server в Kubernetes

### 1. Deployment и Service (обязательно)

Config Server — обычное Spring Boot приложение. Ему нужны те же ресурсы, что и другим сервисам:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-server
  template:
    metadata:
      labels:
        app: config-server
    spec:
      containers:
        - name: config-server
          image: your-registry/config-server:latest
          ports:
            - containerPort: 8888
---
apiVersion: v1
kind: Service
metadata:
  name: config-server
spec:
  selector:
    app: config-server
  ports:
    - name: http
      port: 8888
      targetPort: 8888
```

**Service** нужен, чтобы `tasks-service` мог обратиться к Config Server по DNS-имени `config-server:8888` внутри кластера. Без Service клиенты не найдут Config Server.

### 2. VirtualService (не обязателен)

**VirtualService для Config Server не нужен**, если вы не хотите открывать его снаружи кластера. Config Server — **внутренний** сервис, к нему обращаются только другие микросервисы. Внешний доступ к нему обычно не требуется.

Если по какой-то причине нужен внешний доступ (например, для отладки), тогда можно добавить маршрут в существующий Gateway:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: config-server-vs
spec:
  hosts:
    - "*"
  gateways:
    - tasks-app-gateway
  http:
    - match:
        - uri:
            prefix: /config
      rewrite:
        uri: /
      route:
        - destination:
            host: config-server
            port:
              number: 8888
```

Но для учебной задачи **достаточно Deployment + Service**.

### 3. Sidecar Istio (опционально)

Если namespace помечен для инъекции, sidecar появится автоматически. Это не мешает работе — Config Server будет доступен по mTLS внутри mesh. Если sidecar не нужен, можно отключить аннотацией `sidecar.istio.io/inject: "false"`.

## 📂 Структура папки `config-repo`

Имена файлов подчиняются строгому правилу:

```
config-repo/
├── application.yml              ← общие настройки для ВСЕХ сервисов
├── tasks-service.yml            ← настройки для tasks-service (все профили)
├── tasks-service-dev.yml        ← настройки для tasks-service в профиле dev
├── tasks-service-prod.yml       ← настройки для tasks-service в профиле prod
├── spring-cloud-gateway.yml     ← настройки для Gateway
└── config-server.yml            ← настройки для самого Config Server (опционально)
```

**Правило именования:** `{application.name}-{profile}.yml`. Config Server сопоставляет `spring.application.name` клиента с именем файла.

Например, если у `tasks-service` в `application.yml` указано:
```yaml
spring:
  application:
    name: tasks-service
```
и активный профиль `default`, Config Server отдаст содержимое `tasks-service.yml` + `application.yml`.

### Что можно вынести в config-repo

Всё, что **не является секретом**:

- **Параметры подключения к БД** (без пароля): URL, имя пользователя, настройки пула.
- **Настройки логирования**: уровни для пакетов, формат.
- **Кастомные свойства**: флаги фич, лимиты, таймауты.
- **Настройки Kafka**: адреса брокеров, топики.
- **Общие настройки** в `application.yml`: например, `management.endpoints.web.exposure.include`.

**Пароли и токены** лучше хранить в **Kubernetes Secret** и подключать через `spring.config.import: configtree:/mnt/secrets/`. Config Server для секретов — не лучшая практика, потому что Git-репозиторий может быть доступен большему числу людей.

## 🔗 Как `tasks-service` общается с Config Server

В `application.yml` у `tasks-service` указано:

```yaml
spring:
  config:
    import: optional:configserver:http://config-server:8888
```

**Разбор:**

- `configserver:` — говорит Spring Boot: «импортируй настройки из Config Server».
- `http://config-server:8888` — адрес Config Server **внутри кластера**. `config-server` — имя Kubernetes Service.
- `optional:` — если Config Server недоступен, приложение **всё равно запустится** с локальными настройками. Без `optional:` оно упадёт с ошибкой.

### Что происходит при старте `tasks-service`

1. Spring Boot читает локальный `application.yml`.
2. Видит `spring.config.import: optional:configserver:http://config-server:8888`.
3. Отправляет GET-запрос: `http://config-server:8888/tasks-service/default`.
4. Config Server клонирует Git-репозиторий, находит `tasks-service.yml` и `application.yml`, объединяет их и возвращает JSON с настройками.
5. Spring Boot применяет полученные настройки **поверх** локальных. **Приоритет:** Config Server > локальный `application.yml`.
6. Приложение стартует с итоговой конфигурацией.

## 🛠️ Настройка самого Config Server

В `application.yml` Config Server указывает, где лежит Git-репозиторий:

```yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-username/config-repo.git
          default-label: main
          clone-on-start: true
```

**Ключевые параметры:**

| Параметр | Что делает |
|---|---|
| `spring.cloud.config.server.git.uri` | URL Git-репозитория с конфигами |
| `default-label` | Ветка по умолчанию (`main`, `master`) |
| `clone-on-start` | Клонировать репозиторий при старте (а не при первом запросе) |
| `username` / `password` | Если репозиторий приватный |

Для **приватного репозитория** лучше использовать SSH-ключ или токен, переданный через Kubernetes Secret.

## 🎯 Схема целиком

```
┌─────────────────┐
│ GitHub          │
│ config-repo/    │
│  ├─ application.yml
│  ├─ tasks-service.yml
│  └─ gateway.yml
└────────┬────────┘
         │ git clone
         ▼
┌─────────────────┐         ┌──────────────────┐
│ Config Server   │◀────────│ tasks-service    │
│ (K8s Service)   │  HTTP   │ spring.config.   │
│ порт 8888       │  GET    │ import:configserver│
└─────────────────┘         └──────────────────┘
                                    │
                                    │ применяет настройки
                                    ▼
                            приложение стартует
```

## 📌 Важные нюансы

**1. Config Server не «толкает» конфиги.** Клиент сам запрашивает их при старте. Если вы изменили файл в Git, `tasks-service` **не узнает об этом**, пока не перезапустится (или пока не вызвать `/actuator/refresh`).

**2. Порядок запуска.** Config Server должен быть запущен **до** того, как стартуют клиенты. В Kubernetes это решается `initContainer` с ожиданием, или `optional:` в `spring.config.import` — тогда клиент стартует даже если Config Server ещё не готов, но с локальными настройками.

**3. Config Server сам может быть клиентом.** Если у Config Server есть свои настройки в том же Git-репозитории (файл `config-server.yml`), он может читать их оттуда. Но обычно его `application.yml` лежит локально в образе.

**4. Безопасность.** Config Server отдаёт настройки **без аутентификации** по умолчанию. Если он доступен снаружи кластера — любой может прочитать конфиги. Поэтому его **не открывают через Istio Ingress** без необходимости.

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| Нужен ли Deployment? | ✅ Да |
| Нужен ли Service? | ✅ Да, чтобы клиенты нашли его по DNS |
| Нужен ли VirtualService? | ❌ Нет, если не нужен внешний доступ |
| Что класть в config-repo? | Всё, кроме секретов |
| Как клиент получает настройки? | `spring.config.import: configserver:http://config-server:8888` |
| Когда клиент запрашивает конфиги? | При старте |
| Что если Config Server упал? | С `optional:` клиент стартует с локальными настройками |

Config Server — это **внутренний сервис**, который работает как «раздатчик настроек» для других микросервисов. Он не участвует в обработке пользовательских запросов и не нуждается в VirtualService. Достаточно Deployment + Service, а остальное — вопрос дисциплины хранения конфигов в Git.



> не, спасибо, этого достаточно. давай вернемся к config-server. получается, я создал папку config-repo, в нее положил конфиги например tasks-service. Указал на нее ссылку в Config-server. При старте он прочитает из нее конфиги, а мой микросервис при старте прочитает конфиги из config-server. так? я хочу попробовать запустить на самом простом примере: 
у меня в tasks-service в application.yaml есть настройка подключения в бд:
  mongodb:
    uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}

> мне достаточно вынести ее в config-repo/tasks-service.yaml? а как микросервис узнает из какого файла читать? ну то есть tasks-service откуда знает что надо читать из config-repo/tasks-service.yaml?

Да, вы всё поняли правильно. Разберу по шагам и отвечу на главный вопрос — «откуда tasks-service знает, какой файл читать».

## 🔄 Полная цепочка

```
GitHub (config-repo)
   └── tasks-service.yaml          ← здесь лежит mongodb.uri
         │
         │ Config Server клонирует репо при старте
         ▼
Config Server (:8888)
   └── отдаёт GET /tasks-service/default  →  JSON с настройками
         │
         │ tasks-service запрашивает при старте
         ▼
tasks-service
   └── применяет mongodb.uri поверх локального application.yaml
```

## 🎯 Откуда tasks-service знает, какой файл читать

**Ключ — в `spring.application.name`.** Config Server использует его как имя файла.

У вас в `tasks-service/application.yaml` уже есть:

```yaml
spring:
  application:
    name: tasks-service
```

Когда `tasks-service` стартует, он отправляет в Config Server запрос:

```
GET http://config-server:8888/tasks-service/default
                                    ↑             ↑
                              application.name   profile
```

Config Server видит:
- **`tasks-service`** → ищет файл `tasks-service.yml` (или `.yaml`, или `.properties`).
- **`default`** → активный профиль (если `spring.profiles.active` не задан, используется `default`).

**Совпадение обязательно.** Если `spring.application.name` = `tasks-service`, а файл называется `task-service.yaml` (без `s`) — Config Server ничего не найдёт.

## 📝 Что нужно сделать

### 1. В `config-repo` создайте `tasks-service.yaml`

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}
```

**Важно:** имя файла — ровно `tasks-service.yaml`, совпадает с `spring.application.name`.

### 2. В `tasks-service/application.yaml` оставьте только «точку входа»

```yaml
spring:
  application:
    name: tasks-service
  config:
    import: optional:configserver:http://config-server:8888
```

Секцию с `mongodb.uri` можно **убрать** из локального файла — она теперь живёт в Git.

### 3. Запушьте `config-repo` на GitHub

```bash
cd config-repo
git add tasks-service.yaml
git commit -m "Add tasks-service config"
git push
```

### 4. Убедитесь, что Config Server указывает на этот репозиторий

В `config-server/application.yaml`:

```yaml
server:
  port: 8888
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-username/config-repo.git
          default-label: main
          clone-on-start: true
```

### 5. Запустите Config Server и проверьте

Разверните Config Server в Kubernetes (Deployment + Service), затем проверьте его изнутри кластера:

```bash
kubectl port-forward svc/config-server 8888:8888

curl http://localhost:8888/tasks-service/default | jq
```

Должны увидеть JSON с вашими настройками:

```json
{
  "name": "tasks-service",
  "profiles": ["default"],
  "propertySources": [
    {
      "name": "https://github.com/.../tasks-service.yaml",
      "source": {
        "spring.data.mongodb.uri": "mongodb://localhost:27017/tasksdb"
      }
    }
  ]
}
```

Если видите `mongodb.uri` — Config Server работает, и `tasks-service` его получит при старте.

## 🔍 Как проверить, что tasks-service получил настройки

После перезапуска `tasks-service` посмотрите его логи:

```bash
kubectl logs deploy/tasks-service -c tasks-service | grep -i "config\|mongodb"
```

Ищите строки вроде:
```
Located property source: [BootstrapPropertySource {name='bootstrapProperties-configClient'}]
Fetching config from server at : http://config-server:8888
Located environment: name=tasks-service, profiles=[default], ...
```

Или проверьте через actuator:

```bash
kubectl port-forward svc/tasks-service 8080:80
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.name | contains("configserver"))'
```

Там будет видно, что `mongodb.uri` пришёл из Config Server, а не из локального файла.

## ⚠️ Три важных нюанса

### 1. `optional:` — что это значит

```yaml
spring.config.import: optional:configserver:http://config-server:8888
```

Префикс `optional:` говорит: «если Config Server недоступен — не падай, запускайся с локальными настройками». **Для учебного проекта это удобно.**

Без `optional:` если Config Server не запущен — `tasks-service` **упадёт** при старте с ошибкой. В production это правильно: лучше не запуститься, чем работать с неправильными настройками.

### 2. Порядок применения настроек

Config Server имеет **приоритет** над локальным `application.yaml`. Если одно и то же свойство есть в обоих местах — победит Config Server.

Это удобно: например, локально у вас `mongodb://localhost`, а в Git — реальный адрес. В кластере применится адрес из Git.

### 3. Обновление настроек

Config Server читает Git **при старте** (или при первом запросе, если `clone-on-start: false`). Если вы **изменили файл в Git**:

- `tasks-service` **не узнает** об этом автоматически.
- Нужно либо **перезапустить** `tasks-service`, либо вызвать `/actuator/refresh` (если настроен Spring Cloud Bus — но это отдельная тема).

Для учебной задачи — перезапуск пода:

```bash
kubectl rollout restart deploy/tasks-service
```

## 📌 Что можно вынести в config-repo

Для начала — **только `mongodb.uri`**, как вы планируете. Потом можно добавить:

- Настройки подключения к Kafka (`spring.kafka.bootstrap-servers`).
- Уровни логирования (`logging.level.*`).
- Кастомные свойства приложения (`app.feature.enabled`, `app.limits.max-tasks`).
- Настройки Redis, если он используется в приложении.

**Что не стоит выносить:** пароли, токены, `client_secret`. Для них — Kubernetes Secret + `spring.config.import: configtree:/mnt/secrets/`.

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| Как tasks-service узнаёт, какой файл читать? | По `spring.application.name` = имя файла в config-repo |
| Достаточно ли вынести только `mongodb.uri`? | Да, но нужно убрать её из локального yaml (или оставить как fallback) |
| Что если файл назван иначе? | Config Server не найдёт настройки, приложение запустится с локальными |
| Как обновить настройки в Git? | Перезапустить под tasks-service |
| Нужен ли VirtualService для Config Server? | Нет, это внутренний сервис |

Схема простая: **Git → Config Server → tasks-service при старте**. Имя файла в Git = `spring.application.name` клиента. Всё остальное — детали.

> помоги разобраться с git. адрес моего репозитория: https://github.com/dmitriy0704/todos-microcervice, папка config-repo локально лежит прям в корне проекта: todos-microcervice/config-repo, рядом с todos-microcervice/tasks-service. Ветка feature/configurations. В конфиге config-server я указал:

 > config:
      server:
        git:
          uri: https://github.com/dmitriy0704/todos-microcervice
          search-paths: config-repo
          default-label: feature/configurations

> но почему- то на Github полный путь к папке config-repo указан с учетом ветки:
> https://github.com/dmitriy0704/todos-microcervice/tree/feature/configurations/config-repo

> в итоге, что мне писать в config-server/application.yaml?

Тут путаница между **двумя разными URL**. Давайте разберём.

## 🔍 Два разных URL для одного репозитория

| URL | Для чего |
|---|---|
| `https://github.com/dmitriy0704/todos-microcervice/tree/feature/configurations/config-repo` | **Веб-интерфейс GitHub** — то, что вы видите в браузере, когда листаете репо |
| `https://github.com/dmitriy0704/todos-microcervice.git` | **Git clone URL** — то, что нужно Config Server |

`/tree/feature/configurations/config-repo` — это **часть веб-интерфейса**, а не структура репозитория. Config Server не поймёт такой URL — он попытается клонировать репозиторий с именем `todos-microcervice/tree/feature/configurations/config-repo`, которого не существует.

## ✅ Правильный конфиг для вашего случая

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/dmitriy0704/todos-microcervice.git
          search-paths: config-repo
          default-label: feature/configurations
```

**Разбор параметров:**

- **`uri`** — корень репозитория с `.git` на конце. Именно этот URL вы бы использовали в `git clone`.
- **`search-paths`** — подпапка внутри репозитория, где лежат конфиги. У вас — `config-repo`.
- **`default-label`** — ветка, из которой читать конфиги. У вас — `feature/configurations`.

## ⚠️ Возможная проблема: слэш в имени ветки

Ветка `feature/configurations` содержит `/`. В большинстве случаев Spring Cloud Config с ней справляется, но иногда бывают нюансы — особенно при запросе с явным профилем. Если что-то не заработает, попробуйте для теста создать простую ветку без слэша (например, `config-test`) и переключиться на неё:

```yaml
default-label: config-test
```

Для учебного проекта это лишняя возня, так что сначала попробуйте с `feature/configurations`.

## 🧪 Как проверить

1. **Убедитесь, что файл действительно в репозитории**:

```bash
# Смотрим, что в ветке лежит
git ls-tree -r feature/configurations --name-only | grep config-repo
```

Должны увидеть что-то вроде:
```
config-repo/tasks-service.yaml
```

2. **Запустите Config Server** и проверьте:

```bash
kubectl port-forward svc/config-server 8888:8888
curl http://localhost:8888/tasks-service/default | jq
```

3. **Смотрите логи Config Server при старте**:

```bash
kubectl logs deploy/config-server | grep -i "git\|clone\|config-repo"
```

Ищите строки:
- `Adding property source: ...` — успешно нашёл конфиги
- `Cannot clone or checkout repository` — проблема с URI или веткой
- `No such label: feature/configurations` — проблема с веткой

## 🎯 Что важно понять

**Config Server работает так:**

1. Клонирует репозиторий по `uri` в локальную директорию.
2. Делает `git checkout` на ветку из `default-label` (если не указана — `main` или `master`).
3. Смотрит в подпапку `search-paths`.
4. Ищет файл с именем, соответствующим запросу: `<application.name>-<profile>.yaml` или `<application.name>.yaml`.

В вашем случае:
- Клонирует `https://github.com/dmitriy0704/todos-microcervice.git`
- Чекаутит `feature/configurations`
- Заходит в `config-repo/`
- Ищет `tasks-service.yaml` (при запросе `/tasks-service/default`)

## 💎 Итог

```yaml
uri: https://github.com/dmitriy0704/todos-microcervice.git   # ← clone URL, с .git
search-paths: config-repo                                     # ← папка внутри репо
default-label: feature/configurations                         # ← ветка
```

URL вида `/tree/feature/configurations/config-repo` из браузера — **никогда не используйте** в конфиге. Это интерфейс GitHub, а не адрес для клонирования. Правильный URL — тот, что выдаёт кнопка «Code → HTTPS» на странице репозитория.


> в логах config-server - только 7 строчек про запуск приложения и все. при попытке отправить post- запрос на tasks-service в его консоли выводится ошибка про то, что не возможно сохранить объект - я же в конфиге tasks-service убрал настройки подключения к БД и перенес их в config-repo. по идее, config-server должен был прочитать их

Логи Config Server — ключевая подсказка. **Если там только старт приложения и ни строчки про Git — значит, он не клонировал репозиторий.** Скорее всего, он либо не знает, что он Config Server, либо не получил запрос от `tasks-service`.

Разберём по цепочке.

## 🔍 Шаг 1: Проверьте, что Config Server действительно Config Server

**Самая частая ошибка:** в `pom.xml` добавлена зависимость, но приложение не активирует сервер. Или зависимость вообще забыта.

Проверьте `pom.xml` config-server:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

И в главном классе должна быть аннотация:

```java
@SpringBootApplication
@EnableConfigServer    // ← вот эта
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**Проверьте напрямую** — откройте в браузере:

```bash
kubectl port-forward svc/config-server 8888:8888
curl http://localhost:8888/tasks-service/default
```

Если в ответе **не JSON с настройками**, а что-то вроде `{"timestamp":..., "status":404}` или вообще HTML — Config Server не работает как сервер конфигураций.

**Как выглядит правильный ответ:**
```json
{
  "name": "tasks-service",
  "profiles": ["default"],
  "propertySources": [
    {
      "name": "https://github.com/.../tasks-service.yaml",
      "source": {
        "spring.data.mongodb.uri": "mongodb://..."
      }
    }
  ]
}
```

## 🔍 Шаг 2: Проверьте, что tasks-service обращается к Config Server

**Вторая частая ошибка:** `tasks-service` не знает, что надо идти в Config Server. Это не происходит «само собой» — нужна зависимость и настройка.

### Проверьте `pom.xml` tasks-service:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

**Без этой зависимости** `spring.config.import: configserver:...` **игнорируется молча**. Приложение просто стартует с локальными настройками, даже не пытаясь обратиться к Config Server.

### Проверьте версию Spring Cloud

В Spring Boot 2.4+ механизм bootstrap изменился. Есть два варианта:

**Вариант A (современный, рекомендуется):**
```yaml
spring:
  config:
    import: optional:configserver:http://config-server:8888
```

**Вариант B (legacy, через bootstrap):**
Добавьте в `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```
И тогда можно использовать `bootstrap.yaml` со старым синтаксисом:
```yaml
spring:
  cloud:
    config:
      uri: http://config-server:8888
```

**Смешивать нельзя.** Если используете `spring.config.import` — не нужен `bootstrap`. Если используете `bootstrap.yaml` — не нужен `spring.config.import`.

## 🔍 Шаг 3: Посмотрите логи tasks-service при старте

Это самый информативный шаг:

```bash
kubectl logs deploy/tasks-service -c tasks-service | head -100
```

Ищите строки:

| Строка в логах | Что означает |
|---|---|
| `Fetching config from server at : http://config-server:8888` | ✅ Запрос отправлен |
| `Located environment: name=tasks-service, profiles=[default]` | ✅ Config Server ответил |
| `Located property source: ...` | ✅ Настройки получены |
| `Could not locate PropertySource` | ❌ Config Server ответил, но конфиг не найден |
| `Connection refused` / `UnknownHost` | ❌ Config Server недоступен |
| **Ничего из этого** | ❌ Зависимость `spring-cloud-starter-config` не подключена |

## 🔍 Шаг 4: Проверьте, что Config Server видит Git

Если `curl http://localhost:8888/tasks-service/default` вернул **пустой** `propertySources` или `404`:

**Причина A:** Config Server не нашёл файл `tasks-service.yaml` в репозитории.

Проверьте, что файл лежит **именно** в `config-repo/tasks-service.yaml` в ветке `feature/configurations`:

```bash
cd todos-microcervice
git checkout feature/configurations
git ls-tree -r HEAD --name-only | grep tasks-service
```

Должны увидеть:
```
config-repo/tasks-service.yaml
```

Если файл не в этой ветке, или в другой папке, или назван иначе — Config Server его не найдёт.

**Причина B:** Config Server не смог склонировать репозиторий (приватный? нет доступа?).

Смотрите логи при старте **подробно**:

```bash
kubectl logs deploy/config-server | grep -i "git\|clone\|repo\|config"
```

При `clone-on-start: true` вы должны видеть что-то вроде:
```
Adding property source: ...
Fetching config from server at : ...
```

Если этих строк **нет вообще** — значит, Config Server не инициировал клонирование. Это происходит, если:
- URI неправильный.
- Ветка (`default-label`) не существует.
- Нет прав на репозиторий.

## 🛠️ Пошаговый план

1. **Проверьте `curl http://localhost:8888/tasks-service/default`** — что возвращает?

2. **Проверьте логи config-server** — есть ли там вообще `git`-активность?

3. **Проверьте `pom.xml` tasks-service** — есть ли `spring-cloud-starter-config`?

4. **Проверьте логи tasks-service при старте** — пытается ли он обратиться к Config Server?

Пришлите вывод этих четырёх проверок — и станет точно ясно, где разрыв.

## 💡 Быстрая диагностика одной командой

Изнутри пода tasks-service:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- wget -qO- http://config-server:8888/tasks-service/default
```

Если это сработает и вернёт JSON — проблема в tasks-service (не запрашивает).  
Если вернёт `404` — проблема в config-repo (файл не найден).  
Если не подключится вообще — проблема в сети или config-server.

Пришлите вывод — и определим точно.

> 1. Проверьте curl http://localhost:8888/tasks-service/default — что возвращает?
{"name":"tasks-service","profiles":["default"],"label":null,"version":"a8d25938f65ffdc0d9a5e3ac68b2e64517bbaeaf","state":"","propertySources":[{"name":"https://github.com/dmitriy0704/todos-microcervice.git/config-repo/tasks-service.yaml","source":{"spring.data.mongodb.uri":"${MONGO_URI:mongodb://localhost:27017/tasksdb}"}}]}% 
> 2. Проверьте логи config-server — есть ли там вообще git-активность?
При запросе curl http://localhost:8888/tasks-service/default
появились в консоли config-server появились строчки:
Adding property source: Config resource 'file [/tmp/config-repo-16814943375764487429/config-repo/tasks-service.yaml]' via location 'file:/tmp/config-repo-16814943375764487429/config-repo/'
> 3.Проверьте pom.xml tasks-service — есть ли spring-cloud-starter-config?
есть
> 4. Проверьте логи tasks-service при старте — пытается ли он обратиться к Config Server?
ConnectException: Connection refused (connect failed)

Отличная диагностика! Теперь картина полностью ясна. Разберу по пунктам.

## ✅ Что работает

| Проверка | Результат |
|---|---|
| Config Server отдаёт конфиг | ✅ Идеально — JSON с `mongodb.uri` из Git |
| Git-репозиторий клонируется | ✅ В логах видно `Adding property source: ... tasks-service.yaml` |
| `spring-cloud-starter-config` в tasks-service | ✅ Есть |

**Config Server работает корректно.** Проблема в **сетевом соединении** между `tasks-service` и `config-server`.

## ❌ Что не работает: `Connection refused`

```
ConnectException: Connection refused (connect failed)
```

Это **не** `UnknownHost` (DNS бы сказал `UnknownHostException`). Значит:

- **DNS резолвит** `config-server` — имя найдено.
- **TCP-соединение отвергается** — на том конце либо никто не слушает нужный порт, либо Service пустой.

## 🔍 Проверьте три вещи

### 1. Service `config-server` и его endpoints

```bash
kubectl get svc config-server
kubectl get endpoints config-server
kubectl get pods -l app=config-server
```

**Что смотреть:**

- `kubectl get svc config-server` — какой порт у Service? Должен быть `8888`.
- `kubectl get endpoints config-server` — есть ли там IP пода? Если `ENDPOINTS` = `<none>` — селектор Service не совпадает с метками пода, или под не `Ready`.
- `kubectl get pods -l app=config-server` — под в статусе `Running`, `READY 1/1` или `2/2`?

**Наиболее вероятная причина:** в `Service` селектор не совпадает с метками пода. Например, Service ищет `app: config-server`, а у пода метка `app: configserver` или `name: config-server`.

### 2. Порт в `spring.config.import`

```yaml
spring:
  config:
    import: optional:configserver:http://config-server:8888
```

Порт `8888` должен совпадать с портом Service. Если Service слушает `80` и форвардит на `8888` — тогда в URL должно быть `http://config-server:80` (или просто `http://config-server` — по умолчанию 80).

Проверьте:
```bash
kubectl get svc config-server -o yaml | grep -A5 ports
```

### 3. Namespace

```bash
kubectl get pods -A | grep -E "config-server|tasks-service"
```

Если `config-server` и `tasks-service` в **разных** namespace — короткое имя `config-server` не сработает. Нужно полное DNS-имя:

```yaml
spring:
  config:
    import: optional:configserver:http://config-server.<namespace>.svc.cluster.local:8888
```

## 🎯 Главный вопрос: почему `Connection refused`, а не таймаут

`Connection refused` означает, что пакет **дошёл** до какого-то хоста, но **порт закрыт**. Это бывает в трёх случаях:

**A. Service существует, но endpoints пустые.** `kube-proxy` отвергает соединение, потому что некуда его перенаправлять. Это самая частая причина.

**B. Порт в URL не совпадает с портом Service.** Например, вы стучитесь на `8888`, а Service слушает `80`.

**C. Sidecar Istio блокирует соединение.** Если `config-server` без sidecar, а в namespace включён `STRICT` mTLS — Envoy отвергнет соединение. Но у вас `PeerAuthentication` был `PERMISSIVE`, так что это маловероятно.

## 🛠️ Пошаговое решение

### Шаг 1: Проверьте endpoints

```bash
kubectl get endpoints config-server
```

Если пусто — исправьте Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: config-server
spec:
  selector:
    app: config-server      # ← должно совпадать с меткой пода!
  ports:
    - name: http
      port: 8888
      targetPort: 8888
```

Проверьте метку пода:
```bash
kubectl get pod -l app=config-server --show-labels
```

### Шаг 2: Проверьте связь изнутри tasks-service

```bash
kubectl exec deploy/tasks-service -c tasks-service -- \
  wget -qO- http://config-server:8888/tasks-service/default
```

Если здесь `Connection refused` — проблема в Service/endpoints. Если JSON — значит, сеть работает, и `tasks-service` **при старте** просто не успел подключиться (Config Server был не готов).

### Шаг 3: Перезапустите tasks-service

Если Config Server стал доступен **после** старта `tasks-service` — он не перезапросит конфиг сам. Нужно перезапустить:

```bash
kubectl rollout restart deploy/tasks-service
kubectl rollout status deploy/tasks-service
kubectl logs deploy/tasks-service -c tasks-service | grep -i "config\|mongodb"
```

## ⚠️ Важный нюанс с `optional:`

Вы убрали `mongodb.uri` из локального `application.yaml` и оставили её только в Git. **Но `optional:` в `spring.config.import` означает:** если Config Server недоступен — приложение запустится **без** этой настройки. Оно **не упадёт при старте**, но:

- `mongodb.uri` будет отсутствовать.
- При первом запросе на сохранение — ошибка, потому что некуда подключаться.

**Это ровно то, что вы наблюдаете.** `tasks-service` стартовал, не получил конфиг, продолжил работать «пустым» — а потом упал на запросе.

**Два варианта:**

**Вариант A:** Уберите `optional:` → если Config Server недоступен, tasks-service **не запустится** вообще. Это правильно для production:

```yaml
spring:
  config:
    import: configserver:http://config-server:8888
```

**Вариант B:** Оставьте `optional:` для отладки, но убедитесь, что Config Server **точно доступен** к моменту старта tasks-service.

**Также оставьте fallback в локальном `application.yaml`:**

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}
```

Тогда, даже если Config Server недоступен, приложение запустится с локальным значением. Config Server **переопределит** его, когда станет доступен.

## 💎 Итог

| Что проверить | Команда | Ожидаемо |
|---|---|---|
| Endpoints config-server | `kubectl get endpoints config-server` | IP:8888, не пусто |
| Связь из tasks-service | `kubectl exec ... wget http://config-server:8888/...` | JSON |
| Namespace | `kubectl get pods -A \| grep config-server` | совпадает с tasks-service |
| Порт в URL | `kubectl get svc config-server -o yaml` | порт совпадает |

**Наиболее вероятная причина** — Service `config-server` имеет пустые endpoints (селектор не совпадает с метками пода), либо `tasks-service` стартовал раньше, чем Config Server стал готов.

Проверьте endpoints — это первое, что нужно сделать. Пришлите вывод `kubectl get endpoints config-server` — и станет ясно, куда копать.

> config-server   10.244.1.143:8888   83m

Endpoints корректные — `10.244.1.143:8888`. Значит, Service работает, порт правильный. **Проблема была во времени старта.**

## 🎯 Что произошло

`tasks-service` стартовал **раньше**, чем `config-server` стал готов принимать запросы. В момент старта `tasks-service` попытался подключиться, получил `Connection refused`, и — поскольку в `spring.config.import` стоит `optional:` — **продолжил запуск без конфига**.

`tasks-service` **не перезапрашивает** конфиг сам. Он обращается к Config Server **только один раз — при старте**. Поэтому, даже когда Config Server поднялся, `tasks-service` уже работает с пустыми настройками.

## ✅ Решение: перезапустите tasks-service

```bash
kubectl rollout restart deploy/tasks-service
kubectl rollout status deploy/tasks-service
```

Проверьте логи при старте:

```bash
kubectl logs deploy/tasks-service -c tasks-service --tail=100 | grep -i "config\|mongodb\|located"
```

Теперь должны увидеть строки, которых раньше не было:

```
Fetching config from server at : http://config-server:8888
Located environment: name=tasks-service, profiles=[default], ...
Located property source: [BootstrapPropertySource ...]
```

Это значит, что конфиг получен.

## 🔍 Проверьте, что настройка применилась

После перезапуска:

```bash
kubectl port-forward svc/tasks-service 8080:80
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.name | contains("configserver"))'
```

Или проще — сделайте POST-запрос на создание задачи. Если раньше была ошибка про невозможность сохранить — теперь должна пройти.

## ⚠️ На будущее: три способа избежать этой проблемы

### 1. Убрать `optional:` из `spring.config.import`

```yaml
spring:
  config:
    import: configserver:http://config-server:8888
```

Тогда `tasks-service` **не запустится**, пока не получит конфиг. Если Config Server недоступен — Kubernetes будет рестартовать под, пока Config Server не появится. Это **правильно для production**.

### 2. Оставить fallback в локальном `application.yaml`

Если хотите `optional:` для устойчивости, оставьте **значение по умолчанию** локально:

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}
```

Тогда при недоступном Config Server приложение хотя бы не упадёт при запросах — просто будет использовать localhost. Но в кластере это всё равно не сработает, так что это костыль.

### 3. Init-контейнер для ожидания Config Server

Добавьте в `tasks-service` init-контейнер, который подождёт доступности Config Server **до** старта основного контейнера:

```yaml
spec:
  initContainers:
    - name: wait-for-config
      image: busybox
      command:
        - sh
        - -c
        - |
          until wget -qO- http://config-server:8888/actuator/health; do
            echo "waiting for config-server..."
            sleep 2
          done
  containers:
    - name: tasks-service
      # ...
```

Это самый надёжный вариант, но избыточный для учебного проекта.

## 💎 Итог

| Проверка | Результат |
|---|---|
| Config Server работает | ✅ |
| Git-репозиторий читается | ✅ |
| Endpoints config-server | ✅ `10.244.1.143:8888` |
| Сеть работает | ✅ |
| Проблема | ⏱️ `tasks-service` стартовал раньше Config Server |

**Одна команда для проверки:**

```bash
kubectl rollout restart deploy/tasks-service && \
sleep 20 && \
kubectl logs deploy/tasks-service -c tasks-service --tail=50 | grep -i "config\|mongodb"
```

Если увидите `Located property source` — всё заработало.

## 📌 Что делать дальше

1. **Перезапустите tasks-service** — конфиг подтянется.
2. **Уберите `optional:`** из `spring.config.import` — чтобы такая ситуация больше не повторялась молча.
3. **Проверьте, что Config Server стартует раньше.** В production для этого используют init-контейнеры или `optional:` + fallback. Для учебного проекта достаточно просто перезапустить под.

`Connection refused` был **одноразовой** проблемой при старте. Config Server работает, Git читается, сеть в порядке — осталось только перечитать конфиг.

> выполнил все три шага, но Connection refused остался

Раз `Connection refused` остался после перезапуска — значит, диагноз «стартовал раньше» не подходит. Нужно копать глубже. Разберём по шагам.

## 🔍 Шаг 1: Убедитесь, что вы смотрите свежие логи

Очень частая ловушка: `kubectl logs` показывает логи текущего пода, но если вы **не пересобирали образ**, новый под запускает **старый код** с `optional:` и старым `application.yaml`. Изменения в исходниках не применятся, пока вы не:

```bash
# 1. Пересобрать образ
docker build -t your-registry/tasks-service:latest .

# 2. Загрузить в minikube
minikube image load your-registry/tasks-service:latest

# 3. Перезапустить под
kubectl rollout restart deploy/tasks-service
```

**Проверьте это первым делом.** Если вы меняли `application.yaml`, но не пересобирали образ — все правки остались только в исходниках.

## 🔍 Шаг 2: Проверьте время в логах

```bash
kubectl logs deploy/tasks-service -c tasks-service --tail=200 --timestamps | grep -i "connection refused\|config"
```

Посмотрите на **timestamp** у строки с `Connection refused`. Если время — «до» вашего последнего `rollout restart`, это **старые** логи, и проблема уже могла быть решена.

Если логи действительно свежие:

```bash
# Сколько раз под перезапускался
kubectl get pods -l app=tasks-service
# Если RESTARTS > 0 — под падает
kubectl describe pod -l app=tasks-service | grep -A10 "Last State"
```

## 🔍 Шаг 3: Проверьте связь изнутри пода прямо сейчас

Это ключевой тест. Зайдите в под и попробуйте подключиться к Config Server:

```bash
kubectl exec -it deploy/tasks-service -c tasks-service -- sh
```

Внутри пода:

```bash
# Проверяем DNS
nslookup config-server

# Проверяем TCP-соединение
wget -qO- http://config-server:8888/tasks-service/default

# Если wget нет — используйте busybox-style через /dev/tcp (если bash)
```

**Результаты:**

| Что видите | Что означает |
|---|---|
| `nslookup` не находит `config-server` | Проблема с DNS |
| `wget` возвращает `Connection refused` | Проблема с Service / endpoints / портом |
| `wget` возвращает JSON | Сеть работает, проблема в приложении |
| `wget` висит и таймаутит | Network policy или iptables блокирует |

## 🔍 Шаг 4: Проверьте, что tasks-service и config-server в одном namespace

```bash
kubectl get pods -A | grep -E "config-server|tasks-service"
```

Если они в **разных** namespace — короткое имя `config-server` не сработает. Нужно полное DNS-имя:

```yaml
spring:
  config:
    import: configserver:http://config-server.default.svc.cluster.local:8888
```

## 🔍 Шаг 5: Проверьте sidecar Istio

Если `tasks-service` имеет sidecar, а `config-server` — нет (или наоборот), могут быть проблемы с mTLS.

```bash
kubectl get pods -l app=tasks-service
kubectl get pods -l app=config-server
```

Посмотрите колонку `READY`:
- `2/2` — есть sidecar
- `1/1` — нет sidecar

**Если у `tasks-service` sidecar, а у `config-server` нет** — Envoy может блокировать соединение, если где-то включён `STRICT` mTLS. Проверьте:

```bash
kubectl get peerauthentication -A
```

Если `STRICT` — либо уберите его, либо добавьте sidecar к config-server.

## 🔍 Шаг 6: Проверьте `application.yaml` прямо внутри пода

Убедитесь, что в работающем поде действительно тот конфиг, что вы ожидаете:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- cat /app/application.yaml
```

Если внутри `optional:` всё ещё есть — образ не пересобран.

## 📋 Что прислать для точного диагноза

Выполните и пришлите вывод:

```bash
# 1. Свежие логи
kubectl logs deploy/tasks-service -c tasks-service --tail=50 --timestamps

# 2. Проверка изнутри пода
kubectl exec deploy/tasks-service -c tasks-service -- wget -qO- http://config-server:8888/tasks-service/default 2>&1

# 3. Namespace обоих
kubectl get pods -A | grep -E "config-server|tasks-service"

# 4. READY-статус
kubectl get pods -l app=tasks-service
kubectl get pods -l app=config-server

# 5. PeerAuthentication
kubectl get peerauthentication -A
```

## 💡 Наиболее вероятные причины (по частоте)

| № | Причина | Как проверить |
|---|---|---|
| 1 | **Образ не пересобран** — правки в `application.yaml` не применились | `kubectl exec ... cat application.yaml` |
| 2 | **Разные namespace** — `config-server` не резолвится | `kubectl get pods -A \| grep config-server` |
| 3 | **Смотрите старые логи** — `--previous` не указан, но под падал | `kubectl describe pod` → `Last State` |
| 4 | **Sidecar не даёт соединиться** | `kubectl get peerauthentication -A` |
| 5 | **Порт в URL не тот** — например, `8888` vs `80` | `kubectl get svc config-server -o yaml \| grep port` |

Начните с Шага 3 — зайдите в под `tasks-service` и попробуйте `wget` до Config Server. Это сразу локализует проблему: сеть, DNS или приложение.

> 1. kubectl logs deploy/tasks-service -c tasks-service --tail=50 --timestamps

> 2026-10-02T11:54:16.005721877Z 2026-10-02T11:54:16.005Z  INFO 1 --- [tasks-service] [           main] dev.folomkin.tasks.TasksApplication      : Started TasksApplication in 6.003 seconds (process running for 6.72)

> 2. kubectl exec deploy/tasks-service -c tasks-service -- wget -qO- http://config-server:8888/tasks-service/default 2>&1

> OCI runtime exec failed: exec failed: unable to start container process: exec: "wget": executable file not found in $PATH
command terminated with exit code 127

> 3. kubectl get pods -A | grep -E "config-server|tasks-service"

> default                config-server-9b8bfc594-9wfgj                2/2     Running     1 (32m ago)    68m
default                tasks-service-84f557999d-pc6rx               2/2     Running     0              8m17s

> 4. kubectl get pods -l app=tasks-service - Running
kubectl get pods -l app=config-server - Running

> 5. kubectl get peerauthentication -A
default     default-permissive   PERMISSIVE   9d


Свежие логи показывают **важную аномалию**: в них нет строки `Fetching config from server`. Если бы `spring-cloud-starter-config` работал, вы бы обязательно увидели эту строку при старте.

## 🎯 Главное наблюдение

```
Started TasksApplication in 6.003 seconds
```

И **ничего** про Config Server — ни `Fetching config`, ни `Located environment`, ни `Connection refused`. Приложение стартовало, как будто Config Server вообще не существует.

**Это значит:** `spring.config.import: configserver:...` **не срабатывает**. Spring Boot даже не пытается обратиться к Config Server.

## 🔍 Три возможные причины

### Причина A: Образ не пересобран

Самая частая. Вы правили `application.yaml` в исходниках, но образ в Minikube остался старым.

Проверьте содержимое `application.yaml` **внутри работающего пода**. Так как `wget` нет, используйте `cat`:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- cat /app/application.yaml
```

Путь может быть другим — сначала найдите:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'find / -name "application*.yaml" 2>/dev/null'
```

Если внутри **старый** `application.yaml` (без `spring.config.import`) — пересоберите образ:

```bash
docker build -t your-registry/tasks-service:latest .
minikube image load your-registry/tasks-service:latest
kubectl rollout restart deploy/tasks-service
```

### Причина B: Зависимость `spring-cloud-starter-config` не попала в образ

Проверьте зависимости внутри пода:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'ls /app/BOOT-INF/lib/ | grep -i config'
```

Должны увидеть файлы:
```
spring-cloud-config-client-*.jar
spring-cloud-starter-config-*.jar
```

Если их нет — `pom.xml` не содержит зависимость (или она scope `provided`/`test`), либо Maven не собрал её.

### Причина C: `spring.config.import` игнорируется

В Spring Boot 2.4+ механизм bootstrap по умолчанию отключён. Если у вас **старая версия** Spring Cloud (2020.x и ниже) — `spring.config.import` не поддерживается, нужен `bootstrap.yaml`.

Проверьте версию Spring Boot в логах старта:

```bash
kubectl logs deploy/tasks-service -c tasks-service | head -20
```

Ищите строку:
```
 :: Spring Boot ::                (v3.x.x)
```

| Версия Spring Boot | Как настраивать Config Client |
|---|---|
| 2.4+ | `spring.config.import` в `application.yaml` |
| до 2.4 | `bootstrap.yaml` + `spring-cloud-starter-bootstrap` |

## 🛠️ Проверка связи через sidecar

`wget` в вашем контейнере нет, но **в `istio-proxy` есть `curl`**. Используйте sidecar:

```bash
kubectl exec deploy/tasks-service -c istio-proxy -- curl -s http://config-server:8888/tasks-service/default
```

Это проверит связь **изнутри того же пода**, но другим контейнером. Если вернёт JSON — сеть работает, проблема в приложении. Если `Connection refused` — проблема в сети.

## 📋 Полный план диагностики

Выполните по порядку:

```bash
# 1. Что за версия Spring Boot
kubectl logs deploy/tasks-service -c tasks-service | head -20 | grep -i "spring boot"

# 2. Есть ли config-client в образе
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'ls /app/BOOT-INF/lib/ 2>/dev/null | grep -i config || ls /deployments/lib/ 2>/dev/null | grep -i config || find / -name "*config-client*.jar" 2>/dev/null'

# 3. Что в application.yaml внутри пода
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'find / -name "application*.yaml" -exec cat {} \; 2>/dev/null'

# 4. Проверка связи через sidecar
kubectl exec deploy/tasks-service -c istio-proxy -- curl -s http://config-server:8888/tasks-service/default

# 5. Полные логи старта
kubectl logs deploy/tasks-service -c tasks-service | head -100
```

Пришлите вывод этих команд — и станет ясно, в чём именно проблема.

## 💡 Наиболее вероятный диагноз

Судя по тому, что в логах **вообще нет** упоминания Config Server, ставлю на **Причину A или B** — образ не пересобран, или зависимость `spring-cloud-starter-config` не попала в classpath.

**Проверьте первым делом:**

```bash
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'ls /app/BOOT-INF/lib/ | grep config'
```

Если там нет `spring-cloud-config-client*.jar` — пересоберите образ с правильным `pom.xml`.

## 📌 Дополнительно: `Connection refused` из предыдущих логов

Возможно, вы видели `Connection refused` в **предыдущем** поде, который уже удалён. Сейчас в текущем поде его нет, потому что **Spring Boot вообще не пытается** подключиться к Config Server — просто игнорирует настройку.

Как только вы почините dependency или пересоберёте образ, `Fetching config from server` появится в логах, и тогда — либо всё заработает, либо вы увидите настоящую ошибку, с которой можно работать.

> погоди, в tasks-service я нашел ": Fetching config from server at : http://config-server:8888", но потом идет сразу что-то странное:
> 2026-10-02T12:13:46.234Z  INFO 1 --- [tasks-service] [           main] o.s.c.c.c.ConfigServerConfigDataLoader   : Fetching config from server at : http://config-server:8888
> -10-02T12:13:48.541Z  INFO 1 --- [tasks-service] [localhost:27017] org.mongodb.driver.cluster               : Exception in monitor thread while connecting to server localhost:27017
> com.mongodb.MongoSocketOpenException: Exception opening socket
	at com.mongodb.internal.connection.SocketStream.lambda$open$0(SocketStream.java:85) ~[mongodb-driver-core-5.8.1.jar!/:na]
	at java.base/java.util.Optional.orElseThrow(Unknown Source) ~[na:na]
	at com.mongodb.internal.connection.SocketStream.open(SocketStream.java:85) ~[mongodb-driver-core-5.8.1.jar!/:na]
	at com.mongodb.internal.connection.InternalStreamConnection.open(InternalStreamConnection.java:233) ~[mongodb-driver-core-5.8.1.jar!/:na]
	at com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitor.setupNewConnectionAndGetInitialDescription(DefaultServerMonitor.java:282) ~[mongodb-driver-core-5.8.1.jar!/:na]
	at com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitor.lookupServerDescription(DefaultServerMonitor.java:253) ~[mongodb-driver-core-5.8.1.jar!/:na]
	at com.mongodb.internal.connection.DefaultServerMonitor$ServerMonitor.run(DefaultServerMonitor.java:203) ~[mongodb-driver-core-5.8.1.jar!/:na]
Caused by: java.net.ConnectException: Connection refused (connect failed)
> а вот в самом конце уже:
> 2026-10-02T12:13:50.246Z  INFO 1 --- [tasks-service] [           main] dev.folomkin.tasks.TasksApplication      : Started TasksApplication in 6.208 seconds (process running for 6.997)


Ага, вот теперь всё стало на свои места! Это **совсем другая ошибка**, чем казалось. Разберём.

## 🎯 Что на самом деле происходит

`Connection refused` — **не к Config Server**, а **к MongoDB**!

Смотрите внимательно на лог:
```
Exception in monitor thread while connecting to server localhost:27017
                                                       ^^^^^^^^^^^^^^^
```

Приложение пытается подключиться к MongoDB по адресу **`localhost:27017`** — и, естественно, не находит её, потому что MongoDB в кластере работает на другом адресе.

**А Config Server работает!** Строка `Fetching config from server at : http://config-server:8888` — это доказательство.

## 🔍 Почему MongoDB подключается к `localhost`

Смотрим на содержимое `config-repo/tasks-service.yaml`:

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}
```

Это **placeholder** с fallback-значением:
- `${MONGO_URI:...}` — «возьми переменную окружения `MONGO_URI`, а если её нет — используй значение после двоеточия».
- Значение после двоеточия — `mongodb://localhost:27017/tasksdb`.

В вашем поде `tasks-service` **нет переменной окружения `MONGO_URI`**. Значит, Spring использует fallback — `localhost:27017`, где никакой MongoDB нет.

**Config Server тут ни при чём.** Он честно отдал конфиг, `tasks-service` его получил, применил — но в самом конфиге написано «иди на localhost».

## 🛠️ Как исправить

У вас есть **три варианта**. Выберите один.

### Вариант A: Использовать имя Kubernetes Service для MongoDB

Замените в `config-repo/tasks-service.yaml`:

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://mongodb.default.svc.cluster.local:27017/tasksdb
```

Где `mongodb` — имя вашего Kubernetes Service для MongoDB. Проверьте его:

```bash
kubectl get svc | grep -i mongo
```

Например, если Service называется `tasks-mongodb-service`, то:
```yaml
uri: mongodb://tasks-mongodb-service.default.svc.cluster.local:27017/tasksdb
```

Запушьте изменения в Git, перезапустите `tasks-service`:
```bash
git add config-repo/tasks-service.yaml
git commit -m "Fix MongoDB URI"
git push

kubectl rollout restart deploy/tasks-service
```

### Вариант B: Оставить placeholder, но задать `MONGO_URI` в Deployment

Оставьте конфиг в Git как есть:

```yaml
uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}
```

А в `tasks-service-deployment.yaml` добавьте переменную окружения:

```yaml
spec:
  template:
    spec:
      containers:
        - name: tasks-service
          env:
            - name: MONGO_URI
              value: mongodb://mongodb.default.svc.cluster.local:27017/tasksdb
```

Плюс: адрес MongoDB **не хранится в Git**, можно менять через Deployment. Минус: конфигурация «размазана» — часть в Git, часть в Deployment.

### Вариант C: Использовать Kubernetes Secret (правильный, но сложнее)

Для паролей и URI с credentials — использовать Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-credentials
type: Opaque
stringData:
  MONGO_URI: mongodb://user:password@mongodb:27017/tasksdb
```

В Deployment:
```yaml
env:
  - name: MONGO_URI
    valueFrom:
      secretKeyRef:
        name: mongodb-credentials
        key: MONGO_URI
```

## 📊 Как проверить, что проблема именно в этом

1. **Проверьте переменные окружения в поде:**
   ```bash
   kubectl exec deploy/tasks-service -c tasks-service -- env | grep MONGO
   ```
   Если пусто — `MONGO_URI` не задан, и Spring использует `localhost` из fallback.

2. **Проверьте, где находится MongoDB:**
   ```bash
   kubectl get svc | grep -i mongo
   kubectl get pods | grep -i mongo
   ```
   Имя Service — это адрес, который нужно использовать в URI.

3. **Проверьте, что Config Server отдал правильное значение:**
   ```bash
   curl http://localhost:8888/tasks-service/default | jq '.propertySources[0].source'
   ```
   Там будет строка `mongodb://localhost:27017/tasksdb` — прямое доказательство.

## 💎 Итог

| Что было | Что на самом деле |
|---|---|
| Думали: `Connection refused` к Config Server | Реально: `Connection refused` к MongoDB |
| Config Server не работал | Config Server **работает** — `Fetching config from server` в логах |
| `spring.config.import` не срабатывал | Срабатывает — конфиг получен |
| Проблема в Config Server | Проблема в **неправильном URI MongoDB** |

**Config Server у вас полностью настроен и работает.** Осталось только поправить адрес MongoDB в `config-repo/tasks-service.yaml` — заменить `localhost:27017` на реальное имя Kubernetes Service для MongoDB.

Проверьте:
```bash
kubectl get svc | grep -i mongo
```

И пришлите имя Service — я подскажу точную строку URI.

> curl http://localhost:8888/tasks-service/default | jq '.propertySources[0].source':
  "spring.data.mongodb.uri": "mongodb://tasks-mongodb-service.default.svc.cluster.local:27017/tasksdb"
> kubectl get svc | grep -i mongo:
tasks-mongodb-service   ClusterIP   10.98.226.155    <none>        27017/TCP           15d
> новый адрес применился, но в логах tasks-service все равно про localhost написано..
> я что-то упускаю или не так делаю

Стоп — это очень важный момент. Config Server отдаёт **правильный** адрес:
```
mongodb://tasks-mongodb-service.default.svc.cluster.local:27017/tasksdb
```
А `tasks-service` всё ещё пытается идти на `localhost`. Значит, **полученный из Config Server конфиг не применяется** или **переопределяется чем-то другим**.

## 🔍 Три возможные причины

### Причина A: Под не перезапущен после обновления Git

`tasks-service` запрашивает конфиг **только при старте**. Если вы обновили `config-repo`, запушили в Git, но **не перезапустили под** — он работает со старым конфигом, который получил при своём старте.

**Проверьте:**
```bash
kubectl get pods -l app=tasks-service
```
Колонка `AGE` — если под старше, чем ваш последний `git push` — перезапустите:
```bash
kubectl rollout restart deploy/tasks-service
kubectl rollout status deploy/tasks-service
```

Затем:
```bash
kubectl logs deploy/tasks-service -c tasks-service --tail=30 | grep -i "mongodb\|Fetching"
```

### Причина B: Локальный `application.yaml` переопределяет Config Server

В `tasks-service/src/main/resources/application.yaml` **мог остаться** старый `spring.data.mongodb.uri` с `localhost`. Тогда возможен конфликт: локальный property source может иметь **более высокий приоритет**, чем Config Server, если не настроен `bootstrap`.

**Проверьте, что внутри пода:**
```bash
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'find / -name "application*.yaml" -exec cat {} \;' 2>/dev/null | grep -A2 -B2 mongo
```

Если там есть `mongodb://localhost:27017` — **удалите** эту строку из локального `application.yaml`. В нём должно остаться только:
```yaml
spring:
  application:
    name: tasks-service
  config:
    import: configserver:http://config-server:8888
```

Пересоберите образ:
```bash
docker build -t your-registry/tasks-service:latest .
minikube image load your-registry/tasks-service:latest
kubectl rollout restart deploy/tasks-service
```

### Причина C: Переменная окружения `MONGO_URI` в поде

Если в Deployment задана `env: MONGO_URI=...localhost...` — она **имеет приоритет над всем**. Spring подставит её в placeholder `${MONGO_URI:...}`.

**Проверьте:**
```bash
kubectl exec deploy/tasks-service -c tasks-service -- env | grep -i mongo
```

Если увидите `MONGO_URI=mongodb://localhost:27017/...` — вот причина. Уберите её из Deployment или переопределите на правильный адрес.

## 🎯 Как точно узнать, откуда берётся `localhost`

Самый надёжный способ — посмотреть **все property sources** через actuator. Добавьте actuator, если его нет:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Затем:
```bash
kubectl port-forward svc/tasks-service 8080:80
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.properties["spring.data.mongodb.uri"]) | {name, value: .properties["spring.data.mongodb.uri"].value}'
```

Вы увидите **все места**, где задан `mongodb.uri`, в порядке приоритета. **Первый** в списке — тот, что реально применяется:

| Что увидите в `name` | Что означает |
|---|---|
| `systemEnvironment` с `MONGO_URI` | Переменная окружения в Deployment |
| `applicationConfig: [file:.../application.yaml]` | Локальный файл в образе |
| `configService:...` или `bootstrapProperties-...` | Config Server |
| `bootstrapProperties-...` | Config Server через bootstrap |

Если в списке **первым** идёт локальный `application.yaml` — он побеждает Config Server. Это и есть ваша проблема.

## 📋 Пошаговый план

1. **Перезапустите под:**
   ```bash
   kubectl rollout restart deploy/tasks-service
   kubectl rollout status deploy/tasks-service
   ```

2. **Проверьте свежие логи:**
   ```bash
   kubectl logs deploy/tasks-service -c tasks-service --tail=50 | grep -i "mongodb\|Fetching\|Located"
   ```
   Если `localhost` исчез — проблема была в неперезапущенном поде.

3. **Если `localhost` остался** — проверьте `env`:
   ```bash
   kubectl exec deploy/tasks-service -c tasks-service -- env | grep -i mongo
   ```

4. **Проверьте локальный `application.yaml` внутри пода:**
   ```bash
   kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'find / -name "application*.yaml" -exec sh -c "echo === {} ===; cat {}" \;' 2>/dev/null
   ```

5. **Проверьте `actuator/env`** — где именно сидит `localhost`.

## 💡 Наиболее вероятный сценарий

Учитывая, что Config Server отдаёт правильный URI, а `tasks-service` всё ещё использует `localhost`, я ставлю на **комбинацию A и B**:

1. В **локальном** `application.yaml` `tasks-service` остался `spring.data.mongodb.uri: mongodb://localhost:27017/tasksdb`.
2. Локальный property source имеет **более высокий приоритет**, чем Config Server, потому что `spring.config.import` в Spring Boot **не заменяет** локальные настройки, а **добавляет** их. По умолчанию локальные `application.yaml` **выигрывают** у Config Server.

**Решение:** удалить `mongodb.uri` из локального `application.yaml`. Тогда единственным источником останется Config Server.

## ⚠️ Важное правило Spring Cloud Config

**Приоритет property sources** (от высшего к низшему):
1. Аргументы командной строки
2. **Переменные окружения** (`MONGO_URI`)
3. Config Server (удалённые настройки)
4. Локальный `application.yaml`

Казалось бы, Config Server выше локального. Но есть нюанс: если в локальном `application.yaml` **явно указано** `spring.data.mongodb.uri`, то оно может **перебить** значение из Config Server, если `spring.config.import` обрабатывается **до** загрузки локального файла, и локальный файл потом «перезатирает».

**Правильный подход:** локальный `application.yaml` содержит **только**:
- `spring.application.name`
- `spring.config.import`
- настройки, которые **не должны** переопределяться (например, `server.port`, `management.*`)

Всё остальное — в Git.

Пришлите вывод `kubectl exec ... env | grep -i mongo` и `actuator/env` — и я точно скажу, какой source побеждает.


> kubectl exec deploy/tasks-service -c tasks-service -- env | grep -i mongo
> MONGO_URI=mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin
TASKS_MONGODB_SERVICE_SERVICE_PORT=27017
TASKS_MONGODB_SERVICE_PORT_27017_TCP_PROTO=tcp
TASKS_MONGODB_SERVICE_PORT_27017_TCP_PORT=27017
TASKS_MONGODB_SERVICE_PORT_27017_TCP=tcp://10.98.226.155:27017
TASKS_MONGODB_SERVICE_PORT_27017_TCP_ADDR=10.98.226.155
TASKS_MONGODB_SERVICE_PORT=tcp://10.98.226.155:27017
TASKS_MONGODB_SERVICE_SERVICE_HOST=10.98.226.155
> MONGO_URI у меня прописан в tasks-service-deployment,  а в config-repo/tasks-service.yaml вот так:
    mongodb:
      uri: mongodb://tasks-mongodb-service.default.svc.cluster.local:27017/tasksdb
> что оставить, что удалить?
> actuator не работает:
> {
  "timestamp": "2026-10-02T13:28:23.966Z",
  "status": 404,
  "error": "Not Found",
  "path": "/actuator/env"
}


Теперь всё стало ясно. Причин сразу три, и они накладываются.

## 🔍 Что происходит

**1. `MONGO_URI` в Deployment содержит credentials** (rootadmin:supersecretpassword). Это нормально — но не должно быть в Git.

**2. В `config-repo` теперь прописан «чистый» URI без credentials:**
```yaml
uri: mongodb://tasks-mongodb-service.default.svc.cluster.local:27017/tasksdb
```
Без логина/пароля MongoDB откажет в доступе — не `Connection refused`, а `Authentication failed`. Но у вас в логах `Connection refused` на `localhost` — значит, применяется **не этот** URI.

**3. Локальный `application.yaml` в образе `tasks-service`** — скорее всего, там остался:
```yaml
uri: ${MONGO_URI:mongodb://localhost:27017/tasksdb}
```
Раз в env есть `MONGO_URI=mongodb://rootadmin:...`, этот placeholder должен был подставить его... но подставляется `localhost`. Значит, **либо под не перезапущен**, либо в локальном файле **жёстко прописан** `localhost` без placeholder.

## ✅ Что оставить, что удалить

**Правильная схема — три уровня:**

| Где | Что хранить | Почему |
|---|---|---|
| **Kubernetes Secret** | `MONGO_URI` с credentials | Пароли не должны быть в Git |
| **Deployment** | Ссылка на Secret | Проброс credentials в под |
| **config-repo** | `${MONGO_URI:fallback}` | Placeholder без секрета |
| **Локальный `application.yaml`** | Только `name` + `config.import` | Никаких `mongodb.uri` |

### Шаг 1: Локальный `application.yaml` в `tasks-service`

Приведите к минимуму:

```yaml
spring:
  application:
    name: tasks-service
  config:
    import: configserver:http://config-server:8888
```

**Удалите** оттуда `spring.data.mongodb.uri` полностью.

### Шаг 2: `config-repo/tasks-service.yaml`

Оставьте placeholder, чтобы можно было переопределить через env:

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGO_URI:mongodb://tasks-mongodb-service.default.svc.cluster.local:27017/tasksdb}
```

Тут **два уровня**: если `MONGO_URI` задан — используется он; если нет — fallback без credentials (для локального запуска).

### Шаг 3: `tasks-service-deployment.yaml`

Уберите хардкод и используйте Secret:

```yaml
env:
  - name: MONGO_URI
    valueFrom:
      secretKeyRef:
        name: mongodb-credentials
        key: MONGO_URI
```

Secret:

```bash
kubectl create secret generic mongodb-credentials \
  --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin'
```

### Шаг 4: Пересоберите и перезапустите

```bash
docker build -t your-registry/tasks-service:latest .
minikube image load your-registry/tasks-service:latest
kubectl rollout restart deploy/tasks-service
kubectl rollout status deploy/tasks-service
kubectl logs deploy/tasks-service -c tasks-service --tail=50 | grep -i "mongodb\|Fetching\|Located"
```

## 🎯 Почему сейчас `localhost` побеждает

**Наиболее вероятно:** под не перезапущен **после** последнего `git push` в `config-repo`. `tasks-service` запрашивает конфиг один раз при старте. Если в момент старта в Git был ещё старый URI с `localhost` — он его и получил.

Проверьте:
```bash
kubectl get pod -l app=tasks-service
```
Колонка `AGE`. Если под старше, чем ваш последний `git push` — перезапустите.

**Второй вариант:** в локальном `application.yaml` внутри образа всё ещё прописан `mongodb.uri: mongodb://localhost:27017/tasksdb` **без placeholder** — тогда он может перебить Config Server из-за порядка property sources.

Проверьте, что внутри пода:
```bash
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'find / -name "application*.yaml" 2>/dev/null -exec sh -c "echo === {} ===; cat {}" \;'
```

## 🔍 Про actuator

`actuator/env` возвращает 404 — значит, **actuator не подключён**. Добавьте в `pom.xml` `tasks-service`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

И в `application.yaml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,env,configprops
```

Тогда `actuator/env` покажет **все источники** `mongodb.uri` в порядке приоритета — сразу увидите, кто побеждает.

## 📋 Чек-лист

- [ ] Локальный `application.yaml` — только `name` + `config.import`
- [ ] `config-repo/tasks-service.yaml` — `${MONGO_URI:fallback}`
- [ ] Deployment — `MONGO_URI` из Secret
- [ ] Образ пересобран (`minikube image load`)
- [ ] Под перезапущен (`rollout restart`)
- [ ] Actuator добавлен для диагностики

## 💎 Короткий ответ

**Оставить:**
- `MONGO_URI` в Deployment — но через **Secret**, а не хардкодом.
- В `config-repo` — placeholder `${MONGO_URI:...}`.

**Удалить:**
- `spring.data.mongodb.uri` из локального `application.yaml` в `tasks-service`.

После этого `MONGO_URI` из Secret будет единственным источником credentials, Config Server — источником структуры, а локальный файл перестанет вмешиваться.

Перезапустите под после правок и проверьте логи — `localhost` должен исчезнуть. Если останется — пришлите содержимое локального `application.yaml` из пода.

> я уже сто раз и перезапускал и пересобирал и все равно localhost вылазит. но при запуске config-server в логах вижу эту строчку:
Adding property source: Config resource 'file [/tmp/config-repo-13854923085928424745/config-repo/tasks-service.yaml]' via location 'file:/tmp/config-repo-13854923085928424745/config-repo/'
> может это какой-то кеш? и его почистить надо?

Спасибо за подробности — это ключевая зацепка. Дело не в кеше `/tmp/config-repo-*`, а в **порядке property sources** в Spring Boot 2.4+. Строка из логов Config Server, которую вы видите, как раз это подтверждает: сервер **добавляет** property source, но клиент может его **переопределить** локальным `application.yaml`.

## 🔍 Почему `localhost` побеждает

В Spring Boot 2.4+ механизм загрузки конфигурации изменился. Раньше (через `bootstrap.yml`) свойства из Config Server имели **высший приоритет** и не могли быть переопределены локально. Теперь, при использовании `spring.config.import` в `application.yaml`, порядок **обратный**:

1. **Локальный `application.yaml`** (внутри jar) — имеет **высший** приоритет.
2. **Config Server** (импортированный через `spring.config.import`) — **переопределяется** локальным файлом.

Это официально задокументированное поведение, и оно объясняет всё: даже если Config Server отдаёт правильный URI, локальный `application.yaml` с `mongodb://localhost:27017` **перебивает** его.

## 🎯 Проверьте главное

Убедитесь, что в **локальном** `application.yaml` внутри работающего пода **нет** строки с `mongodb.uri`:

```bash
kubectl exec deploy/tasks-service -c tasks-service -- sh -c 'find / -name "application*.yaml" 2>/dev/null -exec sh -c "echo === {} ===; cat {}" \;' | grep -B2 -A2 mongo
```

Если увидите `uri: mongodb://localhost:27017/tasksdb` — **вот он, источник проблемы**. Он может быть даже с плейсхолдером `${MONGO_URI:...}`, но если `MONGO_URI` из env по какой-то причине не подставляется, сработает fallback `localhost`.

## 🛠️ Решение: три варианта

### Вариант A (самый простой): Убрать `mongodb.uri` из локального файла

В `tasks-service/src/main/resources/application.yaml` **удалите** секцию `spring.data.mongodb` полностью. Оставьте только:

```yaml
spring:
  application:
    name: tasks-service
  config:
    import: configserver:http://config-server:8888
```

После этого единственным источником `mongodb.uri` останется Config Server (или переменная `MONGO_URI` из Deployment, если она есть).

Затем **обязательно**:

```bash
# 1. Пересобрать образ
docker build -t your-registry/tasks-service:latest .

# 2. Загрузить в minikube
minikube image load your-registry/tasks-service:latest

# 3. Перезапустить под
kubectl rollout restart deploy/tasks-service
kubectl rollout status deploy/tasks-service
```

**Проверьте, что образ действительно обновился** — иногда `minikube image load` не перезаписывает образ с тем же тегом, если под уже его использует. Убедитесь, что `kubectl get pod` показывает новый под.

### Вариант B: Использовать `bootstrap.yaml` (старый подход)

Если хотите, чтобы Config Server имел **высший** приоритет (как раньше), вернитесь к bootstrap:

1. Добавьте зависимость:
   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-bootstrap</artifactId>
   </dependency>
   ```

2. Удалите `spring.config.import` из `application.yaml`.

3. Создайте `src/main/resources/bootstrap.yaml`:
   ```yaml
   spring:
     application:
       name: tasks-service
     cloud:
       config:
         uri: http://config-server:8888
   ```

Теперь Config Server будет иметь высший приоритет, и локальный `application.yaml` не сможет его переопределить.

### Вариант C: Разрешить переопределение через настройки

Добавьте в `bootstrap.yaml` (только при использовании bootstrap):
```yaml
spring:
  cloud:
    config:
      allowOverride: true
      overrideNone: true
```

Но это работает **только** с bootstrap-подходом. При `spring.config.import` эти флаги игнорируются.

## 📌 Про кеш Config Server

Вы правы, что Config Server кеширует репозиторий в `/tmp/config-repo-<randomid>`. Но **это не ваша проблема**:

- `curl http://localhost:8888/tasks-service/default` возвращает **правильный** URI — значит, Config Server отдаёт актуальные данные.
- Кеш можно очистить, указав `spring.cloud.config.server.git.basedir` в не-temp директорию, или использовать `force-pull` для принудительного обновления. Но в вашем случае это не нужно — сервер уже отдаёт то, что надо.

## 💎 Итог

| Что проверить | Команда | Ожидаемый результат |
|---|---|---|
| Локальный `application.yaml` в поде | `kubectl exec ... cat application.yaml` | **Нет** строки `mongodb.uri` |
| Образ пересобран | `kubectl get pod -l app=tasks-service` | Новый под (AGE маленький) |
| `MONGO_URI` в env | `kubectl exec ... env \| grep MONGO` | Есть, с правильным URI |
| Config Server отдаёт URI | `curl http://localhost:8888/tasks-service/default \| jq` | Правильный URI |

**Наиболее вероятная причина:** локальный `application.yaml` внутри образа всё ещё содержит `mongodb://localhost:27017`. Удалите его, пересоберите образ, и `localhost` исчезнет.

Если после удаления локальной настройки `localhost` **всё равно** появляется — значит, `MONGO_URI` из env не подставляется в плейсхолдер Config Server. В этом случае проверьте, что в `config-repo/tasks-service.yaml` синтаксис плейсхолдера **правильный**: `${MONGO_URI:mongodb://fallback:27017/db}` — без лишних пробелов и скобок.

> 03-1_config_mongodb.yaml

> Автоматическое обновление конфигурации

Да, вы правы, `@RefreshScope` — это ключевая аннотация для этой задачи. В Spring Cloud есть два подхода к обновлению конфигурации: **ручной** (через `@RefreshScope` и специальный эндпоинт) и **автоматический** (с помощью Spring Cloud Bus).

### 🎯 Ручное обновление: `@RefreshScope` + `/actuator/refresh`

Этот способ самый простой и не требует дополнительной инфраструктуры.

1.  **Добавьте зависимость** `spring-boot-starter-actuator` в `pom.xml` вашего `tasks-service`.
2.  **Разрешите доступ к эндпоинту** `refresh` в `application.yaml`:
    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: refresh
    ```
3.  **Помечайте аннотацией** `@RefreshScope` те бины, которые должны динамически перечитывать конфигурацию. Например, ваш контроллер или сервис, который использует настройки из Config Server.
4.  **Чтобы применить изменения**, после правки файла в Git и пуша, отправьте `POST`-запрос на эндпоинт `/actuator/refresh` вашего сервиса. Spring перечитает конфигурацию и пересоздаст помеченные бины с новыми значениями.

Важно: этот метод работает только для бинов, аннотированных `@RefreshScope`. Без этой аннотации бин, инициализированный один раз, не будет обновлен.

### 🚀 Автоматическое обновление: Spring Cloud Bus

Этот способ избавляет от необходимости вручную вызывать `/actuator/refresh` на каждом сервисе. Он использует брокер сообщений (например, RabbitMQ или Kafka) для рассылки сигнала об обновлении всем заинтересованным сервисам.

1.  **Добавьте зависимость** `spring-cloud-starter-bus-amqp` (для RabbitMQ) или `spring-cloud-starter-bus-kafka` в **Config Server** и во **все клиентские сервисы** (в вашем случае — `tasks-service` и `spring-cloud-gateway`).
2.  **Настройте подключение к брокеру** в конфигурации этих сервисов.
3.  **Настройте Git Webhook** (если ваш репозиторий на GitHub/GitLab). Он будет отправлять `POST`-запрос на эндпоинт `/actuator/busrefresh` **Config Server** при каждом пуше в ветку конфигурации.
4.  **Как это работает**: Config Server получает сигнал от Git, публикует событие в брокер сообщений, а все клиенты, подписанные на него, получают команду на обновление. Это эквивалентно тому, как если бы вы вручную вызвали `/actuator/refresh` на каждом из них.

### 💡 Альтернатива: `@ConfigurationProperties`

Если вы используете `@ConfigurationProperties` для группировки настроек, их привязка к `Environment` также обновляется при вызове `/actuator/refresh`, и для них `@RefreshScope` не требуется. Это может быть удобнее для структурированных конфигураций.

### 📌 Важный нюанс для Spring Boot 4

Учитывая, что вы уже столкнулись с изменением префикса `spring.data.mongodb` → `spring.mongodb`, при настройке `@RefreshScope` стоит быть внимательным. Убедитесь, что свойства, которые вы хотите обновлять, действительно доступны через механизм `Environment` в новой версии.

### 💎 Итог

| Подход | Как работает | Когда использовать |
| :--- | :--- | :--- |
| **Ручной** (`@RefreshScope` + `/actuator/refresh`) | Вы вручную дергаете эндпоинт на каждом сервисе | Для быстрого прототипа или если сервисов немного |
| **Автоматический** (Spring Cloud Bus) | Сигнал от Git автоматически рассылается всем сервисам | В production среде, когда сервисов много |

> 03-02_vault.md

> 03-03_configmap_secret.md



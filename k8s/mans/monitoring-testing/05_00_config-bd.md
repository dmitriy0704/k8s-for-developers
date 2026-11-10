# Подключение Postgres, Prometheus, Grafana, Loki

> я закончил изучение микросервисов по учебнику. ты мне очень помог, спасибо большое! но есть еще пара тем, в которых я разбираюсь очень поверхностно, но они важные - мониторинг и тестирование. у меня подключена mongodb, я бы хотел для начала подключить postgres, потом подключить grafana/prometheus/loki, чтобы посмотреть как вообще это все выглядит по умолчанию. и потом хочу потестировать хотя бы какие то базовые инструменты для тестирования и посмотреть как это выглядит на графане.

Поздравляю с завершением учебника! Это отличный рубеж. Переход к мониторингу и тестированию — логичный и очень важный следующий шаг, который превращает набор работающих сервисов в систему, за которой можно наблюдать и которую можно проверять на прочность. Давайте разберем ваш план по шагам.

### 🐘 Часть 1: Подключение PostgreSQL

Добавление второй базы данных — хороший способ попрактиковаться. Процесс похож на то, что вы делали с MongoDB, но со своими нюансами.

1.  **Развертывание PostgreSQL в Kubernetes**: Вам понадобится `Deployment` (или `StatefulSet` для сохранения данных) и `Service`. Для учебных целей можно начать с простого `Deployment` и `emptyDir` для данных, но для реалистичности лучше использовать `PersistentVolumeClaim` (PVC).

    Пример манифеста `postgres-deployment.yaml`:
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: postgres
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: postgres
      template:
        metadata:
          labels:
            app: postgres
        spec:
          containers:
            - name: postgres
              image: postgres:16-alpine
              env:
                - name: POSTGRES_DB
                  value: tasksdb
                - name: POSTGRES_USER
                  value: admin
                - name: POSTGRES_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-credentials
                      key: password
              ports:
                - containerPort: 5432
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: postgres
    spec:
      selector:
        app: postgres
      ports:
        - port: 5432
          targetPort: 5432
    ```
    Не забудьте создать Secret `postgres-credentials` с паролем.

2.  **Настройка Spring Boot**: В `pom.xml` добавьте драйвер PostgreSQL и стартер Spring Data JPA (если используете JPA):
    ```xml
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    ```

3.  **Конфигурация подключения**: В `config-repo` (Spring Cloud Config) добавьте настройки для PostgreSQL:
    ```yaml
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/tasksdb
        username: admin
        password: ${POSTGRES_PASSWORD}
        driver-class-name: org.postgresql.Driver
    ```
    Пароль снова передавайте через переменную окружения из Secret, как вы делали с MongoDB.

### 📊 Часть 2: Стек мониторинга (Prometheus + Grafana + Loki)

Самый простой способ развернуть весь стек — использовать Helm-чарты. У вас уже есть опыт с Helm для Redis.

1.  **Prometheus + Grafana (kube-prometheus-stack)**: Этот чарт устанавливает Prometheus, Grafana, Alertmanager и набор готовых дашбордов для Kubernetes.
    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    kubectl create ns monitoring
    helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
    ```
    После установки Grafana будет доступна по `port-forward`:
    ```bash
    kubectl port-forward -n monitoring service/prometheus-grafana 3000:80
    ```
    Логин: `admin`, пароль: `prom-operator`.

2.  **Loki для сбора логов**: Loki — это «Prometheus для логов». Установите его вместе с Promtail (агент, который собирает логи подов).
    ```bash
    helm repo add grafana https://grafana.github.io/helm-charts
    helm repo update
    helm install loki grafana/loki-stack -n monitoring
    ```
    Чарт `loki-stack` также может автоматически добавить Loki как источник данных в Grafana, если вы установите его в тот же namespace.

3.  **Настройка Spring Boot для отправки метрик**: Чтобы Prometheus собирал метрики вашего приложения, добавьте зависимость `micrometer-registry-prometheus` в `pom.xml` и настройте `application.yaml`:
    ```yaml
    management:
      endpoints:
        web:
          exposure:
            include: health,info,prometheus
      metrics:
        tags:
          application: ${spring.application.name}
    ```
    После этого Prometheus сможет забирать метрики с эндпоинта `/actuator/prometheus`. В `kube-prometheus-stack` уже настроен автоматический поиск подов с аннотацией `prometheus.io/scrape: "true"`.

### 🚀 Часть 3: Нагрузочное тестирование и визуализация

Для нагрузочного тестирования отлично подходит **Gatling**. Он хорошо интегрируется с Kubernetes и может экспортировать метрики в Prometheus.

1.  **Создание простого теста**: Gatling использует DSL на Scala (или Java). Пример простого сценария для вашего `tasks-service`:
    ```scala
    import io.gatling.core.Predef._
    import io.gatling.http.Predef._
    import scala.concurrent.duration._

    class BasicSimulation extends Simulation {
      val httpProtocol = http
        .baseUrl("http://tasks-service") // Имя вашего сервиса в K8s
        .acceptHeader("application/json")

      val scn = scenario("Create Tasks")
        .exec(http("create_task")
          .post("/tasks/create")
          .header("Content-Type", "application/json")
          .body(StringBody("""{"title": "Load Test Task"}"""))
          .check(status.is(201)))

      setUp(
        scn.inject(
          rampUsers(50).during(10.seconds), // 50 пользователей за 10 секунд
          constantUsersPerSec(20).during(30.seconds) // 20 RPS в течение 30 секунд
        )
      ).protocols(httpProtocol)
    }
    ```
    Запустить тест можно локально через Maven/Gradle плагин или в Kubernetes как `Job`.

2.  **Интеграция с Grafana**: Чтобы видеть метрики Gatling в Grafana, нужно добавить в `pom.xml` зависимость `gatling-graphite` и настроить экспорт метрик в Graphite (или напрямую в Prometheus через плагин). Однако, даже без этого, вы сможете наблюдать за **метриками самого приложения** (CPU, память, latency HTTP-запросов, количество ошибок) в стандартных дашбордах Grafana, которые идут в комплекте с `kube-prometheus-stack`. Этого уже будет достаточно, чтобы увидеть, как сервис реагирует на нагрузку.

### 💎 Итог и следующие шаги

1.  **PostgreSQL**: Создайте `Deployment`/`Service` и настройте подключение в Spring Boot.
2.  **Мониторинг**: Установите `kube-prometheus-stack` и `loki-stack` через Helm. Настройте экспорт метрик в Spring Boot.
3.  **Тестирование**: Напишите простой сценарий в Gatling и запустите его. Наблюдайте за метриками в Grafana.

Этот план даст вам полноценную картину: вы увидите, как ваше приложение ведет себя под нагрузкой, какие метрики собираются, и как логи соотносятся с событиями в системе. Если на каком-то из этапов возникнут вопросы (например, с настройкой Gatling или дашбордов Grafana), спрашивайте.

> я хотел спросить по поводу настройки подключения postgres к моему tasks-service. я делаю по примеру подключения к mongodb. я еще раз проговорю, что я хочу подключить postgres не второй базой после mongo, а заменить mongo на postgres.

> в mongo-deployment.yaml есть 
конфиги переменной среды
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: rootadmin
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: supersecretpassword
            - name: MONGO_INITDB_DATABASE
              value: tasksdb
и в tasks-service-deployment.yaml - в deployment моего сервиса есть 
         env:
            - name: SERVER_PORT
              value: "8091"
            - name: MONGO_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-credentials
                  key: MONGO_URI

> и есть secret :
kubectl create secret generic mongodb-credentials \
                  --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin'
то есть логин/пароль к БД есть в deployment самой бд и в secret. это правильно или я что-то упустил?

Отвечу прямо: **паттерн рабочий, но в нём есть дублирование, которое для PostgreSQL лучше сразу исправить.** Вы не «упустили» ничего критичного — для учебного проекта так делают часто. Но есть три проблемы, которые стоит понимать.

## 🔍 Что не так с текущим MongoDB-подходом

У вас пароль `supersecretpassword` живёт **в трёх местах**:

| Где | Как | Проблема |
|---|---|---|
| `mongo-deployment.yaml` | Plaintext в `MONGO_INITDB_ROOT_PASSWORD` | Если файл в Git — пароль утёк |
| Secret `mongodb-credentials` | Base64 | Нормально, но... |
| Команда `kubectl create secret --from-literal=...` | Plaintext в shell history | Виден в `~/.bash_history`, CI-логах |

**Главная проблема — дублирование.** Если вы захотите сменить пароль, его нужно поменять **в двух местах** (в Deployment БД и в Secret приложения). Забыли одно — приложение не подключится. Классический «drift».

## ✅ Правильный паттерн для PostgreSQL: один Secret — один источник правды

Официальный образ `postgres` требует **обязательную** переменную `POSTGRES_PASSWORD` — без неё контейнер не стартует. И рекомендует передавать её через Secret.

**Идея:** создаём **один** Secret с тремя ключами. И PostgreSQL, и `tasks-service` ссылаются на **него же**. Пароль живёт **в одном месте**.

### Шаг 1. Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: default
type: Opaque
stringData:
  POSTGRES_USER: admin
  POSTGRES_PASSWORD: supersecretpassword
  POSTGRES_DB: tasksdb
```

`stringData` позволяет писать plaintext — Kubernetes сам закодирует в base64. **Не коммитьте этот файл в Git.** Применяйте так:

```bash
kubectl apply -f postgres-secret.yaml
```

Или через `kubectl create secret` (как вы делали), но тогда пароль попадёт в history. Для учебного проекта — приемлемо, для production — нет.

### Шаг 2. PostgreSQL Deployment — ссылается на Secret

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_DB
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc   # создайте PVC отдельно
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**Никакого plaintext-пароля в манифесте.** Всё через `secretKeyRef`.

### Шаг 3. `tasks-service` — тот же Secret

Для Spring Boot + JDBC нужны **три отдельные** переменные (не один URI, как у MongoDB):

```yaml
# в tasks-service-deployment.yaml
env:
  - name: SERVER_PORT
    value: "8091"
  - name: SPRING_DATASOURCE_USERNAME
    valueFrom:
      secretKeyRef:
        name: postgres-credentials
        key: POSTGRES_USER
  - name: SPRING_DATASOURCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-credentials
        key: POSTGRES_PASSWORD
```

А **URL** (не секрет — хост, порт, имя БД) — в `config-repo/tasks-service.yaml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/tasksdb
```

Или тоже через env:

```yaml
- name: SPRING_DATASOURCE_URL
  value: "jdbc:postgresql://postgres:5432/tasksdb"
```

**Spring Boot автоматически свяжет** `SPRING_DATASOURCE_USERNAME` → `spring.datasource.username`, `SPRING_DATASOURCE_PASSWORD` → `spring.datasource.password`, `SPRING_DATASOURCE_URL` → `spring.datasource.url` (relaxed binding).

### Почему не один `POSTGRES_URI`, как у MongoDB?

MongoDB-драйвер принимает всё в одной строке (`mongodb://user:pass@host:port/db`). JDBC работает иначе: URL — отдельно, username/password — отдельно. Можно, конечно, собрать URL с credentials, но тогда пароль снова окажется в строке, которую сложнее ротировать. **Разделение чище.**

## 📊 Сравнение: было vs стало

| | MongoDB (было) | PostgreSQL (правильно) |
|---|---|---|
| Где пароль | Deployment БД + Secret приложения | **Только Secret** |
| Где логин | Deployment БД + Secret приложения | **Только Secret** |
| Кто читает | БД (plaintext) + приложение (Secret) | БД (Secret) + приложение (Secret) |
| Смена пароля | В двух местах | **В одном месте** |
| Plaintext в Git | Да (в Deployment) | **Нет** |

## ⚠️ Важные подводные камни PostgreSQL

### 1. `POSTGRES_PASSWORD` обязателен

Образ `postgres` **не стартует** без `POSTGRES_PASSWORD`. Нельзя «оставить пустым» — контейнер упадёт с ошибкой `Database is uninitialized and superuser password is not specified`.

### 2. Init-скрипты работают только на **пустом** volume

`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` применяются **только при первом старте**, когда `/var/lib/postgresql/data` пуст. Если вы:
1. Запустили PostgreSQL с паролем `A`.
2. Поменяли Secret на пароль `B`.
3. Перезапустили под.

— **пароль останется `A`**. Env-переменная изменится, но `initdb` уже не запустится. Чтобы применить новый пароль, нужно либо `ALTER USER admin WITH PASSWORD 'B';` через `psql`, либо удалить PVC и начать с чистого листа.

**Для учебного проекта:** если что-то пошло не так с паролем — проще удалить PVC:
```bash
kubectl delete pvc postgres-pvc
kubectl delete pod -l app=postgres
```

### 3. Spring Boot 4 — префикс `spring.datasource` стабилен

В отличие от MongoDB (где вы нашли изменение `spring.data.mongodb` → `spring.mongodb`), для JDBC префикс `spring.datasource.*` **не менялся**. Но проверьте: после пересборки посмотрите в логи, что HikariCP стартует без ошибок.

### 4. Код приложения придётся переписать

Это **не только конфигурация**. MongoDB → PostgreSQL — смена парадигмы:

| MongoDB | PostgreSQL |
|---|---|
| `@Document` | `@Entity` |
| `@Id` (String) | `@Id @GeneratedValue` (Long/UUID) |
| `MongoRepository<Task, String>` | `JpaRepository<Task, Long>` |
| `MongoTemplate` | `JdbcTemplate` / `EntityManager` |
| Нет схемы | `schema.sql` или Flyway/Liquibase |
| Нет транзакций в классическом смысле | `@Transactional` |

Плюс зависимости в `pom.xml`:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```
И **удалить** `spring-boot-starter-data-mongodb`.

## 💡 Что делать по шагам

1. **Создайте Secret** `postgres-credentials` с тремя ключами.
2. **Разверните PostgreSQL** — Deployment + Service + PVC, env через `secretKeyRef`.
3. **В `config-repo`** добавьте `spring.datasource.url`.
4. **В `tasks-service-deployment.yaml`** добавьте `SPRING_DATASOURCE_USERNAME` и `SPRING_DATASOURCE_PASSWORD` из Secret.
5. **Перепишите код**: `@Document` → `@Entity`, `MongoRepository` → `JpaRepository`.
6. **Удалите MongoDB** — Deployment, Service, Secret `mongodb-credentials`, `MONGO_URI` из env.
7. **Пересоберите** и проверьте логи: `HikariPool-1 - Starting...`, `HikariPool-1 - Start completed`.

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| Текущий паттерн (пароль в Deployment + Secret) — правильно? | ⚠️ Работает, но дублирование. Для PostgreSQL лучше исправить |
| Где должен жить пароль? | **Только в Secret.** Один источник правды для БД и приложения |
| Нужен ли plaintext в Deployment БД? | **Нет.** Через `secretKeyRef` |
| Один URI или три переменные? | Для JDBC — **три**: URL, username, password |
| Где URL? | `config-repo` (не секрет) или env |
| Что важно помнить? | `POSTGRES_PASSWORD` обязателен; init только на пустом volume; код надо переписать |

Вы ничего не «упустили» — просто паттерн для MongoDB был учебным упрощением. Для PostgreSQL сделайте «single source of truth» через один Secret, и это будет ближе к production-практике.

> получается логин и пароль базы данных подтягиваются в двух местах: в postgres-deployment и в tasks-service-deployment? а url БД только в config-repo?

Да, вы поняли **абсолютно точно**. Именно так и должно быть. Давайте зафиксируем картину.

## 🗺️ Где что живёт

```
┌─────────────────────────────────────────────────────────────────┐
│  Secret postgres-credentials                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ POSTGRES_USER:     admin                                  │  │
│  │ POSTGRES_PASSWORD: supersecretpassword                    │  │
│  │ POSTGRES_DB:       tasksdb                                │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────┬─────────────────────────────────┬────────────────┘
               │                                 │
               │ secretKeyRef                    │ secretKeyRef
               ▼                                 ▼
┌──────────────────────────┐      ┌──────────────────────────────────┐
│ postgres-deployment      │      │ tasks-service-deployment         │
│                          │      │                                  │
│ env:                     │      │ env:                             │
│  POSTGRES_USER     ◀─────┼──────┼──▶ SPRING_DATASOURCE_USERNAME    │
│  POSTGRES_PASSWORD ◀─────┼──────┼──▶ SPRING_DATASOURCE_PASSWORD    │
│  POSTGRES_DB       ◀─────┼──────┘                                  │
│                          │      │ (URL — НЕ здесь)                  │
└──────────────────────────┘      └──────────────────────────────────┘
                                                │
                                                │ spring.datasource.url
                                                ▼
                                  ┌──────────────────────────────────┐
                                  │ config-repo/tasks-service.yaml   │
                                  │                                  │
                                  │ spring:                          │
                                  │   datasource:                    │
                                  │     url: jdbc:postgresql://      │
                                  │       postgres:5432/tasksdb      │
                                  └──────────────────────────────────┘
```

**Кратко:**

| Данные | Где хранится | Кто читает |
|---|---|---|
| Логин | Secret | **PostgreSQL** (для initdb) + **tasks-service** (для подключения) |
| Пароль | Secret | **PostgreSQL** + **tasks-service** |
| Имя БД | Secret | **PostgreSQL** (создать при init) |
| **URL / хост / порт** | **config-repo** | **Только tasks-service** |

## 🎯 Почему именно так

### 1. Логин/пароль — в Secret, потому что это **секрет**

Один и тот же Secret читают **два разных пода**:
- **PostgreSQL** — чтобы при первом старте создать суперпользователя с этим паролем.
- **tasks-service** — чтобы аутентифицироваться при подключении.

Если пароль поменять в Secret — оба пода увидят новое значение (после перезапуска). **Один источник правды.**

### 2. URL — в config-repo, потому что это **не секрет**

`jdbc:postgresql://postgres:5432/tasksdb` — это:
- Имя сервиса (`postgres`) — видно в `kubectl get svc`.
- Порт (`5432`) — стандартный.
- Имя БД (`tasksdb`) — не секрет.

URL **не нужно скрывать**. И он **не нужен самому PostgreSQL** — база не знает, по какому адресу к ней будут подключаться. URL нужен **только клиенту** (`tasks-service`).

Поэтому URL логично держать в Config Server — там он версионируется, меняется без `kubectl`, и его можно переопределить для разных сред (dev/prod).

### 3. Почему URL не в Secret

Технически можно положить и URL в Secret. Но:

- Secret — для **чувствительных** данных. URL без credentials — не чувствительный.
- Secret нужно обновлять через `kubectl` — неудобно для обычной настройки.
- Config Server даёт **версионирование** и **аудит** (кто и когда менял).

**Правило:** в Secret — только то, что нельзя показывать. Всё остальное — в Config Server.

## ⚠️ Один нюанс: `POSTGRES_DB` — это не «имя БД для подключения»

Обратите внимание: `POSTGRES_DB` в Secret нужен **PostgreSQL**, чтобы **создать** базу `tasksdb` при первом старте. Это **не** то же самое, что URL для подключения.

`tasks-service` **не читает** `POSTGRES_DB` из Secret. Он берёт имя БД **из URL** в config-repo:

```yaml
url: jdbc:postgresql://postgres:5432/tasksdb
                                     ^^^^^^^
                                     имя БД здесь
```

**Важно:** если вы поменяете `POSTGRES_DB` в Secret на `newdb`, но забудете поменять URL в config-repo — `tasks-service` будет ломиться в `tasksdb`, которой уже нет. Это **два разных места**, и они должны совпадать.

**Альтернатива (чище):** не дублировать имя БД. Оставить `POSTGRES_DB` только в Secret для PostgreSQL, а в URL использовать **то же значение** через переменную:

```yaml
# config-repo/tasks-service.yaml
spring:
  datasource:
    url: jdbc:postgresql://postgres:5432/${POSTGRES_DB:tasksdb}
```

И пробросить `POSTGRES_DB` из Secret в `tasks-service`:

```yaml
# tasks-service-deployment.yaml
- name: POSTGRES_DB
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: POSTGRES_DB
```

Тогда имя БД живёт **в одном месте** — в Secret. Но это уже оптимизация; для учебного проекта достаточно просто **помнить**, что `POSTGRES_DB` и URL должны совпадать.

## 📊 Итоговая таблица

| Компонент | Что читает | Откуда |
|---|---|---|
| **PostgreSQL** | `POSTGRES_USER` | Secret |
| **PostgreSQL** | `POSTGRES_PASSWORD` | Secret |
| **PostgreSQL** | `POSTGRES_DB` (создать базу) | Secret |
| **tasks-service** | `SPRING_DATASOURCE_USERNAME` | Secret |
| **tasks-service** | `SPRING_DATASOURCE_PASSWORD` | Secret |
| **tasks-service** | `spring.datasource.url` | **config-repo** |

## 💎 Ответ на ваш вопрос

| Вопрос | Ответ |
|---|---|
| Логин/пароль подтягиваются в **двух** местах? | **Да** — в `postgres-deployment` (для initdb) и в `tasks-service-deployment` (для подключения). Оба — из **одного** Secret |
| URL только в config-repo? | **Да** — только `tasks-service` его читает, PostgreSQL он не нужен |
| Это правильно? | **Да**, это стандартный паттерн: секреты — в Secret, конфигурация — в Config Server |

Вы всё поняли верно. Единственное, что стоит запомнить: **имя БД (`tasksdb`) фигурирует и в Secret (`POSTGRES_DB`), и в URL (`config-repo`)** — эти два значения должны совпадать. Если хотите убрать дублирование — используйте `${POSTGRES_DB:...}` в URL, как показано выше.

> конечно же с первого раза не заработало.
под tasks-service завис на стадии init:
tasks-service-7d59fc7774-st9fv               1/2     Init:2/3           0                15m

Зависание на стадии `Init:2/3` означает, что два из трёх init-контейнеров уже успешно отработали, а третий выполняется и не может завершиться. В поде с Istio обычно есть как минимум `istio-init`, а третий контейнер — это, скорее всего, либо нативный сайдкар `istio-proxy`, либо **ваш собственный init-контейнер**, который вы могли добавить для ожидания PostgreSQL. Дальнейшие действия зависят от того, какой именно контейнер завис.

### 🎯 Шаг 1: Определите, какой контейнер завис

Выполните команду и посмотрите на блок `Init Containers` в самом низу вывода:

```bash
kubectl describe pod tasks-service-7d59fc7774-st9fv | grep -A 30 "Init Containers:"
```

Вас интересует состояние (State) каждого init-контейнера. Ищите тот, у которого `State: Running` и `Restart Count: 0` — это и есть «застрявший» контейнер.

### 🔍 Шаг 2: Диагностика в зависимости от типа контейнера

#### 🐘 Если завис ваш init-контейнер ожидания PostgreSQL (`wait-for-postgres`)

Скорее всего, вы добавили контейнер с циклом `until pg_isready ...` или `until nc -z postgres 5432 ...`. Причина зависания — **сервис PostgreSQL не существует или недоступен по указанному адресу**.

**Проверьте логи этого init-контейнера** (подставьте его имя из вывода `describe`):

```bash
kubectl logs tasks-service-7d59fc7774-st9fv -c wait-for-postgres
```

Вы увидите повторяющееся сообщение `Waiting for PostgreSQL...` или ошибку вроде `bad address 'postgres'`. Это подтвердит диагноз.

**Что нужно проверить:**
1. **Существует ли Service `postgres`?** `kubectl get svc postgres` — если его нет, init-контейнер будет ждать вечно.
2. **Есть ли у Service готовые эндпоинты?** `kubectl get endpoints postgres` — если `ENDPOINTS` пуст, значит, под PostgreSQL ещё не запустился или не прошёл проверку готовности.
3. **Совпадает ли имя хоста** в команде ожидания с реальным именем Service.

#### 🛡️ Если завис `istio-init`

Этот контейнер настраивает iptables-правила. Его зависание обычно связано с проблемами прав доступа или конфликтом уже существующих правил.

**Проверьте логи:**

```bash
kubectl logs tasks-service-7d59fc7774-st9fv -c istio-init
```

Ищите ошибки `iptables-restore`, `Permission denied` или `xtables other problem`. Если видите подобное, самое быстрое решение — **удалить под**, чтобы он пересоздался с чистого листа:

```bash
kubectl delete pod tasks-service-7d59fc7774-st9fv
```

Deployment создаст новый под, и `istio-init` применит правила без конфликтов.

#### 🚪 Если завис `istio-proxy` (нативный сайдкар)

В новых версиях Istio прокси может запускаться как init-контейнер (native sidecar) и оставаться в состоянии `Running` навсегда. Это **нормально**, если он отмечен как готовый. Однако если он завис и не переходит в состояние готовности, проверьте его логи:

```bash
kubectl logs tasks-service-7d59fc7774-st9fv -c istio-proxy
```

Ищите ошибки подключения к `istiod` или проблемы с сертификатами.

### 📌 Что делать дальше

Пожалуйста, пришлите вывод команды `kubectl describe pod ... | grep -A 30 "Init Containers:"`. Это сразу покажет, какой именно контейнер завис и на каком этапе. После этого можно будет дать точное решение.

Если окажется, что проблема в вашем init-контейнере ожидания БД, самым надёжным решением будет **временно убрать его из манифеста** и проверить, запустится ли `tasks-service`. Это поможет изолировать проблему: если без него под стартует и падает с ошибкой подключения к PostgreSQL, значит, нужно исправлять конфигурацию БД, а не ждать в init-контейнере.

> в итоге, сервис tasks-service при старте показывает статус CrashLoopBackOff, а в логах:
org.hibernate.exception.JDBCConnectionException: Unable to obtain isolated JDBC connection [The connection attempt failed.] [n/a]
Caused by: org.postgresql.util.PSQLException: The connection attempt failed.
Caused by: java.net.UnknownHostException: postgres
	at java.base/sun.nio.ch.NioSocketImpl.connect(Unknown Source) ~[na:na]
	at java.base/java.net.SocksSocketImpl.connect(Unknown Source) ~[na:na]
	at java.base/java.net.Socket.connect(Unknown Source) ~[na:na]
что то с подключением?

`UnknownHostException: postgres` — это **проблема DNS**, а не подключения. Kubernetes не может найти Service с именем `postgres` в namespace `default`. Это значит, что **самого Service либо нет, либо он называется иначе**.

Обратите внимание на разницу:

| Ошибка | Что означает |
|---|---|
| `UnknownHostException: postgres` | **DNS не знает такого имени** — Service отсутствует |
| `ConnectException: Connection refused` | Имя есть, но за ним нет подов |
| `PSQLException: password authentication failed` | Всё работает, но пароль неверный |

У вас **первый** случай — Service `postgres` не существует.

## 🔍 Шаг 1: Проверьте, что вообще есть в кластере

```bash
# Все сервисы в namespace
kubectl get svc

# Есть ли postgres?
kubectl get svc postgres

# Все поды, связанные с postgres
kubectl get pods -A | grep -i postgres

# Есть ли endpoints
kubectl get endpoints postgres
```

**Скорее всего, вы увидите одно из трёх:**

1. **`Error from server (NotFound): services "postgres" not found`** — Service не создан вообще.
2. **Service есть, но называется иначе** — например, `postgres-service`, `tasks-postgres-service` (как было с MongoDB: `tasks-mongodb-service`).
3. **Service есть, но в другом namespace** — тогда короткое имя не резолвится.

## 🎯 Шаг 2: Исправьте имя в config-repo

Если Service называется иначе (скажем, `tasks-postgres-service`), поправьте URL в `config-repo/tasks-service.yaml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://tasks-postgres-service:5432/tasksdb
```

Проверьте **точное** имя через `kubectl get svc`. У MongoDB у вас было `tasks-mongodb-service` — вероятно, по аналогии PostgreSQL тоже назван `tasks-postgres-service` или `postgres-service`.

## 🛠️ Шаг 3: Если Service вообще не создан

Примените манифест:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: default
spec:
  selector:
    app: postgres         # ← должно совпадать с меткой пода!
  ports:
    - port: 5432
      targetPort: 5432
```

И проверьте, что **метка пода** совпадает с селектором:

```bash
kubectl get pod -l app=postgres --show-labels
```

Если под PostgreSQL имеет метку `app: postgres-db`, а селектор ищет `app: postgres` — endpoints будут пустыми. Тогда либо поправьте селектор, либо метку пода.

## 🔄 Шаг 4: Перезапустите tasks-service

После исправления имени в `config-repo` (или создания Service) **обязательно**:

```bash
# 1. Проверьте, что Config Server отдаёт новый URL
kubectl port-forward svc/config-server 8888:8888 &
curl http://localhost:8888/tasks-service/default | jq '.propertySources[].source | select(."spring.datasource.url")'

# 2. Перезапустите tasks-service
kubectl rollout restart deploy/tasks-service
kubectl rollout status deploy/tasks-service

# 3. Смотрите логи
kubectl logs deploy/tasks-service -c tasks-service --tail=50 | grep -i "postgres\|hikari\|unknownhost"
```

**Важно:** `tasks-service` читает конфиг из Config Server **только при старте**. Если вы поменяли URL в Git, запушли — Config Server его отдаст, но `tasks-service` увидит новое значение **только после перезапуска пода**.

## 📋 Полная диагностика одной командой

```bash
# 1. Какие сервисы есть
echo "=== Services ==="; kubectl get svc

# 2. Endpoints postgres
echo "=== Endpoints ==="; kubectl get endpoints postgres

# 3. Поды postgres и их метки
echo "=== Pods ==="; kubectl get pods -l app=postgres --show-labels

# 4. Что Config Server отдаёт
echo "=== Config ==="; kubectl port-forward svc/config-server 8888:8888 & sleep 2; curl -s http://localhost:8888/tasks-service/default | jq '.propertySources[].source | select(."spring.datasource.url")'

# 5. Что видит tasks-service в env
echo "=== Env ==="; kubectl exec deploy/tasks-service -c tasks-service -- env | grep -i "datasource\|postgres" 2>/dev/null
```

## 💡 Наиболее вероятный сценарий

Учитывая, что с MongoDB у вас был Service `tasks-mongodb-service`, а не `mongodb`, я почти уверен, что:

- **PostgreSQL развёрнут**, но Service называется `tasks-postgres-service` (или как-то похоже).
- **В `config-repo` вы указали `postgres`**, а не реальное имя.

**Проверьте `kubectl get svc`** — увидите точное имя. Затем поправьте URL в `config-repo`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://<точное-имя-из-kubectl-get-svc>:5432/tasksdb
```

Запушьте в Git, перезапустите `tasks-service` — и ошибка `UnknownHostException` исчезнет.

## ⚠️ Ещё одна частая ловушка

Если PostgreSQL и `tasks-service` в **разных namespace** (например, PostgreSQL в `data`, а `tasks-service` в `default`), короткое имя `postgres` не резолвится. Нужно полное DNS-имя:

```yaml
url: jdbc:postgresql://postgres.data.svc.cluster.local:5432/tasksdb
```

Проверьте:
```bash
kubectl get pods -A | grep -i "postgres\|tasks-service"
```

Если namespace разные — используйте `<service>.<namespace>.svc.cluster.local`.

## 💎 Итог

| Симптом | Причина | Решение |
|---|---|---|
| `UnknownHostException: postgres` | Service `postgres` не существует | Проверить точное имя через `kubectl get svc` |
| `Connection refused` | Service есть, endpoints пустые | Исправить селектор или дождаться готовности PostgreSQL |
| `password authentication failed` | Всё работает, пароль неверный | Проверить Secret |

**Начните с `kubectl get svc`** — это сразу покажет, как называется ваш PostgreSQL Service. Пришлите вывод — и я скажу точную строку для `config-repo`.

> да, проблема была в названии сервиса в урле бд в config-repo. теперь новая проблема не проходит post-запрос. как раз тот, над которым навешан circuit-breaker, retry и Bulkhead.
Failed to bind properties under 'resilience4j.circuitbreaker.instances.mongodb.record-exceptions[0]' to java.lang.Class<java.lang.Throwable>:
    Property: resilience4j.circuitbreaker.instances.mongodb.record-exceptions[0]
    Value: "com.mongodb.MongoException"
    Origin: class path resource [application.yaml] from app.jar - 94:13
    Reason: failed to convert java.lang.String to java.lang.Class<java.lang.Throwable> (caused by java.lang.ClassNotFoundException: com.mongodb.MongoException)


 Отличная ошибка — она **сама себя объясняет**. Вы перешли с MongoDB на PostgreSQL, но в `application.yaml` остались **классы исключений MongoDB**. Spring Boot пытается их загрузить, не находит (потому что драйвер MongoDB вы, скорее всего, удалили из `pom.xml`) и падает при старте.

## 🔍 Что именно произошло

```
Failed to bind properties under 'resilience4j.circuitbreaker.instances.mongodb.record-exceptions[0]'
Value: "com.mongodb.MongoException"
Reason: ClassNotFoundException: com.mongodb.MongoException
```

Spring Boot при старте читает конфигурацию Resilience4j и **загружает классы** из `recordExceptions`, `ignoreExceptions`, `retryExceptions`. Если класса нет в classpath — приложение **не стартует вообще**. Это не «ошибка в рантайме», а **ошибка биндинга конфигурации**.

## 🎯 Почему MongoDB-классы больше не доступны

Вы удалили зависимость:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

Вместе с ней ушёл и `mongodb-driver-core`, где жили `com.mongodb.MongoException`, `MongoSocketOpenException` и т.д.

## 🛠️ Как исправить

### Шаг 1. Замените `mongodb` на `postgres` в конфиге

У вас в `application.yaml` секция:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongodb:                          # ← старое имя
        recordExceptions:
          - com.mongodb.MongoException  # ← класс MongoDB
          ...
```

**Переименуйте instance** (чтобы не путаться) и **замените классы** на PostgreSQL/JDBC:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      postgres:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        ignoreExceptions:
          - io.github.resilience4j.bulkhead.BulkheadFullException
        recordExceptions:
          - org.springframework.dao.DataAccessResourceFailureException
          - org.springframework.dao.QueryTimeoutException
          - org.springframework.dao.TransientDataAccessResourceException
          - org.springframework.dao.CannotAcquireLockException
          - org.springframework.jdbc.CannotGetJdbcConnectionException
          - java.sql.SQLException
          - java.net.ConnectException
          - java.net.SocketTimeoutException

  retry:
    instances:
      postgres:
        maxAttempts: 2
        waitDuration: 100ms
        retryExceptions:
          - org.springframework.dao.TransientDataAccessResourceException
          - java.net.ConnectException

  bulkhead:
    instances:
      postgres:
        maxConcurrentCalls: 10
        maxWaitDuration: 100ms
```

### Шаг 2. Обновите аннотации в коде

Если у вас было:

```java
@Bulkhead(name = "mongodb", fallbackMethod = "fallbackCreateTask")
@CircuitBreaker(name = "mongodb", fallbackMethod = "fallbackCreateTask")
@Retry(name = "mongodb", fallbackMethod = "fallbackCreateTask")
```

Стало:

```java
@Bulkhead(name = "postgres", fallbackMethod = "fallbackCreateTask")
@CircuitBreaker(name = "postgres", fallbackMethod = "fallbackCreateTask")
@Retry(name = "postgres", fallbackMethod = "fallbackCreateTask")
```

**Имя в аннотации должно совпадать с именем instance в `application.yaml`.** Если оставить `name = "mongodb"` — Resilience4j создаст instance с дефолтными настройками (без ваших `recordExceptions`), и CircuitBreaker будет вести себя непредсказуемо.

### Шаг 3. Проверьте `retryExceptions` и `ignoreExceptions` по всему файлу

Пройдитесь поиском по `application.yaml`:

```bash
grep -n "com.mongodb\|mongodb" src/main/resources/application.yaml
```

И в `config-repo/tasks-service.yaml` тоже:

```bash
grep -n "com.mongodb\|mongodb" config-repo/tasks-service.yaml
```

**Все** упоминания `com.mongodb.*` нужно заменить. Если какое-то осталось — приложение снова не стартует.

## 📊 Какие исключения записывать для PostgreSQL

JDBC/Spring Data JPA оборачивает ошибки драйвера в **свои** исключения. Не нужно ловить `org.postgresql.util.PSQLException` напрямую — Spring конвертирует его в иерархию `DataAccessException`.

| Исключение | Что означает |
|---|---|
| `DataAccessResourceFailureException` | Не удалось подключиться к БД |
| `CannotGetJdbcConnectionException` | Не удалось получить соединение из пула (HikariCP) |
| `QueryTimeoutException` | Запрос превысил таймаут |
| `TransientDataAccessResourceException` | Временная ошибка (retry имеет смысл) |
| `CannotAcquireLockException` | Не удалось взять блокировку |
| `SQLException` | Общая ошибка SQL (fallback) |
| `ConnectException` / `SocketTimeoutException` | Сетевые ошибки |

**Что НЕ записывать** (как и раньше):

- `MethodArgumentNotValidException` — ошибка валидации.
- `EntityNotFoundException` / `NoSuchElementException` — «не найдено».
- `DataIntegrityViolationException` — нарушение constraint (ошибка клиента, а не БД).
- `NullPointerException` — баг в коде.

## 🎯 Про имя instance

Можно оставить старое имя `mongodb` (просто убрать Mongo-классы) — код продолжит работать. Но лучше переименовать в `postgres`, чтобы:

1. **Метрики** в actuator/Grafana имели осмысленное имя (`resilience4j_circuitbreaker_state{name="postgres"}`).
2. **Логи** не путали (`CircuitBreaker 'mongodb' is OPEN` при работающем PostgreSQL выглядит странно).
3. **Будущий вы** не задавал вопросов «почему mongodb, если у нас postgres?».

## 📋 Пошаговый план

1. **Найдите все `com.mongodb`** в `application.yaml` и `config-repo`:
   ```bash
   grep -rn "com.mongodb" src/ config-repo/
   ```
2. **Замените** на PostgreSQL-исключения из таблицы выше.
3. **Переименуйте** instance `mongodb` → `postgres` (опционально, но рекомендую).
4. **Обновите** `name = "mongodb"` в аннотациях `@CircuitBreaker`, `@Retry`, `@Bulkhead` на `name = "postgres"`.
5. **Проверьте**, что в `pom.xml` **нет** `spring-boot-starter-data-mongodb` и **есть** `postgresql` + `spring-boot-starter-data-jpa`.
6. **Пересоберите** и перезапустите:
   ```bash
   docker build -t myorg/tasks-service:1.0 .
   minikube image load myorg/tasks-service:1.0
   kubectl rollout restart deploy/tasks-service
   ```
7. **Проверьте логи** — не должно быть `ClassNotFoundException`:
   ```bash
   kubectl logs deploy/tasks-service -c tasks-service --tail=50 | grep -i "error\|exception\|postgres"
   ```

## ⚠️ Один нюанс с `retryExceptions`

Для PostgreSQL **Retry опасен** — если `INSERT` упал после того, как транзакция частично применилась, повторный `INSERT` может создать **дубликат** (если нет unique constraint) или упасть с `DataIntegrityViolationException`. Для `createTask` (POST) я бы **не ретраил** запись, а только **чтение** (`getTask`, `listTasks`).

Если оставляете Retry на `createTask` — убедитесь, что:
- Есть **unique constraint** на `id` или бизнес-ключ.
- `retryExceptions` содержит только `TransientDataAccessResourceException` (временные ошибки), но **не** `DataIntegrityViolationException`.

Либо уберите `@Retry` с `createTask` вовсе — оставьте только `@CircuitBreaker` + `@Bulkhead`.

## 💎 Итог

| Проблема | Причина | Решение |
|---|---|---|
| `ClassNotFoundException: com.mongodb.MongoException` | В `application.yaml` остались Mongo-классы после удаления драйвера | Заменить на PostgreSQL/JDBC-исключения |
| CircuitBreaker ведёт себя странно | `name = "mongodb"` в аннотации, а instance переименован | Синхронизировать имена |
| Retry создаёт дубликаты | Retry на `createTask` (POST) | Убрать Retry с записи |

**Начните с `grep -rn "com.mongodb" src/ config-repo/`** — увидите все места, которые надо заменить. После правок приложение стартует, а CircuitBreaker будет корректно считать ошибки PostgreSQL.

> я все сделал, но теперь при самом первом запросе в логах выводится сообщение из fallback:
Не удалось создать задачу, попробуйте позже отсюда:
 throw new ServiceUnavailableException(
                "Не удалось создать задачу, попробуйте позже"
        );


 Раз fallback срабатывает **на первом же запросе** (а не после 5 ошибок, как было бы при `OPEN`), значит **сам метод бросил исключение**. CircuitBreaker тут ни при чём — он просто поймал ошибку и вызвал fallback. Нужно понять, **что именно** бросил метод.

## 🔍 Шаг 1: Найдите настоящее исключение в логах

Ваш fallback для «всего остального» пишет:

```java
log.error("MongoDB call failed after retries, task={}, error={}", task.getId(), t.getMessage(), t);
```

(или похожее — в зависимости от того, как вы переименовали)

**Найдите эту строку в логах** — там будет класс исключения и stack trace:

```bash
kubectl logs deploy/tasks-service -c tasks-service --tail=200 | grep -A 30 "Не удалось создать задачу\|MongoDB call failed\|fallback"
```

Или посмотрите **всё**, что было перед fallback:

```bash
kubectl logs deploy/tasks-service -c tasks-service --tail=100
```

Ищите строки с `Caused by:` или `Exception`.

## 🎯 Наиболее вероятные причины (по частоте)

### Причина 1: Таблица `tasks` не создана

Вы перешли с MongoDB (schemaless) на PostgreSQL (строгая схема). Hibernate/JPA **не создаёт таблицу автоматически**, если не настроен `ddl-auto`.

**Проверьте `config-repo/tasks-service.yaml`** — есть ли:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update   # ← должно быть update или create
    show-sql: true       # полезно для отладки
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
```

**Если `ddl-auto` нет** — Hibernate ничего не делает со схемой, и первый `INSERT` падает с:

```
org.postgresql.util.PSQLException: ERROR: relation "tasks" does not exist
```

**Решение:** добавьте `ddl-auto: update` (для учебного проекта) или `create-drop` (создаст при старте, удалит при остановке). В production — `validate` + Flyway/Liquibase.

**Проверьте руками**, есть ли таблица:

```bash
kubectl exec deploy/postgres -- psql -U admin -d tasksdb -c "\dt"
```

Если пусто — таблицы нет, вот и причина.

### Причина 2: Несовпадение имени таблицы

В JPA по умолчанию имя таблицы = имя сущности в snake_case. Если у вас:

```java
@Entity
public class Tasks {   // ← имя класса Tasks, значит таблица "tasks"
```

а Hibernate ищет `tasks`, но создал `tasks` — ок. Но если сущность называется `Task` (единственное число), а таблица `tasks` — может быть mismatch.

**Проверьте `@Table`:**

```java
@Entity
@Table(name = "tasks")   // ← явно укажите имя
public class Tasks { ... }
```

### Причина 3: Проблема с полем `id`

В MongoDB `@Id` был `String`. В JPA нужен **другой тип** и стратегия генерации:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)   // для PostgreSQL SERIAL
private Long id;
```

Если оставить `String id` без `@GeneratedValue` — Hibernate не сможет вставить запись, потому что `id` будет `null`, а колонка `NOT NULL`.

**Проверьте вашу сущность.** Тип `id` должен быть `Long` (или `UUID` с `@GeneratedValue`).

### Причина 4: Несовпадение полей entity и таблицы

Если таблица уже создана (например, из прошлых экспериментов) с одним набором колонок, а entity ожидает другой — будет ошибка:

```
ERROR: column "title" of relation "tasks" does not exist
```

**Решение:** либо `ddl-auto: update` (добавит колонки), либо удалить таблицу и пересоздать:

```bash
kubectl exec deploy/postgres -- psql -U admin -d tasksdb -c "DROP TABLE IF EXISTS tasks;"
```

Затем перезапустить `tasks-service` с `ddl-auto: update`.

### Причина 5: Kafka producer падает

Ваш метод `createTask` делает **две вещи**:

```java
Tasks saved = taskRepository.save(task);
producer.sendTaskCreatedEvent(...);   // ← Kafka!
return saved;
```

Если Kafka недоступна — `producer.sendTaskCreatedEvent()` бросит исключение **после** успешного `save()`. Fallback сработает, хотя задача **уже создана** в БД.

**Проверьте логи Kafka:**

```bash
kubectl logs deploy/tasks-service -c tasks-service | grep -i "kafka\|producer"
```

Ищите `TimeoutException`, `KafkaException`, `Connection to node -1 could not be established`.

**Решение:**
- Проверить, что Kafka запущена: `kubectl get pods | grep kafka`.
- Настроить `retries` и `delivery.timeout.ms` в producer.
- Или сделать отправку события **асинхронной** и не блокировать ответ:

```java
Tasks saved = taskRepository.save(task);
producer.sendTaskCreatedEvent(...);   // fire-and-forget
return saved;
```

Если используете `KafkaTemplate.send()` — он возвращает `CompletableFuture`. Не блокируйте на нём:

```java
kafkaTemplate.send("tasks-events", event);  // не .get()
```

## 🛠️ Пошаговый план диагностики

```bash
# 1. Полные логи с исключением
kubectl logs deploy/tasks-service -c tasks-service --tail=200 | grep -B2 -A20 "fallback\|Exception\|Caused by"

# 2. Есть ли таблица в БД
kubectl exec deploy/postgres -- psql -U admin -d tasksdb -c "\dt"
kubectl exec deploy/postgres -- psql -U admin -d tasksdb -c "\d tasks"

# 3. Настройки JPA
kubectl exec deploy/tasks-service -c tasks-service -- env | grep -i "jpa\|hibernate\|ddl"

# 4. Что в config-repo
curl http://localhost:8888/tasks-service/default | jq '.propertySources[].source | select(."spring.jpa" or ."spring.datasource")'

# 5. Kafka жива?
kubectl get pods | grep -i kafka
```

## 💡 Наиболее вероятный сценарий

Учитывая, что вы **только что перешли на PostgreSQL** и это **первый запрос**, я ставлю на **одно из двух**:

1. **Таблица `tasks` не создана** — нет `ddl-auto: update`.
2. **Тип `id` не изменён** — остался `String` из MongoDB-версии.

Оба варианта дают исключение на `taskRepository.save()`, fallback срабатывает, в Postman — 503.

## 📊 Что прислать

Пришлите:

1. **Строку с исключением** из логов (то, что идёт **перед** `Не удалось создать задачу`):
   ```bash
   kubectl logs deploy/tasks-service -c tasks-service --tail=100 | grep -B5 "Не удалось создать"
   ```

2. **Сущность `Tasks`** — покажите `@Id`, `@GeneratedValue`, `@Table`.

3. **Секцию `spring.jpa`** в `config-repo/tasks-service.yaml`.

4. **Вывод** `kubectl exec deploy/postgres -- psql -U admin -d tasksdb -c "\dt"`.

По этим данным станет точно ясно, что чинить. Скорее всего — либо добавить `ddl-auto: update`, либо поправить тип `id` в сущности.



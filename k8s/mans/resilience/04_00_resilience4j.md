> привет! мне по учебнику дальше нужно настроить resilience4j и как-то попробовать его использовать. поможешь?

Привет! Да, Resilience4j — это отличный следующий шаг. После того как мы разобрали circuit breaker на уровне Istio (DestinationRule), Resilience4j покажет **другой уровень** отказоустойчивости — на уровне приложения. Давай разберёмся.

## 🎯 Где Resilience4j встаёт в вашу схему

У вас уже есть circuit breaker на уровне Istio:
```
Spring Cloud Gateway ──▶ [Istio Envoy] ──▶ tasks-service
                              │
                              └─ DestinationRule: outlierDetection
```

Resilience4j работает **внутри Java-приложения**:
```
tasks-service ──▶ [Resilience4j] ──▶ вызов внешнего сервиса (MongoDB, Kafka, другой микросервис)
                        │
                        └─ @CircuitBreaker, @Retry, @RateLimiter, @Bulkhead
```

**Зачем оба?** Istio защищает на уровне сети (между сервисами). Resilience4j защищает на уровне **бизнес-логики** (внутри метода контроллера или сервиса). Например, Istio выбьет нездоровый под `tasks-service`, а Resilience4j остановит вызов MongoDB, если та начала отвечать медленно. Они **дополняют** друг друга.

## 📦 Зависимости для Spring Boot 4

Добавьте в `pom.xml` `tasks-service`:

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.4.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

**Важно:** Resilience4j использует **AOP** (аспектно-ориентированное программирование) для работы аннотаций. Без `spring-boot-starter-aop` аннотации **молча игнорируются**.

## ⚙️ Настройка в `application.yaml`

Каждое «устройство» (CircuitBreaker, Retry и т.д.) настраивается отдельно. Для начала — минимальный пример для MongoDB:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        # Считать ошибкой только определённые исключения
        recordExceptions:
          - com.mongodb.MongoSocketOpenException
          - com.mongodb.MongoTimeoutException
          - java.net.ConnectException

  retry:
    instances:
      mongoDb:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - com.mongodb.MongoSocketOpenException
          - java.net.ConnectException

  timelimiter:
    instances:
      mongoDb:
        timeoutDuration: 3s
        cancelRunningFuture: true
```

**Разбор ключевых параметров:**

| Параметр | Что означает |
|---|---|
| `slidingWindowSize` | Сколько последних вызовов учитывать |
| `minimumNumberOfCalls` | Минимум вызовов до расчёта процента ошибок |
| `failureRateThreshold` | Порог ошибок (%), после которого цепь размыкается |
| `waitDurationInOpenState` | Сколько ждать перед попыткой закрыть цепь |
| `permittedNumberOfCallsInHalfOpenState` | Сколько тестовых вызовов разрешить в полуоткрытом состоянии |
| `maxAttempts` (retry) | Сколько раз повторять при ошибке |
| `waitDuration` (retry) | Пауза между попытками |

Полный список — в документации Resilience4j.

## 💻 Использование: аннотации на методах

### Пример: вызов MongoDB с защитой

```java
@Service
public class TaskService {

    private final MongoTemplate mongoTemplate;

    public TaskService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    @CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    @Retry(name = "mongoDb")
    @TimeLimiter(name = "mongoDb")
    public Task createTask(Task task) {
        return mongoTemplate.save(task);
    }

    // Fallback — вызывается, когда цепь разомкнута или все попытки исчерпаны
    public Task fallbackCreateTask(Task task, Throwable t) {
        // Вариант 1: вернуть «заглушку»
        // Вариант 2: сохранить в очередь для повторной обработки
        // Вариант 3: бросить кастомное исключение
        throw new ServiceUnavailableException("MongoDB недоступна, попробуйте позже");
    }
}
```

**Что происходит при вызове:**
1. `@Retry` перехватывает вызов и повторяет его до 3 раз при ошибке.
2. `@CircuitBreaker` считает ошибки; если их больше 50% за последние 10 вызовов — размыкает цепь.
3. `@TimeLimiter` прерывает вызов, если он длится дольше 3 секунд.
4. Если цепь разомкнута или все попытки провалились — вызывается `fallbackCreateTask`.

### Пример: вызов внешнего REST-сервиса через WebClient

```java
@Service
public class ExternalApiService {

    private final WebClient webClient;

    @CircuitBreaker(name = "externalApi", fallbackMethod = "fallbackGetData")
    @Retry(name = "externalApi")
    public Mono<String> getData(String id) {
        return webClient.get()
                .uri("/api/data/{id}", id)
                .retrieve()
                .bodyToMono(String.class);
    }

    public Mono<String> fallbackGetData(String id, Throwable t) {
        return Mono.just("Данные временно недоступны");
    }
}
```

## ⚠️ Порядок аспектов: важно для Retry + CircuitBreaker

По умолчанию в Resilience4j порядок такой:
```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( Function ) ) ) ) )
```

**Retry — снаружи, CircuitBreaker — внутри.** Это означает: каждая повторная попытка Retry **проходит через CircuitBreaker**, и все они считаются как отдельные вызовы. Это может **раздувать счётчик ошибок** и быстрее размыкать цепь.

Если хотите, чтобы **CircuitBreaker был снаружи** (размыкался сразу, без многократных Retry), задайте явный порядок в `application.yaml`:

```yaml
resilience4j:
  circuitbreaker:
    circuitBreakerAspectOrder: 1
  retry:
    retryAspectOrder: 2
```

Тогда порядок станет: `CircuitBreaker ( Retry ( Function ) )` — Retry внутри, CircuitBreaker снаружи.

## 📊 Мониторинг через Actuator

Resilience4j автоматически регистрирует метрики, если есть `spring-boot-starter-actuator` и `resilience4j-micrometer`. Добавьте в `application.yaml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,retries,ratelimiters
  endpoint:
    health:
      show-details: always
```

После этого доступны эндпоинты:

```bash
# Состояние всех circuit breaker'ов
curl http://localhost:8080/actuator/circuitbreakers | jq

# События конкретного breaker'а
curl http://localhost:8080/actuator/circuitbreakerevents/mongoDb | jq

# Health с деталями
curl http://localhost:8080/actuator/health | jq '.components.circuitBreakers'
```

**Пример вывода:**
```json
{
  "mongoDb": {
    "state": "CLOSED",
    "failureRate": "0.0%",
    "bufferedCalls": 5,
    "failedCalls": 0,
    "successfulCalls": 5
  }
}
```

## 🧪 Как протестировать

Самый простой способ — **временно сломать MongoDB**:

1. **Отключите MongoDB Service:**
   ```bash
   kubectl scale deploy tasks-mongodb --replicas=0
   ```

2. **Отправьте несколько запросов** на `tasks-service`.

3. **Смотрите логи:**
   ```bash
   kubectl logs deploy/tasks-service -c tasks-service -f | grep -i "circuit\|retry\|fallback"
   ```

4. **Проверьте состояние breaker'а:**
   ```bash
   kubectl port-forward svc/tasks-service 8080:80
   curl http://localhost:8080/actuator/circuitbreakers | jq
   ```

Ожидаемое поведение:
- Первые 5 вызовов — ошибки, breaker `CLOSED`.
- После превышения порога — breaker `OPEN`.
- Следующие вызовы **мгновенно** возвращают fallback (без обращения к MongoDB).
- Через `waitDurationInOpenState` (10 сек) — breaker `HALF_OPEN`, пробует 3 вызова.
- Если MongoDB вернулась — breaker `CLOSED`.

5. **Верните MongoDB:**
   ```bash
   kubectl scale deploy tasks-mongodb --replicas=1
   ```

## 📋 Пошаговый план для вашего проекта

1. **Добавьте зависимости** в `pom.xml` (`resilience4j-spring-boot3`, `actuator`, `aop`).
2. **Настройте** `application.yaml` — секция `resilience4j`.
3. **Добавьте аннотации** на методы, которые работают с внешними системами (MongoDB, Kafka, другие сервисы).
4. **Добавьте fallback-методы** — что возвращать, когда сервис недоступен.
5. **Пересоберите образ** и перезапустите под:
   ```bash
   docker build -t myorg/tasks-service:1.0 .
   minikube image load myorg/tasks-service:1.0
   kubectl rollout restart deploy/tasks-service
   ```
6. **Проверьте actuator:**
   ```bash
   kubectl port-forward svc/tasks-service 8080:80
   curl http://localhost:8080/actuator/circuitbreakers | jq
   ```
7. **Протестируйте** — отключите MongoDB и отправьте запросы.

## 💡 Важное про сочетание с Istio

У вас **два уровня circuit breaker**:
- **Istio DestinationRule** — защищает `spring-cloud-gateway` от нездоровых подов `tasks-service`.
- **Resilience4j** — защищает `tasks-service` от нездоровой MongoDB.

**Не дублируйте** одну и ту же логику на обоих уровнях. Если Istio уже выбивает нездоровый под, Resilience4j не нужно делать то же самое для HTTP-вызовов между сервисами — оставьте Resilience4j для **внутренних** зависимостей (БД, кэш, внешние API).

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| Что такое Resilience4j? | Библиотека отказоустойчивости на уровне Java-кода |
| Где работает? | Внутри приложения, на методах |
| Чем отличается от Istio circuit breaker? | Istio — на уровне сети, Resilience4j — на уровне бизнес-логики |
| Какие паттерны? | CircuitBreaker, Retry, RateLimiter, Bulkhead, TimeLimiter |
| Как настроить? | `application.yaml` + аннотации на методах |
| Как мониторить? | Actuator: `/actuator/circuitbreakers` |
| Нужен ли AOP? | Да, без `spring-boot-starter-aop` аннотации не работают |

Начните с **одного** аннотированного метода — например, `createTask` — и посмотрите, как ведёт себя breaker через actuator. Когда увидите `OPEN` → `HALF_OPEN` → `CLOSED` вживую, механизм станет понятен. Дальше можно добавить Retry, Bulkhead и другие паттерны.

> помоги разобраться. я упустил один момент. когда мы обсуждали DestinationRule, я как-то упустил его и не добавил. а теперь получается он нужен? это и есть circuit breaker на уровне istio? он убирает поды, которые не отвечают? а Resilience4j-CircuitBreaker работает на уровне приложения  и когда включается? если сервис недоступен? так и под его не будет отвечать?

Отличный вопрос — тут действительно легко запутаться, потому что оба механизма называются «circuit breaker», но работают **на разных уровнях** и решают **разные задачи**. Давайте разложим по полочкам.

## 🎯 Два circuit breaker — два разных мира

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Spring Cloud Gateway                                            │
│         │                                                        │
│         │ вызов HTTP                                            │
│         ▼                                                        │
│  ┌──────────────────┐                                            │
│  │ Envoy (sidecar)  │  ← здесь работает Istio Circuit Breaker    │
│  │                  │     (outlierDetection из DestinationRule)  │
│  └────────┬─────────┘                                            │
│           │                                                      │
│           │ TCP-соединение                                       │
│           ▼                                                      │
│  ┌──────────────────┐                                            │
│  │ tasks-service    │                                            │
│  │ ┌──────────────┐ │                                            │
│  │ │ ваш код      │ │  ← здесь работает Resilience4j             │
│  │ │ @CircuitBreaker│                                            │
│  │ └──────┬───────┘ │                                            │
│  │        │         │                                            │
│  │        ▼         │                                            │
│  │   вызов MongoDB  │  ← Resilience4j защищает ЭТОТ вызов        │
│  └──────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

## 🔍 Istio Circuit Breaker (DestinationRule)

**Что делает:** sidecar (Envoy) **клиента** следит за ответами от **инстансов** сервиса-получателя. Если один под `tasks-service` начал отвечать 5xx или таймаутить — Envoy **локально** выбивает его из своего списка балансировки на `baseEjectionTime`.

**Ключевое:**

| Аспект | Значение |
|---|---|
| Где работает | В Envoy sidecar **клиента** |
| Что защищает | Клиента от нездоровых **подов-инстансов** |
| Что делает с подом | Убирает из **своей** балансировки (не из Kubernetes!) |
| Область | **Только для этого клиента** — другие Envoy о проблеме не знают |
| Восстановление | Автоматически через `baseEjectionTime` |

**Важно:** под **не удаляется из Kubernetes**. Он всё ещё в `Endpoints` сервиса. Просто **этот конкретный Envoy** перестаёт ему слать трафик на время. Если другой клиент (например, `spring-cloud-gateway` с другим подом) обратится к тому же «выбитому» поду — он попробует, и если под жив, всё сработает.

**Аналогия:** вы звоните в ресторан, три раза попадаете на занятую линию, и на 10 минут перестаёте туда звонить. Ресторан при этом работает и принимает других клиентов.

## 🔍 Resilience4j CircuitBreaker (в коде)

**Что делает:** перехватывает вызов **метода** в вашем Java-коде. Считает исключения и таймауты. Если их больше порога — размыкает цепь и вызывает `fallback`-метод **вместо** реального вызова.

**Ключевое:**

| Аспект | Значение |
|---|---|
| Где работает | Внутри JVM приложения |
| Что защищает | Приложение от **зависания** на медленных/падающих зависимостях |
| Что делает с зависимостью | Не трогает её, просто перестаёт вызывать |
| Область | **Только для этого экземпляра приложения** |
| Восстановление | Через `waitDurationInOpenState` + пробные вызовы |

**Важно:** Resilience4j **не знает** ничего о подах, Envoy, Kubernetes. Он просто видит, что метод бросает исключение или долго работает — и реагирует.

**Аналогия:** вы звоните в ресторан и вместо гудков слышите «занято». Через три попытки вы решаете: «больше не звоню, закажу пиццу в другом месте». Ресторан тут ни при чём — это ваше решение.

## ⚠️ Отвечаю на ваш вопрос: «если под не отвечает, зачем Resilience4j?»

Ваш вопрос: *«если сервис недоступен, так и под его не будет отвечать — что тут сделает Resilience4j?»*

**Resilience4j делает три важные вещи, которые Istio не делает:**

### 1. Даёт клиенту **осмысленный ответ**, а не ошибку

Istio при недоступности пода просто оборвёт соединение. Ваш код получит `ConnectException` и... что? Упадёт с 500-й? А Resilience4j позволит вернуть **осмысленный fallback**:

```java
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallback")
public Task createTask(Task task) { ... }

public Task fallback(Task task, Throwable t) {
    // Возвращаем кэшированные данные, ставим в очередь, 
    // показываем пользователю понятное сообщение
}
```

### 2. Защищает от **медленных** зависимостей, а не только мёртвых

Istio выбивает под только по **5xx/таймаутам на уровне HTTP**. Но представьте: MongoDB **жива**, но отвечает **очень медленно** (5 секунд на запрос). Istio не выбьет под — он же отвечает! А Resilience4j с `@TimeLimiter` прервёт вызов через 3 секунды и разомкнёт цепь. Это **спасает ваш сервис** от накопления зависших потоков.

### 3. Работает с **внутренними** зависимостями

Istio circuit breaker работает **только для HTTP-вызовов между сервисами через Envoy**. Но:
- **MongoDB** — не HTTP, а TCP-протокол. Istio его outlierDetection для неё не настроен (и обычно не нужен).
- **Kafka** — тоже не HTTP.
- **Redis** — не HTTP.
- **Вызовы внешних API** — Istio их вообще не видит.

Для всех этих зависимостей **Resilience4j — единственная защита**.

## 📊 Сравнительная таблица

| Аспект | Istio DestinationRule | Resilience4j |
|---|---|---|
| **Уровень** | Сеть (Envoy) | Код (Java) |
| **Что защищает** | HTTP-вызовы между сервисами | Любые вызовы в коде |
| **Считает ошибки** | По 5xx/таймаутам на уровне HTTP | По исключениям и таймаутам метода |
| **Реакция** | Убирает под из балансировки | Размыкает цепь, вызывает fallback |
| **Fallback** | Нет (просто не вызывает под) | Да, вы пишете сами |
| **Видит MongoDB/Kafka** | Нет | Да |
| **Видит медленные ответы** | Только HTTP-таймауты | Любые задержки |
| **Область действия** | Локально для каждого Envoy | Локально для каждого экземпляра |

## 🎯 Когда что использовать в вашем проекте

**Istio DestinationRule — для HTTP-вызовов между сервисами:**
```
Spring Cloud Gateway → tasks-service  ← Istio CB защитит Gateway
```
Настройте `DestinationRule` для `tasks-service` — если один под начнёт падать, Gateway перестанет ему слать трафик.

**Resilience4j — для внутренних зависимостей:**
```
tasks-service → MongoDB   ← Resilience4j защитит tasks-service
tasks-service → Kafka     ← Resilience4j защитит tasks-service
tasks-service → Redis     ← Resilience4j защитит tasks-service
```

## 💡 Разница на конкретном сценарии

**Сценарий:** MongoDB упала.

**Что произойдёт с Istio CB:** **ничего**. Istio не видит MongoDB — она не зарегистрирована в mesh как HTTP-сервис. Поды `tasks-service` живы и отвечают на запросы. Istio их не выбивает.

**Что произойдёт с Resilience4j:**
1. Каждый вызов `mongoTemplate.save()` бросает `MongoSocketOpenException`.
2. Resilience4j считает: 5 ошибок из 5 → `failureRate = 100%`.
3. Цепь размыкается: `CLOSED → OPEN`.
4. Следующие вызовы **не идут в MongoDB вообще** — сразу вызывается `fallback`.
5. Пользователь получает осмысленный ответ: «Сервис временно недоступен, попробуйте позже».
6. Через 10 секунд — `HALF_OPEN`, пробные вызовы. Если MongoDB вернулась — `CLOSED`.

**Поды `tasks-service` при этом не падают** — они продолжают принимать запросы и отвечать через fallback. Это **гораздо лучше**, чем если бы каждый запрос ждал 30 секунд таймаута, копил потоки и валил приложение по OOM.

## 📌 Итог

| Вопрос | Ответ |
|---|---|
| DestinationRule — это Istio circuit breaker? | ✅ Да, через `outlierDetection` |
| Он убирает поды? | Из **локальной** балансировки Envoy, **не** из Kubernetes |
| Resilience4j — это то же самое? | ❌ Нет, это **другой уровень** — в коде приложения |
| Когда включается Resilience4j? | Когда метод бросает исключение или таймаутит |
| Зачем он, если под мёртв? | Даёт **fallback** + защищает от **медленных** зависимостей + работает с **не-HTTP** зависимостями |
| Нужны ли оба? | ✅ Да, они дополняют друг друга |

**Простое правило:**
- **Istio CB** → «не долби мёртвый под» (для HTTP-вызовов между сервисами).
- **Resilience4j** → «не зависай на мёртвой зависимости» (для всего, что вызывает ваш код).

Они работают на **разных уровнях** и **не дублируют** друг друга. Istio — «сетевая страховка», Resilience4j — «страховка в коде». В production обычно есть **оба**.

> а DestinationRule я должен был включить?

у меня есть вот такой конфиг

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-service-rule
  namespace: default
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

```

> его надо применить? kubectl apply -f destinationRule.yaml?

Да, чтобы включить circuit breaker на уровне Istio, `DestinationRule` нужно применить — но ваш текущий конфиг этого не делает. Давайте разберём, почему.

### 🎯 Что в вашем `DestinationRule` есть, а чего нет

Ваш манифест описывает **две вещи**:

*   **`loadBalancer`**: стратегия балансировки (Round Robin).
*   **`subsets`**: логические группы подов по меткам `version: v1` и `version: v2`. Это нужно для **канареечных деплоев** (когда `VirtualService` направляет, скажем, 10% трафика на `v2`, а 90% — на `v1`).

**Функции circuit breaker (`outlierDetection`) в вашем конфиге нет.** Без блока `outlierDetection` в `trafficPolicy` Envoy не будет выбивать нездоровые поды — он продолжит слать им трафик, даже если они отвечают ошибками.

### 🛠️ Как добавить circuit breaker

Circuit breaker в Istio настраивается внутри `trafficPolicy` двумя блоками: `connectionPool` (лимиты на соединения) и `outlierDetection` (выбрасывание нездоровых хостов).

**Важное уточнение по `host`:** в вашем манифесте указан `host: my-service`. Это должно быть **реальное DNS-имя вашего Kubernetes Service**, к которому применяется политика. Для вашего проекта это, скорее всего, `spring-cloud-gateway` (чтобы защитить его от падающих подов `tasks-service`) или `tasks-service`.

Вот исправленный манифест, который **и оставляет subsets, и добавляет circuit breaker**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: tasks-service-cb
  namespace: default
spec:
  host: tasks-service  # ⚠️ Укажите реальное имя вашего сервиса
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    # --- Circuit Breaker: лимиты на соединения ---
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
    # --- Circuit Breaker: выбрасывание нездоровых подов ---
    outlierDetection:
      consecutive5xxErrors: 5      # 5 ошибок подряд → выбить под
      interval: 10s                # Проверять каждые 10 секунд
      baseEjectionTime: 30s        # Выбить минимум на 30 секунд
      maxEjectionPercent: 50       # Выбивать не более 50% подов
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

**Разбор ключевых полей `outlierDetection`:**

*   **`consecutive5xxErrors: 5`**: если один под вернёт 5 ошибок (5xx) **подряд**, он будет выброшен из пула балансировки. Один успешный ответ сбрасывает счётчик.
*   **`interval: 10s`**: как часто Envoy сканирует пул на предмет нездоровых хостов.
*   **`baseEjectionTime: 30s`**: минимальное время, на которое под выбивается. При повторных ошибках время увеличивается (30s → 60s → 90s).
*   **`maxEjectionPercent: 50`**: защита от полного «выбивания» сервиса. Даже если все поды плохие, Envoy оставит хотя бы половину, чтобы сервис не умер полностью.

### 📌 Что делать с `subsets` и `loadBalancer`

*   **`subsets`**: если вы не планируете канареечные деплои прямо сейчас, их **можно убрать**. Они не мешают, но и не нужны для circuit breaker. Если оставите — убедитесь, что у ваших подов есть соответствующие метки `version: v1` и `version: v2`, иначе subsets будут пустыми.
*   **`loadBalancer`**: `ROUND_ROBIN` — это стандартная стратегия, оставьте её.

### ✅ Как применить

Да, ваш `DestinationRule` применяется той же командой, что и остальные манифесты:

```bash
kubectl apply -f destinationRule.yaml
```

После применения проверьте, что Istio «увидел» изменения:

```bash
# Проверить, что ресурс создан
kubectl get destinationrule tasks-service-cb -o yaml

# Проверить, что Envoy применил outlier detection (ищите строки с outlierDetection в конфиге кластера)
istioctl proxy-config cluster deploy/spring-cloud-gateway -o json | grep -A5 "tasks-service"
```

### 💡 Важное напоминание: два уровня защиты

Не забывайте, что это **только один** из двух circuit breaker'ов в вашей системе:

*   **Этот `DestinationRule` (Istio)**: защищает `spring-cloud-gateway` от нездоровых **подов** `tasks-service`. Работает на уровне **сети** (Envoy).
*   **Resilience4j (в коде)**: защищает `tasks-service` от нездоровых **внутренних зависимостей** (MongoDB, Kafka, Redis). Работает на уровне **приложения**.

Они **не заменяют**, а **дополняют** друг друга. `DestinationRule` нужен обязательно, если у вас больше одного пода `tasks-service` и вы хотите, чтобы трафик автоматически переставал идти на «упавший» под.

> получается,  область ответственности Resilience4j  - сервис целиком, то есть он настраивается для приложения tasks-service? а DestinationRule настраивается для кластера и следит за подами всех сервисов? tasks-service, user-service?

Оба утверждения нужно скорректировать — в них есть смещение акцентов. Давайте разберём точно.

## 🎯 Resilience4j: не «для сервиса целиком», а «для конкретных вызовов внутри сервиса»

Resilience4j **не защищает tasks-service**. Он защищает **tasks-service от его зависимостей**.

**Ключевая идея:** Resilience4j всегда стоит на стороне **того, кто вызывает**. Он отвечает на вопрос: «что делать этому приложению, если **то, к чему оно обращается**, сломается?»

```
tasks-service ──вызывает──▶ MongoDB
     │
     └─ здесь работает Resilience4j
        (защищает tasks-service от MongoDB)
```

**Если spring-cloud-gateway тоже хочет защиты** от падающего tasks-service — ему нужна **своя** конфигурация Resilience4j:

```
spring-cloud-gateway ──вызывает──▶ tasks-service
     │
     └─ здесь работает Resilience4j
        (защищает gateway от tasks-service)
```

**Итого:** Resilience4j настраивается **в каждом приложении отдельно**, для **конкретных методов**, которые делают **внешние вызовы**.

| Вопрос | Ответ |
|---|---|
| Где живёт? | Внутри **каждого** Java-приложения |
| Что защищает? | Это приложение от его зависимостей |
| Область действия | **Отдельный метод** (аннотация на методе) |
| Нужно ли настраивать в gateway? | Да, если gateway вызывает другие сервисы |
| Нужно ли настраивать в tasks-service? | Да, для MongoDB/Kafka/Redis |

## 🎯 DestinationRule: не «для кластера», а «для конкретного сервиса-получателя»

DestinationRule — это **namespaced-ресурс** (живёт в namespace, например `default`), и каждый DestinationRule описывает **один** сервис-получатель.

```
DestinationRule для tasks-service:
   └─ "как клиенты должны обращаться к tasks-service"

DestinationRule для user-service:
   └─ "как клиенты должны обращаться к user-service"
```

**Один DestinationRule = один сервис.** Не один на кластер, не один на все сервисы.

| Вопрос | Ответ |
|---|---|
| Где живёт? | В namespace (`default`) |
| Для кого настраивается? | Для **одного** сервиса-получателя (`host: tasks-service`) |
| Кто применяет? | Sidecar'ы **всех клиентов**, которые обращаются к этому сервису |
| Нужен ли отдельный для user-service? | Да, если хотите circuit breaker и для него |
| Автоматически мониторит все сервисы? | **Нет.** Только те, для которых создан DestinationRule |

## 📊 Полная картина для вашего проекта

```
┌──────────────────────────────────────────────────────────────────┐
│                        Kubernetes кластер                          │
│                                                                    │
│  ┌────────────────────┐         ┌─────────────────────┐           │
│  │ spring-cloud-      │         │  tasks-service      │           │
│  │ gateway            │         │  (3 пода)           │           │
│  │                    │         │                     │           │
│  │ ┌────────────────┐ │         │  ┌───────────────┐  │           │
│  │ │ Resilience4j   │ │         │  │ Resilience4j  │  │           │
│  │ │ (для вызовов   │ │         │  │ (для вызовов  │  │           │
│  │ │  к tasks-svc)  │ │         │  │  к MongoDB)   │  │           │
│  │ └────────────────┘ │         │  └───────────────┘  │           │
│  └────────┬───────────┘         └──────────┬──────────┘           │
│           │                                │                      │
│           │ HTTP-вызов                     │ TCP-вызов            │
│           │                                │                      │
│           ▼                                ▼                      │
│  ┌─────────────────────────────────┐   ┌──────────────┐           │
│  │ Envoy sidecar (gateway)         │   │  MongoDB     │           │
│  │                                 │   └──────────────┘           │
│  │ DestinationRule для tasks-svc:  │                              │
│  │  - outlierDetection             │                              │
│  │  - connectionPool               │                              │
│  └─────────────────────────────────┘                              │
│           │                                                       │
│           │ выбирает здоровый под                                 │
│           ▼                                                       │
│  ┌─────────────────────────────────┐                              │
│  │ tasks-service под (здоровый)    │                              │
│  └─────────────────────────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
```

**Что здесь важно:**

1. **Resilience4j в gateway** — защищает gateway от проблем с tasks-service (например, если все поды tasks-service медленно отвечают).
2. **Resilience4j в tasks-service** — защищает tasks-service от проблем с MongoDB.
3. **DestinationRule для tasks-service** — говорит Envoy в sidecar'ах **всех клиентов** (gateway, других сервисов), как обращаться с подами tasks-service.

## 🎯 Разные «оси» ответственности

| Ось | Resilience4j | DestinationRule |
|---|---|---|
| **Где физически** | В JVM приложения | В Envoy sidecar |
| **Для кого настраивается** | Для вызывающего приложения | Для вызываемого сервиса |
| **На что реагирует** | На исключения в методах | На HTTP-ответы (5xx, таймауты) |
| **Область видимости** | Локально для экземпляра | Локально для каждого Envoy |
| **Кто «видит» эффект** | Только это приложение | Все клиенты этого сервиса |

## 💡 Пример: два сервиса, две конфигурации

Допустим, у вас есть `tasks-service` и `user-service`, и gateway вызывает оба.

**Resilience4j в gateway (два набора настроек):**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      tasksService:        # ← для вызовов tasks-service
        slidingWindowSize: 10
        failureRateThreshold: 50
      userService:         # ← для вызовов user-service
        slidingWindowSize: 20
        failureRateThreshold: 30
```

**DestinationRule для каждого сервиса (отдельные ресурсы):**
```yaml
# destination-rule-tasks.yaml
kind: DestinationRule
metadata:
  name: tasks-service-cb
spec:
  host: tasks-service         # ← для tasks-service
  trafficPolicy:
    outlierDetection: { ... }

---
# destination-rule-user.yaml
kind: DestinationRule
metadata:
  name: user-service-cb
spec:
  host: user-service          # ← для user-service
  trafficPolicy:
    outlierDetection: { ... }
```

## 📌 Итог

| Вопрос | Ответ |
|---|---|
| Resilience4j настраивается для tasks-service целиком? | **Нет.** Для конкретных методов внутри приложения |
| Resilience4j в gateway нужен? | Да, если gateway вызывает другие сервисы |
| DestinationRule — для кластера? | **Нет.** Один DestinationRule = один сервис-получатель |
| Один DestinationRule на все сервисы? | **Нет.** Нужен отдельный для каждого сервиса |
| Автоматически мониторит все поды? | Только те, для которых создан DestinationRule |

**Коротко:**
- **Resilience4j** — «защита **вызывающего** от **вызываемого**», настраивается в **каждом** приложении.
- **DestinationRule** — «правила обращения **к конкретному** сервису», настраивается **для каждого** сервиса отдельно.

Оба — **per-service**, но с разных сторон: Resilience4j на стороне клиента (в коде), DestinationRule — на стороне получателя (в Envoy клиента, но описывает получателя).

> я добавил зависимости, добавил методы с аннотациями, удалил поды mongodb. делаю запросы  к tasks-service "tasks/create", которые дергают БД. но так как подов БД  нет, возвращается 500 - это понятно, но я делаю запросов 7-8, на каждый возвращается 500 и как будто бы ничего не происходит - в логах tasks-service нет сообщения из fallback, что mongodb недоступна, 
curl http://localhost:8080/actuator/circuitbreakers | jq - все время показывает CLOSED.. у меня еще же есть ratelimit, но я сделал чтобы он после 10 ответов с ошибками возвращал - 429. но по идее circuitbreaker же должен раньше срабатывать?

Ваша ситуация — классическая ловушка при первом знакомстве с Resilience4j, и я вижу здесь сразу три вероятные причины, которые могли наложиться друг на друга. Давайте разберем их по порядку.

### 🔍 Причина 1: Конфигурация `recordExceptions` не включает ваши исключения

Ваш конфиг может быть слишком строгим. Вы указали:
```yaml
recordExceptions:
  - com.mongodb.MongoSocketOpenException
  - com.mongodb.MongoTimeoutException
  - java.net.ConnectException
```
Проблема в том, что реальное исключение, которое «вылетает» при попытке подключения к MongoDB, может быть **обернуто** в другой класс. Например, Spring Data MongoDB часто оборачивает ошибки драйвера в `org.springframework.dao.DataAccessResourceFailureException`. Ваш `recordExceptions` не считает это исключение «ошибкой», поэтому CircuitBreaker видит его как «успех» и остается в состоянии `CLOSED`.

**Решение:** Самый простой способ — сказать Resilience4j считать **любое** `RuntimeException` или `Exception` ошибкой, пока вы не разберетесь с точными классами:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        # ... остальные параметры ...
        recordExceptions:
          - java.lang.Exception
```
Это гарантирует, что CircuitBreaker будет реагировать на все исключения, а не только на те, что вы явно перечислили.

### 🎯 Причина 2: AOP-прокси не перехватывает вызов (внутренний вызов)

Это самая коварная причина. Если ваш метод с аннотацией `@CircuitBreaker` вызывается **изнутри того же класса** (т.е. `this.createTask()`), Spring AOP **не перехватывает** этот вызов. Аннотация просто игнорируется.

**Как проверить:** Убедитесь, что метод, помеченный `@CircuitBreaker`, вызывается из **другого** Spring-бина. Например, если `createTask` находится в `TaskService`, то `TaskController` должен вызывать `taskService.createTask(...)`, а не `this.createTask(...)`.

### ⏱️ Причина 3: Недостаточно вызовов для «разогрева»

Resilience4j не будет считать процент ошибок, пока не наберется `minimumNumberOfCalls` вызовов. Вы указали `minimumNumberOfCalls: 5`. Если вы сделали 7-8 запросов, но **каждый** вызов — это один запрос к вашему `/tasks/create`, то возможно, что не все вызовы дошли до CircuitBreaker (например, из-за кэша или других фильтров).

**Как проверить:** Посмотрите на событие `BUFFERED` в actuator. Или временно установите `minimumNumberOfCalls: 1`, чтобы CircuitBreaker сработал сразу после первой ошибки.

### 🚨 Важное примечание о Rate Limiting и Circuit Breaker

Вы упомянули, что настроили Rate Limit на возврат 429 после 10 ошибок. Обратите внимание: **Rate Limit и Circuit Breaker — это разные механизмы, и они не конкурируют**.

*   **Rate Limit (Istio):** Ограничивает **количество** запросов, которые пропускает Ingress Gateway. Он не смотрит на то, успешные они или нет. Он просто «не пускает» 11-й запрос.
*   **Circuit Breaker (Resilience4j):** Смотрит на **результат** вызовов (ошибка/успех). Он должен «разомкнуться» после, например, 5 ошибок, и **перестать** вызывать MongoDB, даже если запросы все еще идут.

То есть, если Circuit Breaker работает, он должен сработать **раньше**, чем Rate Limit. То, что вы видите 429, а не fallback, — это еще одно подтверждение, что Circuit Breaker **не активен**.

### 📝 Пошаговый план исправления

1.  **Упростите конфиг для отладки:** Временно установите `minimumNumberOfCalls: 1` и `recordExceptions: [java.lang.Exception]`. Это гарантирует, что одна же ошибка должна «разомкнуть» цепь.
2.  **Проверьте AOP:** Убедитесь, что вызов идет через прокси (из другого класса).
3.  **Проверьте actuator:** Зайдите на `/actuator/circuitbreakerevents/mongoDb` и посмотрите, есть ли события `ERROR` или `FAILURE`. Если там только `SUCCESS`, значит, исключение не записывается.
4.  **Добавьте логирование в fallback:** Убедитесь, что fallback-метод действительно вызывается, добавив туда `System.out.println("FALLBACK!");` или `log.error(...)`.
5.  **Проверьте версии:** Убедитесь, что версия `resilience4j-spring-boot3` совместима с вашим Spring Boot 4. Возможно, потребуется обновление до последней версии.

После этих шагов вы должны увидеть, как CircuitBreaker переходит в `OPEN`, а ваш fallback начинает возвращать «заглушку» вместо 500. Если что-то пойдет не так, пришлите вывод `/actuator/circuitbreakerevents`, и мы разберемся дальше.

> работает.  вернул minimumNumberOfCalls: 5, но оставил recordExceptions: java.lang.Exception. в итоге полчучается на 6й запрос с кодом 500 curl http://localhost:8080/actuator/circuitbreakers | jq вернул статус OPEN. вроде все норм,   recordExceptions: java.lang.Exception - наверное надо испраивть? мы же не можем на каждый чих отрубать запросы?))

Отлично, что заработало! Вы абсолютно правы — `java.lang.Exception` это «ядерный вариант» для отладки, в production так делать нельзя. Разберём, почему и что писать вместо него.

## 🎯 Почему нельзя `recordExceptions: java.lang.Exception`

Представьте: пользователь отправляет POST `/tasks/create` с невалидным JSON. Ваш код бросает `MethodArgumentNotValidException` (ошибка валидации, 400). Resilience4j **посчитает это ошибкой**, добавит в статистику, и через 5 таких запросов цепь **разомкнётся** — даже если MongoDB работает идеально.

```
Пользователь 1: невалидный JSON   → MethodArgumentNotValidException → ❌ записано как ошибка
Пользователь 2: невалидный JSON   → MethodArgumentNotValidException → ❌ записано как ошибка
...
Пользователь 5: невалидный JSON   → MethodArgumentNotValidException → ❌ записано как ошибка
                                     ⚠️ Цепь разомкнулась, MongoDB не при чём
```

Теперь **все** запросы получают fallback, хотя проблема была только в кривых запросах клиентов. Это классический сценарий «самосбывающегося пророчества» — вы сами себе устроили outage.

## 🧩 Принцип: записывать только «ошибки зависимости»

Правило простое: **записывать только те исключения, которые означают, что виноват downstream** (MongoDB, Kafka, внешний API). Всё, что относится к **бизнес-логике** или **ошибкам клиента**, записывать **не надо**.

| Категория | Пример | Записывать? |
|---|---|---|
| **Downstream недоступен** | `MongoSocketOpenException`, `ConnectException`, `MongoTimeoutException` | ✅ Да |
| **Downstream отвечает ошибкой** | `DataAccessResourceFailureException`, `MongoException` | ✅ Да |
| **Таймаут** | `TimeoutException`, `SocketTimeoutException` | ✅ Да |
| **Ошибка валидации** | `MethodArgumentNotValidException` | ❌ Нет |
| **Ресурс не найден** | `EntityNotFoundException`, `NoSuchElementException` | ❌ Нет |
| **Бизнес-ошибка** | `InsufficientFundsException`, `TaskAlreadyExistsException` | ❌ Нет |
| **Ошибка программиста** | `NullPointerException`, `IllegalArgumentException` | ❌ Нет (лучше сразу падать и чинить баг) |

## 🛠️ Три подхода к конфигурации

### Подход A: Явный whitelist (самый безопасный)

Перечисляете только те исключения, которые считаете «ошибками зависимости»:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        recordExceptions:
          - com.mongodb.MongoSocketOpenException
          - com.mongodb.MongoSocketReadException
          - com.mongodb.MongoTimeoutException
          - com.mongodb.MongoSocketException
          - com.mongodb.MongoException
          - org.springframework.dao.DataAccessResourceFailureException
          - org.springframework.dao.QueryTimeoutException
          - java.net.ConnectException
          - java.net.SocketTimeoutException
```

**Плюсы:** чёткий контроль, никакие бизнес-ошибки не влияют на цепь.  
**Минусы:** нужно знать все классы исключений, которые может бросить драйвер. Иногда пропустишь какой-то — и цепь не разомкнётся там, где должна.

### Подход B: Исключения-исключения (ignoreExceptions)

Записывать **всё**, кроме явно указанных бизнес-ошибок:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        # записываем все исключения по умолчанию
        ignoreExceptions:
          - org.springframework.web.bind.MethodArgumentNotValidException
          - jakarta.validation.ValidationException
          - dev.folomkin.tasks.exception.TaskNotFoundException
          - dev.folomkin.tasks.exception.ValidationException
          - java.lang.IllegalArgumentException
```

**Плюсы:** «fail-safe» — если появится новое исключение от драйвера MongoDB, оно автоматически попадёт в статистику.  
**Минусы:** если забудете добавить бизнес-исключение в ignore-list — оно будет ломать цепь.

### Подход C: Смешанный (рекомендую для production)

Явно указываете **и** те, что записывать, **и** те, что игнорировать:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        recordExceptions:
          - com.mongodb.MongoException
          - org.springframework.dao.DataAccessException
          - java.net.ConnectException
          - java.io.IOException
        ignoreExceptions:
          - org.springframework.web.bind.MethodArgumentNotValidException
          - dev.folomkin.tasks.exception.TaskNotFoundException
```

Правила приоритета: если исключение попало в `ignoreExceptions`, оно **не** записывается, даже если матчится с `recordExceptions`. Это удобно для точечных исключений.

## 📊 Как узнать реальные классы исключений

Чтобы не гадать, добавьте в fallback-метод логирование класса исключения:

```java
public Task fallbackCreateTask(Task task, Throwable t) {
    log.error("CircuitBreaker fallback for task={}, exception={}", 
              task, t.getClass().getName(), t);
    throw new ServiceUnavailableException("MongoDB недоступна");
}
```

Отключите MongoDB, отправьте пару запросов, посмотрите логи — увидите **точный** класс исключения. Например:

```
exception=org.springframework.dao.DataAccessResourceFailureException
```

И уже это имя вписывайте в `recordExceptions`.

## 💡 Отдельный вопрос: что делать с NPE и другими багами

**Хорошая практика:** ошибки программиста (`NullPointerException`, `IllegalStateException` от неправильного состояния) **не должны** попадать в CircuitBreaker. Пусть приложение падает и алертит — это баг, который надо чинить, а не «прятать» за fallback.

Если CircuitBreaker будет глотать NPE, вы никогда не узнаете, что где-то в коде ошибка — потому что fallback будет возвращать «сервис недоступен», и пользователи будут жаловаться на «вечные проблемы с MongoDB», хотя на самом деле у вас баг в коде.

## 🎯 Рекомендация для вашего учебного проекта

Начните с **Подхода A** — явный whitelist. Впишите туда минимум:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        recordExceptions:
          - com.mongodb.MongoException
          - org.springframework.dao.DataAccessException
          - java.net.ConnectException
          - java.net.SocketTimeoutException
```

Это покроет 95% сценариев с MongoDB. Если увидите, что какое-то исключение «проскакивает» мимо — добавьте его в список по логам.

## 📌 Сравнение подходов

| Подход | Безопасность | Удобство | Когда использовать |
|---|---|---|---|
| A: whitelist | ✅ Высокая | ⚠️ Нужно знать классы | Production, критичные сервисы |
| B: ignore-list | ⚠️ Средняя | ✅ Просто | Быстрый старт, учебные проекты |
| C: смешанный | ✅ Высокая | ⚠️ Сложнее | Большие проекты с чёткими категориями |

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| `java.lang.Exception` — плохо? | ✅ Да, слишком широко |
| Что записывать? | Только «ошибки зависимости»: MongoDB-исключения, таймауты, сетевые ошибки |
| Что НЕ записывать? | Ошибки валидации, «не найдено», бизнес-ошибки, баги |
| Как узнать классы? | Логировать `t.getClass().getName()` в fallback |
| Что выбрать? | Whitelist с `com.mongodb.MongoException` + `DataAccessException` |

Уберите `java.lang.Exception`, добавьте конкретные MongoDB-исключения — и ваш CircuitBreaker будет срабатывать **только** на реальные проблемы с БД, а не на кривые запросы клиентов. Это и есть production-ready подход.


> у меня вопрос возник: метод "create-tasks" с аннотацией @CircuitBreaker из сервиса возвращает созданный объект, а fallback выводит в консоль приложения. в контроллере есть эндпоинт "@PostMapping("/create")" - он просто возвращает от сервиса ответ  - созданный объект. можно ли мне как-то из контроллера в тот же postman возвращать или успешный ответ или ответ от fallback? может как-то настроить ResponseEntity?

Да, можно — и это делается несколькими способами. Всё зависит от того, **как вы хотите сигнализировать клиенту**, что сработал fallback: через HTTP-статус, через специальный объект-ответ, или через отдельный тип исключения.

## 🎯 Сначала разберём, что происходит сейчас

В вашем сервисе:

```java
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
public Task createTask(Task task) {
    return mongoTemplate.save(task);
}

public Task fallbackCreateTask(Task task, Throwable t) {
    System.out.println("MongoDB недоступна");  // ← просто лог
    return null;                                // ← возвращается null
}
```

Контроллер:

```java
@PostMapping("/create")
public Task create(@RequestBody Task task) {
    return taskService.createTask(task);   // ← при fallback вернёт null
}
```

**Проблема:** fallback возвращает `null`, и Spring сериализует его в пустой ответ `200 OK` с телом `null`. Клиент не понимает, что что-то пошло не так.

## 🛠️ Три подхода к решению

### Подход 1: Fallback бросает кастомное исключение

Самый **чистый** способ. Fallback не возвращает `null`, а бросает исключение, которое контроллер (или `@ControllerAdvice`) превращает в правильный HTTP-ответ.

**Шаг 1.** Создайте кастомное исключение:

```java
public class ServiceUnavailableException extends RuntimeException {
    public ServiceUnavailableException(String message) {
        super(message);
    }
}
```

**Шаг 2.** Измените fallback:

```java
public Task fallbackCreateTask(Task task, Throwable t) {
    log.error("MongoDB недоступна, fallback triggered", t);
    throw new ServiceUnavailableException("Сервис временно недоступен, попробуйте позже");
}
```

**Шаг 3.** Обработайте исключение в `@RestControllerAdvice`:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ServiceUnavailableException.class)
    public ResponseEntity<Map<String, String>> handleServiceUnavailable(ServiceUnavailableException e) {
        return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)  // 503
                .body(Map.of(
                    "error", "service_unavailable",
                    "message", e.getMessage()
                ));
    }
}
```

**Шаг 4.** Контроллер **не меняется** — он просто возвращает `Task`:

```java
@PostMapping("/create")
public Task create(@RequestBody Task task) {
    return taskService.createTask(task);
}
```

**Что увидит Postman:**
- MongoDB жива → `201 Created` с телом `{"id": "...", "title": "..."}`
- MongoDB недоступна → `503 Service Unavailable` с телом `{"error": "service_unavailable", "message": "..."}`

Это **production-стандарт**: клиент сразу понимает, что произошло, и может отреагировать (retry, показать сообщение пользователю).

### Подход 2: Fallback возвращает специальный объект-«заглушку»

Если не хотите бросать исключения, fallback может вернуть объект с пометкой «это fallback»:

```java
public Task fallbackCreateTask(Task task, Throwable t) {
    log.warn("Fallback triggered: {}", t.getMessage());
    return Task.builder()
            .id(null)
            .title("SERVICE_UNAVAILABLE")  // маркер
            .description("MongoDB недоступна, попробуйте позже")
            .build();
}
```

**Минус:** клиент получает `200 OK` и должен **сам** проверять поле `title`, чтобы понять, что что-то не так. Это плохо — HTTP-статус не отражает реальность.

### Подход 3: Fallback возвращает `ResponseEntity` (но это плохо)

Технически можно, но **не рекомендуется**, потому что:

- Сервисный слой начинает знать про HTTP.
- Возвращаемый тип становится `ResponseEntity<Task>`, а не `Task`.
- CircuitBreaker не может понять, что «это ошибка», потому что `ResponseEntity` — не исключение.

**Так делать не стоит.** Если очень хочется — оставьте `ResponseEntity` только на уровне контроллера.

## 🎯 Правильный подход для вашего случая: № 1

Рекомендую **Подход 1** — с кастомным исключением. Причины:

1. **Контроллер остаётся чистым** — никаких проверок `if (result == null)`.
2. **HTTP-статус корректен** — 503 Service Unavailable (RFC 7231).
3. **Клиент понимает, что делать** — можно настроить retry с backoff.
4. **Единая точка обработки ошибок** — `@RestControllerAdvice`.
5. **CircuitBreaker работает корректно** — исключение из fallback не считается ошибкой метода (иначе получится рекурсия).

## 📝 Что нужно для `@RestControllerAdvice`

Добавьте зависимость, если её нет (обычно уже есть в `spring-boot-starter-web`):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Полный код обработчика:

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ServiceUnavailableException.class)
    public ResponseEntity<ErrorResponse> handleServiceUnavailable(ServiceUnavailableException e) {
        log.error("Service unavailable: {}", e.getMessage());
        return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(new ErrorResponse("service_unavailable", e.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(err -> err.getField() + ": " + err.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity
                .status(HttpStatus.BAD_REQUEST)
                .body(new ErrorResponse("validation_error", message));
    }

    @ExceptionHandler(TaskNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(TaskNotFoundException e) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("not_found", e.getMessage()));
    }
}
```

```java
public record ErrorResponse(String error, String message) {}
```

## 🎯 Что увидит Postman

**Успешный запрос:**
```
POST /tasks/create
Body: {"title": "test"}
Response: 201 Created
Body: {"id": "abc123", "title": "test", "createdAt": "..."}
```

**CircuitBreaker разомкнут (MongoDB недоступна):**
```
POST /tasks/create
Body: {"title": "test"}
Response: 503 Service Unavailable
Body: {"error": "service_unavailable", "message": "Сервис временно недоступен, попробуйте позже"}
```

**Ошибка валидации (нет title):**
```
POST /tasks/create
Body: {}
Response: 400 Bad Request
Body: {"error": "validation_error", "message": "title: must not be blank"}
```

## ⚠️ Важный нюанс с CircuitBreaker и fallback

У **каждого** метода с `@CircuitBreaker` должен быть свой fallback **с сигнатурой, совместимой с оригиналом**:

```java
// Оригинал
public Task createTask(Task task) { ... }

// Fallback — те же аргументы + Throwable в конце
public Task fallbackCreateTask(Task task, Throwable t) { ... }
```

**Правила:**
- Fallback должен возвращать **тот же тип** (`Task`).
- Fallback должен принимать **все аргументы оригинала** + `Throwable` в конце.
- Fallback может быть `private`, `public`, в том же классе.

Если оригинал возвращает `Task`, а fallback — `ResponseEntity<Task>` — Resilience4j не сможет их сопоставить.

## 💡 Дополнительный совет: не логируйте `System.out.println`

Используйте логгер:

```java
@Slf4j
@Service
public class TaskService {

    public Task fallbackCreateTask(Task task, Throwable t) {
        log.error("Fallback triggered for task={}: {}", task, t.getMessage(), t);
        throw new ServiceUnavailableException("MongoDB недоступна");
    }
}
```

`System.out.println` не попадает в логи структурированно, не имеет уровня, не фильтруется. `log.error` — попадает в ELK/Loki, можно настроить алерты.

## 📊 Сравнение подходов

| Подход | HTTP-статус | Клиент понимает | Простота | Рекомендую |
|---|---|---|---|---|
| 1. Fallback → исключение | ✅ 503 | ✅ Сразу | ✅ Просто | ✅ Да |
| 2. Fallback → заглушка | ❌ 200 | ⚠️ Надо парсить | ✅ Просто | ❌ Нет |
| 3. Fallback → ResponseEntity | ⚠️ Сложно | ✅ | ❌ Плохо | ❌ Нет |

## 💎 Итог

1. **Fallback должен бросать кастомное исключение** — это самый чистый способ.
2. **Контроллер не меняется** — он по-прежнему возвращает `Task`.
3. **`@RestControllerAdvice` превращает исключение** в `503 Service Unavailable` с понятным JSON.
4. **Клиент (Postman) видит** либо `201 Created` при успехе, либо `503` при fallback — и может отреагировать.
5. **Не возвращайте `ResponseEntity` из сервиса** — это нарушает разделение слоёв.

> ничего не работает. точнее работает, но не происходит то, что ожидалось. circutbracker меняет статус только с исключением         recordExceptions:  - java.lang.Exception, fallback не срабатывает. даже если статус изменился на OPEN - в postman - выводится только 500 и все. в логах простыня целая.


Отличная диагностика — раз статус меняется на `OPEN`, но fallback не вызывается, значит проблема **в сигнатуре fallback-метода**. Resilience4j **не может найти** fallback, ловит это исключение, и оно вылетает наружу как 500 с «простыней» в логах.

## 🎯 Почему fallback не вызывается

Resilience4j для вызова fallback использует **reflection**. Если сигнатура метода не совпадает с ожидаемой — он **не найдёт** его и выбросит `NoSuchMethodException` или `IllegalStateException` с текстом вроде:

```
Fallback method 'fallbackCreateTask' not found
```

**Найдите в логах эту строку** — это ключ:

```bash
kubectl logs deploy/tasks-service -c tasks-service | grep -i "fallback\|NoSuchMethod"
```

## 🔍 Правила сигнатуры fallback

Fallback **обязан** соответствовать строгим правилам:

| Правило | Пример |
|---|---|
| **То же имя, что в `fallbackMethod`** | `@CircuitBreaker(fallbackMethod = "fallbackCreateTask")` → метод `fallbackCreateTask` |
| **Тот же возвращаемый тип** | Оригинал `Task` → fallback тоже `Task` |
| **Те же аргументы + `Throwable` в конце** | `createTask(Task)` → `fallbackCreateTask(Task, Throwable)` |
| **Тот же класс** | Fallback — в том же классе, что и оригинал |
| **Доступен для вызова** | `public` (или хотя бы package-private, но лучше `public`) |
| **Не `static`** | Должен быть экземплярным методом |

## ⚠️ Типичные ошибки в вашем коде

Скорее всего, у вас одно из этих:

### Ошибка 1: Fallback возвращает `void` или другой тип

```java
// ❌ Неправильно
public Task createTask(Task task) { ... }

public void fallbackCreateTask(Task task, Throwable t) { ... }  // ← void, а не Task
```

### Ошибка 2: Fallback принимает не все аргументы

```java
// ❌ Неправильно
public Task createTask(Task task) { ... }

public Task fallbackCreateTask(Throwable t) { ... }  // ← потерялся Task
```

### Ошибка 3: `Throwable` не последний аргумент

```java
// ❌ Неправильно
public Task fallbackCreateTask(Throwable t, Task task) { ... }
```

### Ошибка 4: Fallback в другом классе

```java
@Service
public class TaskService {
    @CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    public Task createTask(Task task) { ... }
}

// ❌ Неправильно — fallback должен быть в TaskService, а не в отдельном классе
public class TaskFallback {
    public Task fallbackCreateTask(Task task, Throwable t) { ... }
}
```

## ✅ Правильный пример

```java
@Slf4j
@Service
public class TaskService {

    private final MongoTemplate mongoTemplate;

    public TaskService(MongoTemplate mongoTemplate) {
        this.mongoTemplate = mongoTemplate;
    }

    // Оригинал
    @CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    public Task createTask(Task task) {
        return mongoTemplate.save(task);
    }

    // Fallback — тот же класс, тот же тип, те же аргументы + Throwable
    public Task fallbackCreateTask(Task task, Throwable t) {
        log.error("Fallback triggered for task={}: {}", task, t.getMessage(), t);
        throw new ServiceUnavailableException("MongoDB недоступна, попробуйте позже");
    }
}
```

**Проверьте свой код на каждое из правил.** Особенно:
- Возвращаемый тип совпадает?
- Все аргументы + `Throwable` в конце?
- В том же классе?

## 🔍 Если сигнатура правильная, но fallback всё равно не вызывается

### Проверка 1: AOP-прокси работает?

Если `createTask` вызывается **изнутри того же класса** (`this.createTask(...)`), Spring AOP **не перехватит** вызов. Убедитесь, что метод вызывается из **другого** бина — из контроллера.

```java
// ✅ Правильно
@RestController
public class TaskController {
    private final TaskService taskService;

    @PostMapping("/create")
    public Task create(@RequestBody Task task) {
        return taskService.createTask(task);   // ← через бин, не this.
    }
}
```

### Проверка 2: `spring-boot-starter-aop` подключён?

Без него аннотации **молча игнорируются**. Проверьте `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### Проверка 3: `@RestControllerAdvice` видит исключение?

Если fallback бросает `ServiceUnavailableException`, а `@RestControllerAdvice` в другом пакете и не сканируется Spring — исключение не будет обработано, вернётся 500.

Проверьте, что `@RestControllerAdvice` **в пакете ниже или равном** главному классу приложения:

```java
// dev.folomkin.tasks.TasksApplication         ← главный класс
// dev.folomkin.tasks.exception.GlobalExceptionHandler  ← OK, ниже
// dev.folomkin.other.ExceptionHandler         ← ❌ не будет найден
```

## 📋 Что прислать для точного диагноза

1. **Строку из логов** с `Fallback` или `NoSuchMethod`:
   ```bash
   kubectl logs deploy/tasks-service -c tasks-service | grep -i "fallback\|nosuchmethod\|IllegalState" | tail -20
   ```

2. **Код метода с `@CircuitBreaker`** и **fallback-метод** — полные сигнатуры.

3. **Код `@RestControllerAdvice`** — есть ли там обработка `ServiceUnavailableException`.

4. **Класс `ServiceUnavailableException`** — где он находится, `RuntimeException` ли наследник.

## 💡 Как быстро проверить, вызывается ли fallback

Добавьте в fallback явный лог **в самом начале**:

```java
public Task fallbackCreateTask(Task task, Throwable t) {
    System.out.println("=== FALLBACK CALLED ===");   // ← если этого нет в логах — fallback не вызывается
    log.error("Fallback triggered", t);
    throw new ServiceUnavailableException("...");
}
```

**Если в логах нет `=== FALLBACK CALLED ===`** — Resilience4j не может вызвать fallback, проблема в сигнатуре.

**Если есть** — fallback вызывается, но `ServiceUnavailableException` не обрабатывается, и Spring возвращает 500.

## 🎯 Наиболее вероятный сценарий

Учитывая, что `CircuitBreaker` **меняет статус на OPEN** — Resilience4j **работает**, ловит исключения, но fallback **не находит**. Это **сигнатура**.

Проверьте по чек-листу:

- [ ] Оригинал и fallback **в одном классе**
- [ ] Возвращаемый тип совпадает (`Task` / `Task`)
- [ ] Аргументы совпадают + `Throwable` последний
- [ ] Метод не `static`
- [ ] `fallbackMethod = "..."` — имя **точно** совпадает с именем метода (без опечаток)

## 💎 Итог

| Симптом | Причина | Решение |
|---|---|---|
| Статус меняется на OPEN | CircuitBreaker работает | ✅ |
| Fallback не вызывается | Сигнатура не совпадает | Проверить правила выше |
| 500 вместо 503 | Fallback бросает, но `@ControllerAdvice` не ловит | Проверить пакет `@RestControllerAdvice` |
| «Простыня» в логах | `NoSuchMethodException` от Resilience4j | Найти строку с `Fallback method ... not found` |

**Сначала найдите в логах строку `Fallback method ... not found`** — это 100% диагноз. Пришлите её — и я скажу, что именно поправить в сигнатуре.

> Fallback method ... not found - такого нет текста. 
вот код сервиса

>    @CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    @Retry(name = "mongoDb")
    @TimeLimiter(name = "mongoDb")
    public Tasks createTask(Tasks task) {
        Tasks t = taskRepository.save(task);
        producer.sendTaskCreatedEvent(
                new TaskEvent(
                        task.getId(),
                        task.getTitle(),
                        "USER abc123",
                        "CREATED")
        );
        return t;
    }

>    public Tasks fallbackCreateTask(Tasks task, Throwable t) throws ServiceUnavailableException {
        System.out.println("=== FALLBACK CALLED ===");   // ← если этого нет в логах — fallback не вызывается
        log.error("MongoDB недоступна, fallback triggered", t);
        throw new ServiceUnavailableException("Сервис временно недоступен, попробуйте позже");
    }

> вот код исключения:

> @RestControllerAdvice
public class GlobalExceptionHandler extends RuntimeException {

>    @ExceptionHandler(ServiceUnavailableException.class)
    public ResponseEntity<Map<String, String>> handleServiceUnavailable(ServiceUnavailableException e) {
        return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)  // 503
                .body(Map.of(
                        "error", "service_unavailable",
                        "message", e.getMessage()
                ));
    }
}

> а ничего что я еще активировал DestinationRule для этого же сервиса?

Нашёл сразу две серьёзные проблемы в вашем коде — обе объясняют поведение. DestinationRule тут ни при чём, но разберём и его в конце.

## ❌ Проблема 1: `@TimeLimiter` не работает с синхронными методами

Это **главная причина** «простыни» в логах и отсутствия fallback.

`@TimeLimiter` в Resilience4j **требует**, чтобы метод возвращал `CompletableFuture<T>`, `Mono<T>` или `Flux<T>`. Ваш метод возвращает **обычный `Tasks`** — синхронный. При старте или при первом вызове Resilience4j бросает исключение вроде:

```
TimeLimiter is only supported on methods returning CompletionStage or CompletableFuture
```

или

```
java.lang.IllegalStateException: TimeLimiter 'mongoDb' does not support return type 'Tasks'
```

Эта ошибка **вылетает из AOP-цепочки** и превращается в 500 в Postman. До вызова вашего fallback дело не доходит — исключение происходит **до** того, как CircuitBreaker обработает результат.

### Как исправить

**Вариант A (быстрый):** Уберите `@TimeLimiter`. Он не нужен для синхронного метода — таймаут уже настроен на уровне MongoDB-клиента и Istio.

```java
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
@Retry(name = "mongoDb")
public Tasks createTask(Tasks task) {
    Tasks t = taskRepository.save(task);
    producer.sendTaskCreatedEvent(...);
    return t;
}
```

**Вариант B (если нужен TimeLimiter):** Сделайте метод асинхронным:

```java
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTaskAsync")
@Retry(name = "mongoDb")
@TimeLimiter(name = "mongoDb")
public CompletableFuture<Tasks> createTask(Tasks task) {
    return CompletableFuture.supplyAsync(() -> {
        Tasks t = taskRepository.save(task);
        producer.sendTaskCreatedEvent(...);
        return t;
    });
}

public CompletableFuture<Tasks> fallbackCreateTaskAsync(Tasks task, Throwable t) {
    return CompletableFuture.failedFuture(
        new ServiceUnavailableException("MongoDB недоступна")
    );
}
```

**Для учебного проекта рекомендую Вариант A** — просто уберите `@TimeLimiter`. Он нужен только для асинхронных операций.

## ❌ Проблема 2: `GlobalExceptionHandler extends RuntimeException`

Это **семантическая ошибка**, и хотя она может не ломать обработку, она точно неправильная:

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends RuntimeException {   // ← вот здесь
```

**Почему плохо:**

1. `@RestControllerAdvice` — это **обработчик** исключений, а не само исключение. Он не должен ничего наследовать.
2. Если он случайно попадёт куда-то как исключение — введёт в заблуждение.
3. Spring может попытаться его как-то иначе обработать — вплоть до неожиданного поведения.

**Правильно:**

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {   // ← без extends

    @ExceptionHandler(ServiceUnavailableException.class)
    public ResponseEntity<Map<String, String>> handleServiceUnavailable(ServiceUnavailableException e) {
        log.error("Service unavailable: {}", e.getMessage());
        return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body(Map.of(
                        "error", "service_unavailable",
                        "message", e.getMessage()
                ));
    }
}
```

Также убедитесь, что `ServiceUnavailableException` **сам** extends `RuntimeException`:

```java
public class ServiceUnavailableException extends RuntimeException {
    public ServiceUnavailableException(String message) {
        super(message);
    }
}
```

## 🎯 Наиболее вероятная картина

Учитывая, что CircuitBreaker **меняет статус на OPEN** (значит, ошибки считаются), но fallback **не вызывается** — я вижу такую цепочку:

1. `@TimeLimiter` бросает `IllegalStateException` **до** вызова `createTask`.
2. `@Retry` пытается 3 раза → все 3 попытки падают с тем же `IllegalStateException`.
3. `@CircuitBreaker` считает 3 ошибки → через 5 вызовов (по вашей настройке) переходит в `OPEN`.
4. Fallback **не вызывается**, потому что ошибка происходит не в `createTask`, а **в аспекте `@TimeLimiter`** — Resilience4j не может её перехватить и вызвать fallback.

## 🛠️ Пошаговое исправление

### Шаг 1. Уберите `@TimeLimiter` из метода

```java
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
@Retry(name = "mongoDb")
public Tasks createTask(Tasks task) {
    Tasks t = taskRepository.save(task);
    producer.sendTaskCreatedEvent(...);
    return t;
}
```

И **уберите** секцию `timelimiter` из `application.yaml`, если она там есть.

### Шаг 2. Исправьте `GlobalExceptionHandler`

Уберите `extends RuntimeException`.

### Шаг 3. Убедитесь, что `ServiceUnavailableException` в правильном пакете

`@RestControllerAdvice` сканирует **подпакеты** главного класса приложения:

```
dev.folomkin.tasks.TasksApplication        ← главный
dev.folomkin.tasks.exception.ServiceUnavailableException  ← OK
dev.folomkin.tasks.exception.GlobalExceptionHandler        ← OK
dev.folomkin.other.ExceptionHandler        ← ❌ не найдёт
```

### Шаг 4. Пересоберите и перезапустите

```bash
docker build -t myorg/tasks-service:1.0 .
minikube image load myorg/tasks-service:1.0
kubectl rollout restart deploy/tasks-service
```

### Шаг 5. Проверьте

```bash
# Логи при старте — ищите ошибки про TimeLimiter
kubectl logs deploy/tasks-service -c tasks-service | grep -i "timelimiter\|not supported\|IllegalState"

# Запрос в Postman
# Ожидаемо: 503 с {"error":"service_unavailable","message":"..."}
```

## 📌 Про DestinationRule

**DestinationRule не влияет** на вызов fallback. Он работает на уровне Envoy между сервисами и не видит, что происходит внутри JVM. Единственное, где он может «вмешаться» — если вы смотрите ответ в Postman **через Istio Ingress**. Но даже в этом случае DestinationRule не блокирует запрос, а лишь решает, на какой **под** его отправить.

**Проверить просто:** сделайте `port-forward` напрямую на под `tasks-service` и обратитесь к нему **минуя Istio**:

```bash
kubectl port-forward deploy/tasks-service 8080:8091
curl -X POST http://localhost:8080/tasks/create -H "Content-Type: application/json" -d '{"title":"test"}'
```

Если поведение то же (500, fallback не вызывается) — DestinationRule не при чём, проблема в коде.

## 💎 Итог

| Проблема | Симптом | Решение |
|---|---|---|
| `@TimeLimiter` с синхронным методом | «Простыня» в логах, fallback не вызывается | Убрать `@TimeLimiter` (или сделать метод async) |
| `GlobalExceptionHandler extends RuntimeException` | Может ломать обработку | Убрать `extends RuntimeException` |
| Проверить пакет | `@RestControllerAdvice` не видит исключение | Поместить в пакет под главным классом |
| DestinationRule | Не влияет | Проверить через `port-forward` без Istio |

**Начните с удаления `@TimeLimiter`** — это самая вероятная причина. После этого fallback должен сработать, и Postman вернёт `503 Service Unavailable` вместо 500.

Если после этого всё ещё не работает — пришлите **первые 10 строк** исключения из логов (не всю простыню, а самое начало, где `Caused by:` или имя главного исключения). Там будет точная причина.

> теперь получается вот такая картина: запросы в postman долгие - каждый секунд по 30 и все равно все 500, но на 5 запрос меняется статус CircuitBreaker на OPEN и в логах вижу текст из fallback вот отсюда :     
   throw new ServiceUnavailableException("Сервис временно недоступен, попробуйте позже");
вот такой:
javax.naming.ServiceUnavailableException: Сервис временно недоступен, попробуйте позже
то есть ServiceUnavailableException выводится в логи, но не в ответ в postman. не то чтобы я придираюсь, но на проде никто же не будет искать ответ в логах - нужно показывать в понятном формате. 
плюс post запросы очень долгие - секунд 30 каждый

Отличная находка в логах! Вы **сами нашли** корень проблемы, просто не заметили. Смотрите внимательно на строку:

```
javax.naming.ServiceUnavailableException
      ^^^^^^
```

Это **не ваше исключение**! Это `javax.naming.ServiceUnavailableException` — класс из JNDI (Java Naming and Directory Interface), а не ваш собственный. Скорее всего, IDE автоподставила импорт при написании кода, и вы бросили совсем не то исключение, которое создали.

## ❌ Проблема 1: Неправильный импорт

**Где-то в коде** у вас:

```java
import javax.naming.ServiceUnavailableException;   // ← вот эта строка
```

Вместо:

```java
import dev.folomkin.tasks.exception.ServiceUnavailableException;
```

### Как это ломает всё

1. Fallback бросает **`javax.naming.ServiceUnavailableException`** — checked exception, наследник `NamingException`.
2. Ваш `GlobalExceptionHandler` ловит **`dev.folomkin.tasks.exception.ServiceUnavailableException`** — совсем другой класс.
3. Ни один `@ExceptionHandler` не срабатывает → Spring возвращает **500** с дефолтным ответом.

### Как исправить

Найдите в IDE импорт `javax.naming.ServiceUnavailableException` и замените на свой:

```java
// ❌ Было
import javax.naming.ServiceUnavailableException;

// ✅ Стало
import dev.folomkin.tasks.exception.ServiceUnavailableException;
```

**Как найти:** в IntelliJ IDEA нажмите `Ctrl+Shift+F` (или `Cmd+Shift+F` на Mac) и поищите `javax.naming.ServiceUnavailableException` по всему проекту. Замените везде, где встречается.

Проверьте заодно, что ваш `ServiceUnavailableException` extends `RuntimeException`:

```java
package dev.folomkin.tasks.exception;

public class ServiceUnavailableException extends RuntimeException {
    public ServiceUnavailableException(String message) {
        super(message);
    }
}
```

**Почему `RuntimeException`, а не checked?** Checked исключения через Spring AOP работают плохо, и Resilience4j их не всегда корректно записывает. Для бизнес-исключений всегда используйте `RuntimeException`.

## ⏱️ Проблема 2: 30 секунд на запрос

Это **дефолтный таймаут MongoDB**. Когда под MongoDB удалён, Kubernetes Service не имеет endpoints — пакеты «уходят в никуда», и клиент ждёт, пока истечёт `serverSelectionTimeoutMS` (по умолчанию **30 секунд**). Плюс `@Retry` может добавить ещё попытки.

### Как исправить

**Шаг 1.** Уменьшите таймауты в connection string:

```yaml
# config-repo/tasks-service.yaml
spring:
  mongodb:
    uri: ${MONGO_URI:mongodb://tasks-mongodb-service:27017/tasksdb}?serverSelectionTimeoutMS=2000&connectTimeoutMS=2000&socketTimeoutMS=5000
```

Или в Secret, если URI хранится там:

```
mongodb://rootadmin:...@tasks-mongodb-service:27017/tasksdb?authSource=admin&serverSelectionTimeoutMS=2000&connectTimeoutMS=2000&socketTimeoutMS=5000
```

**Разбор параметров:**

| Параметр | Значение | Что делает |
|---|---|---|
| `serverSelectionTimeoutMS` | 2000 | Сколько ждать, пока драйвер найдёт доступный сервер |
| `connectTimeoutMS` | 2000 | Таймаут TCP-подключения |
| `socketTimeoutMS` | 5000 | Таймаут чтения/записи |

После этих правок запрос будет падать **за 2 секунды** вместо 30.

**Шаг 2.** Проверьте `@Retry`. Если `maxAttempts: 3` — попытки накладываются:

```
Одна попытка: 2 сек
3 попытки + backoff: 2 + 0.5 + 2 + 1 + 2 = ~7.5 сек
```

Уменьшите `maxAttempts` до 2 или `waitDuration` до 100ms для учебного проекта.

**Шаг 3.** Уберите `@TimeLimiter`, если ещё не убрали — он не работает с синхронными методами.

## 🎯 Полная картина после исправления

```
POST /tasks/create
       │
       ▼
@CircuitBreaker (OPEN) ────▶ не вызывает createTask
       │
       ▼
fallbackCreateTask ────────▶ бросает dev.folomkin.tasks.exception.ServiceUnavailableException
       │                              (свой, RuntimeException)
       ▼
GlobalExceptionHandler ────▶ ловит его
       │
       ▼
ResponseEntity 503 ────────▶ {"error": "service_unavailable", "message": "..."}
       │
       ▼
Postman ────────────────────▶ 503 Service Unavailable
```

**Время ответа:** ~2 сек (таймаут MongoDB), или мгновенно, если CircuitBreaker уже `OPEN`.

## 📋 Пошаговый план

1. **Найдите и замените импорт** `javax.naming.ServiceUnavailableException` на свой.
2. **Убедитесь, что ваш класс** `ServiceUnavailableException` extends `RuntimeException`.
3. **Добавьте таймауты** в connection string MongoDB: `serverSelectionTimeoutMS=2000&connectTimeoutMS=2000&socketTimeoutMS=5000`.
4. **Уменьшите Retry**: `maxAttempts: 2`, `waitDuration: 100ms`.
5. **Уберите `@TimeLimiter`**, если ещё там.
6. **Пересоберите и перезапустите**:
   ```bash
   docker build -t myorg/tasks-service:1.0 .
   minikube image load myorg/tasks-service:1.0
   kubectl rollout restart deploy/tasks-service
   ```
7. **Проверьте в Postman:**
   - Запросы должны стать быстрыми (~2 сек до fallback).
   - Первые 5 запросов → 503 с JSON от fallback (после исправления импорта).
   - 6-й и последующие → **мгновенный** 503 (CircuitBreaker OPEN).

## 💡 Как понять, что вы всё сделали правильно

**Логи при запросе (MongoDB недоступна):**
```
ERROR ... MongoDB недоступна, fallback triggered
   ...
   Caused by: dev.folomkin.tasks.exception.ServiceUnavailableException
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                вот это — ваше исключение, а не javax.naming
```

**Ответ в Postman:**
```
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "error": "service_unavailable",
  "message": "Сервис временно недоступен, попробуйте позже"
}
```

## 📌 Про IDE автоподстановку

Это **очень частая** ошибка. Когда вы пишете `ServiceUnavailableException`, IDE предлагает первый подходящий по имени класс — а им оказывается `javax.naming`. Всегда проверяйте импорт после того, как IDE что-то подставила. В IntelliJ можно включить настройку **"Optimize imports on the fly"** — она убирает неиспользуемые и неоднозначные импорты.

Ещё один способ защиты — **называть свои исключения уникально**, например `TasksServiceUnavailableException` или `MongoUnavailableException`. Тогда IDE не будет путать с чужими классами.

## 💎 Итог

| Проблема | Причина | Решение |
|---|---|---|
| 500 вместо 503 | Fallback бросает `javax.naming.ServiceUnavailableException` вместо своего | Заменить импорт |
| 30 секунд на запрос | Дефолтный `serverSelectionTimeoutMS=30000` в MongoDB | Уменьшить до 2000ms |
| Retry добавляет задержку | `maxAttempts: 3` + backoff | Уменьшить до 2 или убрать |
| Fallback не вызывается | `@TimeLimiter` на синхронном методе | Убрать |

**Начните с поиска `javax.naming` в проекте** — это ключевая правка. После неё fallback начнёт отдавать 503 в Postman, а таймауты MongoDB сократят ответ до 2 секунд.

> теперь вот что получается: я починил исключение ServiceUnavailableException,  обновил репозиторий с конфигом подключения к mongodb, добавив параметры уменьшающие таймаут запроса, перезагрузил config-server, пересобрал tasks-service и теперь ServiceUnavailableException выводит текст какой нужно в postman, но с первого раза. статус circuitbreacker меняется на 5 запрос, а сообщение из ServiceUnavailableException выбрасывается в postman с первого запроса, плюс длительность запроса так же долгая

На самом деле **всё работает правильно** — просто вы неверно интерпретируете поведение fallback. Разберу по пунктам.

## ✅ Пункт 1: Fallback с первого запроса — это НОРМАЛЬНО

Это **ключевой момент**, который многие путают. Fallback вызывается в **двух разных случаях**:

| Ситуация | Когда | Что происходит |
|---|---|---|
| **Метод бросил исключение** | CircuitBreaker `CLOSED` | Метод **вызывается**, падает → fallback ловит ошибку |
| **CircuitBreaker `OPEN`** | Цепь разомкнута | Метод **не вызывается** → сразу fallback |

**То есть fallback — это реакция на ошибку, а не только на `OPEN`.** Даже когда цепь `CLOSED` (замкнута), ошибка метода приводит к fallback. Это **правильное** поведение.

Как работает ваш сценарий:

```
Запрос 1: CB=CLOSED → createTask() → MongoDB недоступна → Exception → fallback → 503
Запрос 2: CB=CLOSED → createTask() → MongoDB недоступна → Exception → fallback → 503
Запрос 3: CB=CLOSED → ... → fallback → 503
Запрос 4: CB=CLOSED → ... → fallback → 503
Запрос 5: CB=CLOSED → ... → fallback → 503  (набралось 5 ошибок → OPEN)
Запрос 6: CB=OPEN   → createTask() НЕ вызывается → сразу fallback → 503 (мгновенно!)
```

**Поэтому fallback с первого запроса — это правильно.** Ваш CircuitBreaker работает как надо. Разница между `CLOSED` и `OPEN` в **скорости ответа** и в том, **вызывается ли метод вообще**.

## 🎯 Проверьте: на 6-й запрос fallback должен сработать **мгновенно**

Если первые 5 запросов идут ~2 секунды (ждём таймаут MongoDB), а 6-й — **мгновенно** (потому что CB=OPEN и метод не вызывается) — значит, всё работает идеально.

Проверьте так:

```bash
# Отправьте 7 запросов подряд с замером времени
for i in $(seq 1 7); do
  echo -n "Request $i: "
  curl -s -o /dev/null -w "%{http_code} (%{time_total}s)\n" \
    -X POST http://<ingress-address>/tasks/create \
    -H "Content-Type: application/json" \
    -d '{"title":"test"}'
done
```

**Ожидаемый результат:**
```
Request 1: 503 (2.005s)
Request 2: 503 (2.003s)
Request 3: 503 (2.001s)
Request 4: 503 (2.004s)
Request 5: 503 (2.002s)
Request 6: 503 (0.005s)   ← мгновенно, CB=OPEN
Request 7: 503 (0.003s)   ← мгновенно
```

Если 6-й и 7-й запросы **мгновенные** — CircuitBreaker работает корректно.

## ⏱️ Пункт 2: Долгая длительность — таймауты MongoDB не применились

Если **все** запросы идут 30 секунд (а не 2 секунды), значит параметры `serverSelectionTimeoutMS=2000` из `config-repo` **не дошли** до приложения.

### Проверьте, что реально в поде

```bash
# 1. Что отдаёт Config Server
curl http://localhost:8888/tasks-service/default | jq '.propertySources[].source | select(."spring.mongodb.uri")'

# 2. Что реально в переменной MONGO_URI внутри пода
kubectl exec deploy/tasks-service -c tasks-service -- env | grep MONGO_URI

# 3. Что видит actuator
curl http://localhost:8080/actuator/env | jq '.propertySources[] | select(.properties["spring.mongodb.uri"]) | .properties["spring.mongodb.uri"].value'
```

**Возможные причины:**

**Причина A: `MONGO_URI` перебивает конфиг из Git**

В вашем `tasks-service-deployment.yaml` есть:
```yaml
env:
  - name: MONGO_URI
    valueFrom:
      secretKeyRef:
        name: mongodb-credentials
        key: MONGO_URI
```

Этот URI — из Secret, **без таймаутов**. А в `config-repo` вы прописали:
```yaml
uri: ${MONGO_URI:mongodb://tasks-mongodb-service:27017/tasksdb}
```

Плейсхолдер `${MONGO_URI:...}` берёт значение **из переменной окружения** — то есть из Secret. Значит, таймауты, которые вы дописали в `config-repo`, **игнорируются**.

**Решение:** добавьте таймауты **в сам Secret**:

```bash
kubectl create secret generic mongodb-credentials \
  --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin&serverSelectionTimeoutMS=2000&connectTimeoutMS=2000&socketTimeoutMS=5000' \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deploy/tasks-service
```

**Причина B: Под не перезапущен после изменения Secret**

Secret обновляется **мгновенно**, но под получает новое значение переменной окружения **только при перезапуске**. Проверьте:

```bash
kubectl get pod -l app=tasks-service
# AGE должен быть меньше, чем время изменения Secret
```

Если под старый — перезапустите:
```bash
kubectl rollout restart deploy/tasks-service
```

**Причина C: Правильный префикс**

Учитывая, что в Spring Boot 4 вы обнаружили изменение `spring.data.mongodb` → `spring.mongodb`, параметры URI работают на уровне драйвера MongoDB. Таймауты в connection string **не зависят от префикса Spring** — они читаются драйвером напрямую. Так что здесь проблема не в этом.

## 📊 Полная диагностика долгого запроса

```bash
# 1. Проверьте, что в env реально есть таймауты
kubectl exec deploy/tasks-service -c tasks-service -- env | grep MONGO_URI
# Ожидаемо: MONGO_URI=mongodb://...?authSource=admin&serverSelectionTimeoutMS=2000...

# 2. Проверьте, что под перезапущен после правок
kubectl get pods -l app=tasks-service

# 3. Проверьте, что Config Server отдаёт правильное
curl http://localhost:8888/tasks-service/default | jq

# 4. Отправьте 7 запросов с замером времени
for i in $(seq 1 7); do
  echo -n "Request $i: "
  curl -s -o /dev/null -w "%{http_code} (%{time_total}s)\n" \
    -X POST http://localhost:8080/tasks/create \
    -H "Content-Type: application/json" \
    -d '{"title":"test"}'
done
```

## 💡 Что делать

1. **Обновите Secret** с таймаутами:
   ```bash
   kubectl create secret generic mongodb-credentials \
     --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin&serverSelectionTimeoutMS=2000&connectTimeoutMS=2000&socketTimeoutMS=5000' \
     --dry-run=client -o yaml | kubectl apply -f -
   ```

2. **Перезапустите под**:
   ```bash
   kubectl rollout restart deploy/tasks-service
   kubectl rollout status deploy/tasks-service
   ```

3. **Проверьте env в поде**:
   ```bash
   kubectl exec deploy/tasks-service -c tasks-service -- env | grep MONGO_URI
   ```
   Должны увидеть таймауты в конце.

4. **Отправьте 7 запросов** и убедитесь, что:
   - Первые 5: ~2 сек, 503
   - Следующие: ~0 сек, 503

## 📌 Резюме

| Наблюдение | Правильно? | Почему |
|---|---|---|
| Fallback с первого запроса | ✅ Да | Fallback = реакция на ошибку, не только на OPEN |
| CB становится OPEN на 5-й | ✅ Да | `minimumNumberOfCalls: 5` |
| Все запросы 30 секунд | ❌ Нет | Таймауты MongoDB не применились |
| 503 в Postman | ✅ Да | Правильно |

**Ключевое:** fallback работает **всегда при ошибке**, а не только при `OPEN`. Разница в том, что при `CLOSED` метод **вызывается** (и падает через 2 сек), а при `OPEN` — **не вызывается** (ответ мгновенный).

**Единственная проблема** — таймауты MongoDB. Скорее всего, они не применились, потому что `MONGO_URI` из Secret перебивает конфиг из Config Server. Обновите Secret — и запросы станут быстрыми.

Пришлите вывод команд из диагностики выше — и я скажу точно, куда пропали таймауты.

> все заработало, как только я обновил secret. теперь запросы стали короткие - 2-3 секунды. теперь я вижу такую картину: делаю 5 запросов: на 5й меняется статус на OPEN и 6й запрос отвечает мгновенно, затем статус меняется на HALF_OPEN - я делаю вроде 3 запроса и они опять по 2-3 секунды и следующий запрос меняет статус на OPEN и отвечает опять мгновенно.

Это **идеальное поведение** CircuitBreaker — вы наблюдаете ровно ту state machine, которая описана в документации. Давайте разберём, что происходит, и почему это правильно.

## 🔄 State machine CircuitBreaker

```
       ┌──────────────────────────────────────────┐
       │                                          │
       │  CLOSED (замкнут)                        │
       │  Метод вызывается                        │
       │  Ошибки считаются                        │
       │                                          │
       └──────────────┬───────────────────────────┘
                      │ failureRate >= 50%
                      │ (5 ошибок из 5+ вызовов)
                      ▼
       ┌──────────────────────────────────────────┐
       │  OPEN (разомкнут)                        │
       │  Метод НЕ вызывается                     │
       │  Все запросы → fallback мгновенно        │
       │  Ждём waitDurationInOpenState (10s)      │
       └──────────────┬───────────────────────────┘
                      │ прошло 10 секунд
                      ▼
       ┌──────────────────────────────────────────┐
       │  HALF_OPEN (полуоткрыт)                  │
       │  Пропускаем permittedNumberOfCalls (3)   │
       │  запроса для проверки                    │
       │                                          │
       └──┬─────────────────────────────────┬─────┘
          │                                 │
          │ все 3 успешны                   │ хоть один failed
          │                                 │
          ▼                                 ▼
       CLOSED                            OPEN (снова)
```

## 🎯 Что вы наблюдаете — пошагово

**Фаза 1: CLOSED**

1. Запрос 1 → метод вызывается → 2-3 сек → ошибка → fallback → 503
2. Запрос 2 → то же самое
3. Запрос 3 → то же самое
4. Запрос 4 → то же самое
5. Запрос 5 → то же самое, **счётчик ошибок = 5** → переход в **OPEN**

**Фаза 2: OPEN**

6. Запрос 6 → метод **не вызывается** → fallback **мгновенно** → 503
7. Запросы 7, 8, 9… → то же самое, пока не пройдёт 10 секунд

**Фаза 3: HALF_OPEN** (через 10 секунд)

8. Запрос N → метод **вызывается** (пробный) → 2-3 сек → ошибка
9. Запрос N+1 → то же самое
10. Запрос N+2 → то же самое, **3 пробных вызова, все failed** → снова **OPEN**

**Фаза 4: OPEN** — опять мгновенные fallback на 10 секунд.

**Цикл повторяется.**

## ✅ Почему это правильно

Ваша MongoDB **действительно недоступна**. CircuitBreaker делает именно то, что должен:

- **Не долбит мёртвую БД** — пока OPEN, запросы идут в fallback мгновенно, не тратя 2-3 секунды на таймауты.
- **Периодически проверяет** — раз в 10 секунд пропускает 3 запроса, чтобы понять, ожила ли БД.
- **Возвращается в CLOSED**, когда БД восстановится (проверьте — поднимите MongoDB и увидите, как после 3 успешных HALF_OPEN цепь замкнётся).

## 🎛️ Как настроить под себя

Все параметры, которые влияют на это поведение:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        slidingWindowSize: 10                    # размер окна
        minimumNumberOfCalls: 5                  # минимум вызовов до расчёта
        failureRateThreshold: 50                 # % ошибок для размыкания
        waitDurationInOpenState: 10s             # сколько сидеть в OPEN
        permittedNumberOfCallsInHalfOpenState: 3 # сколько пробных в HALF_OPEN
        automaticTransitionFromOpenToHalfOpenEnabled: true  # авто-переход
```

**Что менять:**

| Параметр | Увеличить → | Уменьшить → |
|---|---|---|
| `waitDurationInOpenState` | Реже пробует, дольше «терпит» | Чаще пробует, быстрее восстанавливается |
| `permittedNumberOfCallsInHalfOpenState` | Больше уверенности перед CLOSED | Быстрее возвращается в OPEN при проблемах |
| `slidingWindowSize` | Более «гладкая» статистика | Реакция на свежие ошибки |
| `minimumNumberOfCalls` | Позже размыкается | Раньше размыкается |

## 📊 Что смотреть в actuator

```bash
curl http://localhost:8080/actuator/circuitbreakers | jq
```

Вы увидите:
```json
{
  "circuitBreakers": {
    "mongoDb": {
      "state": "HALF_OPEN",
      "failureRate": "100.0%",
      "bufferedCalls": 3,
      "failedCalls": 3,
      "successfulCalls": 0,
      "notPermittedCalls": 12
    }
  }
}
```

**Полезные счётчики:**
- `notPermittedCalls` — сколько запросов **не дошло** до метода (потому что OPEN). Растёт, когда CB в OPEN.
- `failedCalls` / `successfulCalls` — статистика текущего окна.
- `failureRate` — процент ошибок.

## 🔍 Проверьте события

```bash
curl http://localhost:8080/actuator/circuitbreakerevents/mongoDb | jq
```

Вы увидите историю переходов:
```json
{
  "circuitBreakerEvents": [
    {"type": "STATE_TRANSITION", "stateTransition": "CLOSED_TO_OPEN"},
    {"type": "STATE_TRANSITION", "stateTransition": "OPEN_TO_HALF_OPEN"},
    {"type": "STATE_TRANSITION", "stateTransition": "HALF_OPEN_TO_OPEN"},
    {"type": "STATE_TRANSITION", "stateTransition": "HALF_OPEN_TO_OPEN"}
  ]
}
```

Это **прямое доказательство**, что state machine работает корректно.

## 🧪 Как проверить, что всё восстановится

Поднимите MongoDB:

```bash
kubectl scale deploy tasks-mongodb --replicas=1
```

Дождитесь, пока MongoDB поднимется:
```bash
kubectl get pods -l app=tasks-mongodb -w
```

Затем отправьте запрос. Через `waitDurationInOpenState` CB перейдёт в `HALF_OPEN`, пропустит 3 запроса — они теперь **успешны** — и CB вернётся в `CLOSED`.

**Ожидаемая последовательность событий:**
```
CLOSED → OPEN (при падении MongoDB)
OPEN → HALF_OPEN (через 10 сек)
HALF_OPEN → CLOSED (после 3 успешных запросов)
```

## 💡 Тонкая настройка для production

Для реального проекта значения по умолчанию слишком «агрессивны»:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      mongoDb:
        slidingWindowSize: 100              # больше окно = стабильнее
        minimumNumberOfCalls: 20            # не размыкаться на 5 ошибках
        failureRateThreshold: 30            # 30% ошибок — порог
        slowCallRateThreshold: 50           # если 50% вызовов медленные
        slowCallDurationThreshold: 2s       # что считать "медленным"
        waitDurationInOpenState: 30s        # дольше ждать
        permittedNumberOfCallsInHalfOpenState: 5
```

**Плюс `slowCallRateThreshold`** — это отдельный порог для «медленных» вызовов. Если MongoDB начинает тормозить (не падает, но отвечает 5 секунд), CB тоже разомкнётся. Это **важнее**, чем счётчик ошибок — потому что медленная БД губит сервис сильнее, чем мёртвая.

## 💎 Итог

| Что вы видите | Что это значит |
|---|---|
| 5 запросов с ошибками → OPEN | ✅ `minimumNumberOfCalls: 5` сработал |
| OPEN → мгновенный fallback | ✅ Метод не вызывается, экономия 2-3 сек |
| Через 10 сек → HALF_OPEN | ✅ `waitDurationInOpenState: 10s` истёк |
| 3 пробных → все failed → OPEN | ✅ `permittedNumberOfCallsInHalfOpenState: 3` |
| Цикл повторяется | ✅ Это state machine в действии |

**Всё работает идеально.** Вы наблюдаете классическое поведение CircuitBreaker, которое описано в книгах по отказоустойчивости (Michael Nygard «Release It!»).

**Единственное, что стоит поправить** для production — увеличить `slidingWindowSize` и `minimumNumberOfCalls`, добавить `slowCallRateThreshold`. Текущие значения (`5/50%`) хороши для **демонстрации**, но в реальной жизни сервис может «мигать» на временных сбоях. С окном 100 вызовов и порогом 20 — CircuitBreaker будет реагировать на **устойчивые** проблемы, а не на случайные всплески.

Поздравляю — вы полностью разобрались с CircuitBreaker на обоих уровнях (Istio + Resilience4j). Это очень сильный навык для резюме.

> а что делать с параметрами в урле mongodb?

Параметры в URI MongoDB, которые вы добавили (`serverSelectionTimeoutMS`, `connectTimeoutMS`, `socketTimeoutMS`), — это **часть строки подключения**, которую драйвер MongoDB читает напрямую. С ними можно работать несколькими способами, и выбор зависит от того, что для вас важнее: безопасность или гибкость.

### 🎯 Где могут жить параметры URI

Сейчас у вас параметры лежат в **Kubernetes Secret** (`mongodb-credentials`). Это правильное место для **учётных данных** (логин/пароль), но таймауты — это **не секрет**, а операционная настройка. Их логичнее держать в **Config Server** или в **`application.yaml`** приложения.

Вот три подхода, от самого простого до самого гибкого.

### Вариант 1: Оставить всё в Secret (как сейчас)

Вы уже так сделали, и это работает. Плюс — в одном месте. Минус — чтобы поменять таймаут, нужно обновлять Secret и перезапускать под. Для учебного проекта это нормально.

### Вариант 2: Разделить — credentials в Secret, таймауты в Config Server

Это более «правильный» подход. **Логин и пароль** остаются в Secret (их нельзя в Git), а **таймауты** выносятся в `config-repo/tasks-service.yaml`, который версионируется и не требует `kubectl` для правки.

**В Secret** остаётся только базовый URI с credentials:
```
mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin
```

**В `config-repo/tasks-service.yaml`** добавляете параметры через placeholder:
```yaml
spring:
  mongodb:
    uri: ${MONGO_URI:mongodb://tasks-mongodb-service:27017/tasksdb}?serverSelectionTimeoutMS=2000&connectTimeoutMS=2000&socketTimeoutMS=5000
```

Проблема: `MONGO_URI` из Secret **уже содержит** `?authSource=admin`. Если дописать `?...` второй раз, получится невалидный URI (`?` встречается дважды). Поэтому нужно либо убрать `authSource` из Secret и перенести в Config Server, либо использовать другой подход.

### Вариант 3: Использовать отдельные свойства Spring Boot (рекомендую)

Spring Boot позволяет задавать таймауты **отдельными свойствами**, а не через URI. Тогда URI в Secret остаётся «чистым» (только credentials и `authSource`), а таймауты живут отдельно — в Config Server или в `application.yaml`.

Добавьте в `config-repo/tasks-service.yaml`:
```yaml
spring:
  mongodb:
    uri: ${MONGO_URI:mongodb://tasks-mongodb-service:27017/tasksdb}
    # Таймауты — отдельными свойствами
    connection-timeout: 2000ms
    # Для serverSelectionTimeout и socketTimeout нужен кастомный бин (см. ниже)
```

Но, к сожалению, Spring Boot **не предоставляет** отдельных свойств для `serverSelectionTimeoutMS` и `socketTimeoutMS` в стандартной автоконфигурации. Есть только `spring.mongodb.connection-timeout` (для `connectTimeoutMS`). Для остальных параметров нужно создавать кастомный `MongoClient` с `MongoClientSettings`.

**Пример кастомного бина:**
```java
@Configuration
public class MongoConfig {

    @Value("${spring.mongodb.uri}")
    private String uri;

    @Value("${app.mongodb.server-selection-timeout:2000}")
    private int serverSelectionTimeout;

    @Value("${app.mongodb.socket-timeout:5000}")
    private int socketTimeout;

    @Bean
    public MongoClient mongoClient() {
        return MongoClients.create(
            MongoClientSettings.builder()
                .applyConnectionString(new ConnectionString(uri))
                .applyToClusterSettings(b -> b.serverSelectionTimeout(serverSelectionTimeout, TimeUnit.MILLISECONDS))
                .applyToSocketSettings(b -> b.readTimeout(socketTimeout, TimeUnit.MILLISECONDS))
                .build()
        );
    }
}
```

Тогда в `config-repo`:
```yaml
spring:
  mongodb:
    uri: ${MONGO_URI:mongodb://tasks-mongodb-service:27017/tasksdb}

app:
  mongodb:
    server-selection-timeout: 2000
    socket-timeout: 5000
```

### 💡 Что выбрать для вашего проекта

Для **учебного проекта** проще всего **Вариант 1** — всё в Secret, как у вас сейчас. Это работает, и не требует дополнительного кода.

Если хочется «как в production» — **Вариант 3**. Он даёт:
- **URI в Secret** — только credentials (безопасно).
- **Таймауты в Config Server** — версионируются, меняются без `kubectl`.
- **Гибкость** — можно задавать разные таймауты для разных сред через профили.

Но он требует **кастомного бина `MongoClient`**, и здесь важно не повторить ошибку из начала вашего проекта: **уберите** `spring.data.mongodb.uri` из `application.yaml`, чтобы кастомный бин не конфликтовал с автоконфигурацией.

### 📌 Итог

| Подход | Где URI | Где таймауты | Плюсы | Минусы |
|---|---|---|---|---|
| **1. Всё в Secret** | Secret | Secret | Просто | Менять через `kubectl`, не версионируется |
| **2. URI в Secret, параметры в Config** | Secret | Config Server | Версионирование | Проблема с двумя `?` |
| **3. Отдельные свойства + кастомный бин** | Secret | Config Server | Гибко, версионируется | Нужен код |

Для вашего случая **Вариант 1** — рабочий и достаточный. Если хотите попробовать Вариант 3, я могу помочь с кодом кастомного бина.

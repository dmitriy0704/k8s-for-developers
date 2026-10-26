> можешь рассказать про bulkhead?

Bulkhead — это паттерн изоляции ресурсов. Если CircuitBreaker защищает от **повторных** вызовов сломанного сервиса, то Bulkhead ограничивает **количество одновременных** вызовов, чтобы одна медленная или проблемная зависимость не «съела» все ресурсы приложения.

## 🚢 Аналогия: водонепроницаемые переборки

Название пришло из судостроения: корабль разделён на отсеки (bulkheads). Если один отсек затопит, вода не распространится на остальные, и корабль останется на плаву.

В микросервисах то же самое: если один эндпоинт или зависимость начнёт «тормозить», он должен исчерпать **только свой** лимит потоков/соединений, а не весь пул приложения.

## 🎯 Чем Bulkhead отличается от CircuitBreaker и RateLimiter

| Паттерн | Что ограничивает | Когда срабатывает |
|---|---|---|
| **CircuitBreaker** | Прекращает **вызывать** сломанную зависимость | После N ошибок подряд |
| **RateLimiter** | Ограничивает **скорость** вызовов (за секунду/минуту) | При превышении RPS |
| **Bulkhead** | Ограничивает **одновременные** вызовы | Когда все «слоты» заняты |

**Ключевая идея Bulkhead:** даже если зависимость **не сломана**, но отвечает медленно, одновременно разрешено не более N вызовов. Остальные либо ждут в очереди, либо сразу получают отказ.

## ⚙️ Два типа Bulkhead в Resilience4j

Resilience4j предлагает две реализации, и выбор между ними — это выбор **что именно изолировать**:

| Тип | Механизм | Что изолирует | Ограничения |
|---|---|---|---|
| **`SemaphoreBulkhead`** (по умолчанию) | Семафор считает разрешения | Количество одновременных вызовов **на потоке вызывающего** | Медленный вызов всё равно занимает поток вызывающего до конца |
| **`ThreadPoolBulkhead`** | Отдельный пул потоков + ограниченная очередь | Поток вызывающего **освобождается**; вызов идёт в отдельном пуле | Метод должен возвращать `CompletionStage`/`Future` |

**Когда что выбирать:**
- **Semaphore** — проще, подходит для лёгких вызовов, где вы уверены, что поток вызывающего не «залипнет» надолго.
- **ThreadPool** — если зависимость может **блокировать** поток надолго (JDBC, HTTP без таймаута), и вы хотите защитить основной пул Tomcat/Netty от исчерпания.

## 🛠️ Настройка для Spring Boot 4 / Resilience4j

Добавьте в `application.yaml`:

```yaml
resilience4j:
  bulkhead:
    instances:
      mongoDb:
        maxConcurrentCalls: 10          # сколько вызовов одновременно разрешено
        maxWaitDuration: 100ms          # сколько ждать свободный слот (0 = не ждать)
```

**Параметры:**
- **`maxConcurrentCalls`** — максимальное количество одновременных вызовов. Если все заняты, новый вызов либо ждёт `maxWaitDuration`, либо сразу получает `BulkheadFullException`.
- **`maxWaitDuration`** — время ожидания свободного слота. `0` означает «не ждать вообще», `100ms` — подождать немного.

**Для `ThreadPoolBulkhead`** конфигурация другая:
```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      mongoDb:
        maxThreadPoolSize: 10
        coreThreadPoolSize: 5
        queueCapacity: 20
```

## 💻 Использование: аннотация и fallback

```java
@Bulkhead(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
@CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
@Retry(name = "mongoDb")
public Tasks createTask(Tasks task) {
    // ...
}
```

**Fallback вызывается, когда все слоты заняты** — Resilience4j бросит `BulkheadFullException`. Fallback-метод может вернуть понятный ответ, например `429 Too Many Requests`:

```java
public Tasks fallbackCreateTask(Tasks task, BulkheadFullException e) {
    throw new ServiceUnavailableException("Сервис перегружен, попробуйте позже");
}
```

Или можно сделать fallback с общим `Throwable`:

```java
public Tasks fallbackCreateTask(Tasks task, Throwable t) {
    if (t instanceof BulkheadFullException) {
        throw new ServiceUnavailableException("Слишком много одновременных запросов");
    }
    throw new ServiceUnavailableException("MongoDB недоступна");
}
```

## 📊 Мониторинг через Actuator

Добавьте `bulkheads` в список эндпоинтов:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,retries,bulkheads
```

Затем:
```bash
curl http://localhost:8080/actuator/bulkheads | jq
```

Вы увидите:
```json
{
  "bulkheads": {
    "mongoDb": {
      "availableConcurrentCalls": 8,
      "maxAllowedConcurrentCalls": 10
    }
  }
}
```

**Ключевая метрика:** `resilience4j_bulkhead_available_concurrent_calls` — сколько свободных слотов осталось. Если она падает до `0` — Bulkhead работает, ограничивая нагрузку.

## 🔄 Порядок декораторов

Важно помнить: Resilience4j применяет декораторы в определённом порядке. Согласно документации, порядок такой:

```
Retry → CircuitBreaker → RateLimiter → Bulkhead
```

То есть **Retry оборачивает CircuitBreaker**, а **Bulkhead — внутренний слой**. Это значит, что каждая повторная попытка Retry **заново проходит через Bulkhead** и занимает слот. Учитывайте это при настройке `maxConcurrentCalls` и `maxAttempts`.

## 🆚 Bulkhead vs Istio

У Istio есть похожий механизм — `connectionPool` в `DestinationRule`:

```yaml
trafficPolicy:
  connectionPool:
    http:
      http1MaxPendingRequests: 50
      http2MaxRequests: 100
```

**Разница:**
- **Istio connectionPool** ограничивает **сетевые соединения** между сервисами (на уровне Envoy).
- **Resilience4j Bulkhead** ограничивает **вызовы методов** внутри JVM (например, вызовы MongoDB, Kafka, внешних API).

Они не дублируют друг друга, а работают на разных уровнях — так же, как CircuitBreaker в Istio и Resilience4j.

## 💎 Итог

| Вопрос | Ответ |
|---|---|
| Что такое Bulkhead? | Паттерн изоляции: ограничивает количество **одновременных** вызовов |
| Зачем нужен? | Чтобы одна медленная зависимость не исчерпала все потоки/соединения |
| Два типа | `Semaphore` (проще) и `ThreadPool` (полная изоляция потоков) |
| Как настроить? | `resilience4j.bulkhead.instances.<name>.maxConcurrentCalls` |
| Когда срабатывает fallback? | Когда все слоты заняты → `BulkheadFullException` |
| Как мониторить? | `/actuator/bulkheads`, метрика `available_concurrent_calls` |
| Отличие от Istio? | Istio — сетевые соединения; Resilience4j — вызовы в коде |

Для вашего `tasks-service` Bulkhead будет полезен, если вы заметите, что при наплыве запросов к MongoDB приложение начинает «задыхаться» — все потоки заняты ожиданием ответа от БД. Тогда `maxConcurrentCalls` ограничит давление на MongoDB и оставит потоки для других операций (например, health-check'ов или других эндпоинтов).

> я понял, но у меня есть два вопроса:
>1) мы чуть раньше разбирали rate limit, там ограничивалось количество запросов от одного ip адреса, или как, а Bulkhead - это количество запросов ОТ ВСЕХ ВООБЩЕ?
>2) что считается запросами, количество которых мы ограничиваем? HTTP запросы от разных.. от разных кого? а виртуальные потоки тут участвуют?

Да, вы очень точно уловили суть различий. Это два разных инструмента, которые решают разные задачи, и важно понимать, что именно они ограничивают. Если коротко: **Rate Limiter управляет скоростью поступления запросов, а Bulkhead — количеством одновременно выполняемой работы.**

### 🚦 Rate Limiter vs. Bulkhead: Скорость и Загрузка

Разница между ними фундаментальна. Rate Limiter смотрит на **время** (сколько запросов в секунду), а Bulkhead — на **состояние** (сколько задач выполняется прямо сейчас).

| Характеристика | Rate Limiter (например, в Istio) | Bulkhead (Resilience4j) |
| :--- | :--- | :--- |
| **Что ограничивает** | **Скорость поступления** запросов (RPS) | **Количество одновременных** вызовов (конкурентность) |
| **Ключевой вопрос** | «Не слишком ли быстро приходят запросы?» | «Не слишком ли много работы выполняется одновременно?» |
| **Единица измерения** | Запросы в секунду/минуту | Количество одновременно занятых «слотов» |
| **Область действия** | Глобально для endpoint'а (все клиенты) или для группы (например, по IP) | Локально для конкретного метода или зависимости в одном экземпляре приложения |
| **Реакция на сбой** | Отклоняет запросы (HTTP 429), если превышен лимит скорости | Отклоняет вызовы, если все «слоты» заняты, даже если запросы приходят нечасто |

**Ключевой сценарий, где они расходятся:** представьте, что зависимость (MongoDB) начала отвечать очень медленно, но без ошибок.
*   **Rate Limiter не сработает**: запросы приходят с прежней частотой (например, 10 в секунду), что может быть в пределах лимита.
*   **Bulkhead сработает**: каждый вызов занимает «слот» на всё время долгого ответа. Когда все слоты (например, 10) заняты, следующие вызовы будут мгновенно отклонены, защищая пул потоков приложения от перегрузки.

Именно поэтому эти паттерны не заменяют, а **дополняют** друг друга. Система, которая может столкнуться с замедлением, нуждается в обоих.

### 🎯 Что именно считает Bulkhead и при чём тут потоки?

Этот вопрос ещё важнее, потому что он определяет, как именно работает защита.

**Что считается «запросом» для Bulkhead?**
Это **не** входящий HTTP-запрос от пользователя. Bulkhead считает **вызовы конкретного Java-метода**, который вы аннотировали, например `@Bulkhead(name = "mongoDb") public Tasks createTask(...)`. Каждый раз, когда поток исполнения входит в этот метод, он занимает один «слот» (permit). Метод может быть вызван из разных мест, но для Bulkhead это просто «вызовы целевой операции».

**Роль потоков и виртуальных потоков:**
Здесь всё зависит от типа Bulkhead, который вы выберете:

1.  **`SemaphoreBulkhead` (по умолчанию):** Он просто считает количество вызовов. Вызов **выполняется в том же потоке**, который его инициировал (например, в потоке обработки HTTP-запроса Tomcat). Если вызов `createTask` «зависнет» на 2 секунды, поток, который его вызвал, тоже будет занят на 2 секунды. Семафор лишь не даст *другим* потокам войти в этот метод, если все «слоты» заняты.

2.  **`ThreadPoolBulkhead`:** Этот тип создаёт **отдельный пул потоков** для выполнения защищаемого метода. Поток, который вызвал метод, просто передаёт задачу в этот пул и освобождается. Это обеспечивает полную изоляцию: даже если все потоки в пуле Bulkhead'а «зависнут», основной пул потоков приложения (например, Tomcat) останется свободным для обработки других запросов. Однако этот тип требует, чтобы метод возвращал `CompletableFuture` или `CompletionStage`.

**Про виртуальные потоки (Java 21+):**
Современный подход с виртуальными потоками (Project Loom) меняет правила игры. Виртуальные потоки очень дешёвые, и их можно создавать тысячами. Использование `ThreadPoolBulkhead` с фиксированным пулом платформенных потоков в этом случае считается антипаттерном, так как оно нивелирует преимущества виртуальных потоков.

Рекомендуемый подход — использовать **`SemaphoreBulkhead`** вместе с виртуальными потоками. Виртуальные потоки будут дешево блокироваться на семафоре, а вы получите эффективное ограничение конкурентности без накладных расходов на переключение контекста между потоками ОС.

### 💎 Итог

*   **Rate Limiter** — это про **скорость**. Он защищает от слишком частых запросов.
*   **Bulkhead** — это про **объём одновременной работы**. Он защищает пул потоков приложения от исчерпания из-за медленных вызовов.
*   «Запрос» для Bulkhead — это вызов конкретного метода в вашем коде.
*   Выбор между `SemaphoreBulkhead` и `ThreadPoolBulkhead` — это выбор между простым ограничением конкурентности и полной изоляцией потоков. Для современных приложений с виртуальными потоками предпочтителен `SemaphoreBulkhead`.

Надеюсь, теперь стало понятнее. Если захотите разобрать, как это выглядит в коде для вашего `tasks-service`, дайте знать.

> да, покажи код.

Отлично, давайте соберём всё вместе. Я покажу полный код с Bulkhead, CircuitBreaker и Retry, плюс объясню тонкости, которые важны именно при **совместном** использовании.

## ⚙️ Конфигурация `application.yaml`

```yaml
resilience4j:
  bulkhead:
    instances:
      mongoDb:
        maxConcurrentCalls: 10        # не более 10 одновременных вызовов
        maxWaitDuration: 100ms        # ждать свободный слот до 100ms, потом отказ

  circuitbreaker:
    instances:
      mongoDb:
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
        # ⚠️ ВАЖНО: BulkheadFullException — не ошибка MongoDB,
        # поэтому НЕ должен размыкать circuit breaker
        ignoreExceptions:
          - io.github.resilience4j.bulkhead.BulkheadFullException
        recordExceptions:
          - com.mongodb.MongoException
          - org.springframework.dao.DataAccessException
          - java.net.ConnectException

  retry:
    instances:
      mongoDb:
        maxAttempts: 2                # 2 попытки (1 оригинал + 1 retry)
        waitDuration: 100ms
        retryExceptions:
          - com.mongodb.MongoException
          - java.net.ConnectException
```

**Ключевой момент:** `ignoreExceptions` с `BulkheadFullException`. Без этого каждая перегрузка Bulkhead'а считалась бы ошибкой MongoDB, и CircuitBreaker размыкался бы не из-за проблем с БД, а из-за наплыва запросов.

## 💻 Код `TaskService`

```java
@Slf4j
@Service
public class TaskService {

    private final TaskRepository taskRepository;
    private final KafkaProducer producer;

    public TaskService(TaskRepository taskRepository, KafkaProducer producer) {
        this.taskRepository = taskRepository;
        this.producer = producer;
    }

    @Bulkhead(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    @CircuitBreaker(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    @Retry(name = "mongoDb", fallbackMethod = "fallbackCreateTask")
    public Tasks createTask(Tasks task) {
        Tasks saved = taskRepository.save(task);
        producer.sendTaskCreatedEvent(new TaskEvent(
                saved.getId(),
                saved.getTitle(),
                "USER abc123",
                "CREATED"
        ));
        return saved;
    }

    /**
     * Единый fallback для всех трёх декораторов.
     * Различаем причину отказа через instanceof.
     */
    public Tasks fallbackCreateTask(Tasks task, Throwable t) {
        if (t instanceof BulkheadFullException) {
            log.warn("Bulkhead full: too many concurrent requests, task={}", task.getId());
            throw new ServiceUnavailableException(
                "Сервис перегружен, попробуйте позже"
            );
        }

        if (t instanceof CallNotPermittedException) {
            log.error("CircuitBreaker is OPEN: MongoDB unavailable, task={}", task.getId());
            throw new ServiceUnavailableException(
                "MongoDB временно недоступна, попробуйте позже"
            );
        }

        log.error("MongoDB call failed after retries, task={}, error={}",
                  task.getId(), t.getMessage(), t);
        throw new ServiceUnavailableException(
            "Не удалось создать задачу, попробуйте позже"
        );
    }
}
```

## 🎯 Что происходит при вызове — пошагово

Порядок декораторов по умолчанию:
```
Retry → CircuitBreaker → Bulkhead → createTask
```

**Сценарий 1: всё хорошо**
1. Retry пропускает вызов.
2. CircuitBreaker `CLOSED` — пропускает.
3. Bulkhead: есть свободный слот — пропускает.
4. `taskRepository.save()` → успех.
5. Слот освобождается.

**Сценарий 2: MongoDB упала**
1. Retry вызывает → CB пропускает → Bulkhead даёт слот → `save()` бросает `MongoSocketException`.
2. Retry ждёт 100ms и пробует снова → та же ошибка.
3. Retry исчерпан → пробрасывает `MongoSocketException`.
4. **CircuitBreaker** записывает ошибку (не ignored).
5. Fallback получает `MongoSocketException` → `"Не удалось создать задачу"`.
6. После 5 ошибок → CB `OPEN`.

**Сценарий 3: все 10 слотов Bulkhead заняты**
1. Retry вызывает → CB `CLOSED` пропускает → **Bulkhead бросает `BulkheadFullException`**.
2. Retry: `BulkheadFullException` не в `retryExceptions` → не ретраит, пробрасывает.
3. **CircuitBreaker** видит `BulkheadFullException` — но он в `ignoreExceptions`, **не записывает** как ошибку.
4. Fallback получает `BulkheadFullException` → `"Сервис перегружен"`.
5. CB **не размыкается** — MongoDB-то в порядке, просто нагрузка.

**Это критически важно:** без `ignoreExceptions` перегрузка Bulkhead'а «положила» бы CircuitBreaker, и потом MongoDB не вызывалась бы даже после спада нагрузки.

## 🧪 Как протестировать Bulkhead

**Шаг 1.** Искусственно замедлите MongoDB, чтобы вызовы занимали больше времени:

```bash
# Добавьте latency через сетевой фильтр (необязательно, можно использовать
# любой способ замедлить ответ MongoDB)
```

Или проще — временно поставьте в коде `Thread.sleep(2000)` перед `save()` — **только для теста**.

**Шаг 2.** Отправьте 20 параллельных запросов:

```bash
# С помощью xargs (доступен почти везде)
seq 1 20 | xargs -P 20 -I {} curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://<ingress-address>/tasks/create \
  -H "Content-Type: application/json" \
  -d '{"title":"test-{}"}'
```

Флаг `-P 20` означает «20 параллельных процессов». Каждый делает по одному запросу.

**Ожидаемый результат:**
- 10 запросов получат `201 Created` (слоты Bulkhead'а).
- 10 запросов получат `503 Service Unavailable` с сообщением `"Сервис перегружен"` — **мгновенно**, без ожидания.
- CircuitBreaker **останется `CLOSED`** — потому что `BulkheadFullException` в `ignoreExceptions`.

**Шаг 3.** Проверьте Bulkhead через actuator:

```bash
kubectl port-forward svc/tasks-service 8080:80
curl http://localhost:8080/actuator/bulkheads | jq
```

Вывод:
```json
{
  "bulkheads": {
    "mongoDb": {
      "availableConcurrentCalls": 0,
      "maxAllowedConcurrentCalls": 10
    }
  }
}
```

`availableConcurrentCalls: 0` — все слоты заняты, Bulkhead работает.

## 📊 Мониторинг

Добавьте в `application.yaml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,retries,bulkheads,metrics
  metrics:
    tags:
      application: tasks-service
```

Полезные метрики (через Prometheus или `/actuator/metrics`):

```
resilience4j_bulkhead_available_concurrent_calls{name="mongoDb"}
resilience4j_bulkhead_max_allowed_concurrent_calls{name="mongoDb"}
resilience4j_circuitbreaker_state{name="mongoDb",state="closed"}
resilience4j_circuitbreaker_calls_total{name="mongoDb",kind="not_permitted"}
```

`not_permitted` — счётчик вызовов, которые **не дошли** до метода, потому что CB был `OPEN`. `available_concurrent_calls` — свободные слоты Bulkhead'а.

## 🧵 Про виртуальные потоки

Ваш `tasks-service` использует Spring MVC (Tomcat, платформенные потоки). `SemaphoreBulkhead` ограничивает **количество потоков Tomcat**, которые одновременно висят на вызове MongoDB.

**Проблема без Bulkhead:** если MongoDB тормозит, все 200 потоков Tomcat (дефолт) займутся ожиданием → новые запросы, включая health-check, не смогут обработаться → Kubernetes убьёт под.

**С Bulkhead:** максимум 10 потоков заняты MongoDB. Остальные 190 свободны для health-check, других эндпоинтов и т.д. Под жив, пользователи получают быстрый 503.

Если включите виртуальные потоки (Spring Boot 3.2+ / 4):
```yaml
spring:
  threads:
    virtual:
      enabled: true
```

`SemaphoreBulkhead` продолжит работать корректно — виртуальные потоки дёшево блокируются на семафоре. Но **не используйте `ThreadPoolBulkhead`** с виртуальными потоками — это нивелирует их преимущества (пул платформенных потоков станет узким горлышком).

## 💎 Итог

| Что добавили | Зачем |
|---|---|
| `@Bulkhead` с `maxConcurrentCalls: 10` | Не более 10 потоков ждут MongoDB одновременно |
| `ignoreExceptions: BulkheadFullException` в CB | Перегрузка не должна «положить» CB |
| Единый fallback с `instanceof` | Разные сообщения для разных причин отказа |
| Тест с 20 параллельными запросами | Увидеть Bulkhead в действии |

После этого ваш `tasks-service` получит **три уровня защиты**:
- **Bulkhead** — от перегрузки потоками.
- **CircuitBreaker** — от мёртвой MongoDB.
- **Retry** — от временных сбоев.

Плюс **Istio DestinationRule** — от нездоровых подов, и **Istio Rate Limit** — от слишком частых запросов снаружи. Полный production-ready стек.

Если что-то пойдёт не так при тестировании — пришлите логи и вывод `/actuator/bulkheads`, разберёмся.

> я еще спросить хотел. Bulkhead - считает количество вызовов метода с аннотацией @Bulkhead(.....), а что тогда считает Rate Limit, который мы настраивали вместе с Redis? запросы в секунду? а что понимается под запросом в контексте ratelimit + redis?


Отличный вопрос — тут легко запутаться, потому что «запрос» в разных контекстах означает разное. Давайте разложим.

## 🎯 Rate Limit считает **HTTP-запросы на уровне Envoy**

Rate Limit, который мы настраивали через EnvoyFilter + ratelimit + Redis, работает **на входе в кластер**, в Istio Ingress Gateway. Он считает **каждый HTTP-запрос**, который проходит через Envoy и матчится на правила из `VirtualService`.

**Ключевое:** Rate Limit **не знает** ничего о Java-коде, методах, `@Bulkhead` или `@CircuitBreaker`. Он видит только HTTP-запрос: метод, путь, заголовки, IP клиента. И считает **факт прихода запроса**.

```
Клиент → [Envoy Ingress] → ratelimit → Redis
              │
              └─ здесь считается КАЖДЫЙ HTTP-запрос
                 до того, как он попадёт в приложение
```

**Что считается «запросом» для Rate Limit:**

| Считается | Не считается |
|---|---|
| `POST /tasks/create` от клиента | Вызов `taskRepository.save()` внутри Java |
| `GET /tasks/list` от клиента | Запрос к MongoDB |
| Любой HTTP-запрос, прошедший через Ingress Gateway и матчащийся на правило | Внутренние вызовы между подами (если они не идут через Ingress) |
| | Сообщения в Kafka |

## 📊 Что именно считает Redis

Помните ключи, которые мы видели:

```
tasks-app_remote_address_10.244.0.1_1790358840
tasks-app_generic_key_global_1790358840
```

Разбор:

```
tasks-app_remote_address_10.244.0.1_1790358840
   ↑          ↑            ↑              ↑
 domain   descriptor    значение      timestamp
                                    (начало минуты)
```

**Как это работает:**

1. Запрос приходит на Envoy Ingress.
2. Envoy формирует **дескрипторы** по правилам из `VirtualService`:
   - `remote_address = 10.244.0.1` (IP клиента)
   - `generic_key = global` (общая метка)
3. Envoy отправляет дескрипторы в ratelimit-сервис по gRPC.
4. Ratelimit идёт в Redis: «сколько раз `remote_address=10.244.0.1` встречался в текущей минуте?»
5. Redis отвечает: «3 раза».
6. Ratelimit сравнивает с лимитом (5/мин): «3 < 5 → OK».
7. Ratelimit увеличивает счётчик: «стало 4».
8. Envoy пропускает запрос дальше.

**Временное окно:** `unit: minute` в ConfigMap означает, что счётчик «обнуляется» каждую минуту. Redis-ключ с суффиксом `_1790358840` живёт ровно 60 секунд (TTL), потом удаляется. Новый запрос создаст новый ключ с новым timestamp.

## 🆚 Полная сравнительная таблица

| Аспект | Rate Limit (Envoy + Redis) | Bulkhead (Resilience4j) | CircuitBreaker (Resilience4j) |
|---|---|---|---|
| **Что считает** | HTTP-запросы на входе | Одновременные вызовы метода | Ошибки/успехи метода |
| **Где работает** | Envoy Ingress Gateway | Внутри JVM | Внутри JVM |
| **Что видит** | HTTP: метод, путь, заголовки, IP | Факт входа в метод | Результат метода (исключение/успех) |
| **Единица** | Запросы в минуту (RPS) | Количество слотов | Процент ошибок |
| **Когда срабатывает** | При превышении скорости | Когда все слоты заняты | Когда много ошибок |
| **Временное окно** | Минута/секунда/час | Текущий момент | Скользящее окно вызовов |
| **Что защищает** | Приложение от наплыва клиентов | Пул потоков от исчерпания | От повторных вызовов мёртвой зависимости |
| **Реакция** | 429 Too Many Requests | 503 Service Unavailable | 503 + fallback |

## 💡 Конкретный пример: один запрос — три счётчика

Представьте, что клиент отправляет `POST /tasks/create`:

```
1. Клиент → Envoy Ingress
   │
   ├─ Rate Limit: +1 к счётчику remote_address=10.244.0.1 (минута 1790358840)
   ├─ Rate Limit: +1 к счётчику generic_key=global (минута 1790358840)
   │
   │ Если любой из счётчиков > лимита → 429, запрос НЕ идёт дальше
   │
   ▼
2. Spring Cloud Gateway → tasks-service
   │
   ▼
3. Контроллер → TaskService.createTask()
   │
   ├─ Bulkhead: занимает 1 слот из 10 (проверяет maxConcurrentCalls)
   ├─ CircuitBreaker: проверяет, не OPEN ли цепь
   ├─ Retry: оборачивает вызов
   │
   ▼
4. taskRepository.save() → MongoDB
   │
   ├─ Успех → CircuitBreaker записывает успех, Bulkhead освобождает слот
   └─ Ошибка → CircuitBreaker записывает ошибку, Retry пробует снова
```

**Важно:** Rate Limit срабатывает **до** того, как запрос попадёт в приложение. Если Rate Limit вернул 429, `@Bulkhead` и `@CircuitBreaker` **вообще не вызываются** — запрос не дошёл до кода.

## 🎯 Что понимается под «запросом» в контексте Rate Limit

В нашем setup Rate Limit применялся к пути `/tasks` через `VirtualService`:

```yaml
rateLimits:
  - actions:
      - remoteAddress: {}
  - actions:
      - genericKey:
          descriptorValue: global
```

Это означает:

- **Каждый HTTP-запрос** к `/tasks/*` (создание, список, удаление — любой) **увеличивает счётчик**.
- Счётчик `remote_address` — **отдельный для каждого IP**. Клиент A и клиент B имеют разные счётчики.
- Счётчик `generic_key=global` — **общий для всех**. Каждый запрос от любого клиента увеличивает его.

**Пример с лимитами 5/мин на IP и 100/мин глобально:**

| Время | Клиент | Запрос | remote_address A | remote_address B | global | Результат |
|---|---|---|---|---|---|---|
| 12:34:00 | A | POST /tasks/create | 1 | 0 | 1 | ✅ 200 |
| 12:34:01 | A | POST /tasks/create | 2 | 0 | 2 | ✅ 200 |
| 12:34:02 | B | GET /tasks/list | 2 | 1 | 3 | ✅ 200 |
| 12:34:03 | A | POST /tasks/create | 3 | 1 | 4 | ✅ 200 |
| ... | A | (ещё 2 запроса) | 5 | 1 | 6 | ✅ 200 |
| 12:34:30 | A | POST /tasks/create | **6** | 1 | 7 | ❌ **429** (A превысил 5/мин) |
| 12:34:31 | B | GET /tasks/list | 6 | 2 | 8 | ✅ 200 (B ещё в лимите) |

Ключевой момент: **A получил 429, хотя общий лимит (100) не превышен**. Превышен **персональный** лимит для IP A.

## 📌 Что важно запомнить

| Вопрос | Ответ |
|---|---|
| Rate Limit считает запросы в секунду? | Считает **за единицу времени**, настроенную в ConfigMap (`minute`/`second`/`hour`) |
| Что такое «запрос» для Rate Limit? | **HTTP-запрос** на входе в Envoy, до приложения |
| Считает ли Rate Limit вызовы методов? | **Нет**, только HTTP на входе |
| Знает ли Rate Limit про `@Bulkhead`? | **Нет**, это независимые механизмы |
| Если Rate Limit вернул 429, вызовется ли `@Bulkhead`? | **Нет**, запрос не дойдёт до кода |
| Bulkhead считает HTTP-запросы? | **Нет**, только вызовы конкретного метода |
| Могут ли оба сработать на один запрос? | **Да**, но на разных этапах: Rate Limit — до приложения, Bulkhead — в коде |

## 💎 Итог

**Rate Limit** — это «охранник на входе в клуб»: считает, сколько людей прошло за минуту, и не пускает сверх лимита. Он **не знает**, что происходит внутри.

**Bulkhead** — это «ограничение на количество людей в лифте»: даже если в клуб пустили 100 человек, в лифт (метод `createTask`) влезает только 10 одновременно. Остальные ждут или уходят.

**CircuitBreaker** — это «табличка „лифт сломан“»: после нескольких поломок лифт перестаёт принимать пассажиров, и все идут пешком (fallback).

Все три работают на **разных уровнях** и **не дублируют** друг друга. В production нужны все три — плюс Istio DestinationRule для защиты от нездоровых подов.

> у меня еще два вопроса:
> 1) для rate limit - ip адрес чей именно? прям каких-нибудь условных пользователей или это внешний адрес между подами?
> 2) по поводу retry - он сам отправляет повторные запросы или считает запросы из cirvuit breaker?

см. 04_02_retry.md
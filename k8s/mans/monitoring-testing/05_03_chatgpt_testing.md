> помоги разобраться?
я пишу приложение java spring boot4 в minikube. я делаю тестирование с помощью k6.
виртуальные потоки выключены, вот настройки tomcat:

```yaml
tomcat:
	mbeanregistry:
		enabled: true
		  	accept-count: 200\
```

вот скрипт

```javascript
import http from 'k6/http';
import {check, sleep} from 'k6';

export const options = {
    // vus: 10,          // 10 виртуальных пользователей (Virtual Users)
    // duration: '30s',  // в течение 30 секунд(10/s)

    stages: [
        { duration: '30s', target: 100 },
        { duration: '30s', target: 150 },
        { duration: '30s', target: 200 },
        { duration: '30s', target: 0 },
    ],

    thresholds: {
        http_req_duration: ['p(95)<500'],   // 95% запросов быстрее 200ms
        http_req_failed: ['rate<0.05'],     // меньше 1% ошибок
    },
};

function safeParseJson(str) {
    try {
        return JSON.parse(str);
    } catch {
        return null;
    }
}

export default function () {
    const url = 'http://127.0.0.1:8091/tasks/create';
    const payload = JSON.stringify({
        title: `task-${__VU}-${__ITER}`, // Уникальное имя для каждой задачи
        description: `NEW task-${__VU}-${__ITER}`,
        completed: true
    });

    const params = {
        headers: {
            'Content-Type': 'application/json',
            'X-User-ID': `user-${__VU}`,   // каждый VU — отдельный пользователь
        },
    };

    // Отправляем POST-запрос
    const res = http.post(url, payload, params);

    // Проверяем, что статус ответа — 201 Created
    check(res, {
        'status is 201': (r) => r.status === 201,
        'has task id': (r) => {
            if (r.status !== 201) return false;   // ← не парсим тело на ошибках
            const body = safeParseJson(r.body);
            return body !== null && body.id !== undefined;
        },
        'fast enough': (r) => r.timings.duration < 500,
        // k6 покажет, сколько проверок прошло/провалилось. Если «has task id»
        // провалилась — значит, tasks-service возвращает некорректный JSON.
    });

    // Пауза в 1 секунду, имитируя пользователя, читающего ответ
    sleep(1);
}
```


вот результат

```text

  █ THRESHOLDS 

    http_req_duration
    ✓ 'p(95)<500' p(95)=1.87ms

    http_req_failed
    ✗ 'rate<0.05' rate=100.00%


  █ TOTAL RESULTS 

    checks_total.......: 40467  335.625596/s
    checks_succeeded...: 33.32% 13487 out of 40467
    checks_failed......: 66.67% 26980 out of 40467

    ✗ status is 201
      ↳  0% — ✓ 0 / ✗ 13489
    ✗ has task id
      ↳  0% — ✓ 0 / ✗ 13489
    ✗ fast enough
      ↳  99% — ✓ 13487 / ✗ 2

    HTTP
    http_req_duration....: avg=5.53ms min=0s med=738µs max=30s p(90)=1.57ms p(95)=1.87ms
    http_req_failed......: 100.00% 13489 out of 13489
    http_reqs............: 13489   111.875199/s

    EXECUTION
    iteration_duration...: avg=1s     min=1s med=1s    max=31s p(90)=1s     p(95)=1s    
    iterations...........: 13489   111.875199/s
    vus..................: 3       min=3              max=199
    vus_max..............: 200     min=200            max=200

```

Да, конечно. И здесь уже есть интересные моменты. Судя по результатам, проблема не просто в том, что Tomcat не справляется с количеством потоков. Похоже, при увеличении нагрузки система начинает накапливать запросы, а затем часть из них завершается по таймауту.

Давай разберём последовательно.

## 1. Что говорят результаты k6

Самые показательные значения:

| Метрика               | Результат | Что означает                                       |
| --------------------- | --------- | -------------------------------------------------- |
| Average response time | 32.07 s   | Очень долго                                        |
| Median                | 30 s      | Половина запросов выполняется 30 секунд или дольше |
| P95                   | 59.99 s   | 95% запросов укладываются в 60 секунд              |
| Max                   | 60 s      | Есть запросы, достигающие 60 секунд                |
| Failed requests       | 64.56%    | Почти 2/3 запросов завершились ошибкой             |
| Requests/sec          | 3.06      | Очень низкая пропускная способность                |



Главная проблема

&#x20;P95 = 59.99 секунды.



Обрати внимание на максимальное время — ровно 60 секунд.

У k6 стандартный таймаут HTTP-запроса составляет 60 секунд. Поэтому я бы в первую очередь проверил, не упираются ли запросы в этот таймаут.

При этом пока нельзя утверждать, что все неуспешные запросы завершились именно по таймауту. Нужно посмотреть их статусы и ошибки.

## 2. Как работает твой тест

У тебя настроены stages:

```
stages: [
    { duration: '30s', target: 100 },
    { duration: '30s', target: 150 },
    { duration: '30s', target: 200 },
    { duration: '30s', target: 0 },
],
```

Важный момент: `target` — это количество виртуальных пользователей, а не количество запросов в секунду.

То есть:

- первые 30 секунд k6 постепенно увеличивает количество VU до 100;
- следующие 30 секунд — до 150;
- следующие 30 секунд — до 200;
- последние 30 секунд — постепенно уменьшает до нуля.

Каждый виртуальный пользователь выполняет примерно такой цикл:

```
POST /tasks/create
       |
       v
Ожидание ответа сервера
       |
       v
sleep(1)
       |
       v
Следующая итерация
```

И здесь есть интересная особенность.

Представим, что сервер отвечает за 100 мс. Один VU сможет отправлять примерно один запрос в секунду.

Но если сервер начинает отвечать за 30 секунд, один VU сможет отправлять примерно один запрос каждые 31 секунду.

Получается, чем сильнее перегружается сервер, тем меньше запросов в секунду генерирует твой тест.

Это называется closed workload model — нагрузка зависит от скорости ответов системы.

Для проверки предельной пропускной способности это не всегда удобно.

## 3. Что может происходить внутри Tomcat

У тебя:

```
tomcat:
  mbeanregistry:
    enabled: true
  threads:
    max: 400
    min-spare: 20
  accept-count: 200
```

Если настройки действительно загружены Spring Boot, то Tomcat может использовать до 400 рабочих потоков.

Упрощённо обработка выглядит так:

k6

До 200 виртуальных пользователей

Tomcat Connector

Принимает соединения, очередь accept-count

Tomcat Worker Threads

# 400

Максимальное количество потоков

Tasks Service

Spring Boot → MongoDB

Но есть нюанс.

Если обработка запроса включает обращение к MongoDB, поток Tomcat может ждать ответа от базы данных.

Например:

1. Tomcat принимает запрос.
2. Выделяет рабочий поток.
3. Поток обращается к MongoDB.
4. MongoDB не успевает обрабатывать запросы.
5. Поток Tomcat ждёт освобождения соединения или ответа от базы.
6. Количество занятых потоков растёт.
7. Новые запросы начинают ждать.

Таким образом, даже 400 потоков не гарантируют высокую производительность.

Вполне возможно, что ограничение находится вообще не в Tomcat, а в MongoDB, CPU или ресурсах Kubernetes.

## 4. Что я предлагаю проверить

У тебя уже включён MBean Registry, поэтому можно использовать метрики Tomcat.

В Prometheus попробуй следующие запросы:

Занятые потоки Tomcat:

```
tomcat_threads_busy_threads
```

Текущее количество потоков:

```
tomcat_threads_current_threads
```

Максимальное количество потоков:

```
tomcat_threads_config_max_threads
```

Текущее количество соединений:

```
tomcat_connections_current_connections
```

Особенно интересно посмотреть на `tomcat_threads_busy_threads`.

Если во время нагрузки значение приближается к 400, значит, Tomcat действительно исчерпывает пул рабочих потоков.

Также проверь:

```
kubectl top pods
```

И посмотри:

- CPU;
- Memory;
- не происходит ли CPU throttling;
- нет ли рестартов контейнера.

Отдельно стоит посмотреть логи приложения и MongoDB во время тестирования.

## 5. Я бы немного изменил методику тестирования

Сейчас у тебя одновременно проверяется несколько вещей:

- способность Tomcat обрабатывать запросы;
- способность Spring Boot выполнять бизнес-логику;
- производительность MongoDB;
- пропускная способность всего приложения.

Это нормально для комплексного нагрузочного тестирования, но затрудняет поиск узкого места.

Я бы провёл три теста:

| Тест   | Количество VU | Цель                                      |
| ------ | ------------- | ----------------------------------------- |
| Первый | 10–20         | Проверить базовую производительность      |
| Второй | 50–100        | Найти начало деградации                   |
| Третий | 150–200       | Проверить поведение под высокой нагрузкой |

И пока не менял бы настройки Tomcat.

Ещё один момент: в твоём предыдущем мониторинговом проекте k6 обращался к приложению через `kubectl port-forward`. Если сейчас используется тот же способ, стоит учитывать, что в нагрузочный путь входит ещё и локальный механизм port-forward. Для более достоверного измерения производительности самого сервиса впоследствии можно запустить k6 внутри Kubernetes.

## 6. Давай начнём с трёх вопросов

Мне хочется сначала понять, где именно образуется задержка, а не предлагать тебе просто увеличить количество потоков.

1\. Какие HTTP-статусы возвращают неуспешные запросы?

Пока не знаю

500 Internal Server Error

503 Service Unavailable

Timeout / context deadline exceeded

Другие статусы (400, 404 и т. д.)

2\. Приложение сохраняет задачи в MongoDB?

Да, каждый POST записывает задачу в MongoDB

Нет, запрос не обращается к базе

3\. Как k6 обращается к приложению?

Через kubectl port-forward (127.0.0.1:8091)

Через Kubernetes Ingress

Запущен внутри Kubernetes

Другой способ

Отправить ответы

И ещё одна просьба: если можешь, покажи лог одной неуспешной итерации k6, включая HTTP-статус и тело ответа.

После этого сможем определить, что именно ограничивает производительность: Tomcat, MongoDB, Kubernetes или сам способ проведения нагрузочного тестирования. И уже тогда будем экспериментировать с настройками, а не менять их наугад.

> ой, я забыл сказать, что у меня postgres и вот его настройки
hikari:\
&#x20; maximum-pool-size: 30  \&#xA;*&#x20;&#x20;*&#x63;onnection-timeout: 3000&#x20;

и у меня еще есть bulkhead

```yaml
  bulkhead: #-> Соответствует @Bulkhead(name = "postgres", type = Bulkhead.Type.SEMAPHORE, fallbackMethod = "fallbackCreateTask")
    instances:
      postgres:
        maxConcurrentCalls: 20 #-> Количество одновременных вызовов метода
        maxWaitDuration: 0 #-> 0 - НЕ ждать, сразу fallback; 100ms - Дополнительные запросы будут ожидать до 100ms или будут отклонены

```

```java
    @CircuitBreaker(name = "postgres", fallbackMethod = "fallbackCreateTask")
    @Bulkhead(name = "postgres")
    public Tasks createTask(Tasks task) {
        Tasks savedTask = taskRepository.save(task);
        tasksCreated.increment();   // ← +1 при успехе
        return savedTask;
    }

    public Tasks fallbackCreateTask(Tasks task, Throwable t) throws ServiceUnavailableException {
     if (t instanceof BulkheadFullException) {
            log.warn("Bulkhead full: too many concurrent requests");
        } else {
            log.error("DB error: {}", t.getMessage(), t);
        }
        tasksFailed.increment();
        throw new ServiceUnavailableException(
                "Не удалось создать задачу, попробуйте позже"
        );
    }
}


```


О, вот теперь картина становится гораздо интереснее! И я вижу потенциальное узкое место, которое стоит проверить в первую очередь.

У тебя есть три ограничения на разных уровнях:

1. Tomcat — до 400 рабочих потоков.
2. HikariCP — до 30 соединений с PostgreSQL.
3. Bulkhead — максимум 20 одновременных вызовов метода `createTask()`.

Причём Bulkhead у тебя настроен строже, чем HikariCP.

## 1. Как сейчас работает твоя система

Посмотри на эту схему:

k6

До 200 VU

Tomcat

# 400

Максимум рабочих потоков

Resilience4j Bulkhead

# 20

Одновременно выполняемых вызовов

maxWaitDuration = 0

HikariCP

# 30

Максимум соединений с PostgreSQL

PostgreSQL

Что происходит при высокой нагрузке:

- 20 запросов одновременно могут пройти через Bulkhead.
- 21-й запрос уже не сможет войти в защищённый метод.
- Поскольку `maxWaitDuration: 0`, он не будет ждать освобождения места.
- Resilience4j выбросит `BulkheadFullException`.
- Запрос попадёт в `fallbackCreateTask()`.
- Ты увеличишь `tasksFailed` и вернёшь клиенту ошибку `503` (если соответствующий HTTP-маппинг настроен).

То есть у тебя есть вполне конкретный механизм, который способен объяснить большое количество ошибок при нагрузочном тестировании.

## 2. Почему Bulkhead может быть причиной ошибок

Представим, что каждый запрос к PostgreSQL выполняется 2 секунды.

При Bulkhead = 20:

\\[ Throughput = \frac{20}{2} = 10 \text{ запросов/сек} \\]

Это теоретическая пропускная способность именно защищённого участка при таком времени выполнения.

Если запросы выполняются 5 секунд:

\\[ Throughput = \frac{20}{5} = 4 \text{ запроса/сек} \\]

А теперь вспомни свой результат k6:

```
http_reqs: 460
rate: 3.066551/s
```

Получается, что наблюдаемая пропускная способность около 3 запросов в секунду.

Это ещё не доказывает, что виноват Bulkhead, но очень интересное совпадение. Возможно, запросы долго выполняются, все 20 разрешённых слотов заняты, а остальные получают отказ.

## 3. HikariCP здесь не обязательно является проблемой

Твои настройки:

```
hikari:
  maximum-pool-size: 30
  connection-timeout: 3000
```

Означают:

- максимум 30 соединений с PostgreSQL;
- если соединение недоступно, поток ждёт до 3 секунд;
- после этого Hikari выбрасывает исключение.

У тебя Bulkhead ограничивает одновременное выполнение метода 20 вызовами.

Если только этот метод использует данный пул, то при нормальной работе он физически не должен занимать все 30 соединений.

Но есть нюанс: другие методы приложения тоже могут обращаться к PostgreSQL и использовать те же соединения.

Поэтому я бы пока не уменьшал Hikari и не увеличивал Bulkhead.

## 4. Есть ещё один момент в твоём коде

Посмотри:

```
public Tasks createTask(Tasks task) {
    tasksCreated.increment();
    return taskRepository.save(task);
}
```

Ты увеличиваешь `tasksCreated` до сохранения задачи.

Если PostgreSQL выбросит исключение, задача не сохранится, но метрика `tasksCreated` уже увеличится.

Я бы изменил так:

```
public Tasks createTask(Tasks task) {
    Tasks savedTask = taskRepository.save(task);
    tasksCreated.increment();
    return savedTask;
}
```

А `tasksFailed` оставил бы в fallback.

Так метрики будут отражать реальные результаты операций.

## 5. Что посмотреть в Grafana

Учитывая, что ты уже настроил Prometheus и Grafana, я бы добавил следующие графики.

Bulkhead

```
resilience4j_bulkhead_available_concurrent_calls
```

Показывает количество свободных слотов Bulkhead.

```
resilience4j_bulkhead_max_allowed_concurrent_calls
```

Показывает максимальное разрешённое количество одновременных вызовов.

HikariCP

```
hikaricp_connections_active
```

Активные соединения.

```
hikaricp_connections_idle
```

Свободные соединения.

```
hikaricp_connections_pending
```

Количество потоков, ожидающих соединение.

```
hikaricp_connections_timeout_total
```

Количество таймаутов получения соединения.

Названия метрик могут немного отличаться в зависимости от версий Micrometer и Resilience4j.

Особенно интересна такая ситуация:

| Метрика            | Значение | Интерпретация             |
| ------------------ | -------- | ------------------------- |
| Bulkhead available | 0        | Все 20 слотов заняты      |
| Hikari active      | 20       | 20 соединений заняты      |
| Hikari idle        | 10       | Есть свободные соединения |
| Hikari pending     | 0        | Нет ожидающих соединения  |

Если увидишь такую картину, это будет сильным аргументом в пользу того, что ограничителем выступает Bulkhead, а не HikariCP.

## 6. Мой главный вопрос: что именно возвращает fallback?

Я хочу проверить одну гипотезу: возможно, значительная часть твоих 64,56% ошибок — это именно `BulkheadFullException`, а не проблемы PostgreSQL.

Добавь временно немного более подробное логирование:

```
public Tasks fallbackCreateTask(Tasks task, Throwable t) {
    log.error("Fallback triggered. Exception type: {}, message: {}",
            t.getClass().getName(),
            t.getMessage());

    tasksFailed.increment();

    throw new ServiceUnavailableException(
            "Не удалось создать задачу, попробуйте позже"
    );
}
```

И посмотри, что преобладает:

- `BulkheadFullException` — не хватает слотов Bulkhead.
- `SQLTransientConnectionException` — проблема с получением соединения Hikari.
- `DataAccessException` — ошибка на уровне Spring Data.
- `CallNotPermittedException` — CircuitBreaker уже открылся и блокирует вызовы.

## 7. Что я хочу увидеть от тебя следующим шагом

Давай пока ничего не будем менять в конфигурации.

Покажи мне две вещи:

1. Какое исключение чаще всего появляется в логах `fallbackCreateTask()` после нагрузочного тестирования?
2. Какой статус возвращается клиенту при ошибке — действительно 503?

Это позволит отделить три совершенно разные ситуации:

- Bulkhead не пропускает запросы.
- Hikari не может выдать соединение.
- PostgreSQL не справляется с выполнением запросов.

И ещё: я бы хотел сохранить твой Bulkhead = 20 на первом этапе. Это хорошая возможность научиться видеть, как работает ограничение конкурентности, и экспериментально определить его влияние на пропускную способность приложения.


> ```
resilience4j_bulkhead_available_concurrent_calls - 20 с падениями
resilience4j_bulkhead_max_allowed_concurrent_calls - 20
hikaricp_connections_active - скачок до 5
hikaricp_connections_idle - 30 с падениями
hikaricp_connections_pending - 0

тест кажется вообще не прошел

  █ THRESHOLDS 

    http_req_duration
    ✓ 'p(95)<500' p(95)=1.87ms

    http_req_failed
    ✗ 'rate<0.05' rate=100.00%


  █ TOTAL RESULTS 

    checks_total.......: 40467  335.625596/s
    checks_succeeded...: 33.32% 13487 out of 40467
    checks_failed......: 66.67% 26980 out of 40467

    ✗ status is 201
      ↳  0% — ✓ 0 / ✗ 13489
    ✗ has task id
      ↳  0% — ✓ 0 / ✗ 13489
    ✗ fast enough
      ↳  99% — ✓ 13487 / ✗ 2

    HTTP
    http_req_duration....: avg=5.53ms min=0s med=738µs max=30s p(90)=1.57ms p(95)=1.87ms
    http_req_failed......: 100.00% 13489 out of 13489
    http_reqs............: 13489   111.875199/s

    EXECUTION
    iteration_duration...: avg=1s     min=1s med=1s    max=31s p(90)=1s     p(95)=1s    
    iterations...........: 13489   111.875199/s
    vus..................: 3       min=3              max=199
    vus_max..............: 200     min=200            max=200

 Вот теперь мы получили гораздо более интересные результаты. И, похоже, обнаружили важную вещь: сейчас твой тест вообще не измеряет производительность PostgreSQL.

Судя по метрикам, запросы практически не доходят до базы данных.

Давай разберём почему.

## 1. Самое важное — метрики Bulkhead

Ты получил:

| Метрика                      | Значение |
| ---------------------------- | -------- |
| Available concurrent calls   | 20       |
| Max allowed concurrent calls | 20       |

Причём `available` всё время остаётся равным 20.

Это означает, что Bulkhead практически не занят. Все 20 разрешённых слотов свободны.

Если бы Bulkhead был узким местом, мы ожидали бы увидеть примерно такую картину:

```
available_concurrent_calls: 20 → 10 → 5 → 0
```

А у тебя:

```
available_concurrent_calls: 20
max_allowed_concurrent_calls: 20
```

То есть гипотеза о том, что именно Bulkhead ограничивает нагрузку, пока не подтверждается.

## 2. Что говорят метрики HikariCP

| Метрика | Твой результат |
| ------- | -------------- |
| Active  | Пик 5          |
| Idle    | 30             |
| Pending | 0              |

Здесь тоже всё достаточно интересно.

- `active = 5` — одновременно использовалось максимум 5 соединений.
- `idle = 30` — свободны все 30 соединений.
- `pending = 0` — ни один поток не ожидал получения соединения.

Получается, PostgreSQL и HikariCP, судя по всему, вообще не испытывают серьёзной нагрузки.

Причём это хорошо согласуется с метриками Bulkhead.

## 3. Теперь посмотрим на результат k6

100% HTTP errors

```
http_req_failed: 100%
```

Но обрати внимание на время ответа:

```
avg=5.53ms
med=738µs
p(95)=1.87ms
```

Это очень быстрые ответы.

Получается следующая ситуация:

- k6 отправляет около 112 запросов в секунду;
- практически все запросы завершаются ошибкой;
- среднее время ответа составляет всего 5,53 мс;
- PostgreSQL практически не загружен;
- Bulkhead не занят.

Это совсем не похоже на перегрузку сервера.

Скорее всего, приложение быстро возвращает ошибку ещё до выполнения SQL-запроса.

Возможные причины:

1. HTTP endpoint возвращает 4xx или 5xx.
2. Ошибка валидации входящего JSON.
3. Ошибка авторизации или фильтрации запроса.
4. Ошибка маршрутизации или обработки запроса.
5. Исключение возникает до вызова `createTask()`.

Но пока это гипотезы.

## 4. Почему P95 прошёл, хотя тест провалился?

Вот здесь есть важный нюанс k6.

У тебя два независимых threshold:

```
thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.05'],
},
```

Первый проверяет скорость ответа. Второй — долю неуспешных HTTP-запросов.

Поэтому вполне возможна ситуация:

- сервер отвечает за 1 мс;
- возвращает HTTP 503;
- проверка скорости проходит;
- проверка успешности запроса проваливается.

Именно это, похоже, и происходит.

## 5. Что нужно сделать сейчас

Я предлагаю временно изменить k6-скрипт, чтобы увидеть реальные ответы сервера.

Сразу после `http.post()` добавь:

```
const res = http.post(url, payload, params);

// Временно выводим первые 10 ответов
if (__ITER < 10 && __VU <= 2) {
    console.log(JSON.stringify({
        vu: __VU,
        iteration: __ITER,
        status: res.status,
        error: res.error,
        error_code: res.error_code,
        body: res.body ? res.body.substring(0, 500) : null
    }));
}
```

Остальной скрипт пока не меняй.

Нам нужно увидеть:

- `status` — реальный HTTP-статус;
- `error` — транспортная ошибка, если есть;
- `body` — сообщение от Spring Boot.

Особенно интересны HTTP 400, 404, 500 и 503.

## 6. Ещё одна проверка

Временно добавь в начало `createTask()`:

```
log.info("ENTERED createTask()");
```

И непосредственно перед сохранением:

```
log.info("SAVING TASK TO POSTGRES");
```

В результате мы сможем сопоставить три события:

1. k6 отправил запрос.
2. Spring Boot вошёл в `createTask()`.
3. Spring Boot начал сохранение в PostgreSQL.

Это позволит понять, где именно обрывается обработка.

## Мой текущий диагноз

Пока я бы оценил ситуацию так:

| Компонент     | Предварительный вывод             |
| ------------- | --------------------------------- |
| Tomcat        | Перегрузка не подтверждена        |
| Bulkhead      | Не выглядит постоянно заполненным |
| HikariCP      | Перегрузка не подтверждена        |
| PostgreSQL    | Признаков исчерпания пула нет     |
| HTTP endpoint | Главный подозреваемый             |

Пришли мне результат с `status`, `error` и `body` из k6.

Пока не меняй `maxConcurrentCalls`, `maximum-pool-size` и `threads.max`.

Мне кажется, сейчас гораздо полезнее найти причину мгновенного отказа, чем пытаться увеличивать производительность системы, которая, возможно, даже не доходит до выполнения бизнес-логики.


> INFO[0010] {"vu":1,"iteration":9,"status":201,"error":"","error_code":0,"body":"{\"title\":\"task-1-9\",\"description\":\"NEW task-1-9\",\"completed\":true,\"id\":71713}"}  source=console
INFO[0010] {"vu":2,"iteration":8,"status":201,"error":"","error_code":0,"body":"{\"title\":\"task-2-8\",\"description\":\"NEW task-2-8\",\"completed\":true,\"id\":71719}"}  source=console
█ THRESHOLDS 
  http_req_duration
  ✓ 'p(95)<500' p(95)=17.01ms
  http_req_failed
  ✓ 'rate<0.05' rate=0.13%
█ TOTAL RESULTS 
  checks_total.......: 40425  334.33196/s
  checks_succeeded...: 99.90% 40387 out of 40425
  checks_failed......: 0.09%  38 out of 40425
  ✗ status is 201
    ↳  99% — ✓ 13457 / ✗ 18
  ✗ has task id
    ↳  99% — ✓ 13457 / ✗ 18
  ✗ fast enough
    ↳  99% — ✓ 13473 / ✗ 2
  HTTP
  http_req_duration..............: avg=7.11ms min=1.43ms med=4.66ms max=966.28ms p(90)=10.97ms p(95)=17.01ms
    { expected_response:true }...: avg=7.08ms min=1.43ms med=4.66ms max=966.28ms p(90)=10.9ms  p(95)=16.77ms
  http_req_failed................: 0.13%  18 out of 13475
  http_reqs......................: 13475  111.443987/s
  EXECUTION
  iteration_duration.............: avg=1s     min=1s     med=1s     max=1.97s    p(90)=1.01s   p(95)=1.01s  
  iterations.....................: 13475  111.443987/s
  vus............................: 3      min=3           max=199
  vus_max........................: 200    min=200         max=200
вот в консоли самого сервиса, я еще добавил вывод сразу после сохранения и до возврата результата из метода
2026-11-06T11:02:26.200Z  INFO 1 --- [tasks-service] [nio-8091-exec-9] dev.folomkin.tasks.service.TaskService   : ENTERED createTask()
2026-11-06T11:02:26.200Z  INFO 1 --- [tasks-service] [nio-8091-exec-9] dev.folomkin.tasks.service.TaskService   : SAVING TASK TO POSTGRES
Hibernate: insert into tasks (completed,description,title) values (?,?,?)
2026-11-06T11:02:26.201Z  INFO 1 --- [tasks-service] [nio-8091-exec-9] dev.folomkin.tasks.service.TaskService   : RETURN TASK


Отлично! Теперь у нас совсем другая картина. И самое главное — мы подтвердили, что приложение действительно выполняет полный цикл: HTTP → Spring Boot → Hibernate → PostgreSQL → HTTP response.

Причём результаты очень хорошие.

## 1. Что изменилось

Сравним предыдущий и последний тесты.

| Метрика         | Предыдущий тест | Последний тест |
| --------------- | --------------- | -------------- |
| HTTP RPS        | 111.9           | 111.4          |
| P95             | 1.87 ms         | 17.01 ms       |
| HTTP errors     | 100%            | 0.13%          |
| Среднее время   | 5.53 ms         | 7.11 ms        |
| Успешность HTTP | 0%              | 99.87%         |

Тест практически успешен

Последний тест прошёл оба threshold:

```
p(95)<500  ✓ 17.01ms
rate<0.05  ✓ 0.13%
```

У тебя 18 неуспешных запросов из 13 475.

Это очень хороший результат для текущего учебного эксперимента.

## 2. Что особенно интересно в логах Spring Boot

Ты показал:

```
ENTERED createTask()
SAVING TASK TO POSTGRES
Hibernate: insert into tasks (...)
RETURN TASK
```

Причём временные метки:

```
11:02:26.200 ENTERED createTask()
11:02:26.200 SAVING TASK TO POSTGRES
11:02:26.201 RETURN TASK
```

Видно, что весь участок от входа в сервис до возврата результата занял примерно 1 мс с точностью до отображаемых миллисекунд.

Но есть важное уточнение: это не означает, что непосредственно SQL INSERT гарантированно выполнился за 1 мс. Логи показывают время между событиями с точностью до миллисекунд, а `repository.save()` может выполнять дополнительные операции ORM.

Тем не менее это хороший признак.

## 3. Почему Bulkhead не показывает загрузку?

Теперь я бы обратил внимание на интересный момент.

У тебя:

- 200 VU в пике;
- около 111 запросов/сек;
- Bulkhead разрешает 20 одновременных вызовов;
- Hikari использует максимум 5 соединений.

Это вполне объяснимо.

Твой метод `createTask()` выполняется очень быстро. Поэтому запросы не успевают накапливаться внутри Bulkhead.

Например, если обработка занимает 5 мс, то для поддержания 111 запросов/сек одновременно будет занято в среднем:

\\[ 111 \times 0.005 = 0.555 \\]

То есть в среднем меньше одного слота Bulkhead!

Поэтому совершенно нормально, что Bulkhead остаётся практически свободным.

## 4. Но почему остались 18 ошибок?

Вот это я бы всё-таки исследовал.

У тебя:

```
http_req_failed: 0.13% 18 out of 13475
```

А в проверках:

```
status is 201: 18 failures
has task id: 18 failures
fast enough: 2 failures
```

Обрати внимание: первые две проверки провалились ровно 18 раз.

Скорее всего, это одни и те же 18 запросов, которые получили статус, отличный от 201.

При этом максимальное время ответа:

```
max=966.28ms
```

То есть есть отдельные запросы, которые выполнялись заметно дольше остальных.

Я бы добавил ещё один диагностический блок:

```
if (res.status !== 201) {
    console.error(JSON.stringify({
        vu: __VU,
        iteration: __ITER,
        status: res.status,
        body: res.body,
        duration: res.timings.duration
    }));
}
```

Так мы узнаем причину оставшихся ошибок.

## 5. Что я предлагаю исследовать дальше

Мне кажется, ты сейчас находишься в очень хорошей точке для изучения нагрузочного тестирования.

Я бы провёл следующий эксперимент.

Эксперимент №1 — проверить Bulkhead.

Временно измени:

```
bulkhead:
  instances:
    postgres:
      maxConcurrentCalls: 5
      maxWaitDuration: 0
```

Остальное оставь без изменений.

Запусти тот же k6-тест.

Что мы ожидаем увидеть?

- Если Bulkhead начнёт ограничивать запросы, `available_concurrent_calls` будет доходить до нуля.
- Появятся `BulkheadFullException`.
- Увеличится количество HTTP 503.
- PostgreSQL при этом может оставаться практически незагруженным.

Это позволит на практике увидеть, как работает Bulkhead.

После эксперимента верни значение 20.

Мой вывод: сейчас твоя конфигурация выглядит вполне работоспособной. Tomcat, HikariCP и Bulkhead пока не демонстрируют признаков насыщения. При этом 99,87% успешных HTTP-запросов и P95 в 17 мс — отличный результат для первого нагрузочного эксперимента.

И я бы не стремился сейчас искусственно улучшать эти цифры. Гораздо интереснее научиться находить точку, в которой каждый из компонентов начинает ограничивать производительность.

> сначала шли хороши запросы со статусом 201, потом пошли вот такие
ERRO[0114] {"vu":39,"iteration":101,"status":503,"body":"{\"error\":\"service_unavailable\",\"message\":\"Не удалось создать задачу, попробуйте позже\"}","duration":5.152}  source=console

>это ответ из fallback

>  throw new ServiceUnavailableException(
                "Не удалось создать задачу, попробуйте позже"
        );

>вот результат


>█ THRESHOLDS 
 http_req_duration
  ✓ 'p(95)<500' p(95)=96.26ms
  http_req_failed
  ✗ 'rate<0.05' rate=9.36%
█ TOTAL RESULTS 
  checks_total.......: 39807  330.740045/s
  checks_succeeded...: 93.59% 37257 out of 39807
  checks_failed......: 6.40%  2550 out of 39807
  ✗ status is 201
    ↳  90% — ✓ 12026 / ✗ 1243
  ✗ has task id
    ↳  90% — ✓ 12026 / ✗ 1243
  ✗ fast enough
    ↳  99% — ✓ 13205 / ✗ 64
  HTTP
  http_req_duration..............: avg=20.27ms min=1.29ms med=4.8ms  max=834.26ms p(90)=24.87ms p(95)=96.26ms
    { expected_response:true }...: avg=12.89ms min=1.29ms med=4.53ms max=834.26ms p(90)=16.2ms  p(95)=30.57ms
  http_req_failed................: 9.36%  1243 out of 13269
  http_reqs......................: 13269  110.246682/s
  EXECUTION
  iteration_duration.............: avg=1.02s   min=1s     med=1s     max=2.25s    p(90)=1.02s   p(95)=1.09s  
  iterations.....................: 13269  110.246682/s
  vus............................: 5      min=4             max=199
  vus_max........................: 200    min=200           max=200\


  Вот теперь мы, кажется, подошли непосредственно к тому, ради чего и проводим этот эксперимент. У тебя появилась интересная закономерность: сначала приложение успешно обрабатывает запросы, а после определённого времени начинает возвращать 503 из fallback.

И это уже похоже на срабатывание одного из механизмов Resilience4j.

## 1. Что изменилось в нагрузке

Сравним результаты:

| Метрика            | Предыдущий тест | Новый тест |
| ------------------ | --------------- | ---------- |
| RPS                | 111.4           | 110.2      |
| P95                | 17 ms           | 96.26 ms   |
| HTTP errors        | 0.13%           | 9.36%      |
| Успешные запросы   | 99.87%          | 90.64%     |
| Максимальное время | 966 ms          | 834 ms     |

Обрати внимание: средний RPS практически не изменился, но P95 вырос более чем в 5 раз.

Это означает, что система стала работать менее стабильно: часть запросов начала обрабатываться значительно дольше, а часть вообще отклоняется.

## 2. Самое интересное — время ответа 503

Ты показал:

```
status: 503
duration: 5.152ms
```

Это очень важная деталь.

Запрос завершился всего за 5 мс. То есть он, скорее всего, не ждал PostgreSQL и не выполнял длительный SQL-запрос.

Я вижу два основных подозреваемых:

Вариант №1. BulkheadFullException

```
@Bulkhead(name = "postgres")
```

У тебя:

```
maxConcurrentCalls: 20
maxWaitDuration: 0
```

Если одновременно уже выполняются 20 вызовов `createTask()`, следующий запрос немедленно получает отказ.

Это хорошо согласуется с быстрым ответом 503.

Вариант №2. CallNotPermittedException

У тебя также включён CircuitBreaker:

```
@CircuitBreaker(name = "postgres", fallbackMethod = "fallbackCreateTask")
```

Если CircuitBreaker переходит в состояние OPEN, он начинает отклонять новые вызовы, не обращаясь к PostgreSQL.

Это тоже может давать очень быстрые 503.

И вот второй вариант я бы не стал недооценивать.

## 3. Сейчас нужно определить, какой именно механизм срабатывает

В твоём fallback уже есть проверка:

```
if (t instanceof BulkheadFullException) {
    log.warn("Bulkhead full: too many concurrent requests");
} else {
    log.error("DB error: {}", t.getMessage(), t);
}
```

Но здесь есть недостаток: `CallNotPermittedException` тоже попадёт в ветку `else` и будет ошибочно обозначен как ошибка БД.

Я предлагаю временно заменить fallback на такой:

```
public Tasks fallbackCreateTask(Tasks task, Throwable t) {

    log.error("Fallback triggered. Exception: {}",
            t.getClass().getName());

    if (t instanceof BulkheadFullException) {
        log.warn("BULKHEAD FULL!");
    }

    if (t instanceof CallNotPermittedException) {
        log.warn("CIRCUIT BREAKER OPEN!");
    }

    if (t instanceof TimeoutException) {
        log.warn("TIMEOUT!");
    }

    tasksFailed.increment();

    throw new ServiceUnavailableException(
            "Не удалось создать задачу, попробуйте позже"
    );
}
```

Не забудь импортировать:

```
import io.github.resilience4j.bulkhead.BulkheadFullException;
import io.github.resilience4j.circuitbreaker.CallNotPermittedException;
```

Для `TimeoutException` используй соответствующий класс в зависимости от того, какой механизм таймаута у тебя настроен.

## 4. Ещё хочу проверить метрики CircuitBreaker

Добавь в Grafana следующие PromQL-запросы:

Состояние CircuitBreaker:

```
resilience4j_circuitbreaker_state
```

Количество неуспешных вызовов:

```
resilience4j_circuitbreaker_failure_rate
```

Количество вызовов:

```
resilience4j_circuitbreaker_calls
```

Последняя метрика может потребовать группировки по `kind`, если она экспортируется с соответствующей меткой.

Нас особенно интересует переход CircuitBreaker в OPEN.

Кстати, стандартные настройки Resilience4j CircuitBreaker обычно включают:

- sliding window size = 100;
- minimum number of calls = 100;
- failure rate threshold = 50%;
- wait duration in open state = 60 секунд.

Если ты не переопределял эти параметры, их тоже стоит учитывать.

## 5. Моя гипотеза

Сейчас я бы рассматривал такую последовательность:

1. Начало теста

   Приложение успешно принимает запросы и сохраняет задачи в PostgreSQL.
2. Рост нагрузки

   Время выполнения запросов увеличивается, растёт конкуренция за ресурсы.
3. Срабатывание защитного механизма

   Bulkhead отклоняет лишние вызовы либо CircuitBreaker переходит в OPEN.
4. Fallback

   Приложение возвращает 503 практически мгновенно.

Это пока гипотеза, а не установленная причина.

## 6. Что хочу попросить тебя сделать

Пока не меняй настройки.

Проведи ещё один тест с теми же 200 VU и добавленным логированием типа исключения в fallback.

Мне нужны два результата:

1. Какие исключения преобладают: `BulkheadFullException`, `CallNotPermittedException` или что-то другое?
2. Что показывает `resilience4j_circuitbreaker_state` в момент появления первых 503?

Почему я настаиваю именно на этом?

Потому что Bulkhead и CircuitBreaker решают разные задачи:

- Bulkhead ограничивает количество одновременно выполняемых операций.
- CircuitBreaker прекращает вызовы к потенциально неисправной зависимости на определённое время.

И было бы очень полезно на твоём учебном проекте увидеть работу обоих механизмов отдельно, а затем вместе.

> итак, в логах приложения вот что
Fallback triggered. Exception: io.github.resilience4j.bulkhead.BulkheadFullException
2026-11-06T12:04:18.960Z  WARN 1 --- [tasks-service] [io-8091-exec-29] dev.folomkin.tasks.service.TaskService   : BULKHEAD FULL!
2026-11-06T12:04:18.961Z ERROR 1 --- [tasks-service] [io-8091-exec-54] d.f.t.exception.GlobalExceptionHandler   : Service unavailable: Не удалось создать задачу, попробуйте позже

>результат теста такой:

	█ THRESHOLDS 

	  http_req_duration
	  ✓ 'p(95)<500' p(95)=169.77ms

	  http_req_failed
	  ✗ 'rate<0.05' rate=10.51%


	█ **TOTAL RESULTS** 

	  checks_total.......: 39528  327.289165/s
	  checks_succeeded...: 92.69% 36641 out of 39528
	  checks_failed......: 7.30%  2887 out of 39528

	  ✗ status is 201
	    ↳  89% — ✓ 11790 / ✗ 1386
	  ✗ has task id
	    ↳  89% — ✓ 11790 / ✗ 1386
	  ✗ fast enough
	    ↳  99% — ✓ 13061 / ✗ 115

	  **HTTP**
	  http_req_duration..............: avg=29.46ms min=1.29ms med=4.62ms max=1.08s p(90)=50.1ms  p(95)=169.77ms
	    { expected_response:true }...: avg=15.45ms min=1.29ms med=4.27ms max=1.08s p(90)=21.22ms p(95)=49.53ms 
	  http_req_failed................: 10.51% 1386 out of 13176
	  http_reqs......................: 13176  109.096388/s

	  **EXECUTION**
	  iteration_duration.............: avg=1.03s   min=1s     med=1s     max=2.08s p(90)=1.05s   p(95)=1.17s   
	  iterations.....................: 13176  109.096388/s
	  vus............................: 5      min=4             max=199
	  vus_max........................: 200    min=200           max=200

	  **настройки **
	circuitbreaker: #-> Соответствует @CircuitBreaker(name = "postgres", fallbackMethod = "fallbackCreateTask")&#xA;  instances:
	    postgres:
	      registerHealthIndicator: true
	      slidingWindowSize: 10 # Сколько последних вызовов учитывать. 100 для прода&#xA;      minimumNumberOfCalls: 5 # Минимум вызовов до расчёта процента ошибок. 20 для прода&#xA;      failureRateThreshold: 50 # Порог ошибок (%), после которого цепь размыкается. % ошибок для размыкания. Для прода - 30&#xA;      waitDurationInOpenState: 10s # Сколько ждать перед попыткой закрыть цепь. Сколько сидеть в OPEN. Для прода - 30.&#xA;      permittedNumberOfCallsInHalfOpenState: 3 # Сколько тестовых вызовов разрешить в полуоткрытом состоянии: все 3 успешны - CLOSED, хоть один failed - OPEN (снова), для прода 5&#xA;      automaticTransitionFromOpenToHalfOpenEnabled: true # авто-переход&#xA;      slowCallRateThreshold: 50           # если 50% вызовов медленные&#xA;      slowCallDurationThreshold: 2s       # что считать "медленным"\


Отлично! Теперь у нас есть подтверждение, а не просто предположение.

Причина 503 найдена

Твой Bulkhead действительно переполняется. CircuitBreaker в данном случае не является непосредственной причиной отказов.

Вот ключевая строка:

```
Fallback triggered. Exception:
io.github.resilience4j.bulkhead.BulkheadFullException
```

Это означает, что Resilience4j не разрешил очередному запросу войти в `createTask()`, потому что все разрешённые места уже заняты.

И теперь самое интересное — почему это происходит, если у тебя 200 VU, а Bulkhead разрешает 20 одновременных вызовов?

## 1. Разбираем происходящее

У тебя:

```
bulkhead:
  instances:
    postgres:
      maxConcurrentCalls: 20
      maxWaitDuration: 0
```

И:

```
@CircuitBreaker(name = "postgres", fallbackMethod = "fallbackCreateTask")
@Bulkhead(name = "postgres")
public Tasks createTask(Tasks task) {
    Tasks savedTask = taskRepository.save(task);
    tasksCreated.increment();
    return savedTask;
}
```

Важный момент: Bulkhead ограничивает не количество запросов в секунду, а количество одновременно выполняющихся вызовов метода.

Например:

- 20 вызовов уже находятся внутри `createTask()`;
- приходит 21-й;
- Bulkhead немедленно отказывает ему;
- вызывается fallback;
- клиент получает HTTP 503.

При `maxWaitDuration: 0` очередь ожидания отсутствует.

## 2. Почему Bulkhead переполняется при быстром PostgreSQL?

Посмотрим на твой тест:

| Параметр      | Значение  |
| ------------- | --------- |
| Максимум VU   | 200       |
| RPS           | 109       |
| P95           | 169.77 ms |
| Среднее время | 29.46 ms  |
| Ошибки        | 10.51%    |

Здесь есть интересная математическая зависимость.

По закону Литтла:

	L = \lambda * W

Где:

- \\(L\\) — среднее количество одновременно выполняемых запросов;
- \\(\lambda\\) — интенсивность поступления запросов;
- \\(W\\) — среднее время обработки.

Если взять твои показатели:

\\[ L = 109 \times 0.02946 \approx 3.21 \\]

То есть среднее количество одновременно обрабатываемых HTTP-запросов — около 3,2.

Но это не означает, что Bulkhead не может заполниться!

Почему?

Потому что:

- 29,46 мс — среднее время всех HTTP-запросов;
- Bulkhead защищает конкретный участок приложения;
- распределение времени выполнения может иметь длинный хвост;
- в моменты пиков некоторые запросы могут выполняться значительно дольше.

Например, ты видишь максимальное время 1,08 секунды. Если несколько таких операций совпадут по времени, они могут занять все 20 слотов.

Но здесь я хочу обратить внимание на один нюанс: среднее значение 3,2 само по себе не объясняет заполнение 20 слотов. Нам нужно посмотреть фактическую динамику конкурентности, а не делать вывод только по среднему времени.

## 3. Есть ещё одна потенциальная проблема — CircuitBreaker

Вот твои настройки:

```
circuitbreaker:
  instances:
    postgres:
      slidingWindowSize: 10
      minimumNumberOfCalls: 5
      failureRateThreshold: 50
      waitDurationInOpenState: 10s
      permittedNumberOfCallsInHalfOpenState: 3
      automaticTransitionFromOpenToHalfOpenEnabled: true
      slowCallRateThreshold: 50
      slowCallDurationThreshold: 2s
```

Для экспериментов они вполне подходят, но есть важный нюанс.

У тебя `minimumNumberOfCalls: 5`.

Это означает, что CircuitBreaker начинает рассчитывать процент ошибок уже после пяти учитываемых вызовов.

А `failureRateThreshold: 50` означает, что при достижении 50% ошибок он может перейти в OPEN.

Важный момент: `BulkheadFullException` тоже может учитываться CircuitBreaker как ошибка, в зависимости от порядка выполнения аспектов Resilience4j и настроек обработки исключений.

Поэтому нельзя пока считать, что CircuitBreaker вообще не участвует в происходящем.

У тебя потенциально может получиться цепочка:

1. Bulkhead переполняется.
2. Возникают `BulkheadFullException`.
3. CircuitBreaker учитывает эти исключения как неуспешные вызовы.
4. CircuitBreaker достигает порога ошибок.
5. Переходит в OPEN.
6. Начинает отклонять вызовы уже самостоятельно.

Это стоит проверить отдельно.

## 4. Важное уточнение: частота сбора метрик Grafana

У тебя Prometheus настроен со scrape interval 15 секунд (из предыдущей настройки).

Это может быть причиной того, что ты не видишь заполнения Bulkhead на графике.

Представь:

- в течение 200 мс все 20 слотов заняты;
- следующие 14,8 секунды Bulkhead свободен;
- Prometheus снимает показания раз в 15 секунд.

На графике ты можешь вообще не увидеть момент перегрузки.

Для следующего эксперимента временно установи:

```
global:
  scrape_interval: 1s
```

И смотри на:

```
resilience4j_bulkhead_available_concurrent_calls
```

Особенно интересно минимальное значение за период:

```
min_over_time(
  resilience4j_bulkhead_available_concurrent_calls[1m]
)
```

Если оно достигает нуля, мы увидим кратковременное исчерпание слотов.

## 5. Я бы ещё немного скорректировал CircuitBreaker

Нашёл важную особенность в документации Resilience4j.

При стандартном порядке Spring AOP аспектов CircuitBreaker оборачивает Bulkhead:

```
CircuitBreaker
    |
    v
Bulkhead
    |
    v
createTask()
```

Поэтому `BulkheadFullException` может учитываться CircuitBreaker как ошибка. Это зависит от конфигурации исключений.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://resilience4j.readme.io\&sz=32)

resilience4j+1



У тебя довольно чувствительные настройки:

```
slidingWindowSize: 10
minimumNumberOfCalls: 5
failureRateThreshold: 50
```

Поэтому я бы рекомендовал отдельно исключить `BulkheadFullException` из статистики CircuitBreaker, если твоя цель — отслеживать именно здоровье PostgreSQL.

Например, в конфигурации CircuitBreaker можно использовать:

```
circuitbreaker:
  instances:
    postgres:
      ignoreExceptions:
        - io.github.resilience4j.bulkhead.BulkheadFullException
```

Тогда отказы Bulkhead не будут увеличивать failure rate CircuitBreaker.

Это разделит две ситуации:

- Bulkhead защищает приложение от чрезмерной конкурентности.
- CircuitBreaker реагирует на проблемы PostgreSQL.

Проверить фактическое поведение после изменения можно по метрикам CircuitBreaker и его событиям.

## 6. Какой эксперимент я предлагаю

Я бы сейчас не увеличивал Bulkhead до 50 или 100. Нам нужно сначала понять, почему он достигает предела.

Сделай три последовательных теста:

| Эксперимент | Bulkhead | VU  |
| ----------- | -------- | --- |
| A           | 5        | 200 |
| B           | 20       | 200 |
| C           | 50       | 200 |

Для каждого запиши:

- RPS;
- P95;
- процент HTTP 503;
- количество `BulkheadFullException`;
- минимум `available_concurrent_calls`;
- максимум `hikaricp_connections_active`.

Так мы увидим, как изменение конкурентности влияет на пропускную способность.

И ещё один вопрос, который существенно поможет диагностике:

У тебя в Kubernetes у `tasks-service` сколько реплик?

Одна (`replicas: 1`) или несколько?

Это важно, потому что Bulkhead создаётся внутри экземпляра приложения. Если у тебя две реплики, то `maxConcurrentCalls: 20` означает до 20 вызовов на каждой реплике, а не 20 на весь Kubernetes Deployment.

Именно поэтому перед изменением лимитов я хочу знать количество реплик.
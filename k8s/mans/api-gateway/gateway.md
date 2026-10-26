# Маршрутизация трафика извне: Istio -> Spring Cloud Gateway.

Вы задали отличный вопрос, который часто вызывает путаницу при переходе от чистой сети Istio к гибридной схеме со Spring Cloud Gateway. Давайте разложим всё по полочкам: что нужно оставить, а что — изменить.

## 1. Нужен ли старый gateway.yaml?
Да, он обязательно нужен и остаётся без изменений!
Этот файл описывает физическое открытие портов кластера наружу (в нашем случае — 80-й порт). Он по-прежнему служит «парадными воротами» для всего входящего трафика с вашего компьютера (localhost). Удалять или дублировать его не нужно. Он остаётся один на весь кластер.

## 2. Что происходит с VirtualService для каждого микросервиса?
Вот здесь происходит кардинальное изменение. Старые внешние виртуальные сервисы для tasks-service, users-service и notifications-service (в которых в поле gateways был указан app-gateway) вам больше не нужны — их нужно удалить.

* Почему? Потому что мы больше не хотим, чтобы Istio напрямую пускал трафик с улицы в эти сервисы. Теперь мы хотим, чтобы все внешние запросы шли строго через одно Java-приложение шлюза.


## 3. Какой VirtualService теперь должен быть?
Теперь у вас будет всего один внешний VirtualService, написанный специально для api-gateway. Он заберёт на себя роль диспетчера на входе и перенаправит все запросы с префиксом /api/ на порт 8080 вашего нового Spring-шлюза.
Удалите старые внешние файлы VirtualService микросервисов и примените вместо них один общий файл k8s/api-gateway-vs.yaml:

apiVersion: networking.istio.io/v1alpha3kind: VirtualServicemetadata:
  name: api-gateway-external-vs # 👈 Одно общее имя для входной маршрутизации
  namespace: defaultspec:
  hosts:
    - "localhost" # Или "*" (какой адрес вводите в Postman)
  gateways:
    - app-gateway # 🔗 Подключаемся к вашему единственному неизменному gateway.yaml
  http:
    - match:
        # Ловим абсолютно все запросы, которые начинаются с /api/
        # Например: /api/tasks, /api/users, /api/notifications
        - uri:
            prefix: /api
      route:
        - destination:
            # 🚀 Перенаправляем весь этот поток на ваш новый Spring Cloud Gateway!
            host: spring-cloud-gateway-service.default.svc.cluster.local
            port:
              number: 8080 # Порт, который мы указали в манифесте api-gateway.yaml


## 🗺️ Итоговая схема движения трафика:

   1. Внешний мир: Запрос http://localhost/api/tasks прилетает в кластер.
   2. Gateway (app-gateway): Принимает этот запрос на 80-м порту и передает его во внутреннюю сеть Istio.
   3. Новый VirtualService (api-gateway-external-vs): Видит префикс /api, перехватывает запрос и отправляет его в под api-gateway на порт 8080.
   4. Spring Cloud Gateway (код Java): Принимает запрос /api/tasks, прокручивает через ваши Java-фильтры (авторизация, логи), с помощью StripPrefix=1 отрезает /api и превращает в чистый /tasks.
   5. Внутренняя сеть: Шлюз отправляет запрос на внутренний адрес http://cluster.local, и тот успешно отдаёт данные.

## Резюме:

* gateway.yaml — оставляем (он один открывает порт 80).
* Старые внешние VirtualService микросервисов — удаляем (закрываем прямой доступ с улицы).
* Манифест api-gateway-external-vs.yaml — создаем и применяем (он становится единственным мостом между Istio Gateway и вашим Spring-шлюзом).

Понятна ли эта схема перенаправления? Удалось ли вам удалить старые виртуальные сервисы и применить новый api-gateway-external-vs.yaml? Напишите, если готовы отправить тестовый запрос через этот обновленный конвейер!

# Маппинг маргрутов

В Spring Cloud Gateway сопоставление путей с реальными адресами и портами микросервисов происходит через три ключевых параметра в файле application.yml: Path (предикат), uri (цель) и StripPrefix (фильтр).
Давайте разберем на конкретном примере, как именно движок Spring Cloud Gateway превращает строчки из конфига в реальный сетевой запрос к вашему контейнеру.

## 1. Как устроен маппинг в application.yml
Посмотрим на один из ваших маршрутов:

spring:
  cloud:
    gateway:
      routes:
        - id: tasks-service-route
          uri: http://cluster.local # 👈 2. КУДА слать (Адрес и Порт)
          predicates:
            - Path=/api/tasks/**                               # 👈 1. ЧТО ловить (Входящий адрес)
          filters:
            - StripPrefix=1                                    # 👈 3. КАК изменить путь


## 2. Пошаговая механика сопоставления (Под капотом)
Когда запрос долетает до шлюза, реактивный сервер Netty (на котором работает Spring Cloud Gateway) запускает следующий конвейер:
## Шаг А. Поиск совпадения (Предикат Path)
Шлюз берет URI входящего запроса (например, /api/tasks/create) и начинает по очереди сверять его со всеми маршрутами в списке routes.

* Конструкция /api/tasks/** означает: «Поймай любой запрос, который начинается точно с этих символов, независимо от того, сколько слэшей идет дальше».
* Условие выполнено ➔ Шлюз выбирает этот маршрут (tasks-service-route).

## Шаг Б. Изменение пути (Фильтр StripPrefix)
Шлюз понимает, что в самом tasks-service нет префикса /api. Поэтому включается фильтр StripPrefix=1.

* Он считает блоки между слэшами. Первый блок — это api. Второй — tasks. Третий — create.
* Цифра 1 приказывает: отрезать ровно 1 первый блок.
* Исходный путь /api/tasks/create в памяти шлюза превращается в /tasks/create.

## Шаг В. Склейка и отправка (Параметр uri)
Теперь шлюз берет базовый адрес, указанный в строке uri, и приклеивает к нему измененный на Шаге Б хвост.

* Базовый URI: http://cluster.local
* Хвост: /tasks/create
* Итоговый результат: http://cluster.local


## 3. Как этот адрес превращается в конкретный Под и Порт?
Когда Spring Cloud Gateway выполняет этот итоговый HTTP-запрос, управление переходит к Kubernetes DNS и Сети:

   1. Разрешение имени (DNS): Шлюз отправляет запрос на имя tasks-service.default.svc.cluster.local. Встроенный DNS-сервер кластера (CoreDNS) переводит это имя в виртуальный IP-адрес обычного Kubernetes Service (например, 10.96.0.45).
   2. Поиск порта: Шлюз стучится на порт 80, потому что именно этот порт мы явно указали в uri: ...:80.
   3. Проброс в контейнер (targetPort): Обычный Kubernetes Service принимает запрос на порту 80, смотрит в свои настройки и перенаправляет трафик на реальный IP-адрес пода tasks-app на порт 8091 (ваш targetPort), где его окончательно принимает и обрабатывает ваш Spring Boot контроллер.

## 💡 Важное замечание по поводу портов:
Если вы укажете в uri шлюза порт самого приложения (http://tasks-service...:8091), запрос пойдет в обход правил Kubernetes Service. В рамках Istio Mesh это может привести к ошибкам маршрутизации, так как Istio ожидает, что трафик идет через стандартные порты сервисов (в нашем случае — порт 80). Поэтому схема Шлюз (порт 80) ➔ K8s Service (порт 80) ➔ Под приложения (порт 8091) является самой надежной и правильной.

Теперь логика сопоставления путей и портов стала понятна? Удалось ли вам скорректировать VirtualService в Istio, чтобы сдвинуть дело с мертвой точки и заставить шлюз наконец-то увидеть ваши запросы?

# Еще

Понимаю вашу задачу. Настроить сквозной поток трафика в Kubernetes с Istio, Spring Cloud Gateway и вашим `tasks-service` — это классический сценарий. Давайте разберем его по шагам, чтобы у вас сложилась целостная картина.

### 🗺️ Общая архитектура потока трафика

Согласно учебнику, ваша цепочка будет выглядеть так:

1.  **Внешний клиент** отправляет запрос на входную точку кластера.
2.  **Istio Ingress Gateway** принимает трафик и, благодаря ресурсу `Gateway`, направляет его на **Spring Cloud Gateway**.
3.  **Spring Cloud Gateway** получает запрос и, используя свои правила маршрутизации, перенаправляет его на соответствующий микросервис, в вашем случае — на **`tasks-service`**.
4.  **`tasks-service`** обрабатывает запрос и возвращает ответ.

Важно понимать, что в этой связке **Istio Ingress Gateway** и **Spring Cloud Gateway** выполняют разные роли. Istio — это *инфраструктурный* шлюз (L4-L7), отвечающий за вход в сервисную сеть, mTLS и базовую маршрутизацию. Spring Cloud Gateway — это *прикладной* шлюз (L7), который предоставляет богатые возможности для фильтрации, аутентификации и маршрутизации на уровне приложений.

### ⚙️ Необходимые конфигурации Kubernetes

Для реализации этой схемы вам понадобятся следующие манифесты.

#### 1. `tasks-service` (Deployment и Service)

Это ваш целевой микросервис. Ему нужны стандартные Deployment и Service для работы внутри кластера.

**`tasks-service-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tasks-service
  labels:
    app: tasks-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tasks-service
  template:
    metadata:
      labels:
        app: tasks-service
    spec:
      containers:
        - name: tasks-service
          image: your-registry/tasks-service:latest
          ports:
            - containerPort: 8080 # Укажите порт вашего приложения
---
apiVersion: v1
kind: Service
metadata:
  name: tasks-service
spec:
  selector:
    app: tasks-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

#### 2. Spring Cloud Gateway (Deployment и Service)

Это приложение-шлюз, которое будет принимать трафик от Istio и маршрутизировать его на `tasks-service`.

**`spring-cloud-gateway-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-cloud-gateway
  labels:
    app: spring-cloud-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-cloud-gateway
  template:
    metadata:
      labels:
        app: spring-cloud-gateway
    spec:
      containers:
        - name: spring-cloud-gateway
          image: your-registry/spring-cloud-gateway:latest
          ports:
            - containerPort: 8080 # Порт, на котором слушает ваш Gateway
---
apiVersion: v1
kind: Service
metadata:
  name: spring-cloud-gateway
spec:
  selector:
    app: spring-cloud-gateway
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

#### 3. Istio Gateway и VirtualService

Эти ресурсы настраивают входную точку в сервисную сеть.

- **`Gateway`** определяет, на каких портах и хостах Istio Ingress Gateway будет принимать внешний трафик.
- **`VirtualService`** связывает этот `Gateway` с вашим внутренним сервисом (`spring-cloud-gateway`).

**`istio-gateway.yaml`**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: tasks-app-gateway
spec:
  selector:
    istio: ingressgateway # Стандартный селектор для Istio Ingress Gateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*" # Или укажите конкретный хост, например, "tasks.example.com"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: spring-cloud-gateway-vs
spec:
  hosts:
    - "*" # Должно соответствовать hosts в Gateway
  gateways:
    - tasks-app-gateway
  http:
    - route:
        - destination:
            host: spring-cloud-gateway # Имя Service для Spring Cloud Gateway
            port:
              number: 80
```

### 🧩 Настройка Spring Cloud Gateway

Теперь самое главное — научить Spring Cloud Gateway перенаправлять трафик на `tasks-service`. В Kubernetes-окружении для этого лучше всего использовать интеграцию с `DiscoveryClient` через `spring-cloud-kubernetes`.

1.  **Добавьте зависимость** в `pom.xml` вашего Gateway-приложения:
    ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
    </dependency>
    ```

2.  **Включите автоматическое создание маршрутов** в `application.yml`:
    ```yaml
    spring:
      cloud:
        gateway:
          discovery:
            locator:
              enabled: true
              lower-case-service-id: true # Рекомендуется для Kubernetes
    ```
    Эта настройка заставляет Gateway автоматически создавать маршруты на основе сервисов, зарегистрированных в Kubernetes. Например, для сервиса `tasks-service` автоматически появится маршрут вида `/tasks-service/**`, который будет проксировать запросы на `http://tasks-service`.

3.  **Проверьте маршрутизацию**. Теперь, когда запрос дойдет до Spring Cloud Gateway, он сможет найти `tasks-service` по его DNS-имени внутри кластера (например, `tasks-service.default.svc.cluster.local`) и перенаправить трафик на него.

### 🔧 Финальные штрихи

*   **Внедрение Sidecar-прокси**: Убедитесь, что для ваших пространств имен включена автоматическая инъекция sidecar-прокси Istio. Это необходимо для работы сервисной сети и mTLS:
    ```bash
    kubectl label namespace default istio-injection=enabled
    ```
    После этого все новые поды в пространстве имен `default` (или другом, где вы разворачиваете приложения) будут автоматически получать Envoy-прокси.

*   **Проверка**: После применения всех манифестов проверьте статус подов (`kubectl get pods`) и убедитесь, что у контейнеров `READY` указано `2/2` (основное приложение + sidecar).

Надеюсь, эта схема поможет вам собрать все воедино. Если на каком-то из этапов возникнут вопросы, спрашивайте.


--- 
Разберем оба вопроса по порядку.

## 📝 Настройка `application.yaml` для Spring Cloud Gateway

Да, настраивать нужно, и это ключевой момент. У вас есть **два подхода** — выберите один.

### Вариант A: Автоматическая маршрутизация через Discovery (проще)

Если вы добавили зависимость `spring-cloud-starter-kubernetes-client`, включите автоматический локатор:

**`application.yaml`**
```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
      routes:
        - id: tasks-service-route
          uri: lb://tasks-service
          predicates:
            - Path=/tasks/**
```

**Что здесь важно:**
- `uri: lb://tasks-service` — `lb://` означает "load-balanced", имя `tasks-service` должно **точно совпадать** с именем Kubernetes Service, который вы создали.
- `Path=/tasks/**` — все запросы, начинающиеся с `/tasks/`, пойдут в `tasks-service`.
- **Важно:** Spring Cloud Gateway **не переписывает путь** по умолчанию. То есть запрос `/tasks/create` придет в `tasks-service` как `/tasks/create`. Это как раз то, что нужно для вашего `@RequestMapping("/tasks") @PostMapping("/create")`.

### Вариант B: Явные маршруты без Discovery

Если не хотите использовать Discovery:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: tasks-service-route
          uri: http://tasks-service.default.svc.cluster.local
          predicates:
            - Path=/tasks/**
```

Здесь используется полное DNS-имя сервиса в Kubernetes.

### Дополнительные полезные настройки

```yaml
server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,gateway
```

Последнее позволит смотреть на `http://<gateway>/actuator/gateway/routes` — очень помогает при отладке маршрутов.

---

## 🌐 По какому адресу отправлять запросы

Полный путь запроса зависит от того, **откуда** вы его отправляете.

### Сценарий 1: Изнутри кластера (для отладки, `kubectl port-forward`)

Можно обратиться **напрямую** к Spring Cloud Gateway, минуя Istio:

```bash
kubectl port-forward svc/spring-cloud-gateway 8081:80
```

Затем:
```
POST http://localhost:8081/tasks/create
```

Это удобно для проверки самого Gateway.

### Сценарий 2: Снаружи, через Istio Ingress Gateway (правильный путь)

Это как раз то, что описано в вашем учебнике. Сначала узнайте внешний адрес Ingress Gateway:

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Смотрите на колонку `EXTERNAL-IP` (или `PORT(S)`, если это NodePort / LoadBalancer).

**Если у вас minikube:**
```bash
minikube tunnel   # в отдельном терминале
# или
minikube service istio-ingressgateway -n istio-system --url
```

**Если NodePort** — используйте `http://<IP-ноды>:<NodePort>`.

**Если LoadBalancer** — `http://<EXTERNAL-IP>`.

Затем отправляйте запрос:
```
POST http://<istio-ingress-address>/tasks/create
```

**Почему именно `/tasks/create`, а не `/api-gateway/tasks/create`?**  
Потому что в `VirtualService` вы настроили, что Istio Ingress Gateway направляет **весь** входящий трафик на сервис `spring-cloud-gateway`. А уже сам Spring Cloud Gateway по своему маршруту `Path=/tasks/**` отправляет в `tasks-service`. То есть путь `/tasks/create` проходит через всю цепочку без изменений:

```
Клиент → Istio Gateway (/) → Spring Cloud Gateway (/tasks/**) → tasks-service (/tasks/create)
```

### Пример с curl

```bash
curl -X POST http://<istio-ingress-address>/tasks/create \
  -H "Content-Type: application/json" \
  -d '{"title": "test task"}'
```

---

## ⚠️ Частая ошибка с путями

Многие путаются в префиксах. Смотрите на контроллер:

```java
@RestController
@RequestMapping("/tasks")
public class TasksController {
    @PostMapping("/create")
    public Task create() { ... }
}
```

Полный путь внутри приложения: **`/tasks/create`**.

Значит, и в `Path=` предикате Gateway должен быть префикс, который **не обрезается**, а совпадает с этим путем. Следующее будет **неправильно**:

```yaml
predicates:
  - Path=/api/tasks/**
filters:
  - RewritePath=/api/tasks/(?<segment>.*), /tasks/$\{segment}
```

Это рабочий вариант, но он требует фильтра перезаписи пути. Если не хотите усложнять — используйте `Path=/tasks/**` без фильтров, как в моем примере выше.

---

## 🔎 Как отладить, если не работает

1. **Проверьте маршруты Gateway:**
   ```bash
   kubectl port-forward svc/spring-cloud-gateway 8081:80
   curl http://localhost:8081/actuator/gateway/routes
   ```

2. **Проверьте доступность tasks-service изнутри пода Gateway:**
   ```bash
   kubectl exec -it deploy/spring-cloud-gateway -- curl http://tasks-service/tasks/create -X POST
   ```

3. **Проверьте логи Istio:**
   ```bash
   istioctl proxy-config routes deploy/istio-ingressgateway -n istio-system
   ```

4. **Проверьте конфигурацию sidecar у Gateway:**
   ```bash
   istioctl proxy-config cluster deploy/spring-cloud-gateway
   ```

Если после проверки маршрутов `curl` напрямую к Gateway работает, а через Istio Ingress — нет, значит проблема в `Gateway`/`VirtualService`. Если не работает и напрямую — проблема в маршрутах Spring Cloud Gateway.


# loadbalancer

Хороший вопрос — и очень правильный. Разберу по частям, потому что здесь смешались **две разные проблемы**: одна была настоящей, другая — просто следствие асинхронного прогрева.

## 🎯 Короткий ответ

**Зависимость `spring-cloud-starter-loadbalancer` была настоящим фиксом.** Двухминутная задержка — это отдельный эффект «холодного старта» нескольких асинхронных слоёв, который существовал всегда, просто раньше был замаскирован 503-й из-за отсутствия LB.

## 🔍 Почему `loadbalancer` — это не «просто совпадение»

Без `spring-cloud-starter-loadbalancer` в classpath Spring Cloud не регистрирует **никакой реализации** `ReactorServiceInstanceLoadBalancer`. Когда Gateway видит `uri: lb://tasks-service`, он обращается к балансировщику — а тот либо отсутствует (NoopLoadBalancer), либо всегда возвращает пустой список инстансов.

Прямой DNS работал, потому что он **не использует LB вообще** — просто резолвит hostname и идёт напрямую. Это как раз и подтвердило, что дело не в сети, а именно в LB-звене.

Если бы дело было только в тайминге, `lb://` заработал бы через 2 минуты **и без** добавления зависимости. Но вы этого не проверили (и правильно, потому что пришлось бы ждать дважды).

## ⏱️ Откуда берутся 2 минуты

Даже с правильным `loadbalancer` первый запрос после старта пода идёт «по холодной цепочке». Вот что должно прогреться:

| Слой | Что синхронизируется | Типичное время |
|---|---|---|
| **Spring Cloud Kubernetes Informer** | Начальный List+Watch `endpoints` и `services` через K8s API | 10–60 сек |
| **LoadBalancer cache** | Первое заполнение инстансами (по умолчанию TTL 35 сек) | до 35 сек |
| **Istio Envoy (sidecar Gateway)** | EDS: получение endpoints `tasks-service` от istiod | 5–30 сек |
| **Istio Ingress Envoy** | Синхронизация маршрутов через xDS | несколько секунд |
| **Spring Boot Context** | Подъём Netty, Gateway, Reactor | 5–15 сек |

Итого — 1.5–2.5 минуты **после того, как под `Running`**, — это нормальное поведение для Spring Cloud Kubernetes. Проблема в том, что `Running` **не значит `Ready`**.

## ⚠️ Ключевой нюанс: `Running` ≠ `Ready` ≠ «готов обслуживать»

Ваш под переходит в `Running` раньше, чем:
- Kubernetes Informer получил первую порцию endpoints
- LB-кэш заполнился
- Envoy-sidercar синхронизировал конфигурацию

**Пока всё это не готово**, любой внешний запрос получит 503. Это и есть тот самый «слепой период» в 2 минуты, который вы наблюдали.

## 🛠️ Как это исправить правильно

### 1. Добавьте `startupProbe` в Deployment

Он не даст поду перейти в `Ready`, пока приложение по-настоящему не готово:

```yaml
spec:
  template:
    spec:
      containers:
        - name: spring-cloud-gateway
          startupProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30
```

Тогда Kubernetes **не будет** пускать трафик на под, пока readiness не `UP`, — и вы не увидите 503 в Postman.

### 2. Включите readiness-группу в actuator

В `application.yaml`:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    readinessstate:
      enabled: true
```

Теперь `/actuator/health/readiness` вернёт `UP` только когда все критичные компоненты (включая Discovery и LB) готовы.

### 3. Уменьшите LB-кэш для отладки

По умолчанию кэш LB — 35 секунд. Для отладки можно уменьшить:

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 5s
```

В production оставьте 30+ секунд — меньше нагрузки на API-сервер.

### 4. Включите `holdApplicationUntilProxyStarts`

Это аннотация Istio, которая заставляет приложение ждать старта sidecar:

```yaml
metadata:
  annotations:
    proxy.istio.io/config: |
      holdApplicationUntilProxyStarts: true
```

Помогает в обратном сценарии: когда приложение стартует быстрее Envoy и пытается обратиться наружу раньше, чем sidecar готов.

## 📌 Как в будущем отличать тайминг от реальной ошибки

Простой чек-лист:

1. **Под `Running`** → подождите 30 секунд, отправьте запрос.
2. Если 503 всё ещё — **`kubectl logs deploy/spring-cloud-gateway -c spring-cloud-gateway | grep -i "loadbalancer\|no instance"`**.
   - Есть `No servers available for service` — LB действительно пуст, это баг.
   - Тишина или `Loaded instance: ...` — просто прогрев, ещё чуть-чуть.
3. Проверьте **`/actuator/health/readiness`** — если `OUT_OF_SERVICE`, приложение само говорит «я не готов».

## 💎 Итог

- **Зависимость `loadbalancer` — настоящий фикс.** Без неё `lb://` не работал бы никогда, сколько ни ждать.
- **2 минуты — это нормальный холодный старт** Spring Cloud Kubernetes + Istio в Minikube. Он был и раньше, просто маскировался под 503.
- **Правильное решение** — `startupProbe` + readiness-группа actuator, чтобы Kubernetes не пускал трафик до полной готовности.

Это, кстати, **очень типичная production-задача**: не пускать трафик на под, пока он не прогрелся. В крупных кластерах без `startupProbe` вы бы увидели «мигающие» 5xx при каждом деплое. Так что вы наткнулись на реальный, а не учебный сценарий.


# Маршруты трафика

Отличная идея — давайте соберём всё, что мы прошли, в цельную картину. Это поможет закрепить материал и в будущем быстрее диагностировать похожие ситуации.

## 🗺️ Полный путь трафика

```
┌─────────┐    ┌──────────────────┐    ┌─────────────────────┐    ┌──────────────┐
│ Клиент  │───▶│  Istio Ingress   │───▶│  Spring Cloud       │───▶│ tasks-service│
│Postman/ │    │  Gateway (Envoy) │    │  Gateway (Java)     │    │  (Java)      │
│ curl    │    │                  │    │                     │    │              │
└─────────┘    └──────────────────┘    └─────────────────────┘    └──────────────┘
    :80          10.96.x.x:80             ClusterIP:80              ClusterIP:80
                  (LoadBalancer)          → pod:8080                → pod:8091
```

Запрос `POST http://<ingress-address>/tasks/create` проходит **четыре зоны**, каждая из которых добавляет свой слой логики.

---

## 🌐 Зона 1: Istio Ingress Gateway

**Что это физически:** под `istio-ingressgateway` в namespace `istio-system`, состоящий из одного контейнера `istio-proxy` (Envoy). Это **точка входа** в кластер — сюда приходит весь внешний трафик.

**Что здесь происходит:**

1. **Приём TCP/HTTP-соединения** на порту 80 (или 443) через Service типа `LoadBalancer`. В Minikube адрес выдаётся через `minikube tunnel`.
2. **Поиск маршрута** по конфигурации, полученной от istiod через xDS:
   - `Gateway` определяет, что Ingress Gateway **слушает** порт 80 и принимает хосты `*`.
   - `VirtualService` определяет, **куда** направить трафик: любой путь (`/*`) → сервис `spring-cloud-gateway:80`.
3. **Определение upstream-кластера.** Envoy смотрит в свой EDS-список: есть ли кластер `spring-cloud-gateway.default.svc.cluster.local:80`? Если порт в `VirtualService` не совпадает с портом в Service (`8080` vs `80`) — вы получите `503 NC cluster_not_found`.
4. **Балансировка** между подами `spring-cloud-gateway` (если их несколько).
5. **mTLS** (если включён `STRICT`): Envoy устанавливает TLS-соединение с sidecar целевого пода.

**Ключевые ресурсы:**
- `Gateway` — «слушай порт 80, принимай хосты `*`»
- `VirtualService` — «трафик на `/*` идёт в `spring-cloud-gateway:80`»
- `PeerAuthentication` — режим mTLS (`PERMISSIVE` / `STRICT`)

**Что НЕ делает Istio Ingress Gateway:**
- Не разбирает путь `/tasks/create` семантически (кроме prefix-матчинга).
- Не знает про `tasks-service` — для него существует только `spring-cloud-gateway`.
- Не занимается бизнес-логикой: аутентификацией по JWT, фильтрацией тела и т.д. (это можно, но не в вашей конфигурации).

---

## 🚪 Зона 2: Sidecar Istio в поде `spring-cloud-gateway`

**Что это физически:** контейнер `istio-proxy` (Envoy) внутри того же пода, что и Java-приложение. Внедряется автоматически, потому что namespace помечен `istio-injection=enabled`.

**Что здесь происходит:**

1. **Перехват трафика.** iptables внутри пода перенаправляет весь входящий/исходящий трафик через Envoy на порт 15006/15001.
2. **Терминация mTLS.** Если включён `STRICT`, Envoy расшифровывает соединение от Ingress Gateway.
3. **Передача в приложение** на `localhost:8080`. Приложение думает, что общается напрямую, — на самом деле трафик прошёл через прокси.
4. **Телеметрия.** Envoy пишет access-логи (те самые, где вы видели `503 NC cluster_not_found`), метрики в Prometheus, трейсы.

**Ключевой момент:** у sidecar тоже своя конфигурация от istiod. Если под `tasks-service` **без sidecar**, а mTLS `STRICT` — соединение порвётся с `upstream connect error`.

---

## ☕ Зона 3: Spring Cloud Gateway (Java-приложение)

**Что это физически:** ваше приложение на Spring Boot, слушающее порт 8080 внутри пода. Это **прикладной шлюз** — «умная» точка входа в вашу бизнес-логику.

**Что здесь происходит:**

1. **Маршрутизация по пути.** Gateway получает запрос `POST /tasks/create` и проходит по списку маршрутов:
   ```
   Route matched: tasks-service-route
   Pattern "/tasks/**" matches against value "/tasks/create"
   ```
2. **Определение upstream.** Маршрут `tasks-service-route` имеет `uri: lb://tasks-service`. Префикс `lb://` означает: «используй LoadBalancer, чтобы получить инстанс сервиса `tasks-service`».
3. **Обращение к LoadBalancer.** `ReactiveLoadBalancer` идёт в `KubernetesInformerDiscoveryClient`, который держит локальный кэш endpoints из Kubernetes API (через List+Watch).
4. **Выбор инстанса.** Round-robin по списку `PodIP:8091`. Если список пуст — 503 от Gateway.
5. **Проксирование запроса** на выбранный инстанс. Путь **не меняется** — `/tasks/create` идёт в `tasks-service` как есть.
6. **Возврат ответа** клиенту.

**Ключевые зависимости и настройки:**
- `spring-cloud-starter-gateway-server-webflux` — сам Gateway
- `spring-cloud-starter-kubernetes-client` — Discovery через K8s API
- `spring-cloud-starter-loadbalancer` — **обязателен** для `lb://`, без него 503 всегда
- RBAC `Role` + `RoleBinding` — права на чтение `endpoints`, `services`, `pods`
- `spring.cloud.gateway.server.webflux.routes` — синтаксис для Spring Cloud 2025.1.x

**Что НЕ делает Spring Cloud Gateway:**
- Не управляет mTLS — это забота Istio.
- Не занимается service discovery между подами — эту задачу решает Kubernetes Service + Istio (в фоне), а Gateway использует Discovery только чтобы получить список IP.

---

## 🎯 Зона 4: `tasks-service` и его sidecar

**Что происходит:**

1. Sidecar принимает соединение от Gateway-пода, проверяет mTLS, передаёт в Java-приложение на `localhost:8091`.
2. Spring MVC находит `@RequestMapping("/tasks") @PostMapping("/create")`, вызывает метод контроллера.
3. Контроллер десериализует тело (`@RequestBody` требует `Content-Type: application/json` + непустое тело).
4. Возвращается JSON-ответ, который едет назад по той же цепочке.

---

## 🧩 Разделение ответственности: Istio vs Spring Cloud Gateway

| Задача | Istio (Ingress + sidecar) | Spring Cloud Gateway |
|---|---|---|
| Точка входа снаружи | ✅ LoadBalancer Service | ❌ |
| mTLS между сервисами | ✅ | ❌ |
| Service discovery в кластере | ✅ на уровне Envoy | ✅ на уровне Java (для `lb://`) |
| Балансировка между подами | ✅ | ✅ (клиентская) |
| Роутинг по пути/хосту на входе | ✅ (грубый, prefix) | ✅ (гибкий, predicates/filters) |
| Аутентификация JWT, rate-limiting на уровне приложения | ⚠️ возможно, но неудобно | ✅ удобно |
| Трансформация тела, header'ов, rewrites | ⚠️ ограничено | ✅ фильтры |
| Метрики трафика (4 golden signals) | ✅ из коробки | ⚠️ только actuator |
| Трассировка | ✅ из коробки | ⚠️ нужна настройка |

**Правило:** Istio отвечает за **инфраструктуру** (кто с кем может общаться, шифрование, L4-балансировка). Spring Cloud Gateway — за **приложение** (какие пути, какие фильтры, какая бизнес-логика при входе).

---

## 📋 Итоговый чек-лист, что должно быть настроено

### Kubernetes-слой
- [x] `Deployment` + `Service` для `tasks-service` (`port: 80 → targetPort: 8091`)
- [x] `Deployment` + `Service` для `spring-cloud-gateway` (`port: 80 → targetPort: 8080`)
- [x] Namespace `default` помечен `istio-injection=enabled`
- [x] Оба пода имеют sidecar Istio (`READY 2/2`)
- [x] `Role` + `RoleBinding` для SA `default` с правами на `endpoints`, `services`, `pods`
- [x] (Рекомендуется) `startupProbe` на `/actuator/health/readiness`

### Istio-слой
- [x] `Gateway` с селектором `istio: ingressgateway`, порт 80, hosts `*`
- [x] `VirtualService` привязан к этому Gateway, хост `*`, маршрут `/*` → `spring-cloud-gateway:80`
- [x] **Порт в `VirtualService` = 80 (совпадает с Service)**
- [x] `PeerAuthentication` в режиме `PERMISSIVE` (или `STRICT` при наличии sidecar везде)
- [x] Ingress Gateway доступен снаружи (через `minikube tunnel`)

### Spring Cloud Gateway
- [x] `spring-cloud-starter-gateway-server-webflux`
- [x] `spring-cloud-starter-kubernetes-client`
- [x] **`spring-cloud-starter-loadbalancer`** (критично для `lb://`)
- [x] `spring-boot-starter-actuator` (для диагностики)
- [x] Маршрут: `Path=/tasks/**` → `lb://tasks-service`
- [x] `server.port: 8080`

### Приложение
- [x] `tasks-service` слушает `8091` (совпадает с `targetPort`)
- [x] `@RequestBody` требует `Content-Type: application/json` и непустое тело

---

## 🎓 Чему учит этот путь

1. **503 — это не одна ошибка, а пять разных.** Флаг в логах Envoy (`NC`, `UH`, `UF`, `NR`, `UO`) точно указывает на слой.
2. **`Running` ≠ `Ready`.** Асинхронные слои (Informer, LB-кэш, Envoy xDS) прогреваются 1–2 минуты.
3. **Discovery и LoadBalancer — разные вещи.** Discovery знает список сервисов, LoadBalancer возвращает инстансы. Без явной зависимости `loadbalancer` `lb://` не работает.
4. **Порт в `VirtualService` — это порт Kubernetes Service**, а не `targetPort` и не порт контейнера.
5. **RBAC — не формальность.** Без прав на чтение `endpoints` приложение падает, даже если «всё правильно настроено».
6. **Прямой DNS vs `lb://`.** Прямой DNS — рабочее решение для учебного проекта. `lb://` даёт клиентскую балансировку, но в связке с Istio и K8s Service она избыточна.

Теперь у вас есть рабочая схема, которую можно воспроизводить на любом кластере — от Minikube до managed Kubernetes. Если захотите добавить JWT-аутентификацию, rate-limiting или канареечные деплои — это следующие шаги, и они лягут на эту же архитектуру.
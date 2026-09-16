# Сеть в Kubernetes

# Маршрутизация

### 1️⃣ Что делает LoadBalancer перед Ingress?

#### Коротко:

LoadBalancer — это **вход в кластер**,
Ingress — это **HTTP-маршрутизатор внутри кластера**.

### 📍 Как это выглядит физически

```text
Интернет
   ↓
Cloud LoadBalancer (внешний IP)
   ↓
Ingress Controller (pod в k8s)
   ↓
Gateway
   ↓
Services
```

### 🔹 Что такое LoadBalancer в Kubernetes?

Когда ты создаёшь:

```yaml
type: LoadBalancer
```

Kubernetes просит облако (AWS/GCP/Azure) создать внешний балансировщик.

В итоге появляется:

* публичный IP
* TCP/HTTP балансировка
* распределение трафика на ноды

### Важно:

LoadBalancer балансирует трафик:

👉 между НОДАМИ
НЕ между подами напрямую.

### 🔹 Что делает Ingress?

Kubernetes Ingress — это объект маршрутизации HTTP.

Ingress Controller (например NGINX) получает трафик от LoadBalancer и решает:

```text
api.myapp.com/users → gateway
api.myapp.com/auth → keycloak
```

### 2️⃣ Балансируют ли Service нагрузку?

ДА, но это другой уровень.

### В Kubernetes есть 3 уровня балансировки:

#### 🔵 Уровень 1 — Cloud LoadBalancer

Балансирует:

```
Интернет → Ноды
```


#### 🟢 Уровень 2 — Service (ClusterIP)

Когда у тебя:

```yaml
kind: Service
type: ClusterIP
```

Service через kube-proxy балансирует:

```
Service → Pod1
        → Pod2
        → Pod3
```

Это **внутрикластерная балансировка**.

Работает через:

* iptables
* или IPVS


#### 🟣 Уровень 3 — Ingress Controller

Балансирует HTTP-запросы по backend сервисам.


### 🔥 Итого по балансировке

| Компонент    | Балансирует что    |
|--------------|--------------------|
| LoadBalancer | Между нодами       |
| Service      | Между pod'ами      |
| Ingress      | HTTP маршрутизация |


---

# 🧠 Главное правило Kubernetes networking

В Kubernetes:

> 🟢 Каждый Pod получает свой собственный IP
> 🟢 Все Pod’ы могут напрямую общаться друг с другом
> 🟢 Нет NAT между pod’ами

Это фундаментальная модель.

---

## 📦 1️⃣ Pod Network

Когда создаётся pod:

```id="e3p6tt"
order-service-deployment-xxxxx
```

Он получает IP, например:

```id="s1nr6f"
10.244.1.12
```

Postgres pod:

```id="j7zv8k"
10.244.2.5
```

И они могут обращаться друг к другу напрямую:

```id="1y5l7k"
http://10.244.2.5:5432
```

Но так никто не делает — используют Service.

---

## 🔌 2️⃣ Кто создаёт эту сеть?

Это делает CNI (Container Network Interface).

В Minikube обычно используется:

* Flannel

В проде часто:

* Calico
* Cilium

CNI создаёт overlay-сеть между нодами.

---

## 🌐 3️⃣ Что такое Service на сетевом уровне?

Когда ты создаёшь:

```yaml
kind: Service
type: ClusterIP
```

Kubernetes:

1. Создаёт виртуальный IP (например `10.96.12.34`)
2. Настраивает iptables правила
3. Привязывает Service к pod’ам через Endpoints

---

## Поток запроса внутри кластера

```
order-service pod
     ↓
http://postgres:5432
     ↓
DNS → 10.96.12.34 (ClusterIP)
     ↓
iptables
     ↓
перенаправление на
postgres-0 pod IP
```

---

## 🧠 4️⃣ DNS внутри кластера

В Kubernetes работает:

* CoreDNS

Он автоматически создаёт записи:

```id="sl93ay"
postgres.default.svc.cluster.local
```

Но можно писать просто:

```id="3utvrl"
postgres
```

Потому что namespace добавляется автоматически.

---

## 🧩 5️⃣ Headless Service

Если ты создаёшь:

```yaml
clusterIP: None
```

Тогда:

* виртуальный IP НЕ создаётся
* DNS возвращает IP конкретных pod’ов

Например:

```id="5q5d4k"
order-service-postgres-0.order-service-postgres
```

Это нужно для StatefulSet.

---

## 🚪 6️⃣ Как трафик приходит снаружи?

Путь такой:

```
Internet
   ↓
Cloud Load Balancer
   ↓
Ingress Controller Pod
   ↓
Service
   ↓
Pod
```

Ingress — это просто pod внутри кластера, который проксирует трафик.

---

## 🔄 7️⃣ Load Balancing

Если у тебя:

```yaml
replicas: 3
```

Service создаёт endpoints:

```
order-1 → 10.244.1.10
order-2 → 10.244.1.11
order-3 → 10.244.1.12
```

И iptables делает round-robin.

---

## 🏗 Полная схема сети

```id="bh9ftk"
Node 1
 ├─ order pod (10.244.1.10)
 └─ postgres pod (10.244.1.20)

Node 2
 └─ order pod (10.244.2.15)

ClusterIP service (10.96.0.10)
     ↓
iptables
     ↓
Pod IP
```

---

## 🔥 8️⃣ Важно понять разницу IP

| Тип IP          | Где работает                   |
|-----------------|--------------------------------|
| Pod IP          | внутри кластера                |
| ClusterIP       | виртуальный IP внутри кластера |
| Node IP         | IP ноды                        |
| LoadBalancer IP | внешний IP                     |

---

## 🧠 9️⃣ Почему это круто?

Потому что:

* Pod’ы могут умирать
* IP pod’а меняется
* Service остаётся стабильным

Ты всегда подключаешься к:

```id="b22f3k"
http://service-name
```

---

## 🚀 10️⃣ Что происходит если pod падает?

1. Pod удаляется
2. Deployment создаёт новый
3. Новый pod получает новый IP
4. Service автоматически обновляет endpoints
5. Трафик идёт в новый pod

Без изменения конфигурации.

---

## 🎯 В итоге внутри кластера:

* нет фиксированных IP
* всё через DNS
* всё через Service
* балансировка через iptables
* overlay-сеть через CNI

---

# Kube proxy

Отличный вопрос 👌
`kube-proxy` — это один из ключевых компонентов сети Kubernetes.

Сейчас объясню максимально понятно.

---

## 🔹 Что такое kube-proxy?

**kube-proxy** — это системный компонент Kubernetes, который:

> Обеспечивает работу Service (ClusterIP, NodePort, LoadBalancer)
> и перенаправляет трафик к нужным Pod’ам.

Он запускается **на каждой ноде кластера** как DaemonSet.

---

## 🔹 Где он работает?

На каждой ноде есть pod:

```
kube-proxy-xxxxx
```

Он слушает изменения:

* Service
* Endpoints

И настраивает сетевые правила на ноде.

---

## 🔥 Что именно он делает?

Допустим у тебя есть Service:

```yaml
kind: Service
metadata:
  name: order-service
spec:
  type: ClusterIP
  clusterIP: 10.96.0.15
```

И есть 2 pod’а:

```
10.244.1.10
10.244.1.11
```

Когда другой pod делает запрос:

```
http://order-service:8080
```

Происходит:

1. DNS → `10.96.0.15`
2. Пакет попадает на ноду
3. kube-proxy уже заранее настроил iptables
4. iptables перенаправляет трафик на один из pod IP

---

## 🔹 Как kube-proxy это делает?

Есть 2 режима работы:

### 1️⃣ iptables mode (самый распространённый)

* Создаёт правила iptables
* Работает быстро
* Делает round-robin

Это стандартный режим в Minikube.

---

### 2️⃣ IPVS mode (более производительный)

Использует Linux IP Virtual Server.

Плюсы:

* лучше масштабируется
* быстрее при большом количестве сервисов

В проде часто используют IPVS.

---

## 🔹 Визуально

```
Pod → ClusterIP (10.96.0.15)
           ↓
     iptables (kube-proxy)
           ↓
     10.244.1.10  (order pod)
```

---

## 🔹 Важно понимать

kube-proxy:

* НЕ проксирует трафик как nginx
* НЕ сидит в середине соединения
* НЕ обрабатывает HTTP

Он просто настраивает правила в Linux kernel.

После этого ядро само перенаправляет пакеты.

---

## 🔹 Что будет если kube-proxy выключить?

Service перестанут работать.

Pod → Pod по IP будет работать
Pod → Service работать не будет

---

## 🔹 Связь с Ingress

Ingress:

* это HTTP-прокси
* работает на уровне 7 (L7)

kube-proxy:

* работает на уровне 4 (TCP)
* просто маршрутизирует IP/порт

---

## 🔥 Полная цепочка

```
Frontend
   ↓
Ingress (L7 HTTP)
   ↓
Service (ClusterIP)
   ↓
kube-proxy (iptables)
   ↓
Pod IP
```

---

## 🔹 Почему это гениально?

Потому что:

* pod’ы могут перезапускаться
* IP pod’а меняется
* Service остаётся
* kube-proxy автоматически обновляет правила

Ты никогда не думаешь об IP.

---

## 🔹 Современный тренд

Некоторые кластеры заменяют kube-proxy на:

* Cilium

Cilium может работать без kube-proxy и использовать eBPF (быстрее и
современнее).

---

## 🎯 Коротко

kube-proxy — это:

> Сетевой механизм, который делает так, чтобы Service в Kubernetes реально
> работал.

Он превращает:

```
http://order-service
```

в

```
10.244.x.x
```

через iptables/IPVS.

---

## Kube-proxy и Service

Да 👍 ты почти идеально сформулировал.
Сейчас немного уточню, чтобы картина стала на 100% правильной.

---

## 🔹 Что ты делаешь в Service?

Когда ты создаёшь:

```yaml
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order
  ports:
    - port: 8080
      targetPort: 8080
```

Ты:

1. Даёшь сервису **имя**
2. Указываешь **selector**
3. Описываешь **порт**

⚠️ Важно:
Ты НЕ указываешь pod’ы вручную.

Kubernetes сам находит pod’ы, у которых есть label:

```yaml
labels:
  app: order
```

---

## 🔹 Что делает Kubernetes дальше?

Он создаёт:

#### 1️⃣ ClusterIP (виртуальный IP)

Например:

```
10.96.0.15
```

#### 2️⃣ Endpoints

Список реальных pod IP:

```
10.244.1.10
10.244.1.11
```

Проверить можно:

```bash
kubectl get endpoints order-service
```

---

## 🔥 Где здесь kube-proxy?

kube-proxy на каждой ноде:

* видит новый Service
* видит его Endpoints
* создаёт iptables/IPVS правила

И на уровне Linux ядра настраивает:

```
если трафик идёт на 10.96.0.15:8080
→ перенаправить на 10.244.1.10 или 10.244.1.11
```

---

## 🔁 Что происходит при запросе?

Pod делает запрос:

```
http://order-service:8080
```

Шаги:

1️⃣ DNS → возвращает 10.96.0.15
2️⃣ Пакет уходит на ClusterIP
3️⃣ kube-proxy уже настроил правила
4️⃣ Linux kernel перенаправляет пакет на реальный pod IP

---

## 🎯 То есть ты прав:

> В сервисе я описываю имя и какие pod’ы к нему относятся
> kube-proxy помогает обращаться к pod’ам через service IP

✔ Да, именно так.

---

## 🔥 Главное уточнение

Service — это **не процесс**
Service — это **виртуальный IP + правила маршрутизации**

Он ничего сам не проксирует.
Всё делает Linux kernel через правила, которые настроил kube-proxy.

---

## 🧠 Очень важная мысль

Pod’ы могут:

* перезапускаться
* менять IP
* умирать

Service остаётся.

kube-proxy обновляет правила автоматически.

Ты всегда подключаешься к:

```
http://order-service
```

И никогда к:

```
10.244.1.10
```

---

## 🔍 Мини-схема

```
Pod A
  ↓
order-service (ClusterIP 10.96.0.15)
  ↓
iptables (kube-proxy rules)
  ↓
Pod B
```

---

## 🚀 Почему это мощно?

Потому что это:

* встроенный service discovery
* встроенный load balancing
* отказоустойчивость
* автоматическое обновление маршрутов

И тебе не нужно писать ни строчки кода для этого.

---

# INGRESS

Отличный вопрос 🔥
Сейчас разберём **полную цепочку запроса** — от браузера до твоего Spring Boot
приложения внутри pod’а.

Возьмём пример:

```
http://api.k8s.test/orders
```

И у тебя есть:

* Ingress
* Service `order-service`
* Deployment с pod’ами Spring Boot

---

## 🌍 1️⃣ Браузер → DNS

Пользователь вводит:

```
http://api.k8s.test/orders
```

DNS возвращает:

```
IP Ingress Controller (или LoadBalancer)
```

В Minikube это может быть IP ноды.

---

## 🚪 2️⃣ Запрос попадает в Ingress Controller

В кластере работает, например:

* NGINX Ingress Controller

Это обычный pod с Nginx внутри.

Он получил правило из твоего `ingress.yaml`:

```yaml
- host: api.k8s.test
  http:
    paths:
      - path: /orders
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

Ingress Controller делает:

```
/orders → order-service:8080
```

---

## 🔌 3️⃣ Ingress → Service (ClusterIP)

Ingress теперь делает внутренний HTTP-запрос к:

```
http://order-service:8080
```

DNS внутри кластера (CoreDNS) возвращает:

```
ClusterIP сервиса
например 10.96.0.25
```

---

## ⚙️ 4️⃣ kube-proxy вступает в игру

На ноде уже есть правила, которые настроил kube-proxy:

Если трафик идёт на:

```
10.96.0.25:8080
```

то перенаправить его на один из pod’ов:

```
10.244.1.12:8080
или
10.244.2.7:8080
```

Это load balancing на уровне TCP.

---

## 📦 5️⃣ Пакет приходит в Pod

Теперь запрос реально попадает в контейнер:

```
order-service-deployment-xxxxx
```

Spring Boot слушает:

```yaml
server:
  port: 8080
```

И получает HTTP-запрос:

```
GET /orders
```

---

## 🌱 6️⃣ Spring Boot

Дальше обычная Java логика:

```java
@RestController
@RequestMapping("/orders")
```

Контроллер обрабатывает запрос
Возвращает JSON

---

## 🔄 7️⃣ Ответ идёт обратно

Ответ проходит тем же путём в обратном порядке:

```
Pod
 ↓
Service
 ↓
Ingress Controller
 ↓
Браузер
```

---

## 🔥 Визуально вся цепочка

```
Browser
   ↓
DNS
   ↓
Ingress Controller (NGINX pod)
   ↓
Service (ClusterIP)
   ↓
kube-proxy (iptables rules)
   ↓
Pod IP
   ↓
Spring Boot
```

---

## 🧠 Очень важно понять

### Ingress — это L7 (HTTP)

* работает с путями
* работает с host
* может делать rewrite
* может делать TLS

### Service — это L4 (TCP)

* не понимает HTTP
* просто маршрутизирует IP/порт

### kube-proxy — это Linux networking

* настраивает правила
* не проксирует сам

---

## 🚀 Что происходит при масштабировании?

Если у тебя:

```yaml
replicas: 3
```

Service будет балансировать трафик на 3 pod’а автоматически.

Ingress даже не знает сколько их.

---

## 🧩 Что если pod умрёт?

1. Pod падает
2. Deployment создаёт новый
3. Endpoint обновляется
4. kube-proxy обновляет правила
5. Трафик идёт в новый pod

Без изменения конфигурации.

---

## 🎯 Коротко

Запрос проходит 5 уровней:

1. DNS внешний
2. Ingress Controller
3. Service (ClusterIP)
4. kube-proxy (iptables)
5. Pod → Spring Boot

---

# INGRESS -> POD

```
http://api.k8s.test/orders
```

И у тебя есть:

* Ingress
* Service `order-service`
* Deployment с pod’ами Spring Boot

---

### 🌍 1️⃣ Браузер → DNS

Пользователь вводит:

```
http://api.k8s.test/orders
```

DNS возвращает:

```
IP Ingress Controller (или LoadBalancer)
```

В Minikube это может быть IP ноды.

---

### 🚪 2️⃣ Запрос попадает в Ingress Controller

В кластере работает, например:

* NGINX Ingress Controller

Это обычный pod с Nginx внутри.

Он получил правило из твоего `ingress.yaml`:

```yaml
- host: api.k8s.test
  http:
    paths:
      - path: /orders
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

Ingress Controller делает:

```
/orders → order-service:8080
```

---

### 🔌 3️⃣ Ingress → Service (ClusterIP)

Ingress теперь делает внутренний HTTP-запрос к:

```
http://order-service:8080
```

DNS внутри кластера (CoreDNS) возвращает:

```
ClusterIP сервиса
например 10.96.0.25
```

---

### ⚙️ 4️⃣ kube-proxy вступает в игру

На ноде уже есть правила, которые настроил kube-proxy:

Если трафик идёт на:

```
10.96.0.25:8080
```

то перенаправить его на один из pod’ов:

```
10.244.1.12:8080
или
10.244.2.7:8080
```

Это load balancing на уровне TCP.

---

### 📦 5️⃣ Пакет приходит в Pod

Теперь запрос реально попадает в контейнер:

```
order-service-deployment-xxxxx
```

Spring Boot слушает:

```yaml
server:
  port: 8080
```

И получает HTTP-запрос:

```
GET /orders
```

---

### 🌱 6️⃣ Spring Boot

Дальше обычная Java логика:

```java
@RestController
@RequestMapping("/orders")
```

Контроллер обрабатывает запрос
Возвращает JSON

---

### 🔄 7️⃣ Ответ идёт обратно

Ответ проходит тем же путём в обратном порядке:

```
Pod
 ↓
Service
 ↓
Ingress Controller
 ↓
Браузер
```

---

### 🔥 Визуально вся цепочка

```
Browser
   ↓
DNS
   ↓
Ingress Controller (NGINX pod)
   ↓
Service (ClusterIP)
   ↓
kube-proxy (iptables rules)
   ↓
Pod IP
   ↓
Spring Boot
```

---

### 🧠 Очень важно понять

#### Ingress — это L7 (HTTP)

* работает с путями
* работает с host
* может делать rewrite
* может делать TLS

#### Service — это L4 (TCP)

* не понимает HTTP
* просто маршрутизирует IP/порт

#### kube-proxy — это Linux networking

* настраивает правила
* не проксирует сам

---

### 🚀 Что происходит при масштабировании?

Если у тебя:

```yaml
replicas: 3
```

Service будет балансировать трафик на 3 pod’а автоматически.

Ingress даже не знает сколько их.

---

### 🧩 Что если pod умрёт?

1. Pod падает
2. Deployment создаёт новый
3. Endpoint обновляется
4. kube-proxy обновляет правила
5. Трафик идёт в новый pod

Без изменения конфигурации.

---

### 🎯 Коротко

Запрос проходит 5 уровней:

1. DNS внешний
2. Ingress Controller
3. Service (ClusterIP)
4. kube-proxy (iptables)
5. Pod → Spring Boot

---

# INGRESS API GATEWAY

В Kubernetes **роль API Gateway** (в смысле единой точки входа для внешних
клиентов, маршрутизации, аутентификации, rate limiting, TLS termination и т.д.)
выполняют **несколько разных компонентов**, в зависимости от того, о каком
уровне трафика идёт речь (north-south или east-west) и от твоей архитектуры.

Вот актуальная картина на февраль 2026 года:

### 1. Самый распространённый и классический вариант — **Ingress + Ingress

Controller**

- Ingress — это **ресурс Kubernetes** (не контроллер), который описывает правила
  маршрутизации по HTTP/HTTPS (host, path).
- **Ingress Controller** (например NGINX Ingress, Traefik, HAProxy, Kong
  Ingress) — это тот компонент, который **реально выполняет роль API Gateway**:
    - Принимает внешний трафик
    - Маршрутизирует по правилам из Ingress
    - Завершает TLS
    - Может делать rate limiting, auth, rewrite и т.д. (часто через annotations)
- **Проблемы в 2026 году**:
    - Ingress API **заморожен** (feature-frozen) — новых фич не добавляют.
    - Популярный **Ingress-NGINX Controller** официально **уходит в отставку** (
      deprecation announced в 2025, EOL март 2026) — больше не будет обновлений,
      патчей безопасности.
    - Поэтому многие уже мигрируют или начинают новые проекты не с Ingress.

### 2. Современный и рекомендуемый стандарт — **Kubernetes Gateway API**

- Это **официальный следующий шаг** после Ingress (с 2024–2025 в статусе GA).
- **Gateway API** — не один ресурс, а набор CRD:
    - GatewayClass → определяет тип контроллера
    - Gateway → точка входа (аналог load balancer)
    - HTTPRoute / TCPRoute / GRPCRoute → правила маршрутизации
- **Реальные API Gateway** в кластере — это **контроллеры**, которые реализуют
  Gateway API:
    - **Envoy Gateway** — самый чистый и популярный для простых случаев в 2026 (
      лёгкий, быстрый)
    - **Istio** (с Gateway API вместо старого Gateway/VirtualService)
    - **Cilium** (если используешь eBPF/CNI от Cilium)
    - **Kong**, **Traefik**, **NGINX Plus** и другие тоже поддерживают
- **Преимущества над Ingress**:
    - Поддержка не только HTTP, но и TCP/UDP/gRPC/TLS passthrough
    - Более сложная маршрутизация (headers, query, methods, weighted routing,
      mirroring)
    - Чёткое разделение ролей (infra vs app-team)
    - Кросс-namespace, policy attachment
    - Лучше для multi-tenancy и service mesh
- **В 2026 году** Gateway API — **рекомендуемый способ** для новых проектов и
  для замены Ingress.

### 3. Service Mesh (Istio, Linkerd, Cilium Service Mesh и т.д.)

- Service Mesh может выполнять роль **распределённого API Gateway**, особенно
  для **внутреннего трафика** (east-west).
- Но для **внешнего входа** (north-south) часто используют именно **Gateway** из
  того же mesh (Istio Gateway, Envoy Gateway и т.д.).
- В 2026 году многие **комбинируют**: Gateway API для входа + service mesh для
  внутреннего трафика.

### Краткое сравнение (что чаще всего играет роль API Gateway)

| Компонент / подход                             | Роль API Gateway | North-South (внешний трафик) | East-West (внутри кластера) | Статус в 2026       | Рекомендация сейчас        |
|------------------------------------------------|------------------|------------------------------|-----------------------------|---------------------|----------------------------|
| Ingress + Controller                           | Да (L7)          | Да                           | Нет                         | Устаревает          | Только legacy              |
| Gateway API + Controller                       | Да (L4/L7)       | Да                           | Частично (GAMMA)            | Актуальный стандарт | **Да, для новых проектов** |
| Istio / Linkerd / Cilium                       | Да (как mesh)    | Через Gateway                | Да                          | Очень популярно     | Если нужен mesh            |
| Внешний API Gateway (Kong, Apigee, AWS API GW) | Да               | Да                           | Нет                         | Независимо от k8s   | Гибридные системы          |

### Вывод на февраль 2026

- Если тебе нужен **классический API Gateway** для внешних клиентов (
  маршрутизация, auth, rate limit, TLS) → используй **Kubernetes Gateway API** +
  контроллер (Envoy Gateway — самый простой старт).
- Ingress всё ещё работает, но **не стоит начинать новые проекты** на нём —
  лучше сразу Gateway API.
- Если кластер большой и нужен полный observability, mTLS, retry и т.д. → смотри
  в сторону **service mesh** (Istio или Cilium), где Gateway API тоже
  используется для входа.

## Gateway API + Envoy Gateway в minikube

**Да, конечно, можно** — в minikube очень удобно запускать Spring Boot +
PostgreSQL именно через Gateway API (а не через старый Ingress).

На февраль 2026 года это уже вполне стандартный и рекомендуемый путь для новых
проектов.

### Краткий план (что нужно сделать)

1. Запустить minikube (если ещё не запущен)
2. Установить CRD Gateway API
3. Установить контроллер (самый простой и быстрый — Envoy Gateway)
4. Запустить PostgreSQL (StatefulSet + headless + обычный Service)
5. Запустить Spring Boot (Deployment + Service ClusterIP)
6. Создать Gateway + HTTPRoute для внешнего доступа

### Самый простой и рабочий вариант на сегодня

#### Шаг 1 — minikube + необходимые аддоны

```bash
minikube start --cpus=4 --memory=6144 --driver=docker
minikube addons enable ingress   # на всякий случай, хотя нам не нужен
```

#### Шаг 2 — установка Gateway API CRD

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
# или latest, если хочешь самую свежую
# kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml
```

Проверь:

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

Должны появиться GatewayClass, Gateway, HTTPRoute и т.д.

#### Шаг 3 — установка Envoy Gateway (самый лёгкий контроллер)

```bash
kubectl apply -f https://github.com/envoyproxy/gateway/releases/latest/download/install.yaml
```

Подождать 1–2 минуты, пока поды встанут:

```bash
kubectl get pods -n envoy-gateway-system
```

#### Шаг 4 — PostgreSQL (StatefulSet, как ты правильно сказал)

Создай файл `postgres.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: todo

---
apiVersion: v1
kind: Secret
metadata:
  name: pg-secret
  namespace: todo
stringData:
  POSTGRES_USER: todo
  POSTGRES_PASSWORD: secret123
  POSTGRES_DB: todo

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: todo
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: todo
spec:
  serviceName: postgres
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
          image: postgres:16
          envFrom:
            - secretRef:
                name: pg-secret
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 2Gi
```

Применить:

```bash
kubectl apply -f postgres.yaml
```

#### Шаг 5 — Spring Boot приложение

Предположим, у тебя Dockerfile и образ уже готов (или собери в minikube):

```bash
eval $(minikube docker-env)
docker build -t todo-app:1.0 .
```

Манифест `app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-app
  namespace: todo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: todo-app
  template:
    metadata:
      labels:
        app: todo-app
    spec:
      containers:
        - name: app
          image: todo-app:1.0
          imagePullPolicy: Never   # если локально собран
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:postgresql://postgres.todo.svc.cluster.local:5432/todo
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: pg-secret
                  key: POSTGRES_USER
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pg-secret
                  key: POSTGRES_PASSWORD

---
apiVersion: v1
kind: Service
metadata:
  name: todo-app
  namespace: todo
spec:
  selector:
    app: todo-app
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f app.yaml
```

#### Шаг 6 — Gateway + HTTPRoute (вот где магия)

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: todo-gateway
  namespace: todo
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same

---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: todo-route
  namespace: todo
spec:
  parentRefs:
    - name: todo-gateway
  hostnames:
    - "todo.local"           # или *.nip.io, если хочешь без hosts
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: todo-app
          port: 80
```

```bash
kubectl apply -f gateway.yaml
```

#### Шаг 7 — доступ

```bash
minikube tunnel   # в отдельном терминале (оставь висеть)
```

Получи IP:

```bash
kubectl get gateway todo-gateway -n todo -o jsonpath='{.status.addresses[0].value}'
# обычно 127.0.0.1
```

Добавь в `/etc/hosts`:

```
127.0.0.1 todo.local
```

Теперь открывай в браузере:

http://todo.local

---

# Сеть между нодами

Отличный уровень вопросов 👌
Теперь ты дошёл до **самого ядра Kubernetes-сети**.

---

# 🔥 Главный принцип Kubernetes-сети

В Kubernetes есть базовое требование:

> 📌 Любой Pod может общаться с любым Pod по его IP
> 📌 Без NAT
> 📌 Независимо от того, на каких нодах они находятся

Это называется **flat pod network**.

---

# 🧠 Кто это обеспечивает?

Не сам Kubernetes.

Это делает CNI-плагин.

CNI = Container Network Interface

Kubernetes только вызывает CNI, а реальную сеть создаёт один из плагинов:

* Flannel
* Calico
* Cilium
* и другие

---

# 📦 Что происходит при создании Pod

Когда kubelet создаёт Pod:

1. Создаётся network namespace
2. CNI создаёт виртуальный интерфейс (veth pair)
3. Один конец в pod
4. Второй — в bridge ноды (обычно `cni0`)
5. Pod получает IP (например 10.244.1.5)

---

# 🌐 Как работает сеть между нодами

Допустим:

```
Node1
  Pod A → 10.244.1.5

Node2
  Pod B → 10.244.2.7
```

Pod A хочет отправить пакет Pod B.

---

## Шаг 1 — Pod отправляет пакет

```
src: 10.244.1.5
dst: 10.244.2.7
```

---

## Шаг 2 — Пакет выходит в bridge ноды

Через `cni0`.

---

## Шаг 3 — Ядро Linux смотрит таблицу маршрутизации

Node1 знает:

```
сеть 10.244.2.0/24 находится через Node2
```

---

## Шаг 4 — CNI доставляет пакет

Есть 3 способа:

---

# 🟢 1️⃣ Overlay сеть (Flannel VXLAN)

VXLAN

Node1:

* инкапсулирует пакет внутрь другого IP-пакета
* отправляет его Node2

Node2:

* распаковывает
* передаёт Pod B

Плюсы:

* просто
* работает в любом облаке

Минусы:

* есть оверхед (инкапсуляция)

---

# 🔵 2️⃣ BGP / Direct routing (Calico)

BGP

Ноды обмениваются маршрутами.

Node1 знает напрямую:

```
10.244.2.0/24 → через Node2 IP
```

Без инкапсуляции.

Плюсы:

* быстрее
* меньше latency

Минусы:

* сложнее настройка

---

# 🟣 3️⃣ eBPF (Cilium)

eBPF

Cilium

Использует eBPF программы в ядре Linux:

* маршрутизация
* балансировка
* network policy
* замена kube-proxy

Очень быстро и современно.

---

# 📊 Полный путь пакета

```text
Pod A
  ↓
veth
  ↓
bridge (cni0)
  ↓
Node1 routing table
  ↓
CNI tunnel / BGP
  ↓
Node2
  ↓
bridge
  ↓
veth
  ↓
Pod B
```

---

# 🔥 Важный момент

Pod IP уникальны во всём кластере.

Нет NAT между pod.

Это принципиально отличается от Docker.

---

# 🧠 Где тут kube-proxy?

kube-proxy работает ТОЛЬКО для Service (ClusterIP).

Если Pod общается напрямую по IP другого Pod — kube-proxy не участвует.

---

# 🟡 Что если Pod на той же ноде?

Тогда всё проще:

```
Pod A → bridge → Pod B
```

Без выхода наружу.

---

# ⚡ А как это работает в облаке?

В AWS/GCP:

* либо overlay
* либо VPC-native networking
* иногда pod получает реальный VPC IP

---

# 🎯 Главное понимание

Межнодовая сеть — это:

✔ CNI плагин
✔ маршрутизация в ядре Linux
✔ overlay или BGP
✔ без NAT между pod

---

# 🧩 Визуально

```
Node1                  Node2
--------               --------
Pod A                  Pod B
  |                      |
 veth                  veth
  |                      |
 bridge                bridge
  |                      |
 VXLAN tunnel  <------> VXLAN tunnel
```

---

# 🚀 Почему это важно понимать?

Потому что:

* latency зависит от CNI
* MTU может ломать трафик
* NetworkPolicy работает на этом уровне
* Service Mesh тоже работает поверх этого

---

# Что такое node bridge?

Отличный вопрос 👍
Сейчас разберём **что такое node bridge** и почему без него pod'ы вообще не
могли бы общаться.

---

# 🧠 Коротко

**Node bridge** — это обычный Linux network bridge,
который работает как виртуальный коммутатор внутри ноды.

Чаще всего он называется:

```
cni0
```

Он создаётся CNI-плагином.

---

# 📦 Что происходит при создании Pod

Когда kubelet создаёт pod:

1. Создаётся network namespace
2. CNI создаёт `veth pair` (виртуальный кабель)
3. Один конец → внутрь pod
4. Второй конец → подключается к bridge (`cni0`)

---

# 🔌 Что такое veth pair?

Это виртуальный ethernet-кабель:

```
[veth-pod]  <====>  [veth-node]
```

Если пакет попал в один конец — он выйдет из другого.

---

# 🌉 Теперь про bridge

Linux bridge — это программный L2-коммутатор.

В Kubernetes:

```
Pod A
  |
 veth
  |
cni0  ← виртуальный switch
  |
 veth
  |
Pod B
```

Bridge соединяет:

* все pod'ы на одной ноде
* плюс интерфейс, ведущий наружу

---

# 🔎 Пример на ноде

Если зайти на worker-node:

```bash
ip link
```

Ты увидишь что-то вроде:

```
cni0
vethabc123
vethxyz456
eth0
```

`cni0` — это bridge.

---

# 🧠 Что он делает?

Когда Pod A отправляет пакет Pod B (на той же ноде):

1. Пакет выходит из pod через veth
2. Попадает в bridge
3. Bridge смотрит MAC-адрес назначения
4. Отправляет в нужный veth
5. Пакет попадает в Pod B

---

# 📌 Важно

Bridge работает на уровне L2 (Ethernet).

Он НЕ знает про Kubernetes.

Он просто пересылает кадры как обычный switch.

---

# 🌐 А если pod на другой ноде?

Тогда схема сложнее:

```
Pod A
  ↓
veth
  ↓
bridge (cni0)
  ↓
eth0 ноды
  ↓
overlay/BGP (CNI)
  ↓
другая нода
  ↓
bridge
  ↓
veth
  ↓
Pod B
```

Bridge участвует только внутри ноды.

---

# 🔥 Кто создаёт bridge?

CNI-плагин:

* Flannel
* Calico
* Cilium

Некоторые плагины используют bridge,
некоторые работают без него (например Cilium в eBPF-режиме).

---

# 🧩 Аналогия

Представь ноду как серверную стойку:

* Pod'ы — это серверы
* veth — сетевые кабели
* bridge — сетевой свитч
* eth0 — выход в дата-центр

---

# 🎯 Главное понимание

Node bridge:

✔ это обычный Linux bridge
✔ соединяет pod'ы на одной ноде
✔ работает как виртуальный switch
✔ создаётся CNI

---

# CNI и cni0

Очень хороший вопрос 👌
Ты правильно заметил — тут легко запутаться.

Короткий ответ:

> ❌ `cni0` и CNI — это не одно и то же.
> ✅ CNI — это стандарт/интерфейс и плагины.
> ✅ `cni0` — это Linux bridge, который может создать CNI-плагин.

Теперь аккуратно разложим.

---

# 🟢 1️⃣ Что такое CNI?

Container Network Interface — это **спецификация (стандарт)**.

Она описывает:

* как Kubernetes (через kubelet) должен вызывать сетевой плагин
* какие параметры передавать
* что плагин должен вернуть (IP, интерфейс и т.д.)

То есть CNI — это API/контракт.

---

# 🔵 2️⃣ Что такое CNI-плагин?

Это конкретная реализация стандарта CNI.

Например:

* Flannel
* Calico
* Cilium

Вот они и создают:

* интерфейсы
* bridge
* маршруты
* туннели
* правила

---

# 🟣 3️⃣ Что такое cni0?

`cni0` — это **Linux bridge интерфейс**, который создаёт CNI-плагин (если он
работает в bridge-режиме).

Пример:

```bash
ip link show
```

Ты увидишь:

```
cni0
vethabc123
eth0
```

`cni0` — это просто программный switch внутри Linux.

Он:

* соединяет pod'ы на одной ноде
* не является "CNI"
* не является стандартом

---

# 🧠 Важное различие

| Термин     | Что это                             |
|------------|-------------------------------------|
| CNI        | Спецификация                        |
| CNI plugin | Программа, реализующая спецификацию |
| cni0       | Linux bridge, созданный плагином    |

---

# 🟡 Почему название похожее?

Потому что:

* bridge создан CNI-плагином
* по умолчанию его называют `cni0`

Но это просто имя интерфейса.

---

# 🔥 Важно: не все CNI создают cni0

Например:

## Flannel

Обычно создаёт:

```
cni0 + flannel.1 (VXLAN)
```

## Calico

Может использовать bridge или прямую маршрутизацию.

## Cilium

Вообще может не создавать bridge.
Он использует:
eBPF
и перехватывает пакеты напрямую в ядре.

---

# 📦 Как выглядит процесс при создании pod

1. kubelet создаёт pod
2. kubelet вызывает CNI plugin
3. Плагин:

* создаёт veth pair
* подключает к `cni0` (если используется bridge)
* назначает IP
* настраивает маршруты

---

# 🎯 Итог

* CNI ≠ cni0
* CNI — это стандарт
* CNI plugin — реализация
* cni0 — Linux bridge, который создаёт плагин

---

# 🧩 Аналогия

CNI — это правила дорожного движения
CNI plugin — строитель дороги
cni0 — конкретный мост на дороге

---


# Архитектура

---

# Scheduler и Kubelet

Отличный вопрос 👍
Это уже уровень понимания **внутренностей Kubernetes**.

Разберём два ключевых компонента control plane:

* **kube-scheduler**
* **kubelet**

И как они участвуют в запуске твоего Spring Boot pod'а.


## 🧠 Общая картина

Когда ты делаешь:

```bash
kubectl apply -f orders-deployment.yaml
```

происходит цепочка:

```
API Server → Scheduler → Node → kubelet → Container runtime → Pod запущен
```

Теперь разберём роли.


## 🧭 kube-scheduler

### Что это?

kube-scheduler — это компонент control plane, который **решает, на какую ноду поставить Pod**.

Он НЕ запускает контейнеры.
Он только выбирает node.


### Что делает scheduler?

1. Видит новый Pod без nodeName
2. Анализирует все доступные ноды
3. Проверяет:
  * хватает ли CPU
  * хватает ли RAM
  * taints / tolerations
  * node affinity
  * topology spread
4. Выбирает лучшую ноду
5. Записывает в Pod.spec:

```yaml
nodeName: worker-node-2
```

После этого его работа закончена.

### Пример

Допустим у тебя:

```
Node 1: 2 CPU, 4GB RAM
Node 2: 4 CPU, 8GB RAM
```

Ты создаёшь pod:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
```

Scheduler решает:

> Ок, node-1 подходит → назначаем туда.


### Важно

Scheduler:

* НЕ создаёт pod
* НЕ запускает контейнер
* НЕ следит за здоровьем
* НЕ перезапускает

Он только **назначает ноду**.


## ⚙ kubelet

Теперь самое важное.

kubelet — это агент, который работает **на каждой ноде**.

Если scheduler — мозг,
то kubelet — это рабочий, который реально запускает контейнеры.


### Что делает kubelet?

Когда scheduler назначил pod на node:

1. kubelet на этой ноде видит новый Pod
2. Скачивает образ (docker pull)
3. Создаёт контейнер через container runtime (containerd)
4. Настраивает:

  * volumes
  * secrets
  * configmaps
  * network
5. Запускает контейнер
6. Постоянно проверяет его состояние
7. Отправляет статус обратно в API Server

### Если контейнер упал?

kubelet:

* видит crash
* перезапускает (если restartPolicy позволяет)


### 🔁 Как они работают вместе

```
1. kubectl apply
2. API Server сохраняет Pod
3. Scheduler назначает Node
4. kubelet на Node запускает контейнер
5. kubelet следит за ним
```


### 📦 Где они находятся?

#### Control Plane:

* kube-apiserver
* etcd
* kube-scheduler
* kube-controller-manager

#### Worker Node:

* kubelet
* kube-proxy
* container runtime


### Пример с твоим order-service

Когда ты сделал:

```bash
kubectl apply -f orders-deployment.yaml
```

Произошло:

1. Deployment создал ReplicaSet
2. ReplicaSet создал Pod
3. Pod без nodeName
4. Scheduler выбрал node
5. kubelet на этой node:

  * скачал образ
  * создал контейнер
  * подключил к сети
  * смонтировал secret с БД
  * запустил Spring Boot


### 🔎 Чем scheduler отличается от controller-manager?

| Компонент          | Что делает                          |
| ------------------ | ----------------------------------- |
| Scheduler          | Выбирает node                       |
| Controller Manager | Следит, чтобы было нужное число pod |
| kubelet            | Запускает контейнеры                |


### 💡 Важная мысль

Scheduler думает глобально (по всему кластеру).
kubelet работает локально (на своей ноде).


### 🧨 Что если нода умерла?

1. kubelet перестаёт отвечать
2. API Server помечает node как NotReady
3. Controller Manager создаёт новые pod
4. Scheduler назначает их на другую ноду

### 🎯 Коротко

kube-scheduler → выбирает где
kubelet → запускает и следит


---

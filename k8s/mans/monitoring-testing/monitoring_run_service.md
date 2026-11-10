minikube start --memory 8192 --cpus 4
eval $(minikube docker-env)
kubectl port-forward svc/tasks-service 8091:80
minikube tunnel

brew service start prometheus
brew service start grafana

в папке monitoring(где лежит файл конфига мониторинга) % prometheus --config.file=prometheus.yml --web.enable-remote-write-receiver (этот параметр для передачи данных из k6 в )


Самое интересное начинается, когда вы отправляете метрики k6 в Prometheus. Тогда вы можете смотреть на них в тех же дашбордах Grafana, что и на метрики вашего приложения, и видеть прямую корреляцию: «вот тут k6 дал нагрузку — а вот так на это отреагировал мой сервис».

Для этого при запуске k6 нужно указать флаг -o experimental-prometheus-rw и адрес вашего Prometheus:

K6_PROMETHEUS_RW_SERVER_URL=http://localhost:9090/api/v1/write \
k6 run -o experimental-prometheus-rw script.js

Важно: Для этого ваш Prometheus должен быть запущен с флагом --web.enable-remote-write-receiver. Без него он не будет принимать метрики от k6
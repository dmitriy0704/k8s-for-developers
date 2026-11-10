# Создание

```sh
# Mongo:
kubectl create secret generic mongodb-credentials \
   --from-literal=MONGO_URI='mongodb://rootadmin:supersecretpassword@tasks-mongodb-service:27017/tasksdb?authSource=admin'
        
# Postgres:

kubectl rollout restart deploy/tasks-service
```

# Удаление

```sh

```

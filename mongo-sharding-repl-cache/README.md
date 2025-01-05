# Инструкция по настройке шардирования и репликации MongoDB с кэшированием Redis

## Предварительные требования

- Установленные Docker и Docker Compose
- Подготовленные файлы проекта

## Начальная настройка

1. Запустите кластер:
```bash
docker compose up -d
```

2. Дождитесь, пока все сервисы будут в состоянии healthy:
```bash
docker compose ps
```

## Шаги настройки

1. Инициализация набора реплик сервера конфигурации:
```bash
docker compose exec configSrv mongosh --port 27017

rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [
    { _id: 0, host: "configSrv:27017" }
  ]
})

exit();
```

2. Инициализация набора реплик первого шарда:
```bash
docker compose exec shard1 mongosh --port 27018
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1:27018" },
    { _id: 1, host: "shard1_replica1:27018" },
    { _id: 2, host: "shard1_replica2:27018" }
  ]
})

exit();
```

3. Инициализация набора реплик второго шарда:
```bash
docker compose exec shard2 mongosh --port 27019

rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2:27019" },
    { _id: 1, host: "shard2_replica1:27019" },
    { _id: 2, host: "shard2_replica2:27019" }
  ]
})

exit();
```

4. Добавление шардов в кластер (через mongosh):
```bash
docker compose exec mongos_router mongosh --port 27020
sh.addShard("shard1/shard1:27018,shard1_replica1:27018,shard1_replica2:27018")
sh.addShard("shard2/shard2:27019,shard2_replica1:27019,shard2_replica2:27019")
```

5. Включение шардирования для базы данных и коллекции:
```bash
sh.enableSharding("somedb")
use somedb
db.createCollection("helloDoc")
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
```
## Проверка
Проверка статуса шардирования:
```bash
sh.status()
```
## Добавление тестовых данных

Подключитесь к mongosh и добавьте тестовые документы:
```bash
docker compose exec mongos_router mongosh --port 27020
```

Выполните в mongosh:
```javascript
use somedb
for(let i = 0; i < 1000; i++) {
    db.helloDoc.insertOne({
        _id: `user_${i}`,
        age: Math.floor(Math.random() * 50) + 20,
        name: `User ${i}`
    })
}
```

## Проверка работы
1. Проверьте распределение данных:
```javascript
db.helloDoc.getShardDistribution()
db.helloDoc.countDocuments()
```

2. Проверьте статус реплик:
```javascript
rs.status()
```

3. Проверьте производительность с кэшированием:
```bash
time curl -s http://localhost:8080/helloDoc/users > /dev/null

```
Повторный запрос должен выполниться быстрее 100мс. (кэш живет 60 секунд)
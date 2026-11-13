# Инструкция по запуску и инициализации кластера MongoDB с шардированием и репликацией

Данный проект демонстрирует развертывание отказоустойчивой высоконагруженной архитектуры СУБД MongoDB. Каждый из 2-х шардов развернут в виде реплика-сета (1 Primary + 2 Secondary), что обеспечивает как распределение данных, так и высокую доступность в случае сбоя узлов.

---

## Шаг 1: Запуск контейнеров

Перед началом инициализации необходимо поднять всю инфраструктуру в фоновом режиме:

```bash
docker compose up -d
```

Убедитесь, что все компоненты кластера (`configSrv`, роутеры `mongos-1`/`mongos-2`, приложения и по три узла для каждого шарда `shard1-X`, `shard2-X`) успешно перешли в статус `Running`.

---

## Шаг 2: Инициализация компонентов MongoDB

Выполните следующие команды в терминале для настройки реплика-сетов конфигурации и репликации внутри шардов. Команды используют автоматизированный синтаксис shell-скриптов.

### 1. Инициализация Config Server
Сервер конфигурации отвечает за хранение метаданных и карты распределения чанков:

```bash
docker compose exec -T configSrv mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: 'configReplSet',
  configsvr: true,
  members: [{ _id: 0, host: 'configSrv:27019' }]
})
EOF
```

### 2. Инициализация Шарда 1 (Реплика-сет из 3-х узлов)
Объединяем узлы `shard1-1`, `shard1-2` и `shard1-3` в единый реплика-сет `shard1ReplSet`:

```bash
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: 'shard1ReplSet',
  members: [
    { _id: 0, host: 'shard1-1:27018' },
    { _id: 1, host: 'shard1-2:27018' },
    { _id: 2, host: 'shard1-3:27018' }
  ]
})
EOF
```

### 3. Инициализация Шарда 2 (Реплика-сет из 3-х узлов)
Объединяем узлы `shard2-1`, `shard2-2` и `shard2-3` в единый реплика-сет `shard2ReplSet`:

```bash
docker compose exec -T shard2-1 mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: 'shard2ReplSet',
  members: [
    { _id: 0, host: 'shard2-1:27017' },
    { _id: 1, host: 'shard2-2:27017' },
    { _id: 2, host: 'shard2-3:27017' }
  ]
})
EOF
```

> **Важно:** После инициализации реплика-сетов подождите 5–10 секунд, чтобы узлы успели провести внутреннее голосование и выбрать `Primary`-лидера, прежде чем переходить к Шагу 3.

---

## Шаг 3: Регистрация реплицированных шардов в роутере mongos

Связываем созданные реплика-сеты через первый маршрутизатор `mongos-1`. Мы передаем строку подключения, содержащую все три узла каждого шарда:

```bash
docker compose exec -T mongos-1 mongosh --port 27017 --quiet <<EOF
sh.addShard('shard1ReplSet/shard1-1:27018,shard1-2:27018,shard1-3:27018')
sh.addShard('shard2ReplSet/shard2-1:27017,shard2-2:27017,shard2-3:27017')
EOF
```

---

## Шаг 4: Настройка шардирования коллекции и наполнение данными (>= 1000 документов)

### 1. Включение шардирования для БД и создание хэш-индекса
```bash
docker compose exec -T mongos-1 mongosh --port 27017 --quiet <<EOF
sh.enableSharding("somedb")
use somedb
db.helloDoc.createIndex({ _id: "hashed" })
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
EOF
```

### 2. Генерация и пакетная вставка 1000 документов
Генерируем тестовый массив из 1000 объектов и записываем их через роутер:

```bash
docker compose exec -T mongos-1 mongosh --port 27017 --quiet <<EOF
use somedb
var docs = [];
for (var i = 1; i <= 1000; i++) {
    docs.push({ name: "doc_" + i, createdAt: new Date() });
}
db.helloDoc.insertMany(docs);
EOF
```

---

## Шаг 5: Проверка успешности распределения и репликации (Контроль ревьюера)

Чтобы продемонстрировать проверяющему выполнение всех условий задания (работоспособность приложения, количество документов $\ge 1000$, распределение по шардам и статус реплик), выполните следующие проверки:

### 1. Проверка распределения данных и статуса шардов
```bash
docker compose exec -T mongos-1 mongosh --port 27017 --quiet <<EOF
sh.status()
EOF
```
*В выводе команды в секции `databases -> somedb.helloDoc` должно быть видно разделение чанков (chunks) между `shard1ReplSet` и `shard2ReplSet`.*

### 2. Проверка количества документов на каждом шарде
Вы можете точечно проверить количество документов, зашедших на каждую группу реплик:
```bash
echo "--- Документов в Шарде 1 ---"
docker compose exec -T shard1-1 mongosh --port 27018 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments()'
echo "--- Документов в Шарде 2 ---"
docker compose exec -T shard2-1 mongosh --port 27017 --quiet --eval 'db.getSiblingDB("somedb").helloDoc.countDocuments()'
```

### 3. Проверка статуса реплик (Primary / Secondary)
Убедитесь, что внутри каждого шарда работает по 3 реплики:
```bash
echo "--- Статус реплика-сета Шарда 1 ---"
docker compose exec -T shard1-1 mongosh --port 27018 --quiet --eval 'rs.status().members.map(m => m.name + " -> " + m.stateStr)'
echo "--- Статус реплика-сета Шарда 2 ---"
docker compose exec -T shard2-1 mongosh --port 27017 --quiet --eval 'rs.status().members.map(m => m.name + " -> " + m.stateStr)'
```
*Каждая команда должна вернуть список из 3-х узлов, где один находится в статусе `PRIMARY`, а два других — в `SECONDARY`.*

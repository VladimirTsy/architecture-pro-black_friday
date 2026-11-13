# Инструкция по запуску и инициализации кластера MongoDB с шардированием

Данный проект демонстрирует развертывание высоконагруженной архитектуры СУБД MongoDB с использованием шардирования (на 2 шарда) для распределения нагрузки на запись и чтение.

---

## Шаг 1: Запуск контейнеров

Перед началом инициализации необходимо поднять всю инфраструктуру в фоновом режиме:

```bash
docker compose up -d
```

Убедитесь, что все компоненты кластера (`configSrv`, `shard1`, `shard2`, `mongos`, `pymongo_api`) успешно перешли в статус `Running`.

---

## Шаг 2: Инициализация компонентов MongoDB

Выполните следующие команды в терминале для настройки реплика-сетов конфигурации и хранения данных. Команды используют автоматизированный синтаксис shell-скриптов.

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

### 2. Инициализация Шарда 1
```bash
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: 'shard1ReplSet',
  members: [{ _id: 0, host: 'shard1:27018' }]
})
EOF
```

<h3>3. Инициализация Шарда 2</h3>

```bash
docker compose exec -T shard2 mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: 'shard2ReplSet',
  members: [{ _id: 0, host: 'shard2:27017' }]
})
EOF
```

---

## Шаг 3: Регистрация шардов в роутере mongos

Чтобы маршрутизатор `mongos` объединил независимые шарды в единый кластер, подключитесь к нему и добавьте созданные реплика-сеты:

```bash
docker compose exec -T mongos mongosh --port 27017 --quiet <<EOF
sh.addShard('shard1ReplSet/shard1:27018')
sh.addShard('shard2ReplSet/shard2:27017')
EOF
```

---

## Шаг 4: Настройка шардирования коллекции и наполнение данными (>= 1000 документов)

Для распределения документов по шардам необходимо явно включить шардирование для базы данных, создать хэшированный индекс и наполнить тестовую коллекцию данными.

### 1. Включение шардирования для БД и создание структуры
```bash
docker compose exec -T mongos mongosh --port 27017 --quiet <<EOF
sh.enableSharding("somedb")
use somedb
db.helloDoc.createIndex({ _id: "hashed" })
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
EOF
```

### 2. Генерация и вставка 1000 документов
Данный скрипт генерирует массив из 1000 объектов и выполняет пакетную вставку в коллекцию `helloDoc`:

```bash
docker compose exec -T mongos mongosh --port 27017 --quiet <<EOF
use somedb
var docs = [];
for (var i = 1; i <= 1000; i++) {
    docs.push({ name: "doc_" + i, createdAt: new Date() });
}
db.helloDoc.insertMany(docs);
EOF
```

---

## Шаг 5: Проверка успешности распределения данных (Контроль ревьюера)

Чтобы продемонстрировать проверяющему успешное выполнение всех условий задания, запустите итоговую команду проверки статуса:

```bash
docker compose exec -T mongos mongosh --port 27017 --quiet <<EOF
sh.status()
EOF
```

### На что обратить внимание в выводе команды:
1. В секции **`databases`** должна появиться база данных `somedb`.
2. Напротив коллекции `somedb.helloDoc` вы увидите распределение чанков (chunks), например:
   * `shard1ReplSet` — `2` chunks
   * `shard2ReplSet` — `2` chunks
   
Это подтверждает, что документы физически распределились по двум разным шардам на основе хэш-функции.

# ДЗ Базовые возможности MongoDB
## Установка

При выполнении домашнего задания был использован образ Docker контейнера [mongo](https://hub.docker.com/_/mongo) последней версии (6.0.3 на момент написания отчета).

Для запуска использовалась следующая команда:
```
docker run --name mongodb -d -p 27017:27017 -v $(pwd)/data:/data mongo
```
Для доступа к MongoDB из внешних утилит проброшен порт 27017 из контейнера. Для обмена файлами смонтирован раздел _/data_ в поддиректорию _data_ текущей директории.

Для выполнения команд внутри контейнера (напр. импорт данных) необходимо открыть термин внутри Docker контейнера, для чего выполнить следующую команду:
```
docker exec -it mongodb bash
```

## Инициализация

Создаю пользователя _root_
```
$ mongosh
...
test> use admin
switched to db admin
admin> db.createUser( { user: "root", pwd: "password", roles: [ "userAdminAnyDatabase", "dbAdminAnyDatabase", "readWriteAnyDatabase" ] } )
{ ok: 1 }
```

## Загрузка данных

_Как источник возьму [данные NASA об упавших на Землю метеоритах](https://data.nasa.gov/resource/y77d-th95.json)_

С помощью утилиты _mongoimport_ загружаю данные в коллекцию _meteorite_landings_ БД _otus_ (для выполнения необходимо открыть терминал Docker контейнера):
```
$ docker exec -it mongodb bash
root@0782010aecec:/data/nasa-meteorite-landings# mongoimport -d otus -c meteorite_landings y77d-th95.json --jsonArray -u root --authenticationDatabase admin
Enter password for mongo user:

2022-11-30T13:37:34.385+0000	connected to: mongodb://localhost/
2022-11-30T13:37:34.412+0000	1000 document(s) imported successfully. 0 document(s) failed to import.
```

Провеяем результат загрузки данных:
```
test> use otus
switched to db otus
otus> show collections
meteorite_landings
otus> db.meteorite_landings.countDocuments()
1000
```

## Выборка данных

Отсортирую данные по имени метеорита и выгружу первые 5 документов
```
otus> db.meteorite_landings.find().limit(3).sort({"name":1})
[
  {
    _id: ObjectId("63875c9e291eae97ea0cd14a"),
    name: 'Aachen',
    id: '1',
    nametype: 'Valid',
    recclass: 'L5',
    mass: '21',
    fall: 'Fell',
    year: '1880-01-01T00:00:00.000',
    reclat: '50.775000',
    reclong: '6.083330',
    geolocation: { type: 'Point', coordinates: [ 6.08333, 50.775 ] }
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14b"),
    name: 'Aarhus',
    id: '2',
    nametype: 'Valid',
    recclass: 'H6',
    mass: '720',
    fall: 'Fell',
    year: '1951-01-01T00:00:00.000',
    reclat: '56.183330',
    reclong: '10.233330',
    geolocation: { type: 'Point', coordinates: [ 10.23333, 56.18333 ] }
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14c"),
    name: 'Abee',
    id: '6',
    nametype: 'Valid',
    recclass: 'EH4',
    mass: '107000',
    fall: 'Fell',
    year: '1952-01-01T00:00:00.000',
    reclat: '54.216670',
    reclong: '-113.000000',
    geolocation: { type: 'Point', coordinates: [ -113, 54.21667 ] }
  }
]
```
Посмотрим сколько метеоритов в год падало в последнее время _(последние 5 лет до 2013 года)_:
```
otus> db.meteorite_landings.aggregate([{$group : {_id: "$year", count: {$sum: 1}}},{'$sort': {"_id":-1}},{'$limit':5}])
[
  { _id: '2013-01-01T00:00:00.000', count: 1 },
  { _id: '2012-01-01T00:00:00.000', count: 2 },
  { _id: '2011-01-01T00:00:00.000', count: 4 },
  { _id: '2010-01-01T00:00:00.000', count: 5 },
  { _id: '2009-01-01T00:00:00.000', count: 4 }
]
```
А также какой метеорит самый старый в источнике данных:
```
otus> db.meteorite_landings.find({"year": {$exists: true}}).sort({"year":1}).limit(1)
[
  {
    _id: ObjectId("63875c9e291eae97ea0cd40a"),
    name: 'Nogata',
    id: '16988',
    nametype: 'Valid',
    recclass: 'L6',
    mass: '472',
    fall: 'Fell',
    year: '0861-01-01T00:00:00.000',
    reclat: '33.725000',
    reclong: '130.750000',
    geolocation: { type: 'Point', coordinates: [ 130.75, 33.725 ] }
  }
]
```
## Обновление данных

Обновлю данные по метеориту _Novy-Ergi_, добавив страну где упал метеорит:
```
otus> db.meteorite_landings.find({name: "Novy-Ergi"})
[
  {
    _id: ObjectId("63875c9e291eae97ea0cd410"),
    name: 'Novy-Ergi',
    id: '17934',
    nametype: 'Valid',
    recclass: 'Stone-uncl',
    fall: 'Fell',
    year: '1662-01-01T00:00:00.000',
    reclat: '58.550000',
    reclong: '31.333330',
    geolocation: { type: 'Point', coordinates: [ 31.33333, 58.55 ] }
  }
]
otus> db.meteorite_landings.update({name: "Novy-Ergi"}, {$set: {country: "Russia"}})
...
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
otus> db.meteorite_landings.find({name: "Novy-Ergi"})
[
  {
    _id: ObjectId("63875c9e291eae97ea0cd410"),
    name: 'Novy-Ergi',
    id: '17934',
    nametype: 'Valid',
    recclass: 'Stone-uncl',
    fall: 'Fell',
    year: '1662-01-01T00:00:00.000',
    reclat: '58.550000',
    reclong: '31.333330',
    geolocation: { type: 'Point', coordinates: [ 31.33333, 58.55 ] },
    country: 'Russia'
  }
]
```
Масса объекта в источнике данных хранится в виде строки, добавим атрибут массы в документ в виде числа при условии наличия информации о массе объекта:
```
otus> db.meteorite_landings.find({ "mass": { $exists: true } }).forEach(function (m) { db.meteorite_landings.update({_id: m._id}, {$set: {"massInt" : parseInt(m.mass)}}) })

otus> db.meteorite_landings.find({}, {"mass": 1, "massInt": 1}).limit(10)
[
  {
    _id: ObjectId("63875c9e291eae97ea0cd14a"),
    mass: '21',
    massInt: 21
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14b"),
    mass: '720',
    massInt: 720
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14c"),
    mass: '107000',
    massInt: 107000
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14d"),
    mass: '1914',
    massInt: 1914
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14e"),
    mass: '780',
    massInt: 780
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd14f"),
    mass: '4239',
    massInt: 4239
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd150"),
    mass: '910',
    massInt: 910
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd151"),
    mass: '30000',
    massInt: 30000
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd152"),
    mass: '1620',
    massInt: 1620
  },
  {
    _id: ObjectId("63875c9e291eae97ea0cd153"),
    mass: '1440',
    massInt: 1440
  }
]
```
Удалим все документы, у которых нет информации о массе:
```
otus> db.meteorite_landings.deleteMany({"mass": null})
{ acknowledged: true, deletedCount: 28 }
```

## Индексы

Загрузим еще одну коллекцию ([история сделок](https://dl.dropbox.com/s/gxbsj271j5pevec/trades.json)):
```
root@0782010aecec:/data# mongoimport -d otus -c trades trades.json -u root --authenticationDatabase admin
Enter password for mongo user:

2022-11-30T16:16:54.144+0000	connected to: mongodb://localhost/
2022-11-30T16:16:57.144+0000	[###.....................] otus.trades	30.8MB/232MB (13.3%)
2022-11-30T16:17:00.145+0000	[######..................] otus.trades	61.2MB/232MB (26.4%)
2022-11-30T16:17:03.144+0000	[#########...............] otus.trades	90.6MB/232MB (39.1%)
2022-11-30T16:17:06.144+0000	[############............] otus.trades	119MB/232MB (51.6%)
2022-11-30T16:17:09.144+0000	[###############.........] otus.trades	150MB/232MB (64.7%)
2022-11-30T16:17:12.144+0000	[##################......] otus.trades	179MB/232MB (77.3%)
2022-11-30T16:17:15.144+0000	[#####################...] otus.trades	209MB/232MB (90.2%)
2022-11-30T16:17:17.353+0000	[########################] otus.trades	232MB/232MB (100.0%)
2022-11-30T16:17:17.353+0000	1000001 document(s) imported successfully. 0 document(s) failed to import.
```
И выполним анализ запроса поиска по цене
```
otus> db.trades.explain("executionStats").find({price: 110})
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'otus.trades',
    indexFilterSet: false,
    parsedQuery: { price: { '$eq': 110 } },
    queryHash: 'C06572A4',
    planCacheKey: 'C06572A4',
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    winningPlan: {
      stage: 'COLLSCAN',
      filter: { price: { '$eq': 110 } },
      direction: 'forward'
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 1000001,
    executionTimeMillis: 328,
    totalKeysExamined: 0,
    totalDocsExamined: 1000001,
    executionStages: {
      stage: 'COLLSCAN',
      filter: { price: { '$eq': 110 } },
      nReturned: 1000001,
      executionTimeMillisEstimate: 15,
      works: 1000003,
      advanced: 1000001,
      needTime: 1,
      needYield: 0,
      saveState: 1000,
      restoreState: 1000,
      isEOF: 1,
      direction: 'forward',
      docsExamined: 1000001
    }
  },
  command: { find: 'trades', filter: { price: 110 }, '$db': 'otus' },
...
  ok: 1
}
```
Из ответа найдем время выполнения - _executionTimeMillisEstimate_ для stage _COLLSCAN_ равен 15 ms.

Теперь создадим индекс по цене в новой коллекции
```
otus> db.trades.createIndex({"price": 1})
price_1
```
И повторим анализ запроса:
```
otus> db.trades.explain("executionStats").find({price: 110})
{
  explainVersion: '1',
  queryPlanner: {
    namespace: 'otus.trades',
    indexFilterSet: false,
    parsedQuery: { price: { '$eq': 110 } },
    queryHash: 'C06572A4',
    planCacheKey: 'F0A1562C',
    maxIndexedOrSolutionsReached: false,
    maxIndexedAndSolutionsReached: false,
    maxScansToExplodeReached: false,
    winningPlan: {
      stage: 'FETCH',
      inputStage: {
        stage: 'IXSCAN',
        keyPattern: { price: 1 },
        indexName: 'price_1',
        isMultiKey: false,
        multiKeyPaths: { price: [] },
        isUnique: false,
        isSparse: false,
        isPartial: false,
        indexVersion: 2,
        direction: 'forward',
        indexBounds: { price: [ '[110, 110]' ] }
      }
    },
    rejectedPlans: []
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 1000001,
    executionTimeMillis: 692,
    totalKeysExamined: 1000001,
    totalDocsExamined: 1000001,
    executionStages: {
      stage: 'FETCH',
      nReturned: 1000001,
      executionTimeMillisEstimate: 48,
      works: 1000002,
      advanced: 1000001,
      needTime: 0,
      needYield: 0,
      saveState: 1000,
      restoreState: 1000,
      isEOF: 1,
      docsExamined: 1000001,
      alreadyHasObj: 0,
      inputStage: {
        stage: 'IXSCAN',
        nReturned: 1000001,
        executionTimeMillisEstimate: 25,
        works: 1000002,
        advanced: 1000001,
        needTime: 0,
        needYield: 0,
        saveState: 1000,
        restoreState: 1000,
        isEOF: 1,
        keyPattern: { price: 1 },
        indexName: 'price_1',
        isMultiKey: false,
        multiKeyPaths: { price: [] },
        isUnique: false,
        isSparse: false,
        isPartial: false,
        indexVersion: 2,
        direction: 'forward',
        indexBounds: { price: [ '[110, 110]' ] },
        keysExamined: 1000001,
        seeks: 1,
        dupsTested: 0,
        dupsDropped: 0
      }
    }
  },
  command: { find: 'trades', filter: { price: 110 }, '$db': 'otus' },
...
  ok: 1
}
```
Поменялся stage на FETCH/IXSCAN. Общее время выполнения увеличилось, но из отчета видно, что теперь для выборки данных используется созданный индекс _price_1_.

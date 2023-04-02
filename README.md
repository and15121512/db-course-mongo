### ДЗ 2 Mongo ###

Файл csv с данными брал отсюда (файл uber-raw-data-janjune-15.csv, ~0.5 ГБ):
https://www.kaggle.com/datasets/fivethirtyeight/uber-pickups-in-new-york-city?resource=download

Делаем pull образа монги командой:

*mongodb/mongodb-community-server:6.0-ubi8*

Используя docker-compose файл (в /mongo-db/), поднимаем сервис монги. Прокинул три тома: сами данные БД, csv для загрузки, файл с конфигом (чтобы отключить авторизацию, ибо достаёт). Файл с конфигом в директории /mongo-db/configs, также есть /mongo-db/data для данных БД и /mongo-db/external_data для csv (сам csv не стал заливать в гит, т.к. он ~0.5 ГБ, но при воспроизведении шагов подложить его надо). 

*docker-compose up -d*

Затем нужно импортнуть csv в монгу, для этого есть команда mongoimport. Но сначала надо зайти в сам контейнер:

*docker exec -it {id_контейнера} sh*

И теперь импорт (схема User, коллекция Pickups):

*mongoimport --db Uber --collection Pickups --type csv --headerline --ignoreBlanks --file external_data/uber-raw-data-janjune-15.csv*

Выходим из контейнера, делаем *mongosh* и *use User*, чтобы дальше работать с нужной схемой. Чтобы уметь замерять время выполнения запроса, сразу добавляем профилирование:

*db.setProfilingLevel(2)*

Находим по времени поездки одну конкретную:

*db.Pickups.find({Pickup_date: "2015-01-01 15:39:17"})*

Получим одну запись:

*[
  {
    _id: ObjectId("642995e6a3827fec5e2ed87f"),
    Dispatching_base_num: 'B02512',
    Pickup_date: '2015-01-01 15:39:17',
    Affiliated_base_num: 'B02617',
    locationID: 236
  }
]*

Запрос *show profile* выдаёт время выполнения этого запроса ~8 сек (screen1.png):

*query	Uber.Pickups **8670ms** Sun Apr 02 2023 18:18:11 command:{
  find: 'Pickups',
  filter: { Pickup_date: '2015-01-01 15:39:17' },
  lsid: { id: new UUID("caca4c19-e972-4aaa-8de5-89369680868f") },
  '$db': 'Uber'
}
 ...*

Теперь добавим индекс на поле с временем:

*db.Pickups.createIndex( { Pickup_date: 1 } )*

Повторим запрос, сделаем *show profile* и получим огромный прирост по скорости, до 15 мс (screen2.png):

*command	Uber.system.profile **15ms** Sun Apr 02 2023 19:57:19
command:{
  aggregate: 'system.profile',
  pipeline: [
    { '$match': {} },
    { '$group': { _id: 1, n: { '$sum': 1 } } }
  ],
  cursor: {},
  lsid: { id: new UUID("bb9c3a02-d34b-415d-8c02-6a10e64de8fb") },
  '$db': 'Uber'
}*

Т.е. индексы действительно оправдывают своё применение.

Теперь сделаем оставшиеся CRUD операции.

Создание нового документа (screen3.png):

*db.Pickups.insert({Dispatching_base_num: "B02512", Pickup_date: "2015-01-01 15:39:18", Affiliated_base_num: 'B02617', locationID: 236})*

Удаление документа (screen3.png):

*db.Pickups.deleteOne({Pickup_date: "2015-01-01 15:39:18"})*

Обновление документа (screen3.png):

*db.Pickups.updateOne({Pickup_date: "2015-01-01 15:39:17"}, {$set: {Dispatching_base_num: 'B02598'}})*

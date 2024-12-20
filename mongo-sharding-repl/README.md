Задание 3 Шардирование + Репликация

Все ниже описанные действия выполнялись на основе файла compose.yaml из папки mongo-sharding

В файле compose.yaml я сделал слежующее:
1) Создал 6 контейнеров с уникальными портами и IP адресами
2) В первых трех контейнерах для "--replSet" прописал "shard1". Для следующих трех для "--replSet" прописал "shard1"
3) В volumes прописал путь к базе данных для всех реплик и сервера конфигурации
4) Инициализировал сервер конфигурации:
docker exec -it configSrv mongo --port 27017

> rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
> exit; 
5) Инициализировал первый шард
docker exec -it shard1-replica1 mongo --port 27018

>rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1-replica1:27018" },
        { _id : 1, host : "shard1-replica2:27028" },
		{ _id : 2, host : "shard1-replica3:27038" }
      ]
    }
);
> exit;

6) Инициализировал второй шард
docker exec -it shard2-replica1 mongo --port 27019

>rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 0, host : "shard2-replica1:27019" },
        { _id : 1, host : "shard2-replica2:27029" },
		{ _id : 2, host : "shard2-replica3:27039" }
      ]
    }
);
> exit;

7) Добавил шарды, 
Включил шардирование для базы и коллекции, 
Добавил 1000 записей в базу, проверил что они добавились

docker exec -it mongos_router mongo --port 27020

> sh.addShard( "shard1/shard1-replica1:27018");
> sh.addShard( "shard2/shard2-replica1:27019");

> sh.enableSharding("somedb");
> sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

> use somedb

> for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

> db.helloDoc.countDocuments({}) 
> exit; 

8) Проверка 1:
При открытии http://localhost:8080
отображается информация о том, что в базе есть 2 шарада, в каждом по 3 реплики и 1000 записей

9) Проверка 2

docker exec -it shard2-replica1 mongo --port 27019

> use somedb

> db.helloDoc.countDocuments({}) 

В результате получаем что к шарду 2 привязано 492 записей
Аналогично можно проверить шард 1

10) Проверка 3
Проверяем чтение из secondary сервера реплики
Для этого даем разрешение для чтения из secondary реплики 2-го шарда

docker exec -it shard2-replica2 mongo --port 27029

> use somedb

> db.getMongo().setReadPref('secondary');

> db.helloDoc.countDocuments({}) 

В результате получаем что к шарду 2 привязано 492 записей
Аналогично можно проверить шард 1


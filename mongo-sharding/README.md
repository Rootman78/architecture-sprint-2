Задание 2 Шардирование

В файле compose.yaml я сделал слежующее:
1) Зкомментировал запуск контейнера mongodb1
2) Добавил запуск сервера конфигурации configSrv по Ip 173.17.0.100 с использованием порта 27017
3) Добавил запуск сервера первого шарда c Ip 173.17.0.90 через порт  27018
4) Добавил запуск сервера второго шарда c Ip 173.17.0.80 через порт  27019
5) Добавил запуск сервера вроутера c Ip 173.17.0.70 через порт  27020
6) Добавил в запуск контейнера pymongo_api новый путь для запуска шардированной базы "mongodb://mongos_router:27020/somedb"
6) Прописал настройки сети app-network
7) В volumes прописал путь к базе данных для шардов и сервера конфигурации
8) Инициализировал сервер конфигурации:
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
9) Инициализировал шарды
docker exec -it shard1 mongo --port 27018

> rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1:27018" },
       // { _id : 1, host : "shard2:27019" }
      ]
    }
);
> exit;

docker exec -it shard2 mongo --port 27019

> rs.initiate(
    {
      _id : "shard2",
      members: [
       // { _id : 0, host : "shard1:27018" },
        { _id : 1, host : "shard2:27019" }
      ]
    }
  );
> exit; 
10) Инициализировал роутер, добавил шарды, 
Включил шардирование для базы и коллекции, 
Добавил 1000 записей в базу, проверил что они добавились

docker exec -it mongos_router mongo --port 27020

> sh.addShard( "shard1/shard1:27018");
> sh.addShard( "shard2/shard2:27019");

> sh.enableSharding("somedb");
> sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

> use somedb

> for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

> db.helloDoc.countDocuments({}) 
> exit; 

11) Проверка 1:
При открытии http://localhost:8080
отображается информация о том, что в базе есть 2 шарада и 1000 записей

12) Проверка 2

docker exec -it shard1 mongo --port 27018

> use somedb

> db.helloDoc.countDocuments({}) 

В результате получаем что к шарду 1 привязано 508 записей
Аналогично можно проверить шард 2


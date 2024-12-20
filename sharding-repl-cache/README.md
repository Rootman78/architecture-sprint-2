Задание 4 Шардирование + Репликация + Кэш

Все ниже описанные действия выполнялись на основе файла compose.yaml из папки mongo-sharding-repl

В файле compose.yaml я сделал слежующее:
1) Создал контейнеров redis_single с портом 6379 и уникальныи IP адресами
2) В volumes прописал путь к данным redis
3) Прописал pymongo_api переменную REDIS_URL: "redis://redis_single:6379"
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
отображается информация о том, что в приложении используется кэш

При отключенном кэше скорость чтения 1000 записей примерно 1.1 секунда
При включенном, скорость чтения 1000 записей 20-30 мс

9) Проверка 2
Проверка работы кэш

docker exec -it redis_single sh

# redis-cli ping


В результате получаю ответ "PONG"




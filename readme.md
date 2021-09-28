# MongoDB - Agregação em Sharded Collection

## Docker containers

- Resumo Docker:
    * Rede
    * config-servers: 3
    * shard nodes: 6 (3 shards, cada um com uma réplica)
    * router: 1

### Rede

```shell
docker network create mongo-shard
```

### Config servers

```shell
docker run --name mongo-config01 --net mongo-shard -d mongo mongod --configsvr --replSet configserver --port 27017
docker run --name mongo-config02 --net mongo-shard -d mongo mongod --configsvr --replSet configserver --port 27017
docker run --name mongo-config03 --net mongo-shard -d mongo mongod --configsvr --replSet configserver --port 27017
```

Configurando (executa em um config server qualquer, não precisa ser nos três):

```shell
docker exec -it mongo-config01 mongo
```

Executar:

```javascript
rs.initiate(
   {
      _id: "configserver",
      configsvr: true,
      version: 1,
      members: [
         { _id: 0, host : "mongo-config01:27017" },
         { _id: 1, host : "mongo-config02:27017" },
         { _id: 2, host : "mongo-config03:27017" }
      ]
   }
)
```

### Shards

```shell
# 1
docker run --name mongo-shard1a --net mongo-shard -d mongo mongod --port 27018 --shardsvr --replSet shard01
docker run --name mongo-shard1b --net mongo-shard -d mongo mongod --port 27018 --shardsvr --replSet shard01

# 2
docker run --name mongo-shard2a --net mongo-shard -d mongo mongod --port 27019 --shardsvr --replSet shard02
docker run --name mongo-shard2b --net mongo-shard -d mongo mongod --port 27019 --shardsvr --replSet shard02

# 3
docker run --name mongo-shard3a --net mongo-shard -d mongo mongod --port 27020 --shardsvr --replSet shard03
docker run --name mongo-shard3b --net mongo-shard -d mongo mongod --port 27020 --shardsvr --replSet shard03
```

#### Configurar os shards (deve ser feito no shell de cada um):

```shell
docker exec -it mongo-shard1a mongo --port 27018
```

```javascript
rs.initiate(
   {
      _id: "shard01",
      version: 1,
      members: [
         { _id: 0, host : "mongo-shard1a:27018" },
         { _id: 1, host : "mongo-shard1b:27018" },
      ]
   }
)
```

```shell
docker exec -it mongo-shard2a mongo --port 27019
```

```javascript
rs.initiate(
   {
      _id: "shard02",
      version: 1,
      members: [
         { _id: 0, host : "mongo-shard2a:27019" },
         { _id: 1, host : "mongo-shard2b:27019" },
      ]
   }
)
```

```shell
docker exec -it mongo-shard3a mongo --port 27020
```

```javascript
rs.initiate(
   {
      _id: "shard03",
      version: 1,
      members: [
         { _id: 0, host : "mongo-shard3a:27020" },
         { _id: 1, host : "mongo-shard3b:27020" },
      ]
   }
)
```

### Router

```shell
docker run -p 27017:27017 --name mongo-router --net mongo-shard -d mongo mongos --port 27017 --configdb configserver/mongo-config01:27017,mongo-config02:27017,mongo-config03:27017 --bind_ip_all
```

Config router:

```shell
docker exec -it mongo-router mongo
sh.addShard("shard01/mongo-shard1a:27018")
sh.addShard("shard01/mongo-shard1b:27018") 
sh.addShard("shard02/mongo-shard2a:27019")
sh.addShard("shard02/mongo-shard2b:27019") 
sh.addShard("shard03/mongo-shard3a:27020")
sh.addShard("shard03/mongo-shard3b:27020")
```

## Shard a collection

Os comandos desta seção devem ser executados ainda no mongo-router (processo mongos).

Campos do documento:

- produto
- quantidade
- nomeShard: número cujo valor deve estar entre as faixas das zonas acima para ser direcionado para o shard específico

```javascript
sh.enableSharding("teste_shard")

sh.shardCollection(
  "teste_shard.dados4",
  { "nomeShard" : 1 }
)
```

### Definindo zonas para cada shard

Esta parte é muito importante porque ao definir as zonas de cada shard e a faixa de valores de cada zona nós conseguimos direcionar os documentos de forma direta para cada shard.

```shell
sh.addShardToZone("shard01", "A")
sh.addShardToZone("shard02", "B")
sh.addShardToZone("shard03", "C")

sh.updateZoneKeyRange("teste_shard.dados4", { nomeShard: 0 }, { nomeShard: 9 }, "A")
sh.updateZoneKeyRange("teste_shard.dados4", { nomeShard: 10 }, { nomeShard: 19 }, "B")
sh.updateZoneKeyRange("teste_shard.dados4", { nomeShard: 20 }, { nomeShard: 29 }, "C")
```
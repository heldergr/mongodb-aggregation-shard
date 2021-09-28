# MongoDB - Agregação em Sharded Collection

O objetivo deste trabalho foi testar o uso de agregação de dados com pipeline em collections que são sharded. Usei Docker para subir todos os containers necessários do MongoDB e um Jupyter Notebook simples para adicionar os dados e fazer a consulta utilizando o aggregation pipeline.

- [Docker containers](#docker-containers)
- [Shard a collection](#shard-a-collection)
- [Executando testes](#executando-testes)

## Docker containers

- Resumo de uso do Docker:
    * Rede: 1
    * config-servers: 3 containers
    * shard nodes: 6 containers (3 shards, cada um com uma réplica)
    * router: 1 container

### Rede

```shell
docker network create mongo-shard
```

### Config servers

#### Execução

```shell
docker run --name mongo-config01 --net mongo-shard -d mongo mongod --configsvr --replSet configserver --port 27017
docker run --name mongo-config02 --net mongo-shard -d mongo mongod --configsvr --replSet configserver --port 27017
docker run --name mongo-config03 --net mongo-shard -d mongo mongod --configsvr --replSet configserver --port 27017
```

#### Configuração

A configuração deve executar em apenas um config-server, não é necessário executar nos três.

```shell
docker exec -it mongo-config01 mongo
```

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

#### Execução

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

#### Configuração

A configuração dos shards deve ser feita no shell de cada um deles, mas não é necessário nas duas réplicas, apenas uma.

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

#### Execução

```shell
docker run -p 27017:27017 --name mongo-router --net mongo-shard -d mongo mongos --port 27017 --configdb configserver/mongo-config01:27017,mongo-config02:27017,mongo-config03:27017 --bind_ip_all
```

#### Configuração

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

Os comandos desta seção devem ser executados ainda no shell do mongo-router, da mesma forma que os comandos do item anterior.

Antes de fazer shard em uma collection é necessário habilitar sharding no database.

```javascript
sh.enableSharding("teste_shard")
```

Fazendo shard na collection dados4, do database teste_shard, no campo nomeShard, por range key.

```javascript
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

### Checando quantidade de documentos por shard

```shell
db.dados4.getShardDistribution()
```

## Executando testes

A execução dos testes foi feita na linguagem Python ([Notebook de exemplo](teste.ipynb)). É utilizada a library *pymongo*, recomendada na documentação do MongoDB.

Segue o código fonte:

Obtendo acesso ao database:

```python
from pymongo import MongoClient
client = MongoClient()
db = client.teste_shard
```

Inserindo os dados:

```python
db.dados4.insert_many([
    { 'produto': 'A', 'quantidade': 50, 'nomeShard': 5 },
    { 'produto': 'B', 'quantidade': 40, 'nomeShard': 5 },
    { 'produto': 'C', 'quantidade': 30, 'nomeShard': 5 },
    { 'produto': 'F', 'quantidade': 20, 'nomeShard': 5 },
    { 'produto': 'D', 'quantidade': 10, 'nomeShard': 5 },
    { 'produto': 'A', 'quantidade': 50, 'nomeShard': 14 },
    { 'produto': 'B', 'quantidade': 40, 'nomeShard': 14 },
    { 'produto': 'F', 'quantidade': 30, 'nomeShard': 14 },
    { 'produto': 'C', 'quantidade': 20, 'nomeShard': 14 },
    { 'produto': 'E', 'quantidade': 10, 'nomeShard': 14 },
    { 'produto': 'A', 'quantidade': 50, 'nomeShard': 23 },
    { 'produto': 'E', 'quantidade': 40, 'nomeShard': 23 },
    { 'produto': 'F', 'quantidade': 30, 'nomeShard': 23 },
    { 'produto': 'B', 'quantidade': 20, 'nomeShard': 23 },
    { 'produto': 'C', 'quantidade': 10, 'nomeShard': 23 }
])
```

Testando a agregação por produto:

```python
stages = [
    {
        '$group': {
            '_id': '$produto', 
            'total': {
                '$sum': '$quantidade'
            }
        }
    }, {
        '$sort': {
            'total': -1
        }
    }, {
        '$limit': 3
    }
]

list(db.dados4.aggregate(stages))
```
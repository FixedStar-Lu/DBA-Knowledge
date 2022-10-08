#MongoDB #

# 分片集群下删除Database或Collection的问题

## 问题现象

在分片集群中删除Database或Collection时，它需要到每个Shard上删除对应的数据，并且更新元数据等信息，而这些动作跨多个节点无法做到原子性，可能导致处于操作一个中间状态，复用该集合可能存在数据问题，因此在删除Database或Collection后不建议使用相同的namespace。

## 解决方案

MongoDB4.4
1. 在mongos上执行Drop Database或Collection
2. 在mongos上再次执行Drop操作

MongoDB4.2
1. 在mongos上执行Drop Database或Collection
2. 在mongos上再次执行Drop操作
3. 在所有mongos执行flushRouterConfig

MongoDB4.0或更早
1. 在mongos上执行Drop Database或Collection
2. 连接所有Shard的主节点判断对象是否还存在，存在则删除。删除一个数据库时会删除物理磁盘上的数据文件
3. 连接mongos进入config数据库，从集合chunks、locks、databases和collections中删除对应的元数据
   ```
   ##Drop Database
   use config
   db.collections.remove( { _id: /^DATABASE\./ }, {writeConcern: {w: 'majority' }} )
   db.databases.remove( { _id: "DATABASE" }, {writeConcern: {w: 'majority' }} )
   db.chunks.remove( { ns: /^DATABASE\./ }, {writeConcern: {w: 'majority' }} )
   db.tags.remove( { ns: /^DATABASE\./ }, {writeConcern: {w: 'majority' }} )
   db.locks.remove( { _id: /^DATABASE\./ }, {writeConcern: {w: 'majority' }} )
   
   ##Drop Colleciton
   use config
   db.collections.remove( { _id: "DATABASE.COLLECTION" }, {writeConcern: {w: 'majority' }} )
   db.chunks.remove( { ns: "DATABASE.COLLECTION" }, {writeConcern: {w: 'majority' }} )
   db.tags.remove( { ns: "DATABASE.COLLECTION" }, {writeConcern: {w: 'majority' }} )
   db.locks.remove( { _id: "DATABASE.COLLECTION" }, {writeConcern: {w: 'majority' }} )
   ```
4. 连接shard上的主节点，从缓存中删除对namespace的引用
   ```
   ##Drop Database
   db.getSiblingDB("config").cache.databases.remove({_id:"DATABASE"}, {writeConcern: {w: 'majority' }});
   db.getSiblingDB("config").cache.collections.remove({_id:/^DATABASE.*/}, {writeConcern: {w: 'majority' }});
   db.getSiblingDB("config").getCollectionNames().forEach(function(y) {
   			if(y.indexOf("cache.chunks.DATABASE.") == 0)
   				db.getSiblingDB("config").getCollection(y).drop()
   	})
   
   ##Drop Collection
   db.getSiblingDB("config").cache.collections.remove({_id:"DATABASE.COLLECTION"}, {writeConcern: {w: 'majority' }});
   db.getSiblingDB("config").getCollection("cache.chunks.DATABASE.COLLECTION").drop()
   ```
5. 连接所有mongos节点执行flushRouterConfig

## JIAR
[SERVER-17397：Dropping a Database or Collection in a Sharded Cluster may not fully succeed](https://jira.mongodb.org/browse/SERVER-17397)


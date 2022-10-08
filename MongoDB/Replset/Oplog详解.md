[TOC]

# Oplog详解

## 概述

MongoDB的副本集是基于oplog实现的，oplog记录了主节点的每一次写操作，其保存在local数据库下的一个固定集合。secondary节点通过查询oplog集合就能获取到对应操作记录并进行复制。每个节点都维护着自己的oplog，因此每个成员都能作为同步复制源提供给其它节点使用，并不总是从主节点进行同步复制。

```
shard3:PRIMARY> db.oplog.rs.find().limit(1);
{ "ts" : Timestamp(1584597354, 1), "h" : NumberLong("-3402698259457278602"), "v" : 2, "op" : "n", "ns" : "", "wall" : ISODate("2020-03-19T05:55:54.624Z"), "o" : { "msg" : "initiating set" } }
```

- ts：操作时间
- op：操作类型：插入对应i，更新对应u，删除对应d，n表示no-op空操作
- n：操作的集合

>  Oplog具有幂等性的特点，意味着一个oplog操作无论执行多次和执行一次结果是一样的。

**oplog大小**

在安装配置副本集时，可以手动指定oplog集合的大小，如果未指定则默认为5%的磁盘空闲空间大小，其最小为990M，最大为50G。MongoDB 4.0开始，oplog大小可以超过参数限制，避免删除需要的数据。

从MongoDB 4.4开始可以指定保存oplog条目的最小小时数，mongod仅在以下情况删除oplog

- oplog大小已达参数限制
- oplog条目比系统时钟配置的小时数要老

> MonogDB默认不配置最小保留期，启用需要在参数文件中配置storage.oplogMinRetentionHours或执行replSetResizeOplog

**oplog状态**

副本集oplog的状态可以通过rs.printReplicationInfo()查看

```
configured oplog size:   1561.5615234375MB
log length start to end: 423849secs (117.74hrs)
oplog first event time:  Wed Sep 09 2015 17:39:50 GMT+0800 (CST)
oplog last event time:   Mon Sep 14 2015 15:23:59 GMT+0800 (CST)
now:                     Mon Sep 14 2015 16:37:30 GMT+0800 (CST)
```

其中configured oplog size为集合大小，log length start to end为日志覆盖时间，如果该值太小应该考虑增加oplog的大小。也可以结合db.getReplicationInfo()查看复制是否存在延迟



## 修改oplog大小

在3.6版本中可以通过replSetResizeOplog修改oplog集合的大小，单位为MB。不会影响副本集其它成员，需要一一修改

```
db.adminCommand({replSetResizeOplog:1,size:16384})
```

如果是3.6之前的版本，则需要通过下列方式修改oplog大小

1. 关闭secondary节点

   ```
   use admin
   db.shutdownServer()
   ```

2. 以standalone形式启动(修改端口，注释replset参数，分片环境请注释shardsvr并加入

   ```
   skipShardingConfigurationChecks=true)
   ```

3. 备份当前oplog集合

   ```
   mongodump --db local --collection 'oplog.rs' --port 37017
   ```

4. 保留最后一条oplog记录

   ```
   db = db.getSiblingDB('local')
   db.temp.save( db.oplog.rs.find( { }, { ts: 1, h: 1 } ).sort( {$natural : -1} ).limit(1).next() )
   db.temp.find()
   ```

5. 删除现有oplog集合

   ```
   db = db.getSiblingDB('local')
   db.oplog.rs.drop()
   ```

6. 创建新的oplog集合

   ```
   db.runCommand( { create: "oplog.rs", capped: true, size: (50 * 1024 * 1024 * 1024) } )
   ```

7. 插入最后一条oplog到新集合

   ```
   db.oplog.rs.save( db.temp.findOne()
   db.oplog.rs.find()
   ```

8. 改回参数文件，重新加入副本集，分片环境启动时取消skipShardingConfigurationChecks参数

9. 依次修改其它secondary节点，再通过rs.stepDown()重新选出一个pramary，再对老primary进行修改
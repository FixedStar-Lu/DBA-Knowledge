[TOC]

# MongoDB慢查询日志

## 1. 配置慢查询日志

Database Profiler收集有关正在运行的数据库命令，包括CRUD操作以及配置和管理命令。Profiler将收集的数据写入admin数据库下的system.profile的固定集合。Database Profiler默认情况是OFF的，可以在数据库或实例上启用Profiler

**启用Profiler**

分析级别

| level | 描述                                           |
| :---- | :--------------------------------------------- |
| 0     | 默认级别，profiler处于关闭状态并且没有任何数据 |
| 1     | 将超过slowms时间的操作记录在固定集合中         |
| 2     | 记录所有的操作数据                             |

当前数据库设置分析级别并设置慢操作阈值(毫秒)

```sql
db.setProfilingLevel(1,{slowms:1000})
```

查看分析级别

```sql
> db.getProfilingStatus()
{ "was" : 0, "slowms" : 100, "sampleRate" : 1.0, "ok" : 1 }
```

- was表示当前的分析级别
- slowms表示慢操作阈值，以毫秒为单位
- sampleRate表示分析的慢操作百分比

为整个实例启用分析需要在启动时指定下列参数，或者在参数文件中指定slowOpThresholdMs

```sql
mongod --profile 1 --slowms 15 --slowOpSampleRate 0.5
```

>  MongoDB4.0开始，可以通过db.setProfilingLevel()为mongos配置slowms和sampleRate，其配置仅影响diagnostic log，而对Profiler无效。
>
>  MongoDB4.2开始，副本集中的secondary会记录比slowms阈值更长的oplog条目，这些信息保存在REPL组件下的diagnostic log中，文本格式为:`op: <oplog entry> took <num>ms`。

**设置Filter**

MongoDB4.2.2开始，可以设置一个过滤器(Filter)来控制记录哪些操作，方式有下列两种

- 使用profile命令或db.setProfilingLevel()方法设置filter的值
- 在配置文件中设置filter

例如，设置记录查询超过2秒的操作

```
db.setProfilingLevel( 2, { filter: { op: "query", millis: { $gt: 2000 } } } )
```



## 2. 慢查询日志分析

查看最近10条记录

```sql
db.system.profile.find().limit(10).sort( { ts : -1 } ).pretty()
```

查看指定集合的操作

```sql
db.system.profile.find( { ns : 'mydb.test' } ).pretty()
```

查询操作时间超过500毫秒

```sql
db.system.profile.find( { millis : { $gt : 500 } } ).pretty()
```

查询command外的操作

```sql
db.system.profile.find( { op: { $ne : 'command' } } ).pretty()
```

查询特定时间范围

```sql
db.system.profile.find({
  ts : {
    $gt: new ISODate("2012-12-09T03:00:00Z"),
    $lt: new ISODate("2012-12-09T03:40:00Z")
  }
},{user:0}).sort({millis:-1})
```

查询结果分析

```
{
   "op" : "query",
   "ns" : "test.report",
   "command" : {
      "find" : "report",
      "filter" : { "a" : { "$lte" : 500 } },
      "lsid" : {
         "id" : UUID("5ccd5b81-b023-41f3-8959-bf99ed696ce9")
      },
      "$db" : "test"
   },
   "cursorid" : 33629063128,
   "keysExamined" : 101,
   "docsExamined" : 101,
   "fromMultiPlanner" : true,
   "numYield" : 2,
   "nreturned" : 101,
   "queryHash" : "811451DD",
   "planCacheKey" : "759981BA",
   "locks" : {
      "Global" : {
         "acquireCount" : {
            "r" : NumberLong(3),
            "w" : NumberLong(3)
         }
      },
      "Database" : {
         "acquireCount" : { "r" : NumberLong(3) },
         "acquireWaitCount" : { "r" : NumberLong(1) },
         "timeAcquiringMicros" : { "r" : NumberLong(69130694) }
      },
      "Collection" : {
         "acquireCount" : { "r" : NumberLong(3) }
      }
   },
   "storage" : {
      "data" : {
         "bytesRead" : NumberLong(14736),
         "timeReadingMicros" : NumberLong(17)
      }
   },
   "responseLength" : 1305014,
   "protocol" : "op_msg",
   "millis" : 69132,
   "planSummary" : "IXSCAN { a: 1, _id: -1 }",
   "execStats" : {
      "stage" : "FETCH",
      "nReturned" : 101,
      "executionTimeMillisEstimate" : 0,
      "works" : 101,
      "advanced" : 101,
      "needTime" : 0,
      "needYield" : 0,
      "saveState" : 3,
      "restoreState" : 2,
      "isEOF" : 0,
      "docsExamined" : 101,
      "alreadyHasObj" : 0,
      "inputStage" : {
         ...
      }
   },
   "ts" : ISODate("2019-01-14T16:57:33.450Z"),
   "client" : "127.0.0.1",
   "appName" : "MongoDB Shell",
   "allUsers" : [
      {
         "user" : "someuser",
         "db" : "admin"
      }
   ],
   "user" : "someuser@admin"
}
```

对于任何单个操作，由Database Profiler创建的文档都将包含下列字段的子集：

- system.profile.op：操作类型。包含command、count、query、update等操作

- system.profile.ns：操作的目标集合

- system.profile.command：包含与此操作相关联的完整命令对象的文档，当command文档超过50K，将转换为下列形式

  ```sql
  "command" : {
    "$truncated": <string>,
    "comment": <string>
  }
  ```

  - $truncated为字符串摘要，如果摘要任然超过50K，将以省略号表示

  - 如果将注释传递给操作，则会出现comment字段

- system.profile.originatingCommand：对于从游标中读取的getmore操作，originatingCommand字段包含最初创建该游标的完整命令对象，例如find或aggregate

- system.profile.cursorid：query和getmore操作访问的游标ID

- system.profile.keysExamined：MongoDB为了执行该操作而扫描的索引键的数量，如果比nreturned高得多，可以考虑创建或调整索引

- system.profile.docsExamined：MongoDB为了执行操作而扫描的集合中的文档数量

- system.profile.hasSortStage：HassorTstage是一个布尔值，当查询不能使用索引中的排序来返回请求的排序结果时，改是true。例如，MongoDB必须在从光标接收文档后对文档进行排序

- system.profile.usedDisk：指示任何聚合阶段是否由于内存限制而将数据写入临时文件

- system.profile.ndeleted：操作删除的文档数量

- system.profile.ninserted：操作插入的文档数量

- system.profile.nMatched：与更新操作的查询条件匹配的文档数量

- system.profile.nModified：更新操作修改的文档数量

- system.profile.upsert：指示更新操作的upsert选项值，当upset为true时出现

- system.profile.fromMultiPlanner：指示优化器在为查询选择最终执行计划之前是否评估多个执行计划

- system.profile.replanned：指示查询系统是否删除缓存的执行计划并重新评估所有候选计划

- system.profile.replanReason：指示缓存执行计划被移除的特定原因

- system.profile.keysInserted：写操作插入的索引键的数目

- system.profile.writeConflicts：写操作过程中遇到的冲突数

- system.profile.numYield：为完成其他操作而放弃操作的次数。当需要访问MongoDB还没有完全读入内存的数据时，这允许MongoDB在为生成操作读取数据时，完成内存中有数据的其他操作

- system.profile.queryHash：通过16进程的字符串标识查询结构，可以帮助识别相同查询结构的慢查询

- system.profile.planCacheKey：与查询关联的执行计划缓存键的散列。与queryHash不同，planCacheKey是查询结构和该结构当前可用索引的函数

- system.profile.locks：locks提供了操作期间持有的各种锁类型和锁模式的信息

  | Lock Type                  | Desc                                                       |
  | -------------------------- | ---------------------------------------------------------- |
  | ParallelBatchWriterMode    | 并行批处理写入器模式的锁。早期版本作为global锁信息的一部分 |
  | ReplicationStateTransition | 副本集成员状态转换时获取的锁                               |
  | Global                     | 全局锁                                                     |
  | Database                   | 数据库锁                                                   |
  | Collection                 | 集合锁                                                     |
  | Mutex                      | 互斥锁                                                     |
  | Metadata                   | 元数据锁                                                   |
  | oplog                      | oplog锁                                                    |

  锁定模式

  | Lock Mode | Desc           |
  | --------- | -------------- |
  | R         | 共享锁(S)      |
  | W         | 排它锁(X)      |
  | r         | 意向共享锁(IS) |
  | w         | 意向排它锁(IX) |

  其返回的所信息包含：

  - acquireCount：操作在指定模式下获得锁的次数
  - acquireWaitCount：由于在冲突模式中持有锁，操作必须等待获得acquireCount重新计算锁的次数
  - timeAcquiringMicros：操作为获得锁而等待的累积时间，单位为微妙。timeAcquiringMicros除以acquireWaitCount可以计算平均等待时间
  - deadlockCount：操作在等待获取锁时遇到死锁的次数

- system.profile.storage：提供了存储引擎数据的指标和操作的等待时间，只有指标大于0时才会返回对应指标。具体指标包含：

  - data.bytesRead：操作从磁盘读到缓存的字节数
  - data.timeReadingMicros：操作从磁盘读取所需的时间，单位为微妙
  - data.bytesWritten：操作从缓存写入磁盘的字节数
  - data.timeWritingMicros：操作写入磁盘所需的时间，单位为微妙
  - timeWaitingMicros.cache：操作等待缓存空间的时间，单位为微妙
  - timeWaitingMicros.schemaLock：操作必须等待获取模式锁的时间，单位为微妙
  - timeWaitingMicros.handleLock：操作为获得所需数据句柄上的锁而等待的时间，单位为微妙

- system.profile.nreturned：操作返回的文档数量

- system.profile.responseLength：操作结果文档的字节长度

- system.profile.protocol：MongoDB Wire Protocol请求消息格式

- system.profile.millis：从Mongod的角度来看，从操作开始到操作结束时间，单位为毫秒

- system.profile.planSummary：执行计划摘要

- system.profile.execStats：查询操作执行统计信息的文档

  - stage：作为查询执行的一部分执行的操作描述，COLLSAN为集合扫描，IXSCAN为索引扫描，FETCH为文档检索
  - inputStages：包含作为当前阶段的输入阶段的操作统计信息的数组

- system.profile.ts：操作时间戳

- system.profile.client：客户端地址信息

- system.profile.appName：运行操作的客户端应用程序的标识符

- system.profile.allUsers：会话身份认证的信息，包含用户名，数据库

- system.profile.user：会话认证的用户



## 3. system.profile

由于system.profile集合是一个仅1M大小的固定集合，如果需要增加或减少该结合的大小，必须按照下列步骤操作

1. 禁用分析

   ```
   db.setProfilingLevel(0)
   ```

2. 删除system.profile集合

   ```
   db.system.profile.drop()
   ```

3. 创建一个新的system.profile集合

   ```
   db.createCollection("system.profile" , {capped : true , size : 4000000})
   ```

4. 重新启用分析

   ```
   db.setProfilingLevel(1)
   ```

   > 如果修改的secondary节点的system.profile大小，需要将其以standalone模式启动再进行修改
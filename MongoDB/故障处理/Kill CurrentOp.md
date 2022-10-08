[TOC]

# db.currentOp()

## 输出概述

db.currentOp()用于查看数据库当前正在运行的操作信息，输出结果如下：

```
{
   "inprog": [
        {
          "shard" : <string>,
          "host" : <string>,
          "desc" : <string>,
          "connectionId" : <number>,
          "client_s" : <string>,
          "appName" : <string>,
          "clientMetadata" : <document>,
          "active" : <boolean>,
          "currentOpTime": <string>,
          "opid" : <string>, // "<shard>:<opid>"
          "secs_running" : <NumberLong()>,
          "microsecs_running" : <number>,
          "op" : <string>,
          "ns" : <string>,
          "command" : <document>,
          "originatingCommand" : <document>,
          "planSummary": <string>,
          "msg": <string>,
          "progress" : {
              "done" : <number>,
              "total" : <number>
          },
          "killPending" : <boolean>,
          "numYields" : <number>,
          "locks" : {
              "Global" : <string>,
              "Database" : <string>,
              "Collection" : <string>,
              "Metadata" : <string>,
              "oplog" : <string>
          },
          "waitingForLock" : <boolean>,
          "lockStats" : {
              "Global": {
                 "acquireCount": {
                    "r": <NumberLong>,
                    "w": <NumberLong>,
                    "R": <NumberLong>,
                    "W": <NumberLong>
                 },
                 "acquireWaitCount": {
                    "r": <NumberLong>,
                    "w": <NumberLong>,
                    "R": <NumberLong>,
                    "W": <NumberLong>
                 },
                 "timeAcquiringMicros" : {
                    "r" : NumberLong(0),
                    "w" : NumberLong(0),
                    "R" : NumberLong(0),
                    "W" : NumberLong(0)
                 },
                 "deadlockCount" : {
                    "r" : NumberLong(0),
                    "w" : NumberLong(0),
                    "R" : NumberLong(0),
                    "W" : NumberLong(0)
                 }
              },
              "Database" : {
                 ...
              },
              ...
          }
        },
        ...
    ],
   "ok": <num>,
   "operationTime": <timestamp>,
   "$clusterTime": <document>
 }
```

- host：执行操作的主机名

- desc：客户端描述，包含connectionId

- connectionId：连接标识符

- client：客户端信息

- currentOpTime：操作开始执行的时间

- opid：操作的标识符，可以传递给db.killOp()以终止对应操作

- secs_running：操作持续时间，单位为秒

- op：操作类型

- ns：操作的集合

- command：操作的命令

- locks：操作持有的锁类型和模式，包含以下锁类型和锁模式

  | LockType      | Desc                                     |
  | ------------- | ---------------------------------------- |
  | Global        | 全局锁                                   |
  | MMAPV1Journal | MMAPv1存储引擎专用的锁，用于同步日志写入 |
  | Database      | 数据库锁                                 |
  | Collection    | 集合锁                                   |
  | Metadata      | 元数据锁                                 |
  | oplog         | oplog锁                                  |

  | LockMode | Desc           |
  | -------- | -------------- |
  | R        | 共享锁(S)      |
  | W        | 排它锁(X)      |
  | r        | 意向共享锁(IS) |
  | w        | 意向排它锁(IX) |

- waitingForLock：如果操作正在等待锁，则waitingForLock 为true

- lockStats：对于每种锁类型和模式返回下列信息：

  - acquireCount：操作在指定模式下获得锁的次数
  - acquireWaitCount：由于在冲突模式中持有锁，操作必须等待获得acquireCount重新计算锁的次数
  - timeAcquiringMicros：操作为获得锁而等待的累积时间，单位为微秒
  - deadlockCount：操作在等待获取锁时遇到死锁的次数

[currentOp]:https://docs.mongodb.com/v4.0/reference/command/currentOp/



## 运维操作

查询超过1000秒的操作

```
db.currentOp().inprog.forEach( function (item){ if (item.secs_running > 1000 )print(JSON.stringify(item))})
```

查询所有的查询操作

```
db.currentOp().inprog.forEach( function (item){ if (item.op== "query" ){print(item.opid);}})
```

查询锁等待

```
db.currentOp().inprog.forEach( function (item){ if (item.waitingForLock){ 
	var  lock_info = item[ "opid" ];
	print( "waiting:" ,lock_info,item.op,item.ns);
	}
});
```

杀掉指定会话

```
db.currentOp().inprog.forEach( function (item){ if (item.secs_running > 1000 && item.op== "query")db.killOp(item.opid)})
```


[TOC]

# Replset读写规则

## writeConcern

Write Concern即在操作成功之前必须确认写操作的数据承载成员数量，所有成员只有在成功接收并应用写操作后才能确认写操作。副本集默认Write Concern为w:1，只需要primary确认即可，也可以设置大于1的整数值，要求多个成员进行确认。我们可以通过设置w:majority和j:true来防止Write Concern确认数据的回滚，发出Write Concern写操作的程序将等待指定数量的成员进行确认。确认节点的数量越大，回滚的可能性就越小，但同时也增加客户端的等待时间

![Write operation to a replica set](https://docs.mongodb.com/manual/images/crud-write-concern-w-majority.bakedsvg.svg)

```
db.collection.insert({x: 1}, {writeConcern: { w: "majority" , wtimeout: 5000 }})
```
客户端在写入数据时，可以通过writeConcern来配置写入策略，其包含如下选项
- w：指定数据需要写入多少个节点才会向客户端返回确认，选项默认为1，即写到primary就OK了。`w:majority`则表示需要写入副本集中大多数成员后才完成，提升了数据的写入安全，但会降低写入性能
- j：写入操作的journal持久化后才向客户端返回确认，默认为false
- wtimeout：写入超时时间，仅当w大于1时生效
- 修改副本集默认的writeConcern规则可以执行`conf.settings.getLastErrorDefaults = { w: "majority", wtimeout: 5000 }`

**majority实现**

MongoDB Replset复制是通过secondary节点不断拉取primary上的oplog并重放来实现的，那`w:majority`是如何确保写入到大多数的呢？
1. 客户端发起写入请求，并记录到primary上的oplog中，等待从节点应用写入
2. secondary拉取primary的oplog并重放
3. secondary上的独立线程在oplog时间戳发生更新时，就会向primary发送replSetUpdatePosition命令更新自己的oplog时间戳
4. 当primary发现足够多的节点oplog时间戳已经满足要求了，向客户端返回确认



## readConcern

MongoDB V3.2引入了readConcern来配置读策略，该参数容易与readPreference混淆，两者并不冲突，区别如下：
- readPreference描述了客户端如何将读取操作路由到副本集的成员上。默认情况下，应用程序是将读取操作指向主节点(读首选项为primary)。客户端可以指定读首选项，将读请求发送到secondary节点，其具有下列可选项：
  - primary：默认配置，所有读请求全都发到primary节点上
  - primaryPreferred：优先从primary读取，当primary无法响应时，从secondary读
  - secondary：读请求全都转发到secondary
  - secondaryPreferred：优先从secondary读取，当secondary无法响应时，从primary读取
  - nearest：根据网络距离就近选择

- readConcern决定读取数据时，能读到怎样的数据，其包含如下选项：
  - local：能读取任意数据，默认选项
  - majority：只能读取到"已写入大多数节点的数据"

readConcern设计用于解决脏读问题，例如客户端先在primary读取了一条数据，但该数据还未同步到大多数节点就因为primary down引起的Rollback，那客户端在新primary上无法读取到之前的数据，导致客户端发生了脏读。

> 需要注意区分的是readconcern并不保证读取到的数据是最新的，只是避免了Rollback导致的脏读问题，具体读取哪个节点的是由readPreference策略决定的

**readConcern原理**

配置`readconcern:majority`需要先确认`replication.enableMajorityReadConcern`参数已经开启。配置该参数后，MongoDB会单独起一个snapshot线程，定期采集数据集的快照，并记录快照对应的oplog时间戳，只有当oplog已经应用到大多数节点时，对应的snapshot才会标记为committed，用户读取就只能读取最后一个committed状态的快照数据。

**关闭readconcern**

由于readConcern的snapshot保存在内存中，增加了cache的消耗，对性能存在一定影响，如果需要关闭我们可以设置replication.enableMajorityReadConcern为false，并通过在mongod实例上执行db.serverStatus()查看`storageEngine.supportsCommittedReads`

关闭也会带来一定的影响，例如：
- MongoDB4.0及更早版本中，禁用readConcern:majority将会禁用change stream，V4.2开始则无影响

- 对分片集群事务也存在影响

- 禁用可以防止修改索引的collMod命令回滚


[Read Concern "majority"]:https://docs.mongodb.com/manual/reference/read-concern-majority/?spm=a2c6h.12873639.0.0.29f02afcNY0U16


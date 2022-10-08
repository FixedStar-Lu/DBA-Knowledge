# MongoDB Change Streams

Change Streams允许应用程序访问实时的数据更改，而不存在跟踪oplog的复杂性和风险。应用程序可以利用change streams订阅集合、数据库或整个实例的所有数据更改。因为change streams使用聚合框架，所以应用程序也可以过滤特定的消息内容。



| 对象       | 描述                                                    |
| :--------- | :------------------------------------------------------ |
| collection | 支持除admin、local、config库以外的单个集合              |
| database   | 支持除admin、local、config库以外的数据库，4.0开始       |
| cluster    | 支持整个集群除admin、local、config库以外的集合，4.0开始 |

Change Streams可以用在副本集或分片集群下，需要满足以下条件：

- 必须使用WiredTiger存储引擎，并且Change Stream也适用于MongoDB的静态加密特性
- 必须使用pv1协议(protocol version 1)
- 在4.0或更早版本中，read concern需要启用majority，4.2开启则对此不作要求



## Change Events

一个基本的事件信息结构如下

```
{
   _id : { <BSON Object> },
   "operationType" : "<operation>",
   "fullDocument" : { <document> },
   "ns" : {
      "db" : "<database>",
      "coll" : "<collection>"
   },
   "to" : {
      "db" : "<database>",
      "coll" : "<collection>"
   },
   "documentKey" : { "_id" : <value> },
   "updateDescription" : {
      "updatedFields" : { <document> },
      "removedFields" : [ "<field>", ... ]
   }
   "clusterTime" : <Timestamp>,
   "txnNumber" : <NumberLong>,
   "lsid" : {
      "id" : <UUID>,
      "uid" : <BinData>
   }
}
```

| 字段                            | 含义                                                         |
| :------------------------------ | :----------------------------------------------------------- |
| _id                             | 作为变更流事件的标识符。当恢复变更流时，此值作为resumeAfter参数的resumeToken |
| operationType                   | 操作类型。可以是insert、delete、replace、update、drop、rename、dropDatabase、invalidate中的一个 |
| fullDocument                    | 通过CRUD操作创建或修改的文档。对于insert和replace这表示创建新的文档；对于delete该字段省略；对于update操作，仅当你将变更流的fullDocument设置为updateLookup时，才会显示对应文档的完整内容 |
| ns                              | 受事件影响的数据库或集合                                     |
| ns.db                           | 数据库名                                                     |
| ns.coll                         | 集合名                                                       |
| to                              | 对于操作类型为rename，该字段显示ns集合的新名称               |
| to.db                           | 数据库新名称                                                 |
| to.coll                         | 集合新名称                                                   |
| documentKey                     | 通过CRUD创建或修改的包含_id的文档键值。对于分片集合，还显示文档的完整分片键 |
| updateDescription               | 描述被更新操作更新或删除的字段的文档                         |
| updateDescription.updatedFields | 被更新操作修改的字段，每个字段的值对应于字段的新值，而不是产生新值的操作 |
| updateDescription.removedFields | 由更新操作删除的字段                                         |
| clusterTime                     | 事件关联的oplog记录的时间戳，对于多文档事务中一部分的事件，相关的变更流都为事务提交的时间。在分片集群中，发生在不同分片上的事件可以具有相同的clustertime，但可以与不同的事务关联，甚至不与事务关联。要识别单个事务的事件可以在变更流事件中使用lsid和txnNumber组合 |
| txnNumber                       | 事务号，仅当操作是多文档事务的一部分时才存在                 |
| lsid                            | 与事务关联的会话标识，仅当操作是多文档事务的一部分时才存在   |

Change Streams支持下列变更事件

- insert event
- update event
- replace event
- delete event
- drop event
- rename event
- dropDatabase event
- invalidate event

[Change Events]:https://docs.mongodb.com/manual/reference/change-events/



## 打开Change Streams

对于副本集，可以从任何承载数据的节点发起请求。对于分片集群，则需要从mongos发起操作

```
mongos> cursor=db.users.watch()
mongos> while (!cursor.isExhausted()){
... if (cursor.hasNext()){
... printjson(cursor.next());
... }
... }
{
	"_id" : {
		"_data" : BinData(0,"gmAo2osAAAADRmRfaWQAZF54WFp5P7w6HfddzgBaEARCh2BEkRFL/4ITVepGl3rVBA==")
	},
	"operationType" : "update",
	"ns" : {
		"db" : "test",
		"coll" : "users"
	},
	"documentKey" : {
		"_id" : ObjectId("5e78585a793fbc3a1df75dce")
	},
	"updateDescription" : {
		"updatedFields" : {
			"name" : "luhx"
		},
		"removedFields" : [ ]
	}
}
```

只有在下列情况下，游标才会结束

- 游标被显示的关闭了
- invalidate事件发生
- 分片集群中的分片被删除导致游标关闭

开启change stream需要有对象的find和changeStream的权限，并且事件通知需要满足majority-committed，即变更被大多数节点成员所应用，确保持久化变更。

**修改Change Stream输出**

在配置Change Streams时，可以提供以下管道符的一个或多个来控制变更流输出

- $addFields：向文档添加新字段
- $match：筛选文档，只将匹配指定条件的文档传递到下一个管道阶段
- $project：将带有请求字段的文档传递到管理下一个阶段
- $replaceRoot：用指定的文档替换输入文档
- $replaceWith(V4.2)：用指定的文档替换输入文档
- $redact：根据存储在文档本身中的信息限制文档内容
- $set(V4.2)：向文档添加新字段。$set输出包含所有现有字段和新添加的字段
- $unset(V4.2)：从文档中删除/排除字段

```
pipeline = [
    {'$match': {'fullDocument.username': 'alice'}},
    {'$addFields': {'newField': 'this is an added field!'}}
]
cursor = db.inventory.watch(pipeline=pipeline)
document = next(cursor)
```

**恢复Change Streams**

resume token就是change stream文档的_id值

```
{
   "_data" : <BinData|hex string>
}
```

resume token _data类型依赖于MongoDB版本，在某些情况下，在change stream打开或恢复时的特性兼容版本

| version           | feature Compatibility version | resume token _Data type |
| :---------------- | :---------------------------- | :---------------------- |
| 4.2 and later     | 4.2 or 4.0                    | Hex-encoded string (v1) |
| 4.0.7 and later   | 4.0 or 3.6                    | Hex-encoded string (v1) |
| 4.0.6 and earlier | 4.0                           | Hex-encoded string (v0) |
| 4.0.6 and earlier | 3.6                           | BinData                 |
| 3.6               | 3.6                           | BinData                 |

无论fcv是多少，MongoDB4.0都可以使用bindata或Hex-encoded字符串的resume token，因此4.0部署可以使用3.6集合上打开的change stream中的resume token。

当打开游标时，通过将一个resume token传递给resumeAfter来在特定事件之后恢复Change Streams。对于resume token，使用change stream事件文档的_id值。

这时，oplog需要有足够的历史记录来定位与token或时间戳相关的操作。如果是由invalidate事件关闭的change stream，不能使用resumeAfter来恢复。

```
resume_token = cursor.resume_token
cursor = db.inventory.watch(resume_after=resume_token)
document = next(cursor)
```

MongoDB4.2开始，可以通过startAfter在特定事件后启动一个新的change stream，方法是在打开游标时将一个resume token传递给startAfter。

[Change Streams]:https://docs.mongodb.com/manual/changeStreams/


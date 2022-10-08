# MongoDB Aggregation

聚合操作将多个文档中的值组合在一起并对数据进行各种操作以返回计算结果。MongoDB提供了三种执行聚合的方法：聚合管道、map-reduce、单用途聚合

## 聚合管道
聚合管道(Aggregation Pipeline)是基于数据处理管道概念建模的数据聚合框架。文档进入一个多阶段管道，该管道将文档转换为聚合的结果。例如
```
db.orders.aggregate([
   { $match: { status: "A" } },
   { $group: { _id: "$cust_id", total: { $sum: "$amount" } } }
])
```
首先，\$match阶段按status字段进行筛选值为A的文档传递到下一阶段；其次，\$group阶段按cust_id字段对文档进行分组，计算每个唯一cust_id的金额总和。

通过多个构件来创建一个pipeline，用于对多个文档进行处理。这些构件包含了筛选(filtering)、投射(projecting)、分组(grouping)、排序(sorting)、限制(limiting)和跳过(skiping)

**\$match** 

$match用于对文档集合进行筛选，再对筛选结果做聚合操作，例如查询status=1的文档
```
db.users.aggregate({"$match:{"status":1}})
```
\$match尽可能放在管道最前端的位置，一是能快速过滤数据，二是$match可以用到索引

**\$project** 

投射能够实现提取字段，重命名字段等功能，例如隐藏_id字段，提取_id字段的数据到新的名称里
```
db.users.aggregate({"$project":{"userid":"$_id" , "_id":0}})
```

算术表达式可用于操作数值，指定一组数值就可以对这个表达式进行操作，例如将表达式中的两个值相加
```
db.sales.aggregate({
    "project":{
        "total":{
            "$add":["$sale1","$sale2"]
        }
    }
})
```
- $add：接受一个或多个表达式作为参数，将这些表达式相加
- $subtract：接受两个表达式作为参数，用第一个表达式减去第二个表达式作为结果
- $multiply：接受一个或多个表达式，并且进行相乘
- $divide：接受两个表达式，用第一个表达式处以第二个表达式的商作为结果
- $mod：接受两个表达式，将第一个表达式除以第二个表达式得到的余数作为结果

日期表达式包含了\$year、\$month、\$week、\$dayOfMonth、\$dayOfWeek、\$dayOfYear、\$hour、\$minute、\$second，能够对日期类型进行日期操作

字符串表达式也包含了一些基本的字符串操作：
- $substrBytes：[ \<string expression\>, \<byte index\>, \<byte count\> ]
  第一个参数必须是字符串，从index起始位置截取count个字节，一个中文占三个字节
- $concat：[ \<expression1\>, \<expression2\>, ... ]
  将给定的表达式连接在一起作为返回结果
- $toLower：将字符串转换为小写输出
- $toUpper：将字符串转换为大写输出

逻辑表达式用于控制语句：
- $cmp：两个表达式进行比较，等于返回0，小于返回负数，大于返回正数
- $strcasecmp：比较两个字符串，区分大小写
- \$eq / \$ne / \$gt / \$gte / \$lt / \$lte：比较两个表达式，返回true或false
- $and：如果所有表达式都为true，则返回true，否则返回false
- $or：只有其中一个表达式为true，就返回true，否则返回false
- $not：对表达式取反
- $cond：[booleanExpr,trueExpr,falseExpr]
  如果booleanExpr为true，则返回trueExpr，否则返回falseExpr
- $ifNull：[expr,replacementExpr]
  如果expr是null，则返回replacementExpr，否则返回expr

**$group**

$group操作可以将文档依据特定字段的不同值进行分组
```
db.users.aggregate({"$group":{"_id":"$grade"}})
```

- $sum：求和
- $avg：求平均
- $max：分组内的最大值
- $min：分组内的最小值
- $first：返回分组的第一个值
- $last：返回分组的最后一个值
- $addToSet：如果当前数组不包含指定表达式，那就将它添加到数组中，每个元素只返回一次且无序
- $push：添加到数组中并返回所有值得数组

如果满足以下所有条件，\$group阶段有时可以使用索引来查找每个组中的第一个文档
- \$group阶段之前有一个\$sort阶段对分组字段进行排序
- 在分组字段上有一个与排序顺序匹配的索引
- 在\$group阶段使用的唯一累加器是\$first

**$unwind**

$unwind可以将数组中的每一个值拆分为单独的文档
```
db.users.aggregate({
    "$project":{"address":"$addr"}},
    {"$unwind":"$addr"},
    {"$match":{"addr.city":"shenzhen"}}
)
```
**\$addFields**

将新字段添加到文档。 \$addFields输出包含输入文档中所有现有字段和新添加的字段。相当于\$project
```
{ $addFields: { <newField>: <expression>, ... } }
```
**$sort**

可以根据一个或多个字段进行排序，如果要对大量文档进行排序，建议在$project、$unwind、$group操作前端进行排序，可以使用索引

**$limit**

$limit接收一个数字N，返回结果集中的前N个文档

**$skip**

$skip接收一个数字N，去除结果集中的前N个文档，将剩余结果返回。

**pipeline限制**

- 聚合管道返回的结果集中，单个文档不能超过16MB，超过则会产生错误
- 管道使用的内存不能超过100MB，超过也将产生报错，如果要允许处理大型数据集，可以启用allowDiskUse将数据写入临时文件，\$graphLookup会忽略该选项

**聚合管道在分片集群的行为**

如果pipeline以精确的$match开始，那么只运行在匹配的分片上，如果在多个分片上云上，聚合结果将路由到mongos合并，以下情况除外：
- 如果包含\$out或\$lookup，合并将在primary shard上运行
- 如果包含排序或分组阶段，并且启用了allowDiskUse，合并将在随机的一个分片上运行

更多关于aggregate pipeline的内容请参阅：[aggregation pipeline](https://docs.mongodb.com/manual/core/aggregation-pipeline/)



## MapReduce

MapReduce使用javascript作为查询语言，能够处理很多复杂逻辑。对于大多数聚合操作，Aggregate Pipeline可提供更好的性能和更一致的接口。但是，map-reduce操作提供了Aggregate Pipeline中当前不可用的一些灵活性。

![mapReduce.png](https://upload-images.jianshu.io/upload_images/26125409-7ad1962d03292ebe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

map-reduce执行步骤：
- map：应用于每个符合查询条件的文档，输出键值对
- Shuffle：根据key分组，将产生的键值组成列表放到对应的键中
- reduce：把列表中的值reduce成一个单值，重复进行shuffle-reduce直到每个键的列表只有一个值，mapReduce可以将map-reduce操作的结果作为文档返回，也可以将结果写入集合
- Finalize：可选项，用来进一步压缩或处理聚合结果

除了上述键还有一些其它的键：
- keeptemp：如果为true，在连接关闭时结果集会保存下来，否则不保存
- out：输出集合的名称，如果设置了该选项，系统会自动设置keeptemp：true
- query：在发往map函数前，先用指定条件过滤文档
- sort：在发往map函数前先给文档排序
- limit：发往map函数的文档数量上限
- scope：在javascript代码中使用的变量作用域，例如"scope" : {now : new Date()}
- verbose：是否记录详细的服务器日志


**示例**

1、查询所有键以及计数
```
> use test
> db.users.mapReduce(
        function(){
                for (var key in this){
                emit(key,{count:1});
                }
        },
        function(key,emits){
                total=0;
                for(var i in emits){
                        total += emits[i].count;
                }
                return {"count":total};
        },
        {
                out:"output"
        }
)
--------------执行结果----------------------
{
        "result" : "output",
        "timeMillis" : 18393,
        "counts" : {
                "input" : 985292,
                "emit" : 4926460,
                "reduce" : 49265,
                "output" : 5
        },
        "ok" : 1,
        "operationTime" : Timestamp(1585211994, 6),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1585211994, 6),
                "signature" : {
                        "hash" : BinData(0,"08wqcLkxIGtDS2QIMCX91Wco5tw="),
                        "keyId" : NumberLong("6805801500749594650")
                }
        }
}

```
- result：存放MapReduce结果的集合名
- timeMillis：操作花费的时间，单位为毫秒
- input：发送到map函数的文档个数
- emit：在map函数中emit被调用的次数
- output：结合集合中的文档数量

查询结果集
```
> db.output.find()
{ "_id" : "_id", "value" : { "count" : 985292 } }
{ "_id" : "age", "value" : { "count" : 985292 } }
{ "_id" : "created", "value" : { "count" : 985292 } }
{ "_id" : "i", "value" : { "count" : 985292 } }
{ "_id" : "username", "value" : { "count" : 985292 } }
```

更多关于MapReduce的详细内容请参阅：[Map-Reduce](https://docs.mongodb.com/manual/core/map-reduce/)



## 单用途聚合

MongoDB支持`db.collection.estimatedDocumentCount()`，`db.collection.count()`，`db.collection.distinct()`三种简单的聚合操作，但它们缺乏聚合管道的灵活性和功能
![single_Aggregation.png](https://upload-images.jianshu.io/upload_images/26125409-0ff3ab4e2860b8dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

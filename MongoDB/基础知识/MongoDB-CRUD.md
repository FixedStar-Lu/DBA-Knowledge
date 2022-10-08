[TOC]

# MongoDB CURD操作

## 1. Insert Operation

通过insertOne()的方式将单个文档插入到集合中，如果集合不存在将会自动创建。
```
db.inventory.insertOne([
   { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } }
])
```
通过InsertMany()的方式可以将多个文档插入到集合中
```
db.inventory.insertMany([
   { item: "journal", qty: 25, tags: ["blank", "red"], size: { h: 14, w: 21, uom: "cm" } },
   { item: "mat", qty: 85, tags: ["gray"], size: { h: 27.9, w: 35.5, uom: "cm" } },
   { item: "mousepad", qty: 25, tags: ["gel", "blue"], size: { h: 19, w: 22.85, uom: "cm" } }
])
```
插入行为：
- _id：插入时未指定_id值，系统将自动分配一个ObjectId对象
- 原子性：MongoDB中所有写操作都是单个文档级别的原子操作，如果批量操作时有一个文档插入失败，那么在这个文档之前的所有文档都会插入成功，后续的文档全部失败
- 插入校验：插入数据时，MongoDB会做基本的检查：检查文档结构，检查大小。MongoDB限制所有文档都必须小于16MB，如果要查看BSON文档的大小，可以执行`Object.bsonsize()`
- writeConcern：可选参数。在副本集架构中，可以指定为majority将数据插入大多数节点[[view]](https://docs.mongodb.com/manual/reference/write-concern/)
- ordered：批量插入时，如果为true则有序插入，发生错误则直接返回；false则无序插入，发生错误则继续处理后续文档。默认为true

[db.collection.insertOne()]:(https://docs.mongodb.com/manual/reference/method/db.collection.insertOne/#mongodb-method-db.collection.insertOne)
[db.collection.insertMany()]:(https://docs.mongodb.com/manual/reference/method/db.collection.insertMany/#mongodb-method-db.collection.insertMany)



## 2. Query Operation

1577

```
[
    { item: "journal", qty: 25, size: { h: 14, w: 21, uom: "cm" }, status: "A" },
    { item: "notebook", qty: 50, size: { h: 8.5, w: 11, uom: "in" }, status: "A" },
    { item: "paper", qty: 100, size: { h: 8.5, w: 11, uom: "in" }, status: "D" },
    { item: "planner", qty: 75, size: { h: 22.85, w: 30, uom: "cm" }, status: "D" },
    { item: "postcard", qty: 45, size: { h: 10, w: 15.25, uom: "cm" }, status: "A" }
]

```
查询年龄为qty为50且status为A的文档
```
db.user.find({"qty" : 27 , "status" : "A"})
```
查询size嵌套文档
```
db.user.find({ size: { h: 14, w: 21, uom: "cm" } })
db.user.find({ "size.uom": "in" })
```

> 如果只想返回部分文档列，可以在查询中指定，例如只返回name列：`db.user.find({},{"name" : 1})`。_id总是默认输出

**查询操作符**

操作符 | 描述
-- | --
$lt | 比较符(小于)
$lte | 比较符(小于等于)
$gt | 比较符(大于)
$gte | 比较符(大于等于)
$ne | 比较符(不等于)
$in | 从多个值中查找匹配的文档
$nin | 与$in相反，返回不匹配的文档
$or | 满足任意一个条件即可
$nor | $or取反
$and | 满足所有的给定条件
$not | $not是元条件句，可以用在任何其它条件之上。用于求非
$all | 通过多个元素来匹配数组
$size | 查询指定长度的数组
$slice | 返回某个键匹配的数组元素的一个子集
$mod | 将查询的值除以第一个指定值，若余数为第二个指定值则匹配成功

查询age大于等于18，小于等于30的文档
```
db.mycoll.find({"age" : {"$gte" : 18 , "$lte" : 30}})
```

查询number为725，542，290的文档
```
db.mycoll.find({"number" : {"$in" : [725,542,290]}})
```

查询number不为725，542，290的文档
```
db.mycoll.find({"number" : {"$nin" : [725,542,290]}})
```

查询number为725或name为lhx的文档
```
db.mycoll.find({"$or" : [{"number" : 725} , {"name" : "lhx"}]})
```

查询包含apple和banana的数组
```
db.mycoll.find({"fruit" : {$all : ["apple" , "banana"]}})
```

查询数组长度为3的数组
```
db.mycoll.find({"fruit" : {"$size" : 3}})
```

查询第11-20条的留言
```
db.mycoll.find(criteria , {"comments" : {"$slice" : [10,10]}})
```

**针对null类型的查询**

在针对数据类型为null的字段进行查询时，如果指定了一个不存在的键，则会返回整个集合不包含该键的文档
```
> db.mycoll.find({"a" : null})
{"_id" : ObjectID("4bados9df830a0ok0d52"),"a" : null}
```

查询不存在的b字段
```
> db.mycoll.find({"b" : null})
{"_id" : ObjectID("4bados9df830a0ok0d52"),"a" : null}
{"_id" : ObjectID("4bados9df830a0ok0d53"),"a" : 1}
{"_id" : ObjectID("4bados9df830a0ok0d54"),"a" : 2}
```

如果仅想检查该键的值是否为null，可以设置$exists条件判定键值是否存在
```
db.mycoll.find({"b" :{$in : [null] , "$exists" : true}})
```

**针对正则表达式的查询**

MongoDB使用perl兼容的正则表达式(PCRE)库来匹配正则表达式，任何PCRE支持的正则表达式语法都能被MongoDB接受。建议使用正则表达式之前现在JavaScripts shell中检查一下语法

查询name为joe的文档(正则表达式不区分大小写)
```
db.mycoll.find({"name": /joe/i})
```

**数组与范围查询**

目前存在如下文档：
{"x" : 5}
{"x" : 15}
{"x" : 25}
{"x" : [5 , 25]}

如果现在要查询x的值位于10-20之间的所有文档，通常可能会通过db.mycoll.find({"x" : {"$gt" : 10 , "lt" : 20}})的方式来查询，希望返回{"x" : 15}。但是实际上会返回两个文档：
{"x" : 15}
{"x" : [5 , 25]}

造成数组也返回的原因是因为25大于10，而且5也小于20，因此也符合查询条件。针对这种情况可以进行如下设置
- 如果希望排除非数组，可以通过\$elemMatch要求MongoDB同时使用查询条件的两个语句与一个组元素进行比较
```
db.mycoll.find({"x" : {"\$elemMatch" : {"\$gt" : 10 , "\$lt" : 20}})
```
- 如果查询字段创建了索引，可以使用min()和max()将查询范围限制为$gt和$lt的值
```
db.mycoll.find({"x" : {"$gt" : 10 , "lt" : 20}}).min({"x" : 10}).max({"x" : 20})
```

**$where查询**

\$where子句可以在查询中执行任意的javascript，这样就可以实现更多操作。但$where子句比较慢且不走索引，因此不到迫不得已不建议使用该方式。

比如查询返回两个键值相同的文档，在当前的环境中没有提供相关操作符。这里可以用$where借助javascript实现
```
db.mycoll.find({"where" : function(){
    for (var current in this) {
         for (var other in this) {
              if (current != other && this[currnet] == this[other]) {
                  return true;
              }
         }
     }
     return false;
}});
```

**游标**

数据库使用游标返回find的查询结果，客户端对游标的实现通常可以对最终结果进行有效的控制。

定义一个变量来保存find结果
```
var cursor = db.mycoll.find();
```

cursor.hasNext()检查是否还存在下一个值，cursor.next()获得该值
```
while (cursor.hasNext()) {
    obj=cursor.next();
    print(obj)
}
```

游标还实现了Javascripts的迭代器接口，所以可以在forEach中使用
```
cursor.forEach(function(x)) {
    print(x.name);
})
```

调用find时，并不会立即查询数据库，而是等待真正开始要获取结果时才会立即获取前100个结果或4MB数据(两者中最小)，这样下次调用next或者hasNext就不用再连接服务器获取结果了。当第一组数据获取结束后，会再次用getMore的方式请求更多结果。

**结果集限制**

limit可以限制返回结果的数量，例如只返回三条数据
```
db.mycoll.find().limit(3)
```

Skip则可以略过指定的数据，例如略过前三条数据
```
db.mycoll.find().skip(3)
```

如果将skip用于过滤大量数据，则性能会比较缓慢。例如对数据进行分页。最简单的数据分页方式就是通过skip不断修改偏移量结合limit实现
```
db.mycoll.find().limit(100)
db.mycoll.find().skip(100).limit(100)
db.mycoll.find().skip(200).limit(100)
```

对于数据分页可采用下列方式，而不是使用skip
```
var page1 = db.mycoll.find().sort({"date" : -1}).limit(100)
var latest = null
while (page1.hasNext()) {
     latest = page1.next();
     display(latest)
}
var page2 = db.mycoll.find({"date" : {$gt" : latest.date}});
page2.sort({"date" : -1}).limit(100)
```

sort接收一个键值对对象作为参数，键对应文档的键名，值代表排序方向，1表示升序，-1表示倒序
```
db.mycoll.find().sort({username :1 , age : -1})
```

如果一个键的值是多种类型的，其排序顺序是预先定义好的。优先级从小到大顺序如下：
1. 最小值
2. Null
3. 数字
4. 字符串
5. 对象/文档
6. 数组
7. 二进制数据
8. 对象ID
9. 布尔型
10. 日期型
11. 时间戳
12. 正则表达式
13. 最大值

**高级查询选项**

大部分驱动程序都提供了辅助函数，用于向查询添加各种选项

选项 | 说明
-- | --
$comment | 向查询添加注释
$explain | 强制mongodb报告查询执行计划
$hint | 强制MongoDB使用特定索引
$maxScan | 限制扫描的文档数量
$maxTimeMS | 指定处理游标操作的累积时间限制
$max | 指定查询中使用索引的范围上限
$min | 指定查询中使用索引的范围下限
$orderby | 返回包含根据排序规范排序文档的游标
$query | 包装查询文档
$returnKey | 强制游标仅返回索引中包含的字段
$showDiskLoc | 返回文档的磁盘位置的引用
$natural | 使用磁盘上文档顺序对文档进行排序的特殊排序

```
db.user.find()._addSpecial('$showDiskLoc',true)
```

**查询一致性**

在我们通过查询获取数据之后再对数据进行处理并保存回数据库时，结果集比较大的话，MongoDB可能存在多次返回同一个文档。因为文档预留空间不足，导致原位置无法存放，MongoDB通常会将它们移动到集合尾端，当游标扫描到尾端时就会再次返回这部分数据。

应对这个问题可以设置snapshot，查询就在_id上遍历执行，保证每个文档只返回一次，但快照会使查询变慢，只在必要时使用，例如mongodump备份数据库。
```
db.user.find().snapshot();
```

**getmore查询**

MongoDB默认的查询返回批大小为101，当查询结果超过101之后，需要通过getmore来返回剩余结果集。默认值可以通过指定batchsize来修改

```
db.user.find().batchsize(150).limit(200)
```

## 3. Update Operation

```
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>,
     collation: <document>,
     arrayFilters: [ <filterdocument1>, ... ]
   }
)
```
```
db.collection.updateOne(
   <filter>,
   <update>,
   {
     upsert: <boolean>,
     writeConcern: <document>,
     collation: <document>,
     arrayFilters: [ <filterdocument1>, ... ]
   }
)
```
```
db.collection.updateMany(
   <filter>,
   <update>,
   {
     upsert: <boolean>,
     writeConcern: <document>,
     collation: <document>,
     arrayFilters: [ <filterdocument1>, ... ]
   }
)
```
```
db.collection.findOneAndUpdate(
   <filter>,
   <update>,
   {
     projection: <document>,
     sort: <document>,
     maxTimeMS: <number>,
     upsert: <boolean>,
     returnNewDocument: <boolean>,
     collation: <document>,
     arrayFilters: [ <filterdocument1>, ... ]
   }
)
```

- filter：查询过滤器
- update：如果是要替换文档中的字段，可以使用文档替换模式，如果在原字段进行修改需要使用操作符
- projection：返回文档的字段
- sort：指定排序规则
- maxTimeMS：限定操作时间，单位为毫秒，超时则报错
- upsert：文档不存在时，自动创建一个新文档，默认为false
- returnNewDocument：返回更新前的文档还是更新后的文档，默认为false，返回更新前的文档
- collation：collation用于指定字符串比较规则
- arrayFilters：数组过滤器
- writeConcern：写入关注级别
- multi：是否批量修改，默认为false，更新只能对符合匹配条件的第一个文档进行更新

默认情况下，更新只能对符合匹配条件的第一个文档执行操作。如果需要对多个文档进行操作，需要将update的第四个参数修改为true。为了安全起见，建议显示指定该参数。如果想知道更新了多少文档可以执行getLastError
```
db.runCommand({getLastError:1})
```

**文档替换**

当前数据库存在下列文档记录
```
{
  "_id" : ObjectId("5bf4f9936dd981d267ddd1b0"),
  "name" : "joe",
  "friends" : 32,
  "enemies" : 2
}
```
现在计划将friends和enemies划分到relationships的子文档中，可以做如下修改
```
>var joe=db.U_Test.findOne({"name":"joe"})
>joe.relationships={"frieds":joe.friends,"enemies":joe.enemies}
>delete joe.friends
>delete joe.enemies
>db.U_Test.updateOne({"_id":ObjectId("5bf4f9936dd981d267ddd1b0")},joe)
{
  "_id" : ObjectId("5bf4f9936dd981d267ddd1b0"),
  "name" : "joe",
  "relationships" : {
    "friends" : 32,
    "enemies" : 2
  }
}
```

**修改器**

|               |                                                              |                                                              |
| :------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 修改器        | 作用                                                         | 示例                                                         |
| $set          | 修改指定字段的值，如果字段不存在则创建                       | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{ $set:{“name”:”hengxing”}}); |
| $unset        | 删除指定字段                                                 | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{ $unset:{“address”:1}}); |
| $currentDate  | 将当前时间值赋值给字段，字段不存在则创建                     | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{“$currentDate”:{“lastModified”:true}}); |
| $inc          | 用于增加或减少键值为数字的值，不存在则创建                   | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{“$inc”:{“age”:5}}); |
| $max          | 大于当前字段值，才会更新字段                                 | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”),{ $max:{“age”:30}}); |
| $min          | 小于当前字段值，才会更新字段                                 | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{“$min”:{“age”:25}}); |
| $mul          | 将指定值与字段值相乘                                         | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{“$mul”:{“age”:2}}); |
| $rename       | 字段重命名                                                   | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{“$rename”:{“sex”:”sexs”}}); |
| $setOnInsert  | 如果文档不存在，则将指定值赋给指定字段，存在则退出           | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{“$addToSet”:{“email”:”[1111@mail.com](mailto:1111@mail.com)“}}); |
| $             | 占位符，匹配更新第一个文档元素                               | db.user.update({“email”:”[1111@mail.com](mailto:1111@mail.com)“},{“$set”:{“email.$“:”[3333@mail.com](mailto:3333@mail.com)“}}); |
| $[]           | 占位符，匹配更新所有文档的元素                               | db.user.update({“email”:”[1111@mail.com](mailto:1111@mail.com)“},{“$set”:{“email.$[]”:”[3333@mail.com](mailto:3333@mail.com)“}}); |
| $[identifier] | 占位符，匹配更新所有满足arrayFilters条件的文档的元素         | db.user.update({“_id” : ObjectId(“5e7335dd420cd17d56e7281f”)},{“$set”:{“email.$[elem]”:”[4444@mail.com](mailto:4444@mail.com)“}},{multi:true,arrayFilters:[{“elem”:{$ne:”[5555@mail.com](mailto:5555@mail.com)“}}]}); |
| $push         | 向数组末尾添加一个元素，不存在则创建数组                     | db.U_Test.update({“_id”:ObjectId(“5bf4f9936dd981d267ddd1b0”)},{“$push”:{“comments”:{“name”:”mark”,”email”:”[mark@example.com](mailto:mark@example.com)“}}}) |
| $pop          | 从数组中弹出元素，1为从末尾，-1为头部                        | db.U_Test.update({“_id”:ObjectId(“5bf4f9936dd981d267ddd1b0”)},{“$pop”:{“enemies”:-1}}) |
| $addToSet     | 元素不存在时将元素添加到数组中                               | db.U_Test.update({“_id”:ObjectId(“5bf4f9936dd981d267ddd1b0”)},{“$addToSet”:{“name”:”jack”}}) |
| $pull         | 删除所有符合条件的元素                                       | db.U_Test.update({},{“$pull”:{“top5”:”B”}})                  |
| $pullAll      | 移除数组的所有元素                                           | db.U_Test.update({},{“$pullAll”:{“top5”:”B”}})               |
| $each         | 与$push或者addToSet一起完成批量操作，可以结合$slice限制数组的长度 | db.U_Test.update({“_id”:ObjectId(“5bf4f9936dd981d267ddd1b0”)},{“$push”:{“top5”:{“$each”:[“A”,”B”,”C”,”D”,”E”,”F”],”$slice”:-5}}}) |
| $sort         | 对字段进行排序                                               | db.U_Test.update({“_id”:ObjectId(“5bf4f9936dd981d267ddd1b0”)},{“$sort”:{“enemies”:1}}) |
| $ne           | 只有在集合中尚未存在元素时才将元素添加到数组中               | db.U_Test.update({“_id”:ObjectId(“5bf4f9936dd981d267ddd1b0”),”name”:{“$ne”:”mark”}},{“$set”:{“name”:”mark”}}) |
| $position     | 指定数组添加元素的位置                                       | db.user.update({“_id”:ObjectId(“5e7335dd420cd17d56e7281f”)},{$push: {scores: {$each: [ 40，50 ],$position: 0}}}) |
| $bit          | 执行整数值的按位和、or和异或更新                             |                                                              |

   


**填充因子**

MongoDB不得不移动一个新文档时，例如update使原有文档变大，它会修改集合的填充因子，填充因子是MongoDB为每个新文档预留的增长空间。可以执行db.collection.status()查看填充因子。随着不断的文档移动，填充因子会越来越大，反之则缓慢降低。

移动文档是非常慢的，MongoDB必须将文档原本所占的空间释放掉，然后将文档写入另一片区域。因此尽量让填充因子接近1。如果日志中频繁出现was empty,skipping ahead的字眼，说明数据库目前在频繁移动文档，存在较多碎片，意味着存在性能问题。

如果你的集合插入和删除时会进行大量的移动或者经常打乱数据，可以用usePowerOf2Sizes选项提高磁盘复用率。可以通过collMod命令来设置该选项：
```
db.runCommand({"collMod":collection,"usePowerOf2Sizes":true})
```

这个集合之后进行的所有空间分配，得到的块都是2的幂。只会影响新分配的记录，不对现有数据产生影响。该选项会导致初始空间不再那么高效，建议在需要经常打乱数据的集合上使用

**upsert**

upset是一种特殊更新，如果没有按条件找到对应文档，则以更新条件和更新文档为基础创建一个新的文档,找到则正常更新
```
db.analytics.update({"url":"/blog"},{"$inc":{"pageviews":1}},true)
```

**save**

save是一个shell函数，传入一个文档，如果文档不存在，它会自动创建文档，如果文档存在，它会更新这个文档。要是这个文档带有_id键，save会调用upsert，否则调用insert

**findAndModify**

findAndModify具有原子性，能够在一个操作中返回匹配结果以及更新，适用于大批量查询更新的场景。
```
process=db.runCommand({"findAndModify":"processes",
"query":{"status":"READY"},
"sort":{"priority":-1},
"update":{"$set":{"status":"RUNNING"}}}
).value
```

findAndModify支持很多字段：
- query：检索文档的条件
- sort：排序结果的条件
- update：用于对文档进行匹配更新
- remove：布尔类型，表示是否删除文档
- new：布尔类型，表示返回更新前的文档还是更新后的文档，默认为更新前
- fields：文档中需要返回的字段
- upset：布尔类型，值为true则使用upset，默认为false



## 4. Delete Operation

db.collection.remove()
```
db.collection.remove(
   <query>,
   {
     justOne: <boolean>,
     writeConcern: <document>
   }
)
```
- query:删除文档的条件，如果不指定则删除整个集合
- justOne：是否只删除一个文档，如果为true则只删除一个，false则删除所有匹配的文档
- writeConcern：写入确认，指定为majority可以确保集群中大多数节点都已删除

db.collection.deleteMany()
```
db.collection.deleteMany(
   <filter>,
   {
      writeConcern: <document>,
      collation: <document>
   }
)
```

db.collection.deleteOne()
```
db.collection.deleteOne(
   <filter>,
   {
      writeConcern: <document>,
      collation: <document>
   }
)
```
- collation: collation用于指定字符串比较规则

在3.2版本中通过findOneAndDelete()也可以对文档进行查找删除，并返回删除的文档信息
```
db.collection.findOneAndDelete(
   <filter>,
   {
     projection: <document>,
     sort: <document>,
     maxTimeMS: <number>,
     collation: <document>
   }
)
```
- projection：选择返回的文档字段，省略则返回全部字段
- sort：指定排序方式
- maxTimeMS：指定操作时间限制，超过则报错，单位为毫秒



## 5. Bulk Write

MongoDB3.2支持通过`db.collection.bulkWrite()`的方式执行批量插入，更新和删除操作。Bulk write可以有序操作也可以无序操作，有序操作则按顺序执行操作，如果发生错误，不处理后续的其它操作；无序操作时，MongoDB可以并行执行，发生错误时将继续处理其它操作。Bulk write默认为有序操作，无序操作需要设置`ordered:false`

bulkWrite()支持下列写操作：
- insertOne
- updateOne
- updateMany
- replaceOne
- deleteOne
- deleteMany
```
try {
   db.characters.bulkWrite(
      [
         { insertOne :
            {
               "document" :
               {
                  "_id" : 4, "char" : "Dithras", "class" : "barbarian", "lvl" : 4
               }
            }
         },
         { insertOne :
            {
               "document" :
               {
                  "_id" : 5, "char" : "Taeln", "class" : "fighter", "lvl" : 3
               }
            }
         },
         { updateOne :
            {
               "filter" : { "char" : "Eldon" },
               "update" : { $set : { "status" : "Critical Injury" } }
            }
         },
         { deleteOne :
            { "filter" : { "char" : "Brisbane" } }
         },
         { replaceOne :
            {
               "filter" : { "char" : "Meldane" },
               "replacement" : { "char" : "Tanys", "class" : "oracle", "lvl" : 4 }
            }
         }
      ]
   );
}
catch (e) {
   print(e);
}
```

>针对分片环境的批量插入可能影响集群性能，建议考虑下列策略：1. 预分片 2.无序操作

如果要批量更新不同的数据，平铺的多个操作需要一个个解析去执行，效率非常慢，可以通过下列方式快速执行多个update语句

```
var bulk = db.items.initializeUnorderedBulkOp();
bulk.find( { status: "D" } ).update( { $set: { status: "I", points: "0" } } );
bulk.find( { item: null } ).update( { $set: { item: "TBD" } } );
bulk.execute();
```

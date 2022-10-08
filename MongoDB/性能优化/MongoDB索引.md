# MongoDB索引与执行计划

索引类似于书本中的目录索引页，能够快速查找数据。在无索引的情况下，查询总是遍历整个集合来查找相应的数据。在集合数据量比较大时，全部遍历的效率是低下的。如果创建合适的索引，将通过遍历索引数据来查找对应的文档。

另外，索引数据是有序的，因此我们可以通过索引字段排序，减少了排序消耗。但索引并不是越多越好，因为每次写操作都需要维护对应的索引。

![Index](https://www.mongodb.com/docs/manual/images/index-for-sort.bakedsvg.svg)

**创建索引**

```
db.mycoll.createIndex(<key and index type specification>, <options>)
```

批量创建索引
```
db.getSiblingDB("query_data").runCommand(
  {
    createIndexes: "ng_t_offline_merchant_acq_order",
    indexes: [
        {
            key: {
                aggregator_id:-1
            },
            name: "idx_aggregator_id"
			background:true
        },
        {
            key: {
                sn:-1
            },
            name: "idx_sn",
            background: true
        }
    ],
    writeConcern: { w: "majority" }
  }
)
```
常用选项：
- background：如果集合数据量比较大，在创建索引是会暂时阻塞，如果不希望发生这种情况，可以设置为true在后台创建索引
- name：创建的索引名称通常是以下划线将索引字段和排序方向分隔，可以通过name指定索引名称
- unique：为true则表示唯一索引

> MongoDB在创建集合期间自动在_id字段上创建唯一索引，该索引主要防止客户端插入两个相同字段值的文档。该索引不能被删除



## 1. 索引类型

### 1.1 单键索引

除MongoDB定义的_id索引外，MongoDB还支持在文档的单个字段上创建用户定义的升序/降序索引

![Single Field Indexes](https://docs.mongodb.com/manual/images/index-ascending.bakedsvg.svg)

在针对嵌入文档和嵌入字段创建索引是不同的。嵌入字段上创建索引，可以通过点符号查询来使用索引；而在嵌入文档上创建索引包含了索引中嵌入文档的最大内容，并且要求查询字段顺序与嵌入文档一致。

```
{
	"_id": ObjectId("570c04a4ad233577f97dc459"),
	"score": 1034,
	"location": { state: "NY", city: "New York" }
}
```

创建嵌入字段索引

```
db.records.createIndex( { "location.state": 1 } )
```

使用该索引

```
db.records.find( { "location.state": "NY" } )
```

创建嵌入文档索引

```
db.records.createIndex( { location: 1 } )
```

下列查询能使用索引，但查不到相应数据

```
db.records.find( { location: { city: "New York", state: "NY" } } )
```

[Single Field Indexes]:https://docs.mongodb.com/manual/core/index-single/



### 1.2 复合索引

MongoDB还支持多个字段的用户定义索引，即复合索引。复合索引除了支持所有索引字段的匹配查询外，还支持索引前缀字段的匹配查询。对于复合索引，索引的排序方向很重要，将决定排序能否使用索引

![index-compound-key](https://docs.mongodb.com/manual/images/index-compound-key.bakedsvg.svg)

插入测试数据

```
for (i=0;i<10000;i++) {
  db.users.insert(
    {
      "id":i,
      "username":"user"+i,
      "age":Math.floor(Math.random()*120),
      "created":new Date()
    }
  );
}
```

创建复合索引

```
db.users.createIndex( { "username": 1, "age": -1 } )
```

使用索引前缀字段也可以使用复合索引

```
db.users.find( { username: "user101" } )
```

以下两个查询是等价的，都能使用上述索引

```
db.users.find().sort({"username" : 1 , "age" : -1})
db.users.find().sort({"username" : -1 , "age" : 1})
```

但下列查询将不会使用该索引

```
db.users.find().sort({"username" : 1 , "age" : 1})
```

[Compound Indexes]:https://docs.mongodb.com/manual/core/index-compound/



### 1.3 多键索引(multikey index)

MongoDB用多键索引来存储数组中的索引数据，如果索引包含数组值的字段，MongoDB会为每个元素创建单独的索引条目。可以在即包含标量值又包含嵌套文档的数组上创建多键索引

![index-multikey](https://docs.mongodb.com/manual/images/index-multikey.bakedsvg.svg)

[Multikey Indexes]:https://docs.mongodb.com/manual/core/index-multikey/



### 1.4 地理空间索引

MongoDB支持几种类型的地理空间索引。较为常见的有2dsphere(地球表面类型的地图)索引和2d索引(平面地图和时间连续的数据)

**[2dsphere]**

2dsphere允许使用GeoJSON格式指定点、线和多边形。点可以用形如longitude(经度)、latitude(纬度)两个元素的数组表示

```
{
  "name":"New York",
  "loc":{
    "type":"point",
    "coordinates":[50,2]
  }
}
```

线则是又点组成的数组来表示

```
{
  "name":"shen zhen",
  "loc":{
    "type":"Line",
    "coordinates":[[0,1],[0,2],[1,2]]
  }
}
```

多边形的表示方式与线一样，但是type不同

```
{
  "name":"New England",
  "loc":{
    "type":"Polygon",
    "coordinates":[[0,1],[0,2],[1,2]]
  }
}
```

创建索引时指定2dsphere选项可以创建一个地理空间索引

```
db.user.createIndex({"loc":"2dsphere"})
```

地理空间查询可以包括交集(intersection)、包含(within)、接近(nearness)等类型。查询时，需要将查询的内容指定为{“$geometry”:geoJsonDesc}格式的GeoJSON对象

- 查询交集

  ```
  var shenzhen={
    "type":"Polygon",
    "coordinates":[
      [114.05,22.55],
      [114.12,22.55],
      [114.05,22.53],
      [113.90,22.57]
    ]
  }
  db.users.find({"loc":{"$geoIntersects":{"$geometry":shenzhen}}})
  ```

- 查询包含

  ```
  db.users.find({"loc":{"$geoWithin":{"$geometry":shenzhen}}})
  ```

- 查询附近

  ```
  db.users.find({"loc":{"$near":{"$geometry":shenzhen}}})
  ```

  > $near会对结果进行自动排序，按距离由近到远排序

复合地理空间索引可以利用前缀字段快速过滤部分字段，在进行地理空间查询时能更快得到结果

[2dsphere Indexes]:https://docs.mongodb.com/manual/core/2dsphere/

**[2d索引]**

2d索引适用于二维平面,只能对点进行索引，可以保存一个由点组成的数组，但它不会被练成线。

创建2d索引

```
db.<collection>.createIndex( { <location field> : "2d" ,
                               <additional field> : <value> } ,
                             { <index-specification options> } )
```

其中index-specification可以包含下列选项

```
{ min : <lower bound> , max : <upper bound> , bits : <bit precision> }
```

min和max用来定义位置范围，例如-1000和1000就能构造一个2000 * 2000的空间索引；bits用来定义精度，默认使用26位精度，范围为-180到180，大约为2英尺或者60厘米的精度

查询包含

```
db.<collection>.find( { <location field> :
                         { $geoWithin :
                            { $box|$polygon|$center : <coordinates>
                      } } } )
```

- $box接受一个两元素的数组，第一个元素指定左下角的坐标，第二个元素指定右上角的坐标，以此来绘制一个矩形范围
- $center接受一个两元素的数组，第一个元素用于指定圆心，第二个参数用于指定半径，以此来绘制一个圆形范围
- $polygon可以用多个点的数组来指定多边形范围

查询相近

```
db.<collection>.find( { <location field> :
                         { $near : [ <x> , <y> ]
                      } } )
```

[2d Indexes]:https://docs.mongodb.com/manual/core/2d/



### 1.5 文本索引

文本索引支持对字符串内容的文本进行搜索查询，文本索引可以包含任何值为字符串或字符串元素数组的字段。

创建文本索引

```
db.users.createIndex({"address":"text"})
```

> 一个集合最多可以有一个文本索引，文本索引可以是包含多个字段的复合索引，再使用复合索引时，建议利用前缀索引先过滤一部分数据，再进行文本索引搜索

文本索引的索引名称默认由字段名_text组成，我们也可以在创建文本索引时指定索引名称

```
db.users.createIndex({"address" : "text"},{"name":"idx_addr_text"})
```

删除索引时也需要指定要删除的索引名称

```
db.users.dropIndex("idx_addr_text")
```

与普通的多键索引不同，文本索引中的字段顺序并不重要，可以为每个字段设置不同的权重来控制其优先级，MongoDB将匹配数乘以权重并求和，再以总和来计算文档的score。索引字段的默认权重为1，要调整索引字段的权重，可以在创建索引时设置weights来指定权重

```
db.users.createIndex({"city":"text","address":"text"},{"weights":{"city":3,"address":2}})
```

如果要在集合上的所有字符串字段创建全文索引，可以使用$**通配符来创建

```
db.users.createIndex("$**":"text")
```

与索引数据相关联的默认语言决定了解析词根和忽略停用词的规则，索引的默认语言是English。要指定其它语言就需要在创建索引时设置default_language，默认还不支持中文。并且我们也可以指定一个非语言的字段，然后在文档中通过该字段设置语言

```
> db.users.createIndex( { address : "text" }, { language_override: "idioma" } )
> db.quotes.find()
{ _id: 1, idioma: "portuguese", address: "china guangdong shenzhen" }
{ _id: 2, idioma: "spanish", address: "china jiangxi nanchang" }
{ _id: 3, idioma: "english", address: "china zhejiang hangzhou" }
```

使用文本查询时需要通过$text来查询

```
db.users.find({ $text: { $search: "shenzhen" }})
```

text 索引具有以下存储要求和性能成本：

- text索引可以很大。对于每个插入的文档，每个索引字段中的每个唯一后词形词都包含一个索引条目。
- 创建text索引速度比较慢，所消耗的时间会比较长
- 在text现有集合上建立较大索引时，请确保对打开文件描述符的限制足够高
- text 索引将影响插入吞吐量，因为MongoDB必须在每个新源文档的每个索引字段中为每个唯一的词干词添加一个索引条目

[Text Indexes]:https://docs.mongodb.com/manual/core/index-text/



### 1.6 散列索引

哈希索引支持使用哈希片键进行分片。基于散列的分片使用字段的哈希索引作为片键，以跨分片群集对数据进行分区。哈希索引使用哈希函数来计算索引字段值的哈希，使用散列的分片键对集合进行分片会导致数据分布更加随机

创建哈希索引

```
db.users.createIndex({"_id":"hashed"})
```

> 哈希索引支持对任何单个字段hash，但不支持多键索引

[hashed Indexes]:https://docs.mongodb.com/manual/core/index-hashed/



## 2. 索引属性

**唯一索引**

唯一索引可以确保集合的每个文档索引字段都是唯一的，避免产生重复数据。例如身份证号必须唯一，_id字段的索引就是唯一索引。

Index bucket是有大小限制的，如果键超过了1KB，则不会包含在索引中，也就不会受到唯一约束的限制，可以插入多个1KB长的字符串

在已有的集合上创建唯一索引时，可能已经存在重复值，导致索引创建失败。可以选择找出重复数据并处理，如果希望直接删除重复数据，可以指定dropDups选项，如果遇到重复值，第一个会保留，之后的重复文档都直接删除

```
db.users.createIndex({"username":1},{"unique":true,"dropDups":true})
```

**TTL索引**

TTL索引可以为每个文档设置一个超时时间，当文档超时后，将会被删除。为了防止活跃的会话被删除，可以将索引字段更新为当前时间。TTL索引不能是复合索引。

```
db.users.createIndex({"lastUpdated":1},{"expireAfterSecs":60*60*24});
```

**部分索引**

在MongoDB3.2中推出了部分索引，能够针对符合指定过滤器表达式的文档创建索引，部分索引具有较低的存储要求，降低了索引创建和维护成本。部分索引提供了稀疏索引功能的超集，应该优先于稀疏索引

**稀疏索引**

索引的稀疏属性可以确保索引仅包含具有索引字段的文档条目，索引会跳过没有索引字段的文档。

```
db.users.createIndex({"email":1},{"unique":true,"sparse":true})
```



## 3. 索引优化

**低效的操作符**

有一些查询时完全无法使用索引的，例如$where查询和$exists。通常来说取反的效率是比较低的，$ne查询可以使用索引，但是它需要遍历所有的索引条目；$not有时能够使用索引，但它不知道如何使用索引，它能够对基本的返回和正则表达式进行反转，然后大多数$not查询都会退化为全集合扫描。

如果要针对这种情况进行优化，建议加入一个能够使用索引的语句到查询中，先一步降低结果集的大小，从而使得整体的效率能够提升一点。

**范围查询**

在设计多个字段的索引时，应该将例如{“x” : 27}此类能够精确定位的字段放在索引的前面，而用于范围查询的字段放在最后，这样，查询就能先使用第一个索引键精确匹配结果集，再对这个结果集进行二次的范围查询。这样索引的效率是最高的

**OR查询**

MongoDB在一次查询中只能使用一个索引。但$or是一种特殊情况，它实际上是执行两次查询再合并结果集。合并结果集并不如单次查询快。因此，尽可能还是优先使用$in而不是$or

**索引基数**

索引基数就是集合中字段拥有不同值的数量，比如boolean只有true和false两个值，选择度较低。通常基数越高，越适合键索引，因为能够快速过滤大部分数据。

**强制索引**

某些情况下查询没有使用你希望的索引，可以通过hint来强制走某一个索引，但是强制索引可能带来更坏的情况。因此，建议在上线前测试后重新评估

```
db.mycoll.find({"age" : 14 , "username" : /.*/}).hint({"username" : 1 , "age" : 1})
```

**强制全表扫**

当索引基数比较小时，即查询要返回集合大部分数据，这时索引带来的效率可能还没有全部扫描来的快，因为索引需要扫描两次，全部扫描只要一次。这时我们可能希望能够强制查询走全部扫描。

```
db.mycoll.find({"created_at" : {"\$lt" : "hourAgo"}}).hint({"\$natural" : 1})
```

> $natural有一个副作用，即返回顺序是按照磁盘顺序排列的

**执行计划**

执行计划能够协助分析语句的执行过程以及状态统计，我们可以通过db.collection.explain()来获取操作对应的执行计划，例如：

```
db.products.explain().remove( { category: "apparel" }, { justOne: true } )
```

explain()包含三个选项，用于指定输出的详细模式

- queryPlanner：默认选项，运行查询优化器来为评估中的操作选择最佳计划
- executionStats：查询优化器选择最佳计划，执行最佳计划并返回最佳计划执行情况的统计信息
- allPlansExecution：运行查询优化器选择执行计划并执行最佳计划。在”allPlansExecution”模式下，MongoDB返回描述最佳计划执行情况的统计信息以及在计划选择期间捕获的其他候选计划的统计信息

> 对于更新或删除操作，explain()并不会修改数据库信息



## 4. 管理索引

**查看集合上的索引**

```
db.collection.getIndexes()
```

**列出数据库上的所有索引**

```
db.getCollectionNames().forEach(function(collection) {
   indexes = db[collection].getIndexes();
   print("Indexes for " + collection + ":");
   printjson(indexes);
});
```

**列出指定类型的索引**

```
db.adminCommand("listDatabases").databases.forEach(function(d){
   let mdb = db.getSiblingDB(d.name);
   mdb.getCollectionInfos({ type: "collection" }).forEach(function(c){
      let currentCollection = mdb.getCollection(c.name);
      currentCollection.getIndexes().forEach(function(idx){
        let idxValues = Object.values(Object.assign({}, idx.key));

        if (idxValues.includes("hashed")) {
          print("Hashed index: " + idx.name + " on " + idx.ns);
          printjson(idx);
        };
      });
   });
});
```

**删除指定索引**

```
db.collection.dropIndex({"name":1})
```

**删除集合的所有索引**

```
db.collection.dropIndexes()
```
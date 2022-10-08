[TOC]

# MongoDB执行计划

## 获取执行计划

explain命令支持提供下列命令的执行计划信息：aggregate、count、distinct(v3.2)、find、findAndModify(v3.2)、delete、MapReduce(v4.4)、update。

```sql
{
   explain: <command>,
   verbosity: <string>,
   comment: <any>
}
```

通常建议优先使用db.collection.explain()或cursor.explain()的方式来查看执行计划，其内部封装了explain命令。

```sql
db.products.explain().count( { quantity: { $gt: 50 } } )
```

db.collection.explain()操作会返回下列信息：

- explainVersion：输出格式版本

- command：详细说明所解释的命令

- queryPlanner：查询优化器选择了哪些详细计划并列出了被拒绝的计划

- executionStats：详细说明最终选择的执行计划和落选计划的执行情况

- serverInfo：提供有关MongoDB实例的信息

- serverParameters：详细说明内部参数


db.collection.explain.find()和db.collection.find().explain()看起来相似，但两者存在以下区别：

- db.collection.explain().find()构造允许额外的查询修饰符链接，具体请查看`db.collection.explain().find().help()`。例如下列查询使用了sort和hint修饰符

  ```sql
  db.products.explain("executionStats").find(
     { quantity: { $gt: 50 }, category: "apparel" }
  ).sort( { quantity: -1 } ).hint( { category: 1, quantity: -1 } )
  ```

- db.collection.explain().find()返回一个游标，它需要调用next()或finish()来返回结果。mongosh中交互运行时，会自动调用next()，对于脚本则需要显示调用next()

  ```sql
  var explainResult = db.products.explain().find( { category: "apparel" } ).next();
  ```

**Verbosity**

db.collection.explain()支持verbosity参数，它是可选的，其会影响explain()的行为，并决定返回的信息量。可选的模式包含：

- queryPlanner：默认模式。MongoDB运行查询优化器来评估操作的胜出的计划，并返回评估的queryPlanner信息
- executionStats：MongoDB运行查询优化器来选择胜出的执行计划，执行胜出计划直至完成，并返回描述胜出的计划执行情况的统计数据。对于写操作，返回有关将要执行的更新或删除操作的信息，但不将修改应用到数据库。
- allPlansExecution：MongoDB运行查询优化器以选择胜出的计划并执行胜出的计划。 在“AllplanSexecution”模式下，MongoDB返回描述胜出计划执行情况的统计数据，以及计划选择过程中捕获的其他候选计划的统计数据。对于写操作，返回有关将要执行的更新或删除操作的信息，但不将修改应用到数据库。

> 为了兼容早期的版本，MongoDB将true解释为allPlansExecution，false解释为queryPlanner

## 分析执行计划

下面是一个执行计划queryPlanner部分的示例，描述了优化器选择执行计划的详细信息。其内部包含下列字段：

```
    "queryPlanner" : {
        "mongosPlannerVersion" : 1.0, 
        "winningPlan" : {
            "stage" : "SINGLE_SHARD", 
            "shards" : [
                {
                    "shardName" : "shard1", 
                    "connectionString" : "shard1/10.0.139.161:27017,10.0.139.162:27017,10.0.139.163:27017", 
                    "serverInfo" : {
                        "host" : "10.0.139.161", 
                        "port" : 27017, 
                        "version" : "4.0.14", 
                        "gitVersion" : "1622021384533dade8b3c89ed3ecd80e1142c132"
                    }, 
                    "plannerVersion" : 1.0, 
                    "namespace" : "test1", 
                    "indexFilterSet" : false, 
                    "parsedQuery" : {
                        "$and" : [
                            {
                                "status" : {
                                    "$eq" : "SUCCESS"
                                }
                            }, 
                            {
                                "type" : {
                                    "$eq" : "PALMPAY"
                                }
                            }, 
                            {
                                "type" : {
                                    "$eq" : "coupon"
                                }
                            }
                        ]
                    }, 
                    "winningPlan" : {
                        "stage" : "SORT", 
                        "sortPattern" : {
                            "coupon_time" : -1.0
                        }, 
                        "limitAmount" : 10.0, 
                        "inputStage" : {
                            "stage" : "SORT_KEY_GENERATOR", 
                            "inputStage" : {
                                "stage" : "FETCH", 
                                "filter" : {
                                    "$and" : [
                                        {
                                            "type" : {
                                                "$eq" : "PALMPAY"
                                            }
                                        }, 
                                        {
                                            "type" : {
                                                "$eq" : "coupon"
                                            }
                                        }
                                    ]
                                }, 
                                "inputStage" : {
                                    "stage" : "IXSCAN", 
                                    "keyPattern" : {
                                        "status" : 1.0, 
                                        "activity" : 1.0
                                    }, 
                                    "indexName" : "idx_mo", 
                                    "isMultiKey" : false, 
                                    "multiKeyPaths" : {
                                        "status" : [

                                        ], 
                                        "activity" : [

                                        ]
                                    }, 
                                    "isUnique" : false, 
                                    "isSparse" : true, 
                                    "isPartial" : false, 
                                    "indexVersion" : 2.0, 
                                    "direction" : "forward", 
                                    "indexBounds" : {
                                        "status" : [
                                            "[\"SUCCESS\", \"SUCCESS\"]"
                                        ], 
                                        "activity" : [
                                            "[MinKey, MaxKey]"
                                        ]
                                    }
                                }
                            }
                        }
                    }, 
                    "rejectedPlans" : [

                    ]
                }
            ]
        }
    }
```

- namespace：查询的对象
- indexFilterSet：布尔值，用于指定MongoDB是否对查询范式应用了索引过滤器
- queryHash：表示查询范式的十六进制哈希值，queryHash 可以帮助识别具有相同范式的慢查询
- planCacheKey：与 queryHash 不同的是，当添加/删除影响查询范式的索引，planCacheKey可能会发生变化
- optimizedPipeline：一个布尔值，指示整个聚合管道操作已被优化，由一个查询计划执行阶段来完成，
- winningPlan：详细描述获胜的执行计划信息，其结构呈现为阶段树
- winningPlan.stage：阶段名称
- winningPlan.inputStage：描述子阶段的文档，它向其父级提供文档或索引键。如果有多个子阶段，则为`winningPlan.inputStages`
- rejectedPlans：查询优化器考虑和拒绝的候选计划的数组，没有候选计划则为空

Stage(阶段)是对操作的描述，其主要包含以下操作类型：

- COLLSCAN：对集合进行扫描
- IXSCAN：对索引进行扫描
- FETCH：用于检索文档
- SHARD_MERGE：用于合并来自不同SHARD的结果集
- SHARDING_FILTER：用于从SHARD中过滤孤立的文档

[Explain Results]:https://docs.mongodb.com/manual/reference/explain-results/

**执行计划如何选择索引**

MongoDB会根据查询语句的条件来筛选索引，例如`db.col1.find({type:2 , status:4 , create_time:{$gte:new Date(2021,08,20)}})`可能选择三个字段上的索引或者包含这些字段的复合索引，其中就会产生多个可能的执行计划方案。但最终执行计划只会选择一个最优的，其判断的方法公式如下：

- 扫描次数(Q)

```python
if(count(collection)*0.29 < 10000):
	scan=10000
else:
	scan=count(collection)*0.29
```

- score(S)

  score主要从下列两个方面考虑：

  - common.advanced：索引扫描时能够命中符合条件的记录数，与总扫描次数相除就是对应的分数值
  - tieBreakers：由noFetchBonus、noSortBonus、noIxisectBonus组成，这三个值主要是控制不选则下面三种状态：STAGE_FETCH、STAGE_SORT、STAGE_AND_HASH/STAGE_AND_SORTED，这三种状态非常影响性能，一旦出现则会设置为0

  因此，score的计算公式为：

  ```
  score=1+(common.advanced/扫描次数)+tieBreakers
  ```
  
  

除此之外，还有一个状态也起着决定性作用，也就是`IS_EOF`，当某个索引的命中数量小于扫描次数(Q)时，就会提前结束扫描，将标志位设置为IS_EOF，直接进入score计算环节，如果状态为IS_EOF，score+1，基本都会被选中为最佳执行计划

```
db.col1.count({type:2})
160000
db.col1.count({status:4})
50000
db.col1.count({create_time:{$gte:new Date(2021,08,20)}})
284320
```

这里status字段索引命中数量最低，所以会最先达到IS_EOF状态，它扫描的记录数量最小，因此score也是最高的，因此这里就会选择status字段上的索引。



## Explain Cache

MongoDB会选择最优的执行计划缓存在cache中，对同一类SQL都会做判断是否采用cache中的执行计划，主要步骤如下：

- 计算cache中索引命中数量乘以10得到需要扫描的数量(Q)
- 用索引扫描Q次，如果索引扫描的记录到collection中扫描筛选符合条件的记录超过101次，则继续使用该执行计划，否则重新生成执行计划；索引扫描命中的数量小于Q，则为达到IS_EOF状态，继续使用该执行计划；
- 如果在扫描过程遇见错误，则会返回FAILURE，也会触发重新生成执行计划

因此，可以看出只要命中数量小于cache*10，就会继续采用cache中的执行计划。这种“偷懒”的方式有时会忽略真正最优的执行计划，导致大量慢查询，反而降低了数据库的性能。[SERVER-32452]([[SERVER-32452\] Replanning may not occur when a plan with an extremely high 'works' value is cached - MongoDB Jira](https://jira.mongodb.org/browse/SERVER-32452))就是描述该问题现象的，目前V4.1.1已优化解决

**临时处理方案**

查看缓存的执行计划

```
db.col1.getPlanCache().listQueryShapes()
db.col1.getPlanCache().getPlansByQuery({type:2 , status:4 , create_time:{$gte:new Date(2021,08,20)}})
```

清理缓存计划

```
db.col1.getPlanCache().clear()
```

[PlanCache.clearPlansByQuery() — MongoDB Manual](https://docs.mongodb.com/manual/reference/method/PlanCache.clearPlansByQuery/)

当然，有时我们需要固定执行计划，可以通过人工干预的方式影响执行计划的选择，主要包含两种方式：

1. 客户端干预，通过hint函数影响索引选择，常用于对比不同索引下的执行效率

   ```
   db.collection.find({col1:{$lte:50}}).sort({col2:1}).hint({col1:1})
   ```

   

2. 服务端干预，通过planCacheSetFilter指定操作对应的执行计划

   ```
   db.runCommand({planCacheSetFilter: "collection",query:{col1:{$lte:50}},sort:{col2:1},indexes:[{col1:1}]})
   ```

   [深入解析 MongoDB Plan Cache]:https://mongoing.com/archives/5624
   [planCacheSetFilter]:https://docs.mongodb.com/v4.0/reference/command/planCacheSetFilter/index.html


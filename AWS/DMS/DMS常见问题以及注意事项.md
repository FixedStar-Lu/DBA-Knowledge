[TOC]

# DMS常见问题以及注意事项

## 注意事项

1. 表名称采取通配符(%)时，后续新增的表只有在重启进程之后才会添加到同步中
2. 

## 常见问题

### 1. MongoDB同步Redshift结构问题

MongoDB同步到Redshift时，由于MongoDB是非结构化数据，初始化同步(Full Load)时表结构采样受源终端节点docsToInvestigate参数影响，可能存在同步丢失部分字段的情况，可尝试加大docsToInvestigate参数

> docsToInvestigate：表示预览指定文档数以确定文档结构。当NestingLevel为one时指定，默认值为1000

增量同步过程中，新的文档中包含之前不存在的字段，则新出现的字段将不会被同步，DMS不支持暂不支持新增字段。可考虑下列方式转变一下思路：

- 创建mongodb源endpoint时，元数据模式选择doc，_id作为单独的列存在

![image-20210826181411884](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20210826181411884.png)

- 利用该endpoint创建同步任务，目标表将包含两个字段，一个是`_id`，一个是`_doc`，doc中包含了初`_id`外的所有数据，格式为JSON

  ![image-20210827112245610](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20210827112245610.png)

- 程序通过json函数解析doc字段获取相应的数据

  ```
  SELECT
  	ID,
  	doc. NAME::varchar,
  	doc.age::int,
  	doc.address.coutry::varchar,
      doc.address.city::varchar,
      doc.address.code::int,
  	doc.phone::int,
  	doc.email::varchar
  FROM
  (
  	SELECT
  		A ._id AS ID,
  		JSON_PARSE (A ._doc) AS doc
  	FROM
  	report_mysql_ng.test_dms_lhx A
  )
  ```

  ![image-20210827112316835](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20210827112316835.png)

  > 注意：
  >
  > 1、JSON_PARSE受key的大小写影响，如果key中包含大写则匹配不了，这时可以采用json_extract_path_text来实现
  >
  > ```
  > SELECT
  > _id as id,
  > json_extract_path_text (_doc, 'sequenceId') AS sequenceId,
  > json_extract_path_text (_doc, 'channel') AS channelchannel,
  > json_extract_path_text (_doc, 'transId') AS transId,
  > json_extract_path_text (_doc, 'apiType') AS apiType,
  > json_extract_path_text (_doc, 'requestBody') AS requestBody,
  > json_extract_path_text (_doc, 'httpStatus') AS httpStatus,
  > json_extract_path_text (_doc, 'responseBody') AS responseBody,
  > timestamp 'epoch' + (json_extract_path_text (_doc, 'createTime','$date')::int8)/1000 * interval '1 second'  AS createTime,
  > json_extract_path_text (_doc, 'url') AS url,
  > json_extract_path_text (_doc, 'requestText') AS requestText,
  > json_extract_path_text (_doc, 'responseText') AS responseText,
  > json_extract_path_text (_doc, 'startTime') AS startTime,
  > json_extract_path_text (_doc, 'endTime') AS endTime,
  > json_extract_path_text (_doc, 'duration') AS duration,
  > json_extract_path_text (_doc, 'transStatus') AS transStatus,
  > json_extract_path_text (_doc, 'responseMessage') AS responseMessage,
  > json_extract_path_text (_doc, 'responseCode') AS responseCode,
  > json_extract_path_text (_doc, 'errorType') AS errorType,
  > json_extract_path_text (_doc, 'exception') AS "exception"
  > FROM
  > channel_transaction_doc
  > ```
  >
  > 2、如果开启了并行加载，请注意参数max_lob_size，该参数限制了lob最大大小，如果行大小超过了参数限制会导致数据被阶段，从而导致JSON解析失败
  
  [在 Amazon Redshift 中接收和查询半结构化数据]:https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/super-overview.html

### 2. MySQL ALTER操作解析问题

DMS语法解析树存在一点问题，例如下列SQL无法进行同步，并且任务状态显示正常，但后续对该表的操作都无法同步

```
alter table tmp.tmp_20210715
   add column test01 int comment "测试字段1";
```

修改为下列格式就不会出现该问题

```
alter table tmp.`tmp_20210715`
   add column test01 int comment "测试字段1";
```

这个问题是由于SQL中包含换行以及多个空格，导致语法解析对表名的取值出现不该有的空格，从而无法正确匹配对应表

```
2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208051 [SOURCE_CAPTURE ]T: SQL statement = 'alter table tmp.tmp_20210715 add column test02 int comment "测试字段2"' (mysql_endpoint_capture.c:1720)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208064 [SOURCE_CAPTURE ]T: Found DB 'tmp' (mysql_endpoint_util.c:794)

2021-07-15T10:48:55:208073 [SOURCE_CAPTURE  ]T:  DDL DB = 'tmp', table = 'tmp_20210715 ', verb = 1  (mysql_endpoint_capture.c:1734)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208080 [SOURCE_CAPTURE ]T: Get DDL type for verb 1 of table 'tmp_20210715 ' (mysql_endpoint_capture.c:1777)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208093 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208099 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208105 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208111 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208116 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208122 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208135 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208140 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208146 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208152 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208157 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208163 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208168 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208185 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208194 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208203 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208212 [SOURCE_CAPTURE ]T: table owner 'tmp', owner exclude pattern '%', match_status=1 (endpointshell.c:6340)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208221 [SOURCE_CAPTURE ]T: table name 'tmp', name exclude pattern '%', match_status=0 (endpointshell.c:6346)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208229 [SOURCE_CAPTURE ]T: table owner 'tmp', owner include pattern 'tmp', match_status=1 (endpointshell.c:6365)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208239 [SOURCE_CAPTURE ]T: table name 'tmp', name include pattern 'tmp', match_status=0 (endpointshell.c:6371)

2021-07-15T18:48:55.000+08:00	2021-07-15T10:48:55:208253 [SOURCE_CAPTURE ]T: Table 'tmp_20210715 ' is not in task scope (mysql_endpoint_capture.c:1831)
```

> 根据AWS支持工程师的反馈，该问题目前仅针对navicat客户端，对workbench之类的未有影响

MySQL同步到MySQL时，新增字段同步到目标端之后会默认指定UTF16的排序规则，导致在进行Join操作时存在因排序规则不一致而导致的`Illegal mix of collations`报错，已跟官方确定为Bug，只能重新加载

### 3. mongodb字段类型问题

在将mongodb同步到redshift的场景下，当mongodb同一字段存在不同类型，例如同时存在string和int类型，该字段的数据将不会被同步。我们可以通过$type来验证字段的类型

```
var group = {
    '_id': {$type:"$score"},
    'count': {
        '$sum': 1
    }
}
db.user_score.aggregate([
	{
	  $group:group
	}
])
```

| Type                       | Number | Alias                 | Notes                      |
| :------------------------- | :----- | :-------------------- | :------------------------- |
| Double                     | 1      | "double"              |                            |
| String                     | 2      | "string"              |                            |
| Object                     | 3      | "object"              |                            |
| Array                      | 4      | "array"               |                            |
| Binary data                | 5      | "binData"             |                            |
| Undefined                  | 6      | "undefined"           | Deprecated.                |
| ObjectId                   | 7      | "objectId"            |                            |
| Boolean                    | 8      | "bool"                |                            |
| Date                       | 9      | "date"                |                            |
| Null                       | 10     | "null"                |                            |
| Regular Expression         | 11     | "regex"               |                            |
| DBPointer                  | 12     | "dbPointer"           | Deprecated.                |
| JavaScript                 | 13     | "javascript"          |                            |
| Symbol                     | 14     | "symbol"              | Deprecated.                |
| JavaScript code with scope | 15     | "javascriptWithScope" | Deprecated in MongoDB 4.4. |
| 32-bit integer             | 16     | "int"                 |                            |
| Timestamp                  | 17     | "timestamp"           |                            |
| 64-bit integer             | 18     | "long"                |                            |
| Decimal128                 | 19     | "decimal"             | New in version 3.4.        |
| Min key                    | -1     | "minKey"              |                            |
| Max key                    | 127    | "maxKey"              |                            |
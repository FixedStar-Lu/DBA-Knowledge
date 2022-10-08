[TOC]

# Logs Insights

Logs Insights能够快速对CloudWatch中日志进行交互式查询，例如对日志进行条件过滤等处理。CloudWatch Logs Insights支持如下命令：

- display：指定查询结果需要显示的字段名称

- fields：从日志事件中检索用于显示的指定字段

- filter：基于一个或多个条件过滤查询的结果，filter命令可以使用各种操作符和表达式

- stats：根据日志字段的值计算汇总统计信息，还可以用by指定一个或多个标准进行分组。支持sum()、avg()、count()、min()和max()运算符

- sort：对检索到的日志事件进行排序

- limit：指定查询返回的日志事件数，默认查询只显示1000行，通过limit指定1000-10000的值可以增加显示的数据

- parse：从日志字段中解析数据并创建一个或多个临时字段，您可以在查询中进一步处理这些字段。 parse 接受 glob 表达式和正则表达式

  - 对于glob表达式，为parse命令提供一个常量字符串，其中每个变量文本块都用星号(*)替换，这些字段被提取到临时字段中，并按照位置顺序在as关键字之后给出别名

    ```
    parse @message "[*] * The error was: *" as level, config, exception
    ```

  - 对于正则表达式，需要用`/`括起来，要提取的匹配字符串的每个部分都包含在一个命名的捕获组中。例如`(?<name>.*)`，其中name是临时字段名称，.*是模式

    ```
    parse @message /\[(?<level>\S+)\]\s+(?<config>\{.*\})\s+The error was: (?<exception>\S+)/
    ```

下面是一个解析slowquery log的例子，可供参考：

原始日志

```
@ingestionTime	1628661485773
@log	502076313352:/aws/rds/instance/prod-nigeria/slowquery
@logStream	prod-nigeria
@message	# Time: 2021-08-11T05:58:03.009224Z
            # User@Host: paydbprod[paydbprod] @ [172.30.11.31] Id: 9119166
            # Query_time: 9.154389 Lock_time: 0.000052 Rows_sent: 1 Rows_examined: 31442833
            SET timestamp=1628661483;
			SELECT count(0) FROM t_coupon_user t, t_coupon c WHERE t.coupon_id = c.id AND now() >= c.end_time;
@timestamp	1628661483009
```

格式化查询

```
filter @logStream = 'prod-nigeria'
 | fields @timestamp, @message
 | parse @message /User@Host: (?<user>.[^\[\]\s]*)/
 | parse @message /Query_time: (?<Query_time>.[^\s]*)/
 | parse @message /Lock_time: (?<Lock_time>.[^\s]*)/
 | parse @message /Rows_sent: (?<Rows_sent>.[^\s]*)/
 | parse @message /Rows_examined: (?<Rows_examined>.[^\s]*)/ 
 | parse @message /SET.*;(?<sql>.*)/
 | filter Query_time>3
 | filter sql not like /(?i)ROLLBACK/
 | filter sql not like /(?i)COMMIT/
 | filter sql not like /(?i)SHOW /
 | display @timestamp,user,Query_time,Lock_time,Rows_sent,Rows_examined,sql
 | limit 10
```

查询结果

|@timestamp | user | Query_time | Lock_time | Rows_sent | Rows_examined | sql |
|---- | ---- | ---- | ---- | ---- | ---- | ---- |
|2021-08-11 05:58:03.009 | paydbprod | 9.1544 | 0.000114 | 100 | 31442833 | SELECT count(0) FROM t_coupon_user t, t_coupon c WHERE t.coupon_id = c.id AND now() >= c.end_time AND t.status = 'Claimed';|
|2021-08-11 05:55:44.983 | paydbprod | 9.0076 | 0.000118 | 1 | 31442754 | SELECT count(0) FROM t_coupon_user t, t_coupon c WHERE t.coupon_id = c.id AND now() >= c.end_time AND t.status = 'Claimed';|

>  Logs Insights还支持很多日期、文本、数值运算函数，具体可参考：[1. CloudWatch Logs Insights query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html) | [2. Sample queries - Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html)


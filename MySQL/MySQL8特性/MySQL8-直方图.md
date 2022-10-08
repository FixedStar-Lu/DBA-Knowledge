[TOC]

---
## 直方图

查询优化器负责将SQL查询转换为尽可能高效的执行计划，但随着数据环境不断变化，查询优化器可能无法找到最佳的执行计划，导致SQL效率低下。造成这种情况的原因是优化器对查询的数据了解的不够充足，例如：每个表有多少行数据，每列中有多少不同的值，每列的数据分布情况。

因此MySQL8.0.3推出了直方图(histogram)功能，直方图是列的数据分布的近似值，其向优化器提供更多的统计信息。比如字段NULL的个数，每个不同值的百分比，最大/最小值等。MySQL的直方图分为：等宽直方图和等高直方图，MySQL会自动分配使用哪种类型的直方图，无法干预
- 等宽直方图：每个bucket保存一个值以及这个值的累计频率
- 等高直方图：每个bucket保存不同值的个数，上下限以及累计频率

直方图同时也存在一定的限制条件：
- 不支持几何类型以及json类型的列
- 不支持加密表和临时表
- 无法为单列唯一索引的字段生成直方图

### 创建直方图

创建语法
```
ANALYZE TABLE tbl_name UPDATE HISTOGRAM ON col_name [, col_name] WITH N BUCKETS;
```
创建直方图时能够同时为多个列创建直方图，但必须指定bucket数量，范围在1-1024之间，默认100。对于bucket数量应该综合考虑其有多少不同值、数据的倾斜度、精度等，建议从较低的值开始，不符合再依次增加。

删除语法
```
ANALYZE TABLE tbl_name DROP HISTOGRAM ON col_name [, col_name];
```

**直方图信息**

MySQL通过字典表column_statistics来保存直方图的定义，每行记录对应一个字段的直方图，已JSON格式保存。
```
root@employees 13:49:  select json_pretty(histogram) from information_schema.column_statistics where table_name='employees' and column_name='first_name';;
{
  "buckets": [
    [
      "base64:type254:QWFtZXI=",
      "base64:type254:QWRlbA==",
      0.010176045588684237,
      13
    ],
  "data-type": "string",
  "null-values": 0.0,
  "collation-id": 255,
  "last-updated": "2020-09-09 05:47:32.548874",
  "sampling-rate": 0.163495700259278,
  "histogram-type": "equi-height",
  "number-of-buckets-specified": 100
}
```
MySQL为employees的first_name字段分配了等高直方图，默认为100个bucket。

当生成直方图时，MySQL会将所有数据都加载到内存中，并在内存中执行所有工作。如果在大表上生成直方图，可能会将几百M的数据读取到内存中的风险，因此我们可以通过参数`hitogram_generation_max_mem_size`来控制生成直方图最大允许的内存量，当指定内存满足不了所有数据集时就会采用采样的方式。
```
root@employees 14:12:  select histogram->>'$."sampling-rate"' from information_schema.column_statistics where table_name='employees' and column_name='first_name';;
+---------------------------------+
| histogram->>'$."sampling-rate"' |
+---------------------------------+
| 0.163495700259278               |
+---------------------------------+
```
从MySQL8.0.19开始，存储引擎自身提供了存储在表中数据的采样实现，存储引擎不支持时，MySQL使用默认采样需要全表扫描，这样对于大表来说成本太高，采样实现避免了全表扫描提高采样性能。

通过INNODB_METRICS计数器可以监视数据页的采样情况，这需要提前开启计数器
```
root@employees 14:26:  SELECT NAME, COUNT FROM INFORMATION_SCHEMA.INNODB_METRICS WHERE NAME LIKE 'sampled%'\G
*************************** 1. row ***************************
 NAME: sampled_pages_read
COUNT: 430
*************************** 2. row ***************************
 NAME: sampled_pages_skipped
COUNT: 456
2 rows in set (0.04 sec)
```
采样率的计算公式为：`sampled_page_read/(sampled_pages_read + sampled_pages_skipped)`

### 优化案例

复制一张表出来，源表不添加直方图，新表添加直方图
```
root@employees 14:32:  create table employees_like like employees;
Query OK, 0 rows affected (0.03 sec)

root@employees 14:33:  insert into employees_like select * from employees;
Query OK, 300024 rows affected (3.59 sec)
Records: 300024  Duplicates: 0  Warnings: 0

root@employees 14:33:  ANALYZE TABLE employees_like update HISTOGRAM on birth_date,first_name;
+--------------------------+-----------+----------+-------------------------------------------------------+
| Table                    | Op        | Msg_type | Msg_text                                              |
+--------------------------+-----------+----------+-------------------------------------------------------+
| employees.employees_like | histogram | status   | Histogram statistics created for column 'birth_date'. |
| employees.employees_like | histogram | status   | Histogram statistics created for column 'first_name'. |
+--------------------------+-----------+----------+-------------------------------------------------------+
```

分别在两张表上查看SQL的执行计划
```

root@employees 14:43:  explain format=json select count(*) from employees where (birth_date between '1953-05-01' and '1954-05-01') and first_name like 'A%';
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "30214.45"
    },
    "table": {
      "table_name": "employees",
      "access_type": "ALL",
      "rows_examined_per_scan": 299822,
      "rows_produced_per_join": 3700,
      "filtered": "1.23",
      "cost_info": {
        "read_cost": "29844.37",
        "eval_cost": "370.08",
        "prefix_cost": "30214.45",
        "data_read_per_join": "520K"
      },
      "used_columns": [
        "birth_date",
        "first_name"
      ],
      "attached_condition": "((`employees`.`employees`.`birth_date` between '1953-05-01' and '1954-05-01') and (`employees`.`employees`.`first_name` like 'A%'))"
    }
  }
}

root@employees 14:45:  explain format=json select count(*) from employees where (birth_date between '1953-05-01' and '1954-05-01') and first_name like 'A%';
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "18744.56"
    },
    "table": {
      "table_name": "employees",
      "access_type": "range",
      "possible_keys": [
        "idx_birth",
        "idx_first"
      ],
      "key": "idx_first",
      "used_key_parts": [
        "first_name"
      ],
      "key_length": "58",
      "rows_examined_per_scan": 41654,
      "rows_produced_per_join": 6221,
      "filtered": "14.94",
      "index_condition": "(`employees`.`employees`.`first_name` like 'A%')",
      "cost_info": {
        "read_cost": "18122.38",
        "eval_cost": "622.18",
        "prefix_cost": "18744.56",
        "data_read_per_join": "874K"
      },
      "used_columns": [
        "birth_date",
        "first_name"
      ],
      "attached_condition": "(`employees`.`employees`.`birth_date` between '1953-05-01' and '1954-05-01')"
    }
  }
}
```

可以看出Cost值从30214.45降到了18744.56，扫描行数从299822降到了41654，性能有所提升

**相关链接**

1、[analyze-table-histogram-statistics-analysis](https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html#analyze-table-histogram-statistics-analysis)

2、[Histogram statistics in MySQL](https://mysqlserverteam.com/histogram-statistics-in-mysql/)

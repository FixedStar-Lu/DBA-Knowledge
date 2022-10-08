#MySQL #JSON
# JSON数据类型
MySQL 5.7.8开始对JSON数据类型的支持，JSON不再以字符串形式存储，而是转换为内部二进制存储格式，能够自动验证JSON内容，无效JSON文档会产生错误。二进制结构能够直接通过键或数组索引查找子对象或嵌套值，而无需遍历整个列。
```
root@test 10:57:  create table tab_json(address json);
Query OK, 0 rows affected (0.04 sec)

root@test 11:02:  insert into tab_json values('{"country":"china","city":"shenzhen"}');
Query OK, 1 row affected (0.01 sec)

root@test 11:03:  insert into tab_json values('{"country":"china","city":"zhejiang","people":1000000}');
Query OK, 1 row affected (0.00 sec)

root@test 11:05:  select * from tab_json;
+-------------------------------------------------------------+
| address                                                     |
+-------------------------------------------------------------+
| {"city": "shenzhen", "country": "china"}                    |
| {"city": "zhejiang", "people": 1000000, "country": "china"} |
+-------------------------------------------------------------+
```

### JSON函数
JSON_TYPE函数需要传入一个JSON参数，并尝试将其解析为JSON值。如果有效，则返回JSON类型，否则返回错误：
```
root@test 11:21:  SELECT JSON_TYPE('["a", "b", 1]');
+----------------------------+
| JSON_TYPE('["a", "b", 1]') |
+----------------------------+
| ARRAY                      |
+----------------------------+

root@test 11:23:  SELECT JSON_TYPE('1');
+----------------+
| JSON_TYPE('1') |
+----------------+
| INTEGER        |
+----------------+

root@test 11:23:  SELECT JSON_TYPE('"ab"');
+-------------------+
| JSON_TYPE('"ab"') |
+-------------------+
| STRING            |
+-------------------+

root@test 11:23:  SELECT JSON_TYPE("ab");
ERROR 3141 (22032): Invalid JSON text in argument 1 to function json_type: "Invalid value." at position 0.
```

**JSON_ARRAY**

JSON_ARRARY函数接收值列表，并返回这些值的JSON数组
```
root@test 11:24:  SELECT JSON_ARRAY("shenzhen","shanghai","beijing");
+---------------------------------------------+
| JSON_ARRAY("shenzhen","shanghai","beijing") |
+---------------------------------------------+
| ["shenzhen", "shanghai", "beijing"]         |
+---------------------------------------------+
```

**JSON_OBJECT**

JSON_OBJECT函数接收键值对，并返回包含JSON对象
```
root@test 11:29:  SELECT JSON_OBJECT('NAME','LU','AGE',25,'ADDR','SHENZHEN');
+-----------------------------------------------------+
| JSON_OBJECT('NAME','LU','AGE',25,'ADDR','SHENZHEN') |
+-----------------------------------------------------+
| {"AGE": 25, "ADDR": "SHENZHEN", "NAME": "LU"}       |
+-----------------------------------------------------+
```

**JSON_MERGE**

JSON_MERGE函数接收两个或多个JSON文档并返回合并的结果
```
root@test 11:34:  SELECT JSON_MERGE('[1,2]','3','4');
+-----------------------------+
| JSON_MERGE('[1,2]','3','4') |
+-----------------------------+
| [1, 2, 3, 4]                |
+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```

**JSON_EXTRACT**

JSON_EXTRACT函数能从JSON文档中查询指定键的值
```
root@test 13:42:  select json_extract(address,'$.city') from test.tab_json;
+--------------------------------+
| json_extract(address,'$.city') |
+--------------------------------+
| "shenzhen"                     |
| "zhejiang"                     |
+--------------------------------+
```
也可以使用column->>path的方式获取
```
root@test 13:37:  select address->>"$.city" from test.tab_json;
+--------------------+
| address->>"$.city" |
+--------------------+
| shenzhen           |
| zhejiang           |
+--------------------+
```

**JSON_SET**

JSON_SET函数替换现有的值，并添加不存在的值
```
root@test 13:56:  SET @j = '["a", {"b": [true, false]}, [10, 20]]';
Query OK, 0 rows affected (0.00 sec)

root@test 13:57:  select @j;
+---------------------------------------+
| @j                                    |
+---------------------------------------+
| ["a", {"b": [true, false]}, [10, 20]] |
+---------------------------------------+
1 row in set (0.00 sec)

root@test 13:57:  SELECT JSON_SET(@j, '$[1].b[0]', 1, '$[2][2]', 2);
+--------------------------------------------+
| JSON_SET(@j, '$[1].b[0]', 1, '$[2][2]', 2) |
+--------------------------------------------+
| ["a", {"b": [1, false]}, [10, 20, 2]]      |
+--------------------------------------------+
```

**JSON_INSERT**

JSON_INSERT函数用于添加新值，但不替换现有值
```
root@test 13:57:  SELECT JSON_INSERT(@j, '$[1].b[0]', 1, '$[2][2]', 2);
+-----------------------------------------------+
| JSON_INSERT(@j, '$[1].b[0]', 1, '$[2][2]', 2) |
+-----------------------------------------------+
| ["a", {"b": [true, false]}, [10, 20, 2]]      |
+-----------------------------------------------+
```

**JSON_REPLACE**

JSON_REPLACE函数替换现有值，并忽略新值
```
root@test 14:13:  SELECT JSON_REPLACE(@j, '$[1].b[0]', 1, '$[2][2]', 2);
+------------------------------------------------+
| JSON_REPLACE(@j, '$[1].b[0]', 1, '$[2][2]', 2) |
+------------------------------------------------+
| ["a", {"b": [1, false]}, [10, 20]]             |
+------------------------------------------------+
```

**JSON_REMOVE**

JSON_REMOVE函数删除匹配的值
```
root@test 14:13:  SELECT JSON_REMOVE(@j, '$[2]', '$[1].b[1]');
+--------------------------------------+
| JSON_REMOVE(@j, '$[2]', '$[1].b[1]') |
+--------------------------------------+
| ["a", {"b": [true]}]                 |
+--------------------------------------+
```

### 数据大小写

通过转换JSON值产生的字符串的字符集为utf8mb4，排序规则为utf8mb4_bin，所以JSON值会区分大小写
```
root@test 11:42:  SET @j = JSON_OBJECT('key', 'value');
Query OK, 0 rows affected (0.00 sec)

root@test 11:43:  SELECT CHARSET(@j), COLLATION(@j);
+-------------+---------------+
| CHARSET(@j) | COLLATION(@j) |
+-------------+---------------+
| utf8mb4     | utf8mb4_bin   |
+-------------+---------------+

root@test 11:39:  SELECT JSON_ARRAY('x') = JSON_ARRAY('X');
+-----------------------------------+
| JSON_ARRAY('x') = JSON_ARRAY('X') |
+-----------------------------------+
|                                 0 |
+-----------------------------------+
```

针对true、false或null也同样区分大小写，它们必须始终以小写形式编写
```
root@test 13:33:  SELECT CAST('null' AS JSON);
+----------------------+
| CAST('null' AS JSON) |
+----------------------+
| null                 |
+----------------------+
1 row in set (0.00 sec)

root@test 13:33:  SELECT CAST('NULL' AS JSON);
ERROR 3141 (22032): Invalid JSON text in argument 1 to function cast_as_json: "Invalid value." at position 0

root@test 13:33:  SELECT ISNULL(null), ISNULL(Null), ISNULL(NULL);
+--------------+--------------+--------------+
| ISNULL(null) | ISNULL(Null) | ISNULL(NULL) |
+--------------+--------------+--------------+
|            1 |            1 |            1 |
+--------------+--------------+--------------+
```

### JSON索引

不能直接对JSON列直接进行索引，只能通过虚拟列来引用JSON中的键值，并创建索引。
```
root@test 14:16:  alter table tab_json add column city varchar(20) GENERATED ALWAYS AS (address->"$.city");
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@test 14:25:  show create table tab_json;
+----------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table    | Create Table                                                                                                                                                                                                                                       |
+----------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| tab_json | CREATE TABLE `tab_json` (
  `address` json DEFAULT NULL,
  `city` varchar(20) COLLATE utf8mb4_unicode_ci GENERATED ALWAYS AS (json_extract(`address`,_utf8mb4'$.city')) VIRTUAL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci |
+----------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

root@test 14:25:  create index idx_city on tab_json(city);
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@test 14:25:  explain select address->"$.city" as city from tab_json where city ='shenzhen';
+----+-------------+----------+------------+------+---------------+----------+---------+-------+------+----------+-------+
| id | select_type | table    | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra |
+----+-------------+----------+------------+------+---------------+----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tab_json | NULL       | ref  | idx_city      | idx_city | 83      | const |    1 |   100.00 | NULL  |
+----+-------------+----------+------------+------+---------------+----------+---------+-------+------+----------+-------+
```

更多关于JSON数据类型的内容参考：[JSON Data Type](https://dev.mysql.com/doc/refman/5.7/en/json.html)
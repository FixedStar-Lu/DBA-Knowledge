#MySQL #Character #Collection
# MySQL字符集和排序规则

查看MySQL实例支持的字符集和排序规则
```
mysql> select * from information_schema.character_sets where character_set_name like 'utf8%';
+--------------------+----------------------+---------------+--------+
| CHARACTER_SET_NAME | DEFAULT_COLLATE_NAME | DESCRIPTION   | MAXLEN |
+--------------------+----------------------+---------------+--------+
| utf8               | utf8_general_ci      | UTF-8 Unicode |      3 |
| utf8mb4            | utf8mb4_general_ci   | UTF-8 Unicode |      4 |
+--------------------+----------------------+---------------+--------+

mysql> select * from information_schema.collations where collation_name like 'utf8mb4%_bin';
+----------------+--------------------+----+------------+-------------+---------+
| COLLATION_NAME | CHARACTER_SET_NAME | ID | IS_DEFAULT | IS_COMPILED | SORTLEN |
+----------------+--------------------+----+------------+-------------+---------+
| utf8mb4_bin    | utf8mb4            | 46 |            | Yes         |       1 |
+----------------+--------------------+----+------------+-------------+---------+
```
information_schema.character_sets：
- CHARACTER_SET_NAME：字符集名称，CI表示大小写不敏感
- DEFAULT_COLLATE_NAME：字符集默认对应的排序规则
- MAXLEN：字符集的最长字节长度，utf8mb4最长为u4个字节，能够保存一些生僻字和emoji表情

information_schema.collations：
- COLLATION_NAME：排序规则名称
- CHARACTER_SET_NAME：排序规则所属的字符集
- IS_DEFAULT：是否是字符集的默认排序规则
- IS_COMPILED：MySQL是否编译

字符集和排序规则主要由下面几个参数控制
```
mysql> show variables like 'character%';
+--------------------------+--------------------------------------------+
| Variable_name            | Value                                      |
+--------------------------+--------------------------------------------+
| character_set_client     | utf8                                       |
| character_set_connection | utf8                                       |
| character_set_database   | utf8mb4                                    |
| character_set_filesystem | binary                                     |
| character_set_results    | utf8                                       |
| character_set_server     | utf8mb4                                    |
| character_set_system     | utf8                                       |
| character_sets_dir       | /data/sandbox/mysql/5.7.38/share/charsets/ |
+--------------------------+--------------------------------------------+

mysql> show variables like 'collation%';
+----------------------+--------------------+
| Variable_name        | Value              |
+----------------------+--------------------+
| collation_connection | utf8_general_ci    |
| collation_database   | utf8mb4_general_ci |
| collation_server     | utf8mb4_general_ci |
+----------------------+--------------------+
```
- character_set_server和collation_server是MySQL Server层的字符集和排序规则
- character_set_client：接收客户端数据的字符集
- character_set_database和collation_database是创建数据库时默认的字符集和排序规则
- character_set_system为系统元数据字符集配置，固定为utf8不可更改
- character_set_results为Server发送数据到客户端时的字符集
- character_set_connection和collation_connection：Server会将客户端发来的SQL从character_set_client转换为character_set_connection对应的字符集和排序规则
- character_set_filesystem：SQL语句涉及的文件名字符集，例如执行Load File加载文件数据时，文件数据的字符集格式


## 字符集乱码问题


## 字符集转换

修改数据库的字符集和排序规则

修改表的字符集和排序规则

修改列的字符集和排序规则

**character introducer**

SQL查询时可以指定字符串的字符集和排序规则，而不受系统参数的影响
```
mysql> SELECT _utf8mb4'测试❤' collate utf8mb4_bin;
```

**convert函数**

通过convert函数转换，但不支持设置排序规则
```

mysql> SELECT convert('测试❤' using UTF8MB4);
+------------------------------------+
| convert('测试❤' using UTF8MB4)     |
+------------------------------------+
| 测试❤                              |
+------------------------------------+

mysql> select charset(convert('测试❤' using UTF8MB4));
+---------------------------------------------+
| charset(convert('测试❤' using UTF8MB4))     |
+---------------------------------------------+
| utf8mb4                                     |
+---------------------------------------------+
```

**SET NAMES**

SET NAMES可以同时设置character_set_client、character_set_connection、character_set_results三个会话参数变量
```
mysql> set names utf8mb4;
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like 'character%';
+--------------------------+--------------------------------------------+
| Variable_name            | Value                                      |
+--------------------------+--------------------------------------------+
| character_set_client     | utf8mb4                                    |
| character_set_connection | utf8mb4                                    |
| character_set_database   | utf8mb4                                    |
| character_set_filesystem | binary                                     |
| character_set_results    | utf8mb4                                    |
| character_set_server     | utf8mb4                                    |
| character_set_system     | utf8                                       |
| character_sets_dir       | /data/sandbox/mysql/5.7.38/share/charsets/ |
+--------------------------+--------------------------------------------+
```
> `SET CHARACTER SET`语句也有相似的效果，但character_set_connection会继承character_set_database的字符集

**Collation Coercibility**
```
mysql> SELECT x FROM T WHERE x = 'Y';
```
这里是用x的排序规则，还是字符'Y'的排序规则，优先级该怎么定义呢？为了解决这些问题，MySQL设置了0-6的7种强制指标，数字越小，优先级越大：
- 0：显示设置了COLLATE子句
- 1：两个不同排序规则的字符串进行拼接
- 2：两个不同的列，存储过程参数或者本地变量
- 3：系统常量。如user()，version()
- 4：简单的字符串
- 5：数值或时间值
- 6：NULL或者包含NULL的表达式
```
mysql> SELECT COERCIBILITY(_utf8'A' COLLATE utf8_bin);
-> 0 
mysql> SELECT COERCIBILITY(VERSION()); 
-> 3 
mysql> SELECT COERCIBILITY('A'); 
-> 4 
mysql> SELECT COERCIBILITY(1000);
-> 5 
mysql> SELECT COERCIBILITY(NULL);
-> 6
```
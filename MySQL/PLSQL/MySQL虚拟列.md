#MySQL #Generated-Column
# MySQL虚拟列

Generated Columns是MySQL5.7推出的一个特性，它是基于其它的列的表达式生成的一个新的虚拟列。例如下面求直角三角形斜边的长度的例子
```sql
mysql> CREATE TABLE triangle ( 
sidea DOUBLE, 
sideb DOUBLE, 
sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb)) 
); 
mysql> INSERT INTO triangle (sidea, sideb) VALUES(1,1),(3,4),(6,8);

mysql> SELECT * FROM triangle;
+-------+-------+--------------------+
| sidea | sideb | sidec              |
+-------+-------+--------------------+
|     1 |     1 | 1.4142135623730951 |
|     3 |     4 |                  5 |
|     6 |     8 |                 10 |
+-------+-------+--------------------+
```
Generated Columns支持VIRTUAL和STORED两种模式：
- VIRTUAL：列值不保存，而是在读取前实时计算出来。这也是默认模式
- STORED：列值会持久化保存，会占用一定的存储空间，性能较差。

Generated Columns表达式遵循下列规则：
- 不能使用不确定函数，例如CONNECTION_ID()，NOW()等
- 不接受Stored functions和loadable functions
- 不接受使用存储过程或function参数
- 不接受使用变量
- 不接受子查询
- 不接受AUTO_INCREMENT属性的列
- 如果表达式传入了非法值将会出现错误

**应用场景**

1. 我们知道如果对索引字段进行函数操作，SQL将不会利用该索引，导致性能变差。这时就可以考虑使用虚拟列进行函数操作，并对虚拟列创建二级索引。
2. 可以将复杂条件定义为虚拟列，统一简化查询，并从表上的多个查询中引用，以确保所有这些都使用完全相同的条件。

**Reference**

[MySQL :: MySQL 5.7 Reference Manual :: 13.1.18.7 CREATE TABLE and Generated Columns](https://dev.mysql.com/doc/refman/5.7/en/create-table-generated-columns.html)
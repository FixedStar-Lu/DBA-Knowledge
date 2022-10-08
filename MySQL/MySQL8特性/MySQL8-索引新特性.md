[TOC]

---

# MySQL8 - 索引新特性

## 1. 函数索引

在MySQL8.0之前对条件字段做函数操作、或者做运算都将不会使用字段上的索引，例如下面的例子
```
root@employees 14:09:  show index from employees;
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299232 |     NULL | NULL   |      | BTREE      |         |               |
| employees |          1 | inx_date |            1 | birth_date  | A         |        4739 |     NULL | NULL   |      | BTREE      |         |               |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
2 rows in set (0.00 sec)

root@employees 14:10:  explain select * from employees where month(birth_date)=9;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299232 |   100.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
```
可以看到SQL执行计划的type为ALL，并没有利用索引，这种情况只能采用[Generated Column](https://note.youdao.com/s/SMuY6Njg)的方式。而在MySQL8.0中推出了函数索引的特性，其是通过虚拟列来实现的，接着就来通过函数索引实现相同的需求，看看有什么不同
```
root@employees 14:35:  alter table employees add index idx_birth_date((month(birth_date)));
Query OK, 0 rows affected (0.67 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@employees 14:36:  explain select * from employees where month(birth_date)=9;
+----+-------------+-----------+------------+------+----------------+----------------+---------+-------+-------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys  | key            | key_len | ref   | rows  | filtered | Extra |
+----+-------------+-----------+------------+------+----------------+----------------+---------+-------+-------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ref  | idx_birth_date | idx_birth_date | 5       | const | 47370 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+----------------+----------------+---------+-------+-------+----------+-------+
```
需要注意的是函数索引写法是固定时，在使用函数索引时必须依照定义时的语法进行使用，否则优化器无法识别

## 2. Index Skip Scan

在存在一个联合索引的情况下，如果查询条件中不包含联合索引的最左字段，则无法使用联合索引。例如存在Index(a,b)，现在执行select * from tab where b=1，这时需要针对b字段建立一个单独的索引。MySQL8.0中引入了Index Skip Scan就是用于优化这种场景。
```
root@employees 15:16:  select * from t1;
+------+--------+
| id   | score  |
+------+--------+
|    0 |   100  |
|    0 |   100  |
|    0 |   200  |
|    0 |   300  |
|    0 |   400  |
|    0 |   500  |
|    0 |   600  |
|    0 |   700  |
|    0 |   800  |
|    1 |   900  |
|    1 |   1000 |
|    1 |   1100 |
|    1 |   1200 |
|    1 |   1300 |
|    1 |   1400 |
|    1 |   1500 |
|    1 |   1600 |
|    2 |   1700 |
|    2 |   1800 |
|    2 |   1900 |
+------+--------+

root@employees 15:16:  select * from t1 where score>500;
```
Index Skip Scan会将查询转换为
```
seect * from tab where a=0 and b>500 
union 
select * from tab where a=1 and b>500
union 
select * from tab where a=2 and b>500
```
可以看出实际上它是将id字段做了distinct然后作为条件再union拼接起来，这种优化只适用于左边字段唯一性较差的情况，例如性别，状态之类的值，否则优化器则不会使用Index Skip Scan来进行优化

## 3. 倒序索引

MySQL8.0之前创建索引只支持ASC正向索引，对于一些desc排序的查询并不是很友好，执行计划通常会出现using filesort。
```
root@employees 15:40:  explain select salary from salaries group by salary order by salary desc;
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+-----------------------------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows  | filtered | Extra                                                     |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+-----------------------------------------------------------+
|  1 | SIMPLE      | salaries | NULL       | range | idx_salary    | idx_salary | 4       | NULL | 81274 |   100.00 | Using index for group-by; Using temporary; Using filesort |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+-----------------------------------------------------------+
1 row in set, 1 warning (0.02 sec)

root@employees 15:41:  explain select salary from salaries group by salary  order by salary asc;
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows  | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+--------------------------+
|  1 | SIMPLE      | salaries | NULL       | range | idx_salary    | idx_salary | 4       | NULL | 81274 |   100.00 | Using index for group-by |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+--------------------------+
```
可以看到倒序排序时，执行计划extra中相对正序多了Using temporary; Using filesort，现在看看8.0中的倒序索引
```
root@employees 15:43:  create index idx_salary on salaries(salary desc);
Query OK, 0 rows affected (7.39 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@employees 15:47:  explain select salary from salaries group by salary order by salary desc;
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows  | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+--------------------------+
|  1 | SIMPLE      | salaries | NULL       | range | idx_salary    | idx_salary | 4       | NULL | 72950 |   100.00 | Using index for group-by |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+-------+----------+--------------------------+
```

## 4. 不可见索引

MySQL8.0中引入了隐藏索引，即该索引对优化器不可见，优化器也不会选择该索引，即使使用force index也无法使用。当我们在做优化时需要评估索引的影响，就可以通过隐藏索引来进行。
```
root@employees 15:47:  create index idx_emp on salaries(emp_no) invisible;
Query OK, 0 rows affected (4.12 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@employees 15:58:  explain select * from salaries force index(idx_emp) where emp_no=10001;
ERROR 1176 (42000): Key 'idx_emp' doesn't exist in table 'salaries'
```
开启use_invisible_indexes优化器选项后，就可以使用隐藏索引
```
root@employees 15:58:  set @@optimizer_switch='use_invisible_indexes=on';
```

[TOC]

---

# MySQL8 - SQL特性

## CTE表达式



## 窗口函数

测试数据

```
root@test 03:53:  select * from t1;
+----+-------+------+
| id | name  | flag |
+----+-------+------+
|  1 | lu    |    1 |
|  2 | heng  |    1 |
|  3 | xing  |    2 |
|  4 | li    |    2 |
|  5 | wang  |    3 |
|  6 | zhang |    2 |
+----+-------+------+
```

### 1. ROW_NUMBER

ROW_NUMBER用于返回分区范围内的行号
```
root@test 03:07:  select id,name,row_number() over w as 'row_number',flag from t1 window w as (partition by flag order by name);
+----+------+------------+------+
| id | name | row_number | flag |
+----+------+------------+------+
|  2 | heng |          1 |    1 |
|  1 | lu   |          2 |    1 |
|  4 | li   |          1 |    2 |
|  3 | xing |          2 |    2 |
|  5 | wang |          1 |    3 |
+----+------+------------+------+

root@test 03:08:  select id,name,row_number() over(partition by flag order by name) as 'row_number',flag from t1;
+----+------+------------+------+
| id | name | row_number | flag |
+----+------+------------+------+
|  2 | heng |          1 |    1 |
|  1 | lu   |          2 |    1 |
|  4 | li   |          1 |    2 |
|  3 | xing |          2 |    2 |
|  5 | wang |          1 |    3 |
+----+------+------------+------+
```

### 2. RANK

RANK用于返回分区中的排名，相同条件下的值获得相同排名，因此结果集中会带有间隙。如果不指定order by则所有数据都获得相同的排名
```
root@test 03:12:  select flag,rank() over w as 'rank' from t1 WINDOW w AS (ORDER BY flag);
+------+------+
| flag | rank |
+------+------+
|    1 |    1 |
|    1 |    1 |
|    2 |    3 |
|    2 |    3 |
|    3 |    5 |
+------+------+
```

如果想要消除间隙，可以使用DENSE_RANK
```
root@test 03:12:  select flag,dense_rank() over w as 'dense_rank' from t1 WINDOW w AS (ORDER BY flag);
+------+------------+
| flag | dense_rank |
+------+------------+
|    1 |      1     |
|    1 |      1     |
|    2 |      2     |
|    2 |      2     |
|    3 |      3     |
+------+------------+
```

**PERCENT_RANK**

PERCENT_RANK用于返回分区值小于当前值的百分比(不包括最高值)，范围为0~1。*(rank - 1) / (rows - 1)*
```
root@test 03:19:  select flag,PERCENT_RANK() over w as 'percent_rank' from t1 WINDOW w AS (ORDER BY flag);
+------+--------------+
| flag | percent_rank |
+------+--------------+
|    1 |       0      |
|    1 |       0      |
|    2 |       0.5    |
|    2 |       0.5    |
|    3 |       1      |
+------+--------------+
```

### 3. NTILE

语法
```
NTILE(N) over_clause
```

将一个分区划分为N个桶(buckets)，为分区中的每一行分配桶号，并返回该分区中当前行的桶号。例如NTILE设置为4，则将行分为4个桶。

MySQL8.0.22中N不能为空，可以定义的数值范围为1~2的63次方，它支持下列任意格式：
- 无符号常量
- 一种位置标记
- 用户定义的变量
- 存储过程中的局部变量

```
root@test 03:43:  select id,ntile(2) over w as 'ntile' from t1 WINDOW w AS (ORDER BY id);
+----+-------+
| id | ntile |
+----+-------+
|  1 |     1 |
|  2 |     1 |
|  3 |     1 |
|  4 |     2 |
|  5 |     2 |
+----+-------+
```

### 4. NTH_VALUE

语法
```
NTH_VALUE(expr, N) [from_first_last] [null_treatment] over_clause
```

NTH_VALUE用于返回窗口函数中第N行中的expr值，如果没有则返回NULL
```
root@test 03:47:  select id,flag,nth_value(name,2) over w as 'nth_value' from t1 WINDOW w AS (partition by flag ORDER BY id);
+----+------+-----------+
| id | flag | nth_value |
+----+------+-----------+
|  1 |    1 | NULL      |
|  2 |    1 | heng      |
|  3 |    2 | NULL      |
|  4 |    2 | li        |
|  5 |    3 | NULL      |
+----+------+-----------+

root@test 03:48:  select id,flag,nth_value(name,3) over w as 'nth_value' from t1 WINDOW w AS (partition by flag ORDER BY id);
+----+------+-----------+
| id | flag | nth_value |
+----+------+-----------+
|  1 |    1 | NULL      |
|  2 |    1 | NULL      |
|  3 |    2 | NULL      |
|  4 |    2 | NULL      |
|  6 |    2 | zhang     |
|  5 |    3 | NULL      |
+----+------+-----------+
```

### 5. LEAD

语法
```
LAG(expr [, N[, default]]) [null_treatment] over_clause
```

在分区中，从当前行之前或之后的N行返回expr值，如果没有返回值为default，如果N和default都缺失则默认值分别为1和default
```
root@test 04:04:  select id,flag,lead(name,1,'default') over w as 'lead' from t1 WINDOW w AS ( ORDER BY id);
+----+------+---------+
| id | flag | lead    |
+----+------+---------+
|  1 |    1 | heng    |
|  2 |    1 | xing    |
|  3 |    2 | li      |
|  4 |    2 | wang    |
|  5 |    3 | zhang   |
|  6 |    2 | default |
+----+------+---------+
```

### 6. LAST_VALUE

语法
```
LAST_VALUE(expr) [null_treatment] over_clause
```

LAST_VALUE用于返回窗口最后一行的expr值
```
root@test 04:12:  select id,flag,LAST_value(name) over w as 'last_value' from t1 WINDOW w AS (ORDER BY flag);
+----+------+------------+
| id | flag | last_value |
+----+------+------------+
|  1 |    1 | heng       |
|  2 |    1 | heng       |
|  3 |    2 | zhang      |
|  4 |    2 | zhang      |
|  6 |    2 | zhang      |
|  5 |    3 | wang       |
+----+------+------------+
```

### 7. LAG

语法
```
LAG(expr [, N[, default]]) [null_treatment] over_clause
```
在分区中，从当前行之前或之后的N行返回expr值，如果没有返回值为default，如果N和default都缺失则默认值分别为1和default。与LEAD的差别在于方向不同

```
root@test 04:15:  select id,flag,lag(name,1,'default') over w as 'lag' from t1 WINDOW w AS ( ORDER BY id);
+----+------+---------+
| id | flag | lag     |
+----+------+---------+
|  1 |    1 | default |
|  2 |    1 | lu      |
|  3 |    2 | heng    |
|  4 |    2 | xing    |
|  5 |    3 | li      |
|  6 |    2 | wang    |
+----+------+---------+
```

### 8. CUME_DIST

返回一个值在一组值中的累积分布;即分区值小于或等于当前行值的百分比
```
root@test 04:19:  select id,flag,cume_dist() over w as 'cume_dist' from t1 WINDOW w AS ( partition by flag order BY id);
+----+------+--------------------+
| id | flag | cume_dist          |
+----+------+--------------------+
|  1 |    1 |                0.5 |
|  2 |    1 |                  1 |
|  3 |    2 | 0.3333333333333333 |
|  4 |    2 | 0.6666666666666666 |
|  6 |    2 |                  1 |
|  5 |    3 |                  1 |
+----+------+--------------------+
```

Tips：[Window Function Descriptions](https://dev.mysql.com/doc/refman/8.0/en/window-function-descriptions.html)
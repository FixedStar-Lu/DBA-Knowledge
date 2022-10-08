[TOC]

---

## 唯一索引范围锁修复
在MySQL8.0.18之前，针对唯一索引或主键索引进行范围条件加锁时，向右遍历过程中，会一直扫描并加next-key锁到第一个不满足条件的记录为止，然后退化为间隙锁，但RR隔离级别下并不会退化，也就是锁的范围扩大了，严格来说这算是一个bug，这一问题直到8.0.18总算被优化了。

5.7测试结果
```
mysql> select version();
+------------+
| version()  |
+------------+
| 5.7.17-log |
+------------+
1 row in set (0.01 sec)

mysql> select @@session.tx_isolation;
+------------------------+
| @@session.tx_isolation |
+------------------------+
| REPEATABLE-READ        |
+------------------------+

mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  0 |    0 |    0 |
|  5 |    5 |    5 |
| 10 |   10 |   10 |
| 15 |   15 |   15 |
| 20 |   20 |   20 |
| 25 |   25 |   25 |
+----+------+------+
6 rows in set (0.00 sec)

mysql> show create table t\G
*************************** 1. row ***************************
     Table: t
Create Table: CREATE TABLE `t` (
`id` int(11) NOT NULL,
`c` int(11) DEFAULT NULL,
`d` int(11) DEFAULT NULL,
PRIMARY KEY (`id`),
KEY `c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```

![5.7](https://static001.geekbang.org/resource/image/b1/6d/b105f8c4633e8d3a84e6422b1b1a316d.png)

按照正常理解，ID索引上只加(10,15]的next-key lock，并且ID是主键具有唯一性，应该到ID=15就应该退化，但实际情况会往前扫描到第一个不满足的条件为止，也就是ID=20的记录，而且由于是范围扫描，索引上会加上(15,20]的next-key lock，因此SESSION B更新ID=20的记录也会被锁住，SESSION C插入ID=16的记录同样会被锁住。
```
mysql> select * from INNODB_LOCKS;
+-------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| lock_id     | lock_trx_id | lock_mode | lock_type | lock_table | lock_index | lock_space | lock_page | lock_rec | lock_data |
+-------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| 3715:43:3:6 | 3715        | X,GAP     | RECORD    | `test`.`t` | PRIMARY    |         43 |         3 |        6 | 20        |
| 3711:43:3:6 | 3711        | X         | RECORD    | `test`.`t` | PRIMARY    |         43 |         3 |        6 | 20        |
| 3714:43:3:6 | 3714        | X         | RECORD    | `test`.`t` | PRIMARY    |         43 |         3 |        6 | 20        |
+-------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+

mysql> select * from INNODB_LOCK_waits;
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 3717              | 3717:43:3:6       | 3711            | 3711:43:3:6      |
| 3716              | 3716:43:3:6       | 3711            | 3711:43:3:6      |
+-------------------+-------------------+-----------------+------------------+
```

在MySQL8.0.18中，后两条SQL不再会被锁住，MOS上关于改问题修复的描述如下：
```
commit d1b0afd75ee669f54b70794eb6dab6c121f1f179
Author: Jakub Łopuszański <jakub.lopuszanski@oracle.com>
Date:   Wed Jul 17 16:34:01 2019 +0200

  Bug #29508068       UNNECESSARY NEXT-KEY LOCK TAKEN

  When doing a SELECT...FOR [SHARE|UPDATE] with a WHERE condition specifying a range,
  we were locking "one row too much".
  This patch fixes locking behaviour in several (hopefuly) most common cases, so that
  we only lock rows and gaps which intersect the searched range.
	- Added MTR to demonstrate current locking policy for end of range
	- Got rid of goto
	- Extracted logic of determining relation between range and row to separate function
	- Extracted reoccuring patterns of modifications of search_tuple so it is easier to add same for stop_tuple
	- Added prebuilt->m_stop_tuple and made sure it is in sync with prebuilt->m_mysql_handler->end_range for during read_range_first() and read_range_next()
	- Added row_can_be_in_range field
	- Do not lock the row (just the gap) if the row is same length and after the stop_tuple
	- Do not lock the row (just the gap) if the row is same length and equal to stop_tuple and strict inequality was used for end of range
	- Do not lock the row (just the gap) if the row is longer than stop_tuple and its prefix is after the stop_tuple
	- Do not lock the row (just the gap) if the row is longer than stop_tuple and its prefixis equal to stop_tuple and strict inequality was used for end of range
	- Do not lock the row nor gap if we already saw a row same length and equal to stop_tuple in previous iteration
	  
	      Reviewed-by: Pawel Olchawa <pawel.olchawa@oracle.com>
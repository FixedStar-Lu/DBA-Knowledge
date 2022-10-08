[TOC]

---

### pager

pager类似于Linux的管道符，可以把输出提供给另一个命令作为输入。pager可以衔接各种Linux命令，下面是几种常见的用法

查找正在运行的查询
```
root@(none) 14:44:  pager grep Query
PAGER set to 'grep Query'
root@(none) 14:44:  show full processlist;
```

查看线程数量，按会话状态分组
```
root@(none) 14:44:  pager awk -F '|' '{print $6}' | sort | uniq -c | sort -r
PAGER set to 'awk -F '|' '{print $6}' | sort | uniq -c | sort -r'
root@(none) 14:46:  show full processlist;
    548  Sleep            
      3  Query            
      3 
      2  Binlog Dump GTID 
      1  Daemon           
      1  Command          
554 rows in set (0.00 sec)
```

翻页查看

在执行show engine innodb status\G的结果集太长，我们可以通过less或者more来实现翻页查看
```
root@(none) 14:51:  pager more
PAGER set to 'more'
root@(none) 14:52:  show engine innodb status\G
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2020-01-15 14:52:20 0x7f16a8b8e700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 31 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 2691 srv_active, 0 srv_shutdown, 3537763 srv_idle
srv_master_thread log flush and writes: 3540397
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 15800
OS WAIT ARRAY INFO: signal count 16115
RW-shared spins 0, rounds 22979, OS waits 8099
RW-excl spins 0, rounds 15133, OS waits 755
RW-sx spins 4163, rounds 100663, OS waits 1746
Spin rounds per wait: 22979.00 RW-shared, 15133.00 RW-excl, 24.18 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 279261
Purge done for trx's n:o < 279261 undo n:o < 0 state: running but idle
History list length 45
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421213158348624, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421213158349536, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
--More--
```

关闭pager
```
root@(none) 14:54:  nopager
PAGER set to stdout
```

### tee

tee和linux命令tee一样，在输出到stdout同时可以指定输出到另一个文件。

```
root@(none) 14:54:  tee /tmp/tabsql
Logging to file '/tmp/tabsql'
root@(none) 14:57:  select * from test.tab1;
+----+------------+------+
| id | name       | comm |
+----+------------+------+
|  1 | lu         | NULL |
|  2 | heng       | NULL |
|  3 | xing       | NULL |
|  4 | luhengxing | NULL |
+----+------------+------+
4 rows in set (0.00 sec)

[root@t-luhx03-v-szzb ~]# cat /tmp/tabsql 
root@(none) 14:57:  select * from test.tab1;
+----+------------+------+
| id | name       | comm |
+----+------------+------+
|  1 | lu         | NULL |
|  2 | heng       | NULL |
|  3 | xing       | NULL |
|  4 | luhengxing | NULL |
+----+------------+------+
4 rows in set (0.00 sec)
```
*Tips：也可以写在my.cnf的[mysql]模块，记录所有会话的操作，但可能会造成磁盘空间爆满*

关闭tee
```
root@(none) 14:57:  notee
Outfile disabled.
```

### edit

edit相当于vi命令，可以用来编辑sql命令，默认编辑上一条执行的SQL，编辑完退出后输入分号回车即可执行编辑后的SQL。

```
root@(none) 15:01:  select * from test.tab1;
+----+------------+------+
| id | name       | comm |
+----+------------+------+
|  1 | lu         | NULL |
|  2 | heng       | NULL |
|  3 | xing       | NULL |
|  4 | luhengxing | NULL |
+----+------------+------+
4 rows in set (0.00 sec)

root@(none) 15:06:  edit
    -> ;
+----+------+------+
| id | name | comm |
+----+------+------+
|  1 | lu   | NULL |
+----+------+------+
1 row in set (0.00 sec)
```

### system

system可以在不退出客户端的情况下执行linux命令并返回信息，类似于Oracle sqlplus里的host命令
```
root@(none) 15:12:  system hostname
t-luhx03-v-szzb
root@(none) 15:13:  
```

### status

status可以查看mysql服务器状态信息，或者直接执行\s
```
root@(none) 14:57:  select * from test.tab1;
+----+------------+------+
| id | name       | comm |
+----+------------+------+
|  1 | lu         | NULL |
|  2 | heng       | NULL |
|  3 | xing       | NULL |
|  4 | luhengxing | NULL |
+----+------------+------+
4 rows in set (0.00 sec)

root@(none) 14:57:  notee
root@(none) 15:12:  system hostname
t-luhx03-v-szzb
root@(none) 15:13:  status
--------------
mysql  Ver 14.14 Distrib 5.7.17, for linux-glibc2.5 (x86_64) using  EditLine wrapper

Connection id:		1114334
Current database:	
Current user:		root@localhost
SSL:			Not in use
Current pager:		stdout
Using outfile:		''
Using delimiter:	;
Server version:		5.7.17-log MySQL Community Server (GPL)
Protocol version:	10
Connection:		Localhost via UNIX socket
Server characterset:	utf8mb4
Db     characterset:	utf8mb4
Client characterset:	utf8mb4
Conn.  characterset:	utf8mb4
UNIX socket:		/service/mysql/data/mysqld.sock
Uptime:			41 days 31 min 7 sec

Threads: 2  Questions: 2609254  Slow queries: 48420  Opens: 380183  Flush tables: 1  Open tables: 1932  Queries per second avg: 0.736
--------------
```

### prompt

修改MySQL登陆提示符
```
mysql> prompt master> ;
PROMPT set to 'master> '
master> 
```

通常我们可以写在my.cnf配置文件的[mysql]模块里面，例如下列格式
```
prompt = "\u@\d \R:\m:\s "
```

\u表示当前用户，\d表示当前数据库，\R:\m:\s为时间格式，最终效果如下
```
root@(mysql) 15:16: 
```

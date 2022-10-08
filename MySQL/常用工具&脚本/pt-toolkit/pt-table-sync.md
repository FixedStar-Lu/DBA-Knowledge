[TOC]

---

## pt-table-sync

### 风险因素

在发现主从数据不一致后，我们可以通过pt-table-sync工具来完成数据修复。由于pt-table-sync会修改数据，因为建议在正式执行前注意以下事项：

- 仔细阅读文档手册 Document
- 在测试环境进行验证
- 提前备份生产数据
- 先利用–dry-run或–print了解同步内容
- 表上需要有主键或唯一键，如果没有则只能直接修改从节点的数据并指定–no-check-slave和–nobin-log

### 常用参数

选项 | 描述
-- | --
--replicate | 如果之前执行pt-table-checksum时结果保存在表中则可以直接使用该表
--databases | 指定数据库
--tables | 指定表
--sync-to-master | 指定DSN
--print | 打印命令
--execute | 执行命令
h | 主机名
u | 用户
p | 密码
P | 端口

对于–replicate和–sync-to-master之间的DSN的关系需要具有以下了解
```
if DSN has a t part, sync only that table:
   if 1 DSN:
      if --sync-to-master:
         The DSN is a slave.  Connect to its master and sync.
   if more than 1 DSN:
      The first DSN is the source.  Sync each DSN in turn.
else if --replicate:
   if --sync-to-master:
      The DSN is a slave.  Connect to its master, find records
      of differences, and fix.
   else:
      The DSN is the master.  Find slaves and connect to each,
      find records of differences, and fix.
else:
   if only 1 DSN and --sync-to-master:
      The DSN is a slave.  Connect to its master, find tables and
      filter with --databases etc, and sync each table to the master.
   else:
      find tables, filtering with --databases etc, and sync each
      DSN to the first.
```

### 示例

构建主从测试环境
```
[root@t-luhx01-v-szzb media]# dbdeployer unpack /media/mysql/mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz 
Unpacking tarball /media/mysql/mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz to /media/mysql/5.7.17
.........100.........200.........300.......379
Renaming directory /media/mysql/mysql-5.7.17-linux-glibc2.5-x86_64 to /media/mysql/5.7.17
[root@t-luhx01-v-szzb media]# dbdeployer deploy replication 5.7.17 --nodes 2 --sandbox-directory test-checksum
Creating directory /root/sandboxes
Installing and starting master
. sandbox server started
Installing and starting slave1
. sandbox server started
$HOME/sandboxes/test-checksum/initialize_slaves
initializing slave 1
Replication directory installed in $HOME/sandboxes/test-checksum
run 'dbdeployer usage multiple' for basic instructions'

[root@t-luhx01-v-szzb ~]# mysql -umsandbox -p -P18418 -h127.0.0.1
msandbox@test 11:00:  show slave hosts;
+-----------+--------+-------+-----------+--------------------------------------+
| Server_id | Host   | Port  | Master_id | Slave_UUID                           |
+-----------+--------+-------+-----------+--------------------------------------+
|       200 | node-2 | 18419 |       100 | 00018419-2222-2222-2222-222222222222 |
+-----------+--------+-------+-----------+--------------------------------------+
```

构建测试数据
```
msandbox@test 10:59:  CREATE TABLE `test`.`t1` (
    -> `id` int(11) NOT NULL AUTO_INCREMENT,
    -> `tcol01` tinyint(4) DEFAULT NULL,
    -> `tcol02` smallint(6) DEFAULT NULL,
    -> `tcol03` mediumint(9) DEFAULT NULL,
    -> `tcol04` int(11) DEFAULT NULL,
    -> `tcol05` bigint(20) DEFAULT NULL,
    -> `tcol06` float DEFAULT NULL,
    -> `tcol07` double DEFAULT NULL,
    -> `tcol08` decimal(10,2) DEFAULT NULL,
    -> `tcol09` date DEFAULT NULL,
    -> `tcol10` datetime DEFAULT NULL,
    -> `tcol11` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    -> `tcol12` time DEFAULT NULL,
    -> `tcol13` year(4) DEFAULT NULL,
    -> `tcol14` varchar(100) DEFAULT NULL,
    -> `tcol15` char(2) DEFAULT NULL,
    -> `tcol16` blob,
    -> `tcol17` text,
    -> `tcol18` mediumtext,
    -> `tcol19` mediumblob,
    -> `tcol20` longblob,
    -> `tcol21` longtext,
    -> `tcol22` mediumtext,
    -> `tcol23` varchar(3) DEFAULT NULL,
    -> `tcol24` varbinary(10) DEFAULT NULL,
    -> `tcol25` enum('a','b','c') DEFAULT NULL,
    -> `tcol26` set('red','green','blue') DEFAULT NULL,
    -> `tcol27` float(5,3) DEFAULT NULL,
    -> `tcol28` double(4,2) DEFAULT NULL,
    -> PRIMARY KEY (`id`)
    -> ) ENGINE=InnoDB;

[root@t-luhx01-v-szzb ~]# mysql_random_data_load -h127.0.0.1 -P18418 -umsandbox -pmsandbox --max-threads=4 test t1 100000
INFO[2021-01-21T11:03:40+08:00] Starting                                     
 3m8s [====================================================================] 100%
INFO[2021-01-21T11:06:51+08:00] 100000 rows inserted
```

删除从库数据
```
msandbox@test 11:08:  delete from t1 where id between 100 and 200;
Query OK, 101 rows affected (0.01 sec)
```

执行pt-table-sync修复
```
[root@t-luhx01-v-szzb ~]# pt-table-sync --sync-to-master h=127.0.0.1,u=msandbox,
p=msandbox,P=18419 --databases=test --tables=t1 --execute
```
> 由于需要基于语句的复制，在执行时会设置binlog_format为statement，因此用户需要有super权限

检查数据一致性
```
[root@t-luhx01-v-szzb ~]# pt-table-checksum --nocheck-replication-filters --no-check-binlog-format --databases=test --tables=t1 h=127.0.0.1,u=msandbox,p=msandbox,P=18418
Checking if all tables can be checksummed ...
Starting checksum ...
            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
01-21T15:10:31      0      0   100000          0       6       0   1.465 test.t1
```

## 单表备份恢复

在复制通道遇到数据异常错误导致中断后，我们可以通过单表备份恢复进行修复，大致步骤如下:

1. 在主库dump数据表，并在从库恢复
```
master> mysqldump -uroot -p --single-transaction --master-data=2 db1 tab1 > table_`date +%Y%m%d`.sql
slave> source table_20210705.sql
```

2. 由于备份表的数据领先于其它表，因此需要设置过滤该表
```
slave> change replication filter replicate_wild_ignore_table=tab1;
```

3. 启动复制通道，并指定回放到备份表的GTID时停止，保持所有表的处于一致的状态
```
slave> start slave until sql_after_gtids='xxx:10001';
```

4. 删除复制过滤条件，重启复制进程
```
slave> change replication filter replicate_wild_ignore_table=();
slave> start slave;
```

> 如果在遇到复制错误时，选择跳过错误恢复复制导致的数据不一致，因为同步一直在进行，因此我们需要将源表只读锁定再进行表备份，然后停止复制进程进行表恢复，最后启动复制解锁表。如果表数据量比较大，可以采用可传输表空间的方式进行恢复
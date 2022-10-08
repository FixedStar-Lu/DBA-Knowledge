[TOC]

---

## pt-table-checksum

pt-table-checksum是用于MySQL主从复制数据一致性校验的工具，在主库上运行，通过将数据分成多个chunk数据块，以数据块为单位使用CRC32计算checksum再进行比较。

会话开始时，会降低innodb_lock_wait_timeout的锁超时时间，只要innodb有锁产生则立即放弃操作，并且在执行checksum前会先通过explain评估语句的执行计划成本，因此对线上业务影响很小。

### 常用参数

选项 | 描述
-- |--
–nocheck-replication-filters | 不检查复制过滤器
–no-check-binlog-format | 非STATEMENT格式时需要指定
–replicate-check-only | 只显示不同步的信息
–replicate | 把checksum信息写入到指定表中
–databases | 指定执行检查的数据库
–tables | 指定执行检查的表
h | master节点的地址
u | 用户
p | 密码
P | 端口

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

执行pt-table-checksum比对数据
```
[root@t-luhx01-v-szzb ~]# pt-table-checksum --nocheck-replication-filters --no-check-binlog-format --databases=test --tables=t1 h=127.0.0.1,u=msandbox,p=msandbox,P=18418
Checking if all tables can be checksummed ...
Starting checksum ...
            TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
01-21T12:08:02      0      1   100000          0       6       0   1.533 test.t1
```

==注意事项：表需要存在主键或唯一键==
#MySQL #DDL #gh-ost #pt-online-change
# MySQL无锁DDL变更

## ONLINE DDL
ONLINE DDL是MySQL 5.6开始引入的，能够在DDL执行期间不影响正常的DML操作，提高数据并发能力。Online DDL可分为三种情况：
- copy(ALGORITHM=copy)：5.6之前DDL的执行方式，由Server层创建临时表，进行数据拷贝。在DDL执行期间，DML无法执行。innodb中不支持inplace方式执行的都会自动使用copy，而MyISAM表只能使用copy方式
- inplace(ALGORITHM=inplace)：DDL所有步骤都在innodb引擎完成，在DDL执行期间不影响DML操作，所有操作都是online的。inplace带有日志记录和重放功能，当在需要rebuild重建表时，会申请row log空间记录DDL期间所有DML操作，再重放至临时表
- inplace(offline)：DDL过程是ONLINE的一定采用inplace，但inplace的DDL并不一定ONLINE，截止到8.0中创建全文索引(fulltext index)或空间索引(spatial index)都是采用inplace但会阻塞DML的情况
  
除了ONLINE DDL内部实现，还通过LOCK选项控制锁，不同的DDL有不同的表现形式，默认mysql尽可能不去锁表：
- LOCK=NONE：DDL过程中允许读写
- LOCK=SHARED：DDL过程会阻塞写请求，允许读请求
- LOCK=DEFAULT：mysql自动判断lock，尽可能不去锁表
- LOCK=EXCLUSIVE：DDL过程中不允许读写请求

```
mysql> alter table test ALGORITHM=inplace,LOCK=NONE,ADD INDEX IDX_custid(cust_id);
```

**不同DDL操作的表现**

操作 | 是否inplace | 是否重建表 | 是否允许DML |是否只修改元数据 | 描述
-- | -- | -- | -- | -- | --
添加索引 | YES | NO | YES | NO | 对全文索引不支持inplace
删除索引 | YES | NO | YES | YES | 仅修改表的元数据
OPTIMIZE TABLE | YES | YES | YES | NO | 从 5.6.17开始使用ALGORITHM=INPLACE，如果表上有全文索引只支持COPY
SET DEFAULT | YES | NO | YES | YES | 仅修改元数据
SET AUTO-INCREMENT | YES | NO | YES | NO | 修改的内存数据
添加外键 | YES | NO | YES | YES | 关闭foreign_key_checks参数时，可以使用inplace，否则采用copy的方式
删除外键 | YES | NO | YES | YES | foreign_key_checks参数不影响
修改列名 | YES | NO | YES | YES | 为了允许DML并发, 请保持相同数据类型，仅改变列名
添加列 | YES | YES | YES | NO | 添加auto-increment自增列时，不允许并发DML，需要重建表使得代价很高
删除列 | YES | YES | YES | NO | 需要重建表使得代价很高
修改列排序 | YES | YES | YES | NO | 需要重建表使得代价很高
修改ROW_FORMAT和KEY_BLOCK_SIZE | YES | YES | YES | NO | 需要重建表使得代价很高
设置列为NULL | YES | YES | YES | NO | 需要重建表使得代价很高
设置列非NULL | YES | YES | YES | | 该操作需要sql_mode设置STRICT_ALL_TABLES或STRICT_TRANS_TABLES，如果列中已经存在空值操作会失败	
修改数据类型 | NO | YES | NO | NO | 只支持ALGORITHM=COPY
添加主键 | YES | YES | YES | NO | 支持ALGORITHM=INPLACE，如果列需要转化为NOT NULL，则不允许采用INPLACE
删除并添加主键 | YES | YES | YES | NO | 需要重建表使得代价很高
删除主键 | NO | YES | NO | YES | 当删除主键无需在同一ALTER TABLE语句中新建时，仅支持COPY
修改字符集 | NO | YES | NO | NO | 如果新的字符集编码不同，需要重建表
ALTER TABLE ENGINE=INNODB重建 | YES | YES | YES | NO | 支持ALGORITHM=INPLACE，带全文索引则不支持

**copy online重建表的流程**
1. 根据原表定义一个新的临时表
2. 对原表加写锁，无法执行DML
3. 在临时表上执行DDL
4. 将原表的数据拷贝到临时表中
5. 释放原表的写锁
6. 删除原表并重命名临时表为原表

**inplace online重建表的流程**

1. 获取MDL写锁
2. 创建临时文件
3. 申请row log空间
4. MDL锁降级为读锁，可执行DML
5. 扫描原表所有数据到临时文件中
6. 将对原表的所有DML操作记录到row log
7. MDL升级为写锁，DML不可执行
8. 重做row log日志
9. 重命名临时文件，删除原表文件
10. 提交事务，释放锁

**查看ONLINE DDL进度**

开启instruments
```
mysql> UPDATE setup_instruments SET ENABLED = 'YES' WHERE NAME LIKE 'stage/innodb/alter%';
Query OK, 7 rows affected (0.00 sec)
Rows matched: 7  Changed: 7  Warnings: 0

mysql> UPDATE setup_consumers SET ENABLED = 'YES' WHERE NAME LIKE '%stages%';
Query OK, 3 rows affected (0.00 sec)
Rows matched: 3  Changed: 3  Warnings: 0
```

查看进度
```
mysql> SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED FROM events_stages_current;
+------------------------------------------------------+----------------+----------------+
| EVENT_NAME                                           | WORK_COMPLETED | WORK_ESTIMATED |
+------------------------------------------------------+----------------+----------------+
| stage/innodb/alter table (read PK and internal sort) |            280 |           1245 |
+------------------------------------------------------+----------------+----------------+
```

## GH-OST

gh-ost是一个用于实现online ddl的开源工具，通过模拟从库，从binlog中获取增量数据，再异步应用到ghost表中。并提供暂停，动态控制，审计和许多操作特权，其优点在于无触发器，对主库影响较小

![gh-ost](https://github.com/github/gh-ost/raw/master/doc/images/gh-ost-general-flow.png)

### 工作模式

**1. 连接到从库，在主库上迁移**

这是gh-ost的默认的模式，gh-ost通过从副本查找到主副本，并连接到主副本。大致步骤如下：

- 在主库创建_gho表(与原表结构一致)，_ghc表(变更日志)，并修改gho表结构
- 读取副本上的二进制日志事件，将更改应用到主库gho表
- 在主库读取原表数据插入gho表
- 主库完成切换表

> 如果主库二进制日志格式是statement，可以使用这种模式。但从库必须启用二进制日志(log_bin,log_slave_update)，还要将格式设置为ROW

**2. 主库上执行**

如果是单实例或者不想连接到从副本执行，也可以在主副本上执行。gh-ost将直接在主库上执行所有操作。
- 主库binlog_format必须设置为ROW
- gh-ost必须使用–allow-on-master选项开启该模式

**3. 在从库上迁移/测试**

将在从库上执行迁移。gh-ost将短暂连接到主库，此后对从库执行所有操作，而无需修改主库内容。在整个操作过程中，gh-ost将进行限制以使从库是最新的
- --migrate-on-replica表示gh-ost必须直接在从库上迁移表。即使复制正在进行，它也将进行转换阶段
- --test-on-replica表示迁移仅用于测试目的。在进行转换之前，复制已停止。交换表，然后交换回去。原表将返回其原始位置

**DDL执行流程**

1. 检查有没有外键和触发器
2. 检查表的主键信息
3. 检查是否主库或从库，是否开启log_slave_updates以及binlog
4. 检查gho和del结尾的临时表是否存在
5. 创建ghc结尾的表，存数据迁移信息，以及binlog信息等
6. 初始化stream的连接，添加binlog的监听
7. 创建gho结尾的临时表，执行DDL在gho结尾的临时表上
8. 开启事务，按照主键ID把源表数据写入到gho结尾的表，再提交，以及binlog apply
9. lock源表，重命名表
10. 清理ghc表

### 一致性保证

**数据迁移一致性保证**

在gh-ost执行过程中对原表和中间表的操作包括：(1)对原表进行数据拷贝(2)期间业务执行的DML(3)中间表应用binlog。由于binlog是基于DML操作产生的，因此3一定排在2后面，因此执行顺序存在以下情况

<table>
  <tr>
    <td rowspan='3'>1->2->3</td>
    <td rowspan='3'>insert ingore into t2 select * from t1 where id>0 and id >10;</td>
    <td>insert into t2(id,name) values(11,'lu');</td>
    <td>replace into t2(id,name) values(11,'lu');</td>
  </tr>
  <tr>
    <td>update t2 set name='lu' where id=10;</td>
    <td>update t2 set name='lu' where id=10;</td>
  </tr>
  <tr>
    <td>delete from t2 where id=10;</td>
    <td>delete from t2 where id=10;</td>
  </tr>
  <tr>
    <td rowspan='3'>2->3->1</td>
    <td>insert into t2(id,name) values(11,'lu');</td>
    <td>replace into t2(id,name) values(11,'lu');</td>
    <td>因为id=11的binlog比数据拷贝来的早，中间表已经有id=11的数据了，插入忽略</td>
  </tr>
  <tr>
    <td>update t2 set name='lu' where id=10;</td>
    <td>id=10的记录还未拷贝，update空记录</td>
    <td>将ID=10的数据拷贝过来</td>
  </tr>
  <tr>
    <td>delete from t2 where id=10;</td>
    <td>id=10的记录还未拷贝，delete空记录</td>
    <td>查询不到id=10的记录，不拷贝</td>
  </tr>
  <tr>
    <td rowspan='3'>2->1->3</td>
    <td>insert into t2(id,name) values(11,'lu');</td>
    <td>insert ignore into t2(id,name) values(1,'a'),(2,'b'),(11,'lu');</td>
    <td>因为id=11的记录已经拷贝了，binlog会采用replace覆盖</td>
  </tr>
  <tr>
    <td>update t2 set values='lu' where id=1;</td>
    <td>将id=1的最新数据拷贝到中间表</td>
    <td>再次执行update操作，数据不变</td
  </tr>
  <tr>
    <td>delete from t2 where id=1;</td>
    <td>数据已删除，复制空行</td>
    <td>数据不存在，binlog执行无效果</td>
  </tr>
</table>

从上表来看，无论按哪种顺序执行，最后结果都是一致的。


**cut-over**

cut-over是用于完成原表和中间表的原子性切换，其原理在于lock table阻塞之后rename优先级总是高于DML，无论是否DML先执行。下面分析以下其执行步骤：

1. 开启一个会话(session-1)创建_del表
2. 执行lock table锁住原表和_del表的写入，DML被阻塞
3. 开启一个新的会话(session-2)设置锁等待时间并执行rename
4. session-2被session-1锁定
5. session-1检查session-2在执行rename并请求MDL锁
6. session-1删除_del表，此时原表的DML依然被阻塞
7. session-1执行unlock table释放锁，rename优先执行切换操作，DML请求可以在新表上执行

看到这不禁想问：要是执行过程中失败了会发生什么呢？会不会出现数据错乱？答案是什么都不会发生，下面就分析一下可能的情况：
- 创建_del表失败：此时程序直接退出
- 加锁失败：程序直接退出，DML正常执行
- session-1在session-2执行rename前异常：session-1持有的锁释放，DML正常执行，rename也会因为表存在而失败
- session-1在sesison-2执行rename被阻塞时异常：释放锁，rename因表存在而失败
- session-2异常：查询不到rename操作，释放锁
- 两个会话都异常：释放锁，rename取消

### 日常操作

**常用参数**

参数 | 描述
-- | --
--allow-on-master | 如果需要直接在主库执行需要设置该参数
--max-load string | 设置多个状态值，以逗号分隔，例如"Threads_running=100,Threads_connected"，当超过设置的值迁移则暂停等待
--critical-load string | 设置多个状态值，以逗号分隔，超过设置的值则迁移直接停止并退出
--chunk-size int | 每次从原表迭代迁移的数据行数，范围为100-100000，默认1000
--initially-drop-ghost-table |  在本次操作前删除之前遗留ghost临时表
--initially-drop-old-table | 在本次操作前删除之前遗留的old表
--initially-drop-socket-file | 删除存在的socket文件
--ok-to-drop-table | DDL完成后自动删除old表
--panic-flag-file string | 当指定该参数后，如果创建该文件，gh-ost立刻中断退出，不会清理产生的临时表和文件
--exact-rowcount | 精确的统计表数据行数而不是预估，即使不准确只是影响进度的计算，实际copy行数是由最大值和最小值确定，与其无关
--serve-socket-file string | socket文件
--assume-rbr | 显示告诉gh-ost日志格式是row格式，如果没有该参数，gh-ost每次都会设置row格式并重启复制
--assume-master-host | 显示告诉gh-ost master地址，如果不提供，gh-ost会根据从库查到master

**操作示例**

单实例DDL则相当于主库DDL，需开启–allow-on-master参数和ROW模式
```
$ gh-ost \
--max-load=Threads_running=25 \
--critical-load=Threads_running=1000 \
--chunk-size=1000 \
--max-lag-millis=1500 \
--user="dba" \
--password="Abcd123#" \
--host=10.0.139.163 \
--port=33006 \
--allow-on-master \
--database="test" \
--table="tab1" \
--verbose \
--alter="add column comm varchar(10)" \
--switch-to-rbr \
--allow-master-master \
--cut-over=default \
--exact-rowcount \
--concurrent-rowcount \
--default-retries=120 \
--panic-flag-file=/tmp/ghost.panic.flag \
--initially-drop-ghost-table \
--initially-drop-old-table \
--initially-drop-socket-file \
--execute
```

主从环境DDL
```
gh-ost \
--max-load=Threads_running=25 \
--critical-load=Threads_running=1000 \
--chunk-size=1000 \
--throttle-control-replicas="10.0.139.161,10.0.139.162" \
--max-lag-millis=1500 \
--user="dba" \
--password="Abcd123#" \
--host=10.0.139.163 \
--port=33006 \
--database="test" \
--table="tab1" \
--verbose \
--alter="add column comm varchar(10)" \
--switch-to-rbr \
--allow-master-master \
--cut-over=default \
--exact-rowcount \
--concurrent-rowcount \
--default-retries=120 \
--panic-flag-file=/tmp/ghost.panic.flag \
--initially-drop-ghost-table \
--initially-drop-old-table \
--initially-drop-socket-file \
--execute
```

从库DDL测试
```
gh-ost \
--user="dba" \
--password="Abcd123#" \
--host=10.0.139.163 \
--port=33006 \
--test-on-replica \
--database="test" \
--table="tab1" \
--verbose \
--alter="eadd column comm varchar(10)" \
--initially-drop-ghost-table \
--initially-drop-old-table \
--max-load=Threads_running=30 \
--switch-to-rbr \
--chunk-size=500 \
--cut-over=default \
--exact-rowcount \
--concurrent-rowcount \
--serve-socket-file=/tmp/gh-ost.test.sock \
--initially-drop-ghost-table \
--initially-drop-old-table \
--initially-drop-socket-file \  
--execute
```
==Tips：上述操作并不会删除临时表，可以手动删除==

暂停任务
```
echo throttle | socat - /tmp/gh-ost.test.tab1.sock
```

恢复任务
```
echo no-throttle | socat - /tmp/gh-ost.test.tab1.sock
```

**操作建议**

1. 仔细阅读相关参数说明
2. 添加参数–test-on-replica尝试几次
3. 对每个主迁移，首先发出一个noop空操作
4. 通过–execute真实执行
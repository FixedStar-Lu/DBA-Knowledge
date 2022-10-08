[TOC]

---

## Clone Plugin

MySQL8.0.17推出了Clone plugin功能，其允许用户对当前示例进行本地或远程克隆，对快速搭建复制环境，备份等场景非常有用。
- 本地克隆：将数据从当前MySQL Server克隆到当前主机上的一个指定目录下面
- 远程克隆：启动克隆的MySQL Server称之为recipient(接收者)，远程数据源的MySQL Server称之为donor(提供者)，在recipient上执行远程克隆，数据会通过网络传输到recipient。克隆过程中会覆盖清除recipient上的所有数据，如果不希望被覆盖需指定克隆数据放在其它目录
- clone plugin支持在复制拓扑中使用。除了克隆数据外，克隆还会携带复制信息(binlog位置)，并将其应用于接收方。我们可以用于在组复制中添加成员，也可以在主从环境中添加新的从节点。当组成员和待加入组的Server都支持Clone plugin，待加入的组成员可以自行选择一个更高效的方式获取数据。
- 克隆插件支持克隆数据加密的和数据页压缩

### 安装Clone Plugin

安装插件
```
root@(none) 10:46:  INSTALL PLUGIN clone SONAME 'mysql_clone.so';
```

查看插件
```
root@(none) 10:47:  select plugin_name,plugin_status from information_schema.plugins where plugin_name='clone';
+-------------+---------------+
| plugin_name | plugin_status |
+-------------+---------------+
| clone       | ACTIVE        |
+-------------+---------------+
```

如果想要防止MySQL Server在没有克隆插件的情况下运行，可以在配置文件设置参数控制克隆插件状态。在插件初始化失败时，强制MySQL Server启动失败
```
clone=FORCE_PLUS_PERMANENT
```

### 本地克隆

本地克隆需要backup_admin的权限
```
GRANT BACKUP_ADMIN ON *.* TO 'clone_user'@'localhost';
```

执行克隆
```
root@(none) 11:11:  CLONE LOCAL DATA DIRECTORY = '/service/mysql/backup/20200515';
```
- mysql用户需要对目录有写权限
- 20200515目录事先不能存在
- 备份路径需要设置绝对路径

查看克隆状态
```
root@performance_schema 16:06:  select * from clone_status;
+------+-------+-----------+-------------------------+-------------------------+----------------+---------------------------------+----------+---------------+-------------+-----------------+---------------+
| ID   | PID   | STATE     | BEGIN_TIME              | END_TIME                | SOURCE         | DESTINATION                     | ERROR_NO | ERROR_MESSAGE | BINLOG_FILE | BINLOG_POSITION | GTID_EXECUTED |
+------+-------+-----------+-------------------------+-------------------------+----------------+---------------------------------+----------+---------------+-------------+-----------------+---------------+
|    1 | 21847 | Completed | 2020-05-15 15:46:23.868 | 2020-05-15 15:46:45.992 | LOCAL INSTANCE | /service/mysql/backup/20200515/ |        0 |               |             |               0 |               |
+------+-------+-----------+-------------------------+-------------------------+----------------+---------------------------------+----------+---------------+-------------+-----------------+---------------+
1 row in set (0.00 sec)

root@performance_schema 16:06:  select * from clone_progress;
+------+-----------+-------------+----------------------------+----------------------------+---------+------------+------------+---------+------------+---------------+
| ID   | STAGE     | STATE       | BEGIN_TIME                 | END_TIME                   | THREADS | ESTIMATE   | DATA       | NETWORK | DATA_SPEED | NETWORK_SPEED |
+------+-----------+-------------+----------------------------+----------------------------+---------+------------+------------+---------+------------+---------------+
|    1 | DROP DATA | Completed   | 2020-05-15 15:46:23.868239 | 2020-05-15 15:46:23.868924 |       1 |          0 |          0 |       0 |          0 |             0 |
|    1 | FILE COPY | Completed   | 2020-05-15 15:46:23.868972 | 2020-05-15 15:46:37.725829 |       2 | 1393161217 | 1393161217 |       0 |          0 |             0 |
|    1 | PAGE COPY | Completed   | 2020-05-15 15:46:37.725944 | 2020-05-15 15:46:37.928775 |       2 |          0 |          0 |       0 |          0 |             0 |
|    1 | REDO COPY | Completed   | 2020-05-15 15:46:37.928857 | 2020-05-15 15:46:38.129620 |       2 |       3072 |       3072 |       0 |          0 |             0 |
|    1 | FILE SYNC | Completed   | 2020-05-15 15:46:38.129712 | 2020-05-15 15:46:45.992192 |       2 |          0 |          0 |       0 |          0 |             0 |
|    1 | RESTART   | Not Started | NULL                       | NULL                       |       0 |          0 |          0 |       0 |          0 |             0 |
|    1 | RECOVERY  | Not Started | NULL                       | NULL                       |       0 |          0 |          0 |       0 |          0 |             0 |
+------+-----------+-------------+----------------------------+----------------------------+---------+------------+------------+---------+------------+---------------+
```

### 远程克隆

远程克隆语法
```
CLONE INSTANCE FROM 'user'@'host':port IDENTIFIED BY 'password' [DATA DIRECTORY [=] 'clone_dir'] [REQUIRE [NO] SSL];
```

将提供者信息添加到接收者上
```
MySQL> SET GLOBAL clone_valid_donor_list = '10.0.139.162:33006';
```

接收者上执行克隆命令
```
MySQL> CLONE INSTANCE FROM 'clone_user'@'10.0.139.162':33006 IDENTIFIED BY 'Abcd123#' ;
```

### 克隆限制

远程克隆存在下列限制：
- 远程克隆操作不支持由mysqlx端口指定的X协议端口
- 提供者和接收者都需要安装插件
- 提供者和接收者都需要创建克隆用户，提供者需要BACKUP_ADMIN权限，接收者需要CLONE_ADMIN权限，其中CLONE_ADMIN包含BACKUP_ADMIN和shutdown权限
- 提供者和接收者版本需要一致
- 提供者和接收者字符集设置需要一致
- 不支持通过mysql router连接提供者
- 克隆插件只克隆存储在InnoDB中的数据。其他存储引擎数据不是克隆的，binlog以及配置文件都不会被复制
- 如果没有设置data directory，接收者默认在克隆前会清除自身数据，并在clone完成后进行重启
- 提供者和接收者需要具有相同的innodb_page_size和innodb_data_file_path系统变量设置
- 如果克隆加密数据或压缩页数据，则提供者和接收者必须具有相同的文件系统块大小
- 如果要克隆加密数据，则需要启用安全连接
- 接收者的系统变量clone_valid_donor_list必须包含提供者的主机地址
- 克隆操作只能串行执行，不能多个克隆操作并行执行，可通过performance_schema.clone_status确认
- 克隆插件以1MB大小的数据包和1M大小的元数据的形式传输数据，max_allowed_packet不能低于2M
- UNDO表空间文件名必须是唯一的。无论提供者的undo表空间放在什么位置，克隆后接收者都存放在innodb_undo_directory系统变量下
- 在Clone过程中DDL以及truncate table会被阻塞

### 克隆流程

克隆流程主要包含：INIT->FILE COPY->PAGE COPY->REDO COPY->Done

**INIT**

需要持有backup lock, 阻止ddl进行

**FILE COPY**

按照文件进行拷贝，同时开启page tracking功能，记录在文件拷贝过程中修改的page。此时会设置buf_pool->track_page_lsn为当前lsn，track_page_lsn在flush page阶段用到

**PAGE COPY**

开启redo archiving功能，从当前点存储新增的redo log，这样从当前点开始的增量修改都不会丢失；同时在page track的page被发送到目标端，确保当前点之前所做的变更一定发送到目标端。

**redo copy**

停止Redo Archiving, 所有归档的日志被发送到目标端，这些日志包含了从page copy阶段开始到现在的所有日志

**Done**

接收者重启实例，通过Crash Recovery应用redo log

### 参考链接 

[1. 初探 Clone Plugin](https://baijiahao.baidu.com/s?id=1641811153267265811&wfr=spider&for=pc)

[2. MySQL 8.0 clone plugin](https://mp.weixin.qq.com/s/RvbmIpss6b1S28EIjw3_BQ)

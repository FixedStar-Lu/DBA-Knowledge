[TOC]

---

# MySQL8 - 备份优化

## Redo Log Archiving

在MySQL8.0中启用了redo log archive的功能，旨在解决一致性备份问题。在之前的版本中，由于redo是固定大小循环写入的，如果备份速度跟不上redo log生成的速度，则无法保持备份一致性。redo log archive在备份启动时同步启动，备份结束时停止，此时可以利用redo log归档进行数据恢复。

在启用redo log archive之前，需要先通过参数innodb_redo_log_archive_dirs设置redo log归档路径
```
root@(none) 21:22:  set global innodb_redo_log_archive_dirs="backup:/service/mysql/archive";
Query OK, 0 rows affected (0.01 sec)
```
其中backup为tag标记，说明其是用于备份，/service/mysql/archive为redo log归档目录，我们也可以通过分号设置多个路径

接下来我们就DO innodb_redo_log_archive_start('backup')来手动启动redo log archive
```
root@(none) 21:41:  SELECT innodb_redo_log_archive_start('backup');
+-----------------------------------------+
| innodb_redo_log_archive_start('backup') |
+-----------------------------------------+
|                                       0 |
+-----------------------------------------+

root@(none) 21:41:  DO innodb_redo_log_archive_start('backup');
Query OK, 0 rows affected (0.09 sec)
```
*注：开启archive的会话必须存活才能生成archive日志*

通过sysbench性能测试工具模拟数据操作后停止redo log archive
```
root@(none) 21:50:  do innodb_redo_log_archive_stop();
Query OK, 0 rows affected (0.05 sec)
```

这时可以看见在归档目录下生成了对应的归档日志
```
[root@t-luhx02-v-szzb archive]# ls -lh
total 527M
-r--r----- 1 mysql mysql 527M Jun 14 21:54 archive.fba19872-aa1a-11ea-b40c-005056abb91e.000001.log
```

Tips：详情请参考：[Redo Log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)

## Backup Lock

在MySQL8.0之前，为保证innoDB与非InnoDB表的一致性，通常会使用flush table with read lock来实现一致性备份。在加锁过程中，MySQL变为只读，无法更新。如果使用xtrackup时没加上rsync参数，锁定的时间会更长，影响也就越大。而在MySQL8.0中引入了LOCK INSTANCE FOR BACKUP轻量级备份锁，并且数据字典也全变为InnoDB引擎了，如果没有非InnoDB的表则使用Backup Log来获取一致性位置，期间不允许DDL，但DML可以执行

加锁
```
root@(none) 09:12:  lock instance for backup;
Query OK, 0 rows affected (0.00 sec)
```

解锁
```
root@(none) 09:12:  unlock instance;
Query OK, 0 rows affected (0.00 sec)
```
## Page Tracking

Page Tracking是针对增量备份的优化，减少不必要的数据页扫描。有点类似与Oracle中的块跟踪，只是MySQL的最小单位为页。当前有三种扫描方式：
- page-track：利用LSN跟踪上一次备份之后被修改的数据页，增量备份仅复制这些修改的页
- Optimistic：扫描上一次备份之后被修改的InnoDB数据文件，查找并复制修改的页，依赖于系统时间
- full-scan：扫描所有数据文件，找出上一次备份之后被修改的数据页

安装备份组件
```
root@(none) 09:30:  INSTALL COMPONENT "file://component_mysqlbackup";
Query OK, 0 rows affected (0.16 sec)
```

全备之前开启page tracking
```
root@(none) 09:30:  SELECT mysqlbackup_page_track_set(true);
+----------------------------------+
| mysqlbackup_page_track_set(true) |
+----------------------------------+
|                       4282913844 |
+----------------------------------+
```

全备后进行增量备份
```
mysqlbackup --incremental-backup-dir=/service/mysql/backup/inc --trace=3 --incremental=page-track --incremental-base=history:last_full_backup backup
```

其中--incremental-base有三种选择：
- last_backup：基于前一次备份做增量备份，前一次可能是全备，也可能是增量备份
- last_full_backup：基于前一次全备做增量备份
- dir：基于前一次的备份目录，前一次可能是全备，也可能是增量备份

Tips:[InnoDB Clone and page tracking](http://dbapub.cn/2020/06/14/MySQL8.0%E5%A4%87%E4%BB%BD%E4%BC%98%E5%8C%96/)
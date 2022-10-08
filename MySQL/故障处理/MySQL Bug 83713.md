# Bug 83713：Slave failed to initialize relay log info after OS crash when use MTS and GTID

在一次物理机异常蓝屏导致一台从数据库hang住后，通过重启恢复之后启动数据库时错误日志中出现如下错误

```
2020-05-07T00:33:05.503723Z 80 [Warning] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
2020-05-07T00:33:56.888307Z 81 [ERROR] Error in Log_event::read_log_event(): 'Event too small', data_len: 0, event_type: 0
2020-05-07T00:33:56.888330Z 81 [ERROR] Error reading relay log event for channel '': slave SQL thread aborted because of I/O error
2020-05-07T00:33:56.888348Z 81 [ERROR] Slave SQL for channel '': Relay log read failure: Could not parse relay log event entry. The possible reasons are: the master's binary log is corrupted (you can check this by running 'mysqlbinlog' on the binary log), the slave's relay log is corrupted (you can check this by running 'mysqlbinlog' on the relay log), a network problem, or a bug in the master's or slave's MySQL code. If you want to check the master's binary log or slave's relay log, you will be able to know their names by issuing 'SHOW SLAVE STATUS' on this slave. Error_code: 1594
2020-05-07T00:33:56.888352Z 81 [ERROR] Error running query, slave SQL thread aborted. Fix the problem, and restart the slave SQL thread with "SLAVE START". We stopped at log 'FIRST' position 386502095
2020-05-06T15:48:25.440945Z 133 [Warning] Storing MySQL user name or password information in the master info repository is not secure and is therefore not recommended. Please consider using the USER and PASSWORD connection options for START SLAVE; see the 'START SLAVE Syntax' in the MySQL Manual for more information.
```

尝试启动slave复制进程也同样遭遇了问题
```
mysql> start slave;
ERROR 1872 (HY000): Slave failed to initialize relay log info structure from the repository
```

保存好Master_Log_File和Read_Master_Log_Pos信息后，尝试reset slave再start slave后，出现如下错误
```
Last_Error: Relay log read failure: Could not parse relay log event entry. The possible reasons are: the master's binary log is corrupted (you can check this by running 'mysqlbinlog' on the binary log), the slave's relay log is corrupted (you can check this by running 'mysqlbinlog' on the relay log), a network problem, or a bug in the master's or slave's MySQL code. If you want to check the master's binary log or slave's relay log, you will be able to know their names by issuing 'SHOW SLAVE STATUS' on this slave
```

经过查询发现这是[Bug #83713](https://bugs.mysql.com/bug.php?id=83713)引起的问题，解决方案如下：
```
reset slave;
start slave IO_THREAD;
stop slave IO_THREAD;
reset slave;
start slave;
```
据bug中的评论信息，该bug在8.0.13版本中依旧存在


## MySQL开启GTID

  

**1）设置ENFORCE_GTID_CONSISTENCY参数**

将参数ENFORCE_GTID_CONSISTENCY设置为WARN，允许主库执行违反GTID一致性校验的SQL，并记录告警到错误日志中，以此观察是否有违反GTID一致性校验的SQL
```
mysql> set global enforce_gtid_consistency=warn;
```

观察一段时间后确认无相关告警记录，则正式开启该参数，开启之后如果有违反GTID一致性的SQL将会直接报错
```
mysql> set global enforce_gtid_consistency=on;
```

**2）设置GTID_MODE**

将MODE设置为OFF_PERMISSIVE模式表示新生成的事务还是匿名事务，但同步复制也支持GTID事务
```
mysql> set global gtid_mode=OFF_PERMISSIVE;
```

将MODE设置为ON_PERMMISSIVE模式表示新生成的事务为GTID事务，但同步复制也支持匿名事务
```
mysql> set global gtid_mode=ON_PERMISSIVE;
```

观察匿名事务是否同步回放完成，输出为0则表示回放完成
```
mysql> show status like 'ONGOING_ANONYMOUS_TRANSACTION_COUNT';
```

刷新binlog日志，确保当前binlog只包含GTID事务
```
mysql> flush logs;
```

将MODE设置为ON正式开启GTID
```
mysql> set global gtid_mode=ON;
```

将下列参数持久化到配置文件中
```
[mysqld]
gtid-mode=on;
enforce-gtid-consistency=1
```

**3）修改从库复制模式采用AUTO_POSITION**
```
mysql> show slave status\G
mysql> stop slave;
mysql> change master to master_auto_position=1;
mysql> start slave;
mysql> show slave status\G
```

## 重建从库
**1）备份数据库**

利用xtrabackup进行数据库备份并复制到新节点上
```
innobackupex --user=backup --password=xxxxx --parallel=10 --stream=xbstream /tmp > /mysqlbackup/backup-20220824.xbstream
```

如果两台机器做了互信可以直接备份到目标节点上，省去复制备份文件的时间
```
innobackupex --user=backup --password='xxx' --stream=xbstream /tmp | ssh root@10.0.139.162 \ "cat - > /service/backup.xbstream"
```

**2）数据恢复**
```
# 解压备份到数据目录
xbstream -x < /mysqlbackup/backup-20220824.xbstream -C /service/mysql/data

# 应用日志
innobackupex --apply-log /service/mysql/data

# 修改目录权限
chown -R mysql.mysql /service/mysql
```

启动数据库
```
systemctl start mysql
```

**3）建立同步复制**

复制的位点信息保存在数据目录下的`xtrabackup_binlog_pos_innodb`文件中
```
mysql> change master to master_host='10.186.65.77', MASTER_PORT=3306,master_user='repl',master_password='xxxxx',MASTER_LOG_FILE='mysql-bin.000002',MASTER_LOG_POS=1922;
```
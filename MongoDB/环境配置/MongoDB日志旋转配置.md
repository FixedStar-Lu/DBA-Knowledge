#MongoDB #LogRotation
# MongoDB日志旋转

日志文件是我们排查分析问题的关键因素，但过大的日志不仅会占用大量的存储空间，并且数据库不断写入一个大文件可能会导致性能问题。通过Log Rotation，我们可以主动避免出现该情况，将日志大小保持在阈值以下

mongodb的日志文件由logpath参数指定，其中记录了错误日志、慢查询日志等信息。日志并不会自动进行rotation，需要我们手动控制。一般有下列两种方式：
- logRotate command：通过在admin数据库下执行db.adminCommand( { logRotate : 1 } )来进行日志旋转
- SIGUSR1：通过SIGUSR1信号来进行旋转

日志旋转的行为根据参数Logrotate(V3.0)的值而不同：
- rename：重命名日志文件，并创建由logpath参数指定的新文件以写入进一步的日志
- reopen：按照linux自身的Logrotate旋转行为关闭并重新打开日志文件，设置该值需要启用logAppend参数，如果采用linux的logrotate工具，应该设置reopen

在2.6或更早版本中，发出logRotate命令时的默认行为与使用rename命令时相同，这时我们需要手动或通过脚本进行压缩和迁移历史日志。当然，我们也可以使用reopen并结合linux的logrotate程序来完成日志旋转

## MongoDB3.x(reopen+logrotate)

对mongod参数文件进行如下配置
```
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
  logRotate: reopen
processManagement: 
  pidFilePath: /var/run/mongodb/mongod.pid
```

在/etc/logrotate.d下创建专用的配置文件
```
$ cat >> /etc/logrotate.d/logrotate-mongod.conf <<EOF
/var/log/mongodb/mongod.log {
  daily
  size 100M
  rotate 10
  missingok
  compress
  delaycompress
  notifempty
  create 640 mongod mongod
  sharedscripts
  postrotate
    /bin/kill -SIGUSR1 `cat /var/run/mongodb/mongod.pid 2>/dev/null` >/dev/null 2>&1
  endscript
}
EOF
```

对于上述配置，Linux的Logrotate工具会执行下列操作：
- 检查日志大小，超过100M时开始执行logrotate
- 将mongod.log文件迁移为mongod.log.1
- 创建一个新的mongod.log文件
- 压缩为文件mongod.log.2，但按压缩和旋转10直到mongod.log.10
- MongoDB继续写入旧文件mongod.log.1(基于inode)
- 通过SIGUSR1完成日志旋转，mongod创建Mongod.log并开始写入它



## MongoDB 2.x或logRotate=rename

由于MongoDB 3.0中引入了logRotate参数，所以当使用logRotate=rename或使用version<=2.x时，日志旋转脚本需要额外的步骤。

对mongod参数文件进行如下配置
```
logAppend=true
logpath=/var/log/mongodb/mongod.log
pidfilePath=/var/run/mongodb/mongod.pid
```

在/etc/logrotate.d下创建专用的配置文件
```
$ cat >> /etc/logrotate.d/mongod.conf <<EOF
/var/log/mongodb/mongod.log {
  daily
  size 100M
  rotate 10
  missingok
  compress
  delaycompress
  notifempty
  create 640 mongod mongod
  sharedscripts
  postrotate
    /bin/kill -SIGUSR1 `cat /var/run/mongodb/mongod.pid 2>/dev/null` >/dev/null 2>&1
    find /var/log/mongodb -type f -size 0 -regextype posix-awk \
-regex "^\/var\/log\/mongodb\/mongod\.log\.[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}-[0-9]{2}-[0-9]{2}$" \
-execdir rm {} \; >/dev/null 2>&1
  endscript
}
EOF
```

在这种情况下，logrotate会执行如下操作：
- 检查日志大小，超过100M时开始执行logrotate
- 将mongod.log文件迁移到mongod.log.1
- 创建新的mongod.log文件
- MongoDB继续写入旧文件mongod.log.1
- 通过SIGUSR1完成日志旋转，并写入mongod.log文件
- Linux通过find查找并删除格式为mongod.log.xxxx-xx-xxTxx-xx-xx，大小字节为0的文件

## Reference
[Automating MongoDB Log Rotation](https://www.percona.com/blog/2018/09/27/automating-mongodb-log-rotation/)


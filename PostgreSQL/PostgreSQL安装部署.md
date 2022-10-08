[TOC]

# PostgreSQL安装部署

## 系统配置

安装依赖包

```
yum -y install coreutils glib2 lrzsz dstat sysstat e4fsprogs xfsprogs \
ntp readline-devel zlib-devel openssl-devel pam-devel libxml2-devel \
libxslt-devel python-devel tcl-devel gcc gcc-c++ make smartmontools \
flex bison perl-devel perl-ExtUtils* openldap-devel \
jadetex  openjade bzip2 openssl
```



配置OS内核(/etc/sysctl.conf)

```
fs.aio-max-nr = 1048576
fs.file-max = 76724600
# 信号量，每组多少信号量-总共多少信号量-每个semop()调用支持多少操作-多少组信号量
kernel.sem = 4096 2147483647 2147483646 512000
# 共享内存段大小限制(建议为内存80%)，单位为页
kernel.shmall = 107374182
# 最大单个共享内存段大小(建议为内存的一半)，单位为字节
kernel.shmmax = 274877906944
# 总共能生成多少共享内存段
kernel.shmmni = 819200
net.core.netdev_max_backlog = 10000
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 4194304
net.ipv4.tcp_max_syn_backlog = 16384
net.core.somaxconn = 16384
net.ipv4.tcp_keepalive_intvl = 20
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_mem = 8388608 12582912 16777216
net.ipv4.tcp_fin_timeout = 5
net.ipv4.tcp_synack_retries = 2
# 开启SYN Cookies。当出现SYN等待队列溢出时，启用cookie来处理，可防范少量的SYN攻击
net.ipv4.tcp_syncookies = 1
# 减少time_wait
net.ipv4.tcp_timestamps = 1
# 如果为1则开启TCP连接中TIME-WAIT套接字的快速回收，但是NAT环境可能导致连接失败，建议服务端关闭它
net.ipv4.tcp_tw_recycle = 0
# 开启重用。允许将TIME-WAIT套接字重新用于新的TCP连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 262144
net.ipv4.tcp_rmem = 8192 87380 16777216
net.ipv4.tcp_wmem = 8192 65536 16777216
# 禁用 numa, 或者在vmlinux中禁止
vm.zone_reclaim_mode = 0
# 本地自动分配的TCP, UDP端口号范围
net.ipv4.ip_local_port_range = 40000 65535
# 单个进程允许打开的文件句柄上限
fs.nr_open=20480000
```

内核配置生效

```
sysctl -p
```

配置OS资源限制(/etc/security/limits.conf)

```
* soft    nofile  1024000
* hard    nofile  1024000
* soft    nproc   unlimited
* hard    nproc   unlimited
* soft    core    unlimited
* hard    core    unlimited
* soft    memlock unlimited
* hard    memlock unlimited
```

禁用透明大页

```
[postgres@t-luhx01-v-szzb ~]$ echo never >  /sys/kernel/mm/transparent_hugepage/enabled
[postgres@t-luhx01-v-szzb ~]$ echo never > /sys/kernel/mm/transparent_hugepage/defrag
[postgres@t-luhx01-v-szzb ~]$ echo "echo never >  /sys/kernel/mm/transparent_hugepage/enabled" >> /etc/rc.local
[postgres@t-luhx01-v-szzb ~]$ echo "echo never > /sys/kernel/mm/transparent_hugepage/defrag" >> /etc/rc.local
```

关闭selinux

```
[postgres@t-luhx01-v-szzb ~]$ vi /etc/selinux/config
SELINUX=disabled
SELINUXTYPE=targeted
```

创建用户

```
[postgres@t-luhx01-v-szzb ~]$ groupadd postgres
[postgres@t-luhx01-v-szzb ~]$ useradd postgres -g postgres
```

创建数据目录

```
[postgres@t-luhx01-v-szzb ~]$ mkdir -p /service/pgsql/{data,backups,scripts,archive_wals}
[postgres@t-luhx01-v-szzb ~]$ chown -R postgres.postgres /service/pgsql
```

## PostgreSQL安装

下载介质

[DownLoad](https://ftp.postgresql.org/pub/source/v11.8/postgresql-11.8.tar.gz)

解压并编译安装

```
[postgres@t-luhx01-v-szzb ~]$ tar -xvf postgresql-11.8.tar.gz
[postgres@t-luhx01-v-szzb ~]$ cd postgresql-11.8
[postgres@t-luhx01-v-szzb ~]$ ./configure --prefix=/usr/local/postgresql --with-openssl
[postgres@t-luhx01-v-szzb ~]$ make world && make install-world -j 4
[postgres@t-luhx01-v-szzb ~]$ chown -R postgres.postgres /usr/local/postgresql
```

修改用户环境变量

```
[root@t-luhx01-v-szzb postgresql-11.8]# su - postgres
[postgres@t-luhx01-v-szzb ~]$ vi ~/.bash_profile
[postgres@t-luhx01-v-szzb ~]$ cat ~/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH
export PGPORT=3433
export PGDATA=/service/pgsql/data
export LANG=en_US.utf8
export PGHOME=/usr/local/postgresql
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH
export DATE=`date +"%Y%m%d%H%M"`
export PATH=$PGHOME/bin:$PATH:.
export MANPATH=$PGHOME/share/man:$MANPATH
export PGHOST=$PGDATA
export PGUSER=postgres
export PGDATABASE=postgres

[postgres@t-luhx01-v-szzb ~]$ source ~/.bash_profile
```

客户端及服务端程序

- clusterdb：是SQL CLUSTER命令的封装，通过索引对数据库中基于堆表的物理文件重新排序，在一定场景下可以节省磁盘访问，加快查下速度
- reindexdb：是SQL REINDEX命令的封装，当索引发生膨胀或索引物理文件损坏时，可以使用reindexdb对指定的表或数据库重建索引并删除旧的索引
- vacuumdb：PostgreSQL数据库的VACUUM、VACUUM FREEZE、VACUUM FULL、VACUUM ANALYZE命令的封装，主要是对数据的物理文件等的垃圾回收
- vacuumlo：用来清理数据库中未引用的大对象
- createdb/dropdb：分别对应CREATE DATABASE和DROP DATABASE，用于创建数据库和删除数据库
- createuser/dropuser：分别对应CREATE USER和DROP USER，用于创建用户和删除用户
- pg_basebackup：对正在运行的PostgreSQL实例的基础备份
- pg_dump和pg_dumpall：以数据库转储方式进行备份
- pg_restore：用于从pg_dump命令创建的非文本格式的备份中恢复数据
- pgbench：运行基准测试工具，可以进行简单的压力测试
- pg_config：获取当前安装的PostgreSQL应用程序的配置参数
- pg_isready：用于检测数据库服务器是否已经允许接受连接
- pg_receivexlog：用于从一个运行的实例中获取事务日志流
- pg_recvlogical：控制逻辑解码复制槽以及来自这种槽的流数据
- pgsql：连接PostgreSQL实例的客户端命令行工具
- initdb：用于创建新的数据库目录
- pg_archivecleanup：清理PostgreSQL WAL归档文件的工具
- pg_controldata：显示数据库服务器的控制信息，例如目录版本，检查点信息等
- pg_ctl：是初始化、启动、停止、控制数据库服务器的工具
- pg_resetwal：清理预写日志并且有选择地重置在pg_control文件中的一些控制信息，当服务器由于控制文件损坏，pg_resetwal可以作为最后的手段
- pg_rewind：在master和slave发生切换后，将旧的master通过同步模式恢复，避免重做基础备份
- pg_test_fsync：通过快速测试，了解系统使用哪一种预写日志的同步方式最快，还可以在发生I/O问题时提供诊断信息
- pg_test_timing：是一种度量系统计时开销以及确认系统时间不会回退的工具
- pg_upgrade：原地升级时的升级工具
- pg_waldump：将预写日志解析为可读的格式
- postgres：PostgreSQL服务器程序

初始化数据库

```
[postgres@t-luhx01-v-szzb ~]$ initdb -D $PGDATA -U postgres -W --lc-collate=C --lc-ctype=en_US.utf8 -E UTF8
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locales
  COLLATE:  C
  CTYPE:    en_US.utf8
  MESSAGES: en_US.utf8
  MONETARY: en_US.utf8
  NUMERIC:  en_US.utf8
  TIME:     en_US.utf8
The default text search configuration will be set to "english".

Data page checksums are disabled.
Enter new superuser password:
Enter it again:
fixing permissions on existing directory /service/pgsql/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default timezone ... PRC
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    pg_ctl -D /service/pgsql/data -l logfile start
```

修改数据库参数

```
[postgres@t-luhx01-v-szzb ~]$ cd $PGDATA
[postgres@t-luhx01-v-szzb data]$ cat postgresql.conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
listen_addresses = '0.0.0.0'
port = 3433  # 监听端口
max_connections = 2000  # 最大允许的连接数
superuser_reserved_connections = 10
unix_socket_directories = '.'
unix_socket_permissions = 0700
tcp_keepalives_idle = 60
tcp_keepalives_interval = 60
tcp_keepalives_count = 10
shared_buffers = 2GB                  # 共享内存，建议设置为系统内存的1/4  .
maintenance_work_mem = 512MB           # 系统内存超过32G时，建议设置为1GB。超过64GB时，建议设置为2GB。超过128GB时，建议设置为4GB。
work_mem = 64MB                        # 1/4 主机内存 / 256 (假设256个并发同时使用work_mem)
wal_buffers = 128MB                    # min( 2047MB, shared_buffers/32 )
dynamic_shared_memory_type = posix
vacuum_cost_delay = 0
bgwriter_delay = 10ms
bgwriter_lru_maxpages = 500
bgwriter_lru_multiplier = 5.0
effective_io_concurrency = 0
max_worker_processes = 128
max_parallel_workers_per_gather = 4        # 建议设置为主机CPU核数的一半。
max_parallel_workers = 4                   # 看业务AP和TP的比例，以及AP TP时间交错分配。实际情况调整。例如 主机CPU cores-2
wal_level = replica
fsync = on
synchronous_commit = off               # 
full_page_writes = on                  # 支持原子写超过BLOCK_SIZE的块设备，在对齐后可以关闭。或者支持cow的文件系统可以关闭。
wal_writer_delay = 10ms
wal_writer_flush_after = 1MB
checkpoint_timeout = 30min
max_wal_size = 2GB                    # shared_buffers*2
min_wal_size = 512MB                     # max_wal_size/4
archive_mode = always
archive_command = '/bin/date'
hot_standby = on
max_wal_senders = 10
max_replication_slots = 10
wal_receiver_status_interval = 1s
max_logical_replication_workers = 4
max_sync_workers_per_subscription = 2
random_page_cost = 1.2
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000.0
min_parallel_table_scan_size = 8MB
min_parallel_index_scan_size = 512kB
effective_cache_size = 2GB                 # 建议设置为主机内存的5/8。
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_min_duration_statement = 5s
log_checkpoints = on
log_connections = off                            # 如果是短连接，并且不需要审计连接日志的话，建议OFF。
log_disconnections = off                         # 如果是短连接，并且不需要审计连接日志的话，建议OFF。
log_error_verbosity = verbose
log_line_prefix = '%m [%p] '
log_lock_waits = on
log_statement = 'ddl'
log_timezone = 'PRC'
log_autovacuum_min_duration = 0
autovacuum_max_workers = 5
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_scale_factor = 0.05
autovacuum_freeze_max_age = 1000000000
autovacuum_multixact_freeze_max_age = 1200000000
autovacuum_vacuum_cost_delay = 0
statement_timeout = 0                                # 单位ms, s, min, h, d.  表示语句的超时时间，0表示不限制。
lock_timeout = 0                                     # 单位ms, s, min, h, d.  表示锁等待的超时时间，0表示不限制。
idle_in_transaction_session_timeout = 2h             # 单位ms, s, min, h, d.  表示空闲事务的超时时间，0表示不限制。
vacuum_freeze_min_age = 50000000
vacuum_freeze_table_age = 800000000
vacuum_multixact_freeze_min_age = 50000000
vacuum_multixact_freeze_table_age = 800000000
datestyle = 'iso, ymd'
timezone = 'PRC'
lc_messages = 'en_US.UTF8'
lc_monetary = 'en_US.UTF8'
lc_numeric = 'en_US.UTF8'
lc_time = 'en_US.UTF8'
default_text_search_config = 'pg_catalog.simple'
shared_preload_libraries='pg_stat_statements'
```

修改数据库认证权限访问控制ACL(pg_hba.conf)

```
host all all 0.0.0.0/0 md5
host replication rep 0.0.0.0/0 md5
host all postgres 0.0.0.0/0 reject
```

pg_hba的每一行内容的格式都为：type database user address method

- type：可选值有local、host、hostssl、hostnosll。local采用unix套接字连接，host使用TCP/IP建立的连接，同时匹配SSL和非SSL连接。默认安装只监听本地localhost，不允许使用TCP/IP，远程连接需要修改配置文件中的listen_addresses地址。hostssl要求客户端和服务端都安装了openssl，编译安装时指定了–with-openssl，postgresql.conf中参数ssl=on
- database：指定生效的数据库，可以设置ALL也可以特定数据库
- user：指定生效的用户
- address：访问白名单地址
- method：可选值有trust、reject、md5和password。reject用于拒绝特定主机，md5和password的区别在于md5会进行双重加密，而password是明文密码
- PG10中新增了scram-sha-256基于SASL的认证方式，不支持旧客户端，PG10之前的客户端连接会报错

启动数据库

```
[postgres@t-luhx01-v-szzb ~]$ pg_ctl start
waiting for server to start....2020-07-08 17:36:35.727 CST [104259] LOG:  00000: listening on IPv4 address "0.0.0.0", port 3433
2020-07-08 17:36:35.727 CST [104259] LOCATION:  StreamServerPort, pqcomm.c:593
2020-07-08 17:36:35.728 CST [104259] LOG:  00000: listening on Unix socket "./.s.PGSQL.3433"
2020-07-08 17:36:35.728 CST [104259] LOCATION:  StreamServerPort, pqcomm.c:587
2020-07-08 17:36:35.975 CST [104259] LOG:  00000: redirecting log output to logging collector process
2020-07-08 17:36:35.975 CST [104259] HINT:  Future log output will appear in directory "log".
2020-07-08 17:36:35.975 CST [104259] LOCATION:  SysLogger_Start, syslogger.c:668
 done
server started
```

关闭数据库

```
[postgres@t-luhx01-v-szzb ~]$ pg_ctl stop [DATADIR] [-m SHUTDOWN-MODE] [-W] [-t SECS] -s
```

其中”-s”参数开启和关闭屏幕上的消息输出，”-t SECS”参数设置超时时间，超过SECS值设置的超时时间后自动退出。其中”-m”参数控制以怎样的模式停止，PostgreSQL支持三种关闭模式：smart、fast、immediate，默认为fast模式。

- smart模式下会等待活动的事务提交结束，并等待客户端主动断开连接后关闭
- fast模式会回滚所有活动的事务，并强制断开客户端的连接并关闭数据库
- immediate模式会立即终止所有服务器进程，当下一次数据库启动时会先进行recover，一般不推荐使用
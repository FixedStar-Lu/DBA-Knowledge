#MySQL #dbdeployer
# dbdeployer构建测试环境
dbdeployer能够快速部署数据库测试环境，一键实现MySQL，MariaDB，TiDB，NDB Cluster，PXC等架构。项目地址：[GitHub - datacharmer/dbdeployer: DBdeployer is a tool that deploys MySQL database servers easily.](https://github.com/datacharmer/dbdeployer)

**安装dbdeployer**
```
[root@classhost01 soft]# tar -xvf dbdeployer-1.68.0.linux.tar.gz 
dbdeployer-1.68.0.linux
[root@classhost01 soft]# mv dbdeployer-1.68.0.linux /usr/local/bin/dbdeployer
[root@classhost01 soft]# chmod +x /usr/local/bin/dbdeployer
[root@classhost01 soft]# dbdeployer --version
dbdeployer version 1.68.0
```

**配置dbdeployer**

默认配置文件为当前用户的 $HOME/.dbdeployer/config.json 作为配置文件，可以通过 dbdeplyoer defaults export 导出并修改配置或者直接通过 dbdeployer defaults update 来更新默认文件，配置文件包含 MySQL 初始信息。

查看配置信息
```
[root@classhost01 soft]# dbdeployer defaults show
# Internal values:
{
        "version": "1.66.0",
        "sandbox-home": "$HOME/sandboxes",
        "sandbox-binary": "$HOME/opt/mysql",
        "use-sandbox-catalog": true,
        "log-sb-operations": false,
        "log-directory": "/root/sandboxes/logs",
        "cookbook-directory": "recipes",
        "shell-path": "/usr/bin/bash",
        "master-slave-base-port": 11000,
        "group-replication-base-port": 12000,
        "group-replication-sp-base-port": 13000,
        "fan-in-replication-base-port": 14000,
        "all-masters-replication-base-port": 15000,
        "multiple-base-port": 16000,
        "pxc-base-port": 18000,
        "ndb-base-port": 19000,
        "ndb-cluster-port": 20000,
        "group-port-delta": 125,
        "mysqlx-port-delta": 10000,
        "admin-port-delta": 11000,
        "master-name": "master",
        "master-abbr": "m",
        "node-prefix": "node",
        "slave-prefix": "slave",
        "slave-abbr": "s",
        "sandbox-prefix": "msb_",
        "imported-sandbox-prefix": "imp_msb_",
        "master-slave-prefix": "rsandbox_",
        "group-prefix": "group_msb_",
        "group-sp-prefix": "group_sp_msb_",
        "multiple-prefix": "multi_msb_",
        "fan-in-prefix": "fan_in_msb_",
        "all-masters-prefix": "all_masters_msb_",
        "reserved-ports": [
                1186,
                3306,
                5432,
                33060,
                33062
        ],
        "remote-repository": "https://raw.githubusercontent.com/datacharmer/mysql-docker-minimal/master/dbdata",
        "remote-index-file": "available.json",
        "remote-completion-url": "https://raw.githubusercontent.com/datacharmer/dbdeployer/master/docs/dbdeployer_completion.sh",
        "remote-tarball-url": "https://raw.githubusercontent.com/datacharmer/dbdeployer/master/downloads/tarball_list.json",
        "pxc-prefix": "pxc_msb_",
        "ndb-prefix": "ndb_msb_",
        "default-sandbox-executable": "default",
        "download-name-linux": "mysql-{{.Version}}-linux-glibc2.17-x86_64{{.Minimal}}.{{.Ext}}",
        "download-name-macos": "mysql-{{.Version}}-macos11-x86_64.{{.Ext}}",
        "download-url": "https://dev.mysql.com/get/Downloads/MySQL",
        "timestamp": "Mon Aug 15 11:30:39 CST 2022"
 }
```

更新mysql二进制包的存放位置
```
[root@classhost01 sandbox]# dbdeployer defaults update sandbox-binary /data/sandbox/mysql
```
更新项目数据目录
```
[root@classhost01 sandbox]# dbdeployer defaults update sandbox-home /data/sandbox/
```
初始化配置
```
[root@classhost01 sandbox]# cd /data/sandbox && dbdeployer init
```

**MySQL版本管理**

dbdeployer 在联网环境下，支持在线下载多种版本的 MySQL 介质，可以根据需要下载
```
[root@classhost01 data]# dbdeployer downloads list
[root@classhost01 mysql]# cd /data/sandbox/mysql && dbdeployer downloads get-unpack mysql-5.7.34-linux-glibc2.12-x86_64.tar.gz
```
如果在内网环境下无法在线下载，也可以选择执行上传对应介质到sandbox-binary目录下面
```
[root@classhost01 mysql]# dbdeployer unpack mysql-5.7.38-linux-glibc2.12-x86_64.tar.gz --sandbox-binary /data/sandbox/mysql
```
**部署实例**

创建单节点实例
```
[root@classhost01 mysql]# dbdeployer deploy single 5.7.38 --my-cnf-options="character_set_server=utf8mb4" --db-user=root --db-password=Abcd123#
```

创建主从实例
```
[root@classhost01 mysql]# dbdeployer deploy replication 5.7.38 --repl-crash-safe --gtid --my-cnf-options="character_set_server=utf8mb4" --db-user=root --db-password=Abcd123# --rpl-password=repl_user --rpl-password=Abcd123#
```

创建单主MGR实例
```
[root@classhost01 mysql]# dbdeployer deploy --topology=group replication 8.0.29 --single-primary --db-user=root --db-password=Abcd123# --rpl-password=repl_user --rpl-password=Abcd123#
```

0创建多主MGR
```
[root@classhost01 mysql]# dbdeployer deploy --topology=all-masters replication 8.0.29 --db-user=root --db-password=Abcd123# --rpl-password=repl_user --rpl-password=Abcd123#
```

**实例管理**

查看已安装的环境
```
[root@classhost01 sandbox]# dbdeployer sandboxes
 msb_5_7_38               :   single   5.7.38   [5738 ]
```

查看运行状态
```
[root@classhost01 sandbox]# dbdeployer global status
# Running "status" on msb_5_7_38
msb_5_7_38 on
```

连接实例，我们可以切换到实例数据库目录下执行use进入数据库，也可以用账号密码进入
```
[root@classhost01 msb_5_7_38]# cd /data/sandbox/msb_5_7_38 && ./use
```

关闭和启动实例
```
[root@classhost01 sandbox]#  dbdeployer global stop msb_5_7_38
# Running "stop" on msb_5_7_38
stop /data/sandbox/msb_5_7_38

[root@classhost01 sandbox]#  dbdeployer global start msb_5_7_38
# Running "start" on msb_5_7_38
```

删除实例
```
[root@t-luhx02-v-szzb ~]# dbdeployer delete msb_5_7_38
```
> 如果不希望实例被误删可以通过`dbdeployer admin lock`对实例进行加锁


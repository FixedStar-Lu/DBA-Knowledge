#MongoDB #mongoexport #mongoimport
# MongoDB备份与恢复

## 1. mongoexport/mongoimport

mongoexport用于导出MongoDB实例中的数据，支持JSON和CSV格式。而mongoimport则提供相反的导入功能。Mongoexport/mongoimport采用严格模式表示。mongoexport需要对数据库进行读访问，因此连接用户至少要有数据库的read角色。mongoimport仅支持UTF-8编码的文件，且连接用户要有readwrite的数据库角色

### 1.1 mongoexport

```
[root@t-zabbix-p-szzb ~]# mongoexport --help
Usage:
    mongoexport <options>

Export data from MongoDB in CSV or JSON format.

See http://docs.mongodb.org/manual/reference/program/mongoexport/ for more information.

general options:
            --help                                      查看帮助
            --version                                   查看工具版本并退出

verbosity options:
    -v, --verbose=<level>                               增加详细日志的等级 (通过多次-v选项来增加输出详细程度，例如-vvvvv)
            --quiet                                     以静默方式运行

connection options:
    -h, --host=<hostname>                               连接的主机名 
            --port=<port>                               连接对应的端口
(连接副本集：-h <replsetName>/<host1:port>,<host2:port>)

ssl options:
            --ssl                                       连接到启用了TSL/SSL支持的mongod或mongos
            --sslCAFile=<filename>                      指定.pem根证书文件
            --sslPEMKeyFile=<filename>                  指定.pem包含的TSL/SSL证书和密钥文件
            --sslPEMKeyPassword=<password>              指定解密证书密钥文件的密码
            --sslCRLFile=<filename>                     指定.pem包含证书吊销列表的文件
            --sslAllowInvalidCertificates               绕过服务器的证书检查，并允许使用无效证书
            --sslAllowInvalidHostnames                  禁用TSL/SSL证书中主机名的验证
            --sslFIPSMode                               使用已安装的OPENSSL库的FIPS模式

authentication options:
    -u, --username=<username>                           登录用户名
    -p, --password=<password>                           登录密码
            --authenticationDatabase=<database-name>    用户对应的认证数据库
            --authenticationMechanism=<mechanism>       验证机制，默认为SCRAM-SHA-1

namespace options:
    -d, --db=<database-name>                            指定数据库名称
    -c, --collection=<collection-name>                  指定集合名称

uri options:
            --uri=mongodb-uri                           URI连接串(mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]])

output options:
    -f, --fields=<field>[,<field>]*                     指定要包含在导出中的字段，使用逗号分割
            --fieldFile=<filename>                      将要包含的字段先记录在文件中，一行一个字段。仅对--type为CSV
            --type=<type>                               输出的文件类型(default: json)
    -o, --out=<filename>                                输出的文件。如果未指定则写入标准输出
            --jsonArray                                 将导出的内容写入单个JSON数组
            --pretty                                    格式美化输出
            --noHeaderLine                              导出类型为CSV格式时，忽略第一行字段名

querying options:
    -q, --query=<json>                                  查询条件
            --queryFile=<filename>                      JSON格式的查询文件
    -k, --slaveOk                                       设置从节点可读，默认为true
            --readPreference=<string>|<json>            设置读首选项，默认为primary
            --forceTableScan                            强制直接扫描数据存储而非_id索引遍历
            --skip=<count>                              跳过部分文档
            --limit=<count>                             指定导出的最大文档数
            --sort=<json>                               指定导出结果的排序。如果不存在支持排序的索引，则结果必须小于32M
```

**示例**

使用--fields选项，导出CSV格式的数据

```
mongoexport --db users --collection contacts --type = csv --fields name，address --out /opt/backups/contacts.csv
```

导出JSON格式

```
mongoexport --db sales --collection contacts --out contacts.json
```

通过身份验证导出远程数据

```
mongoexport --host mongodb1.example.net --port 37017 --username user --password "pass" -- collection contacts --db marketing --out mdb1-examplenet.json
```

导出查询结果

```
mongoexport --db sales --collection contacts --query '{"field"：1}'
```

### 1.2 mongoimport

```
[root@t-luhxdb01-p-szzb ~]# mongoimport --help
Usage:
    mongoimport <options> <file>

Import CSV, TSV or JSON data into MongoDB. If no file is provided, mongoimport reads from stdin.

See http://docs.mongodb.org/manual/reference/program/mongoimport/ for more information.

general options:
            --help                                      查看帮助
            --version                                   查看工具

verbosity options:
    -v, --verbose=<level>                               增加详细日志的等级 (通过多次-v选项来增加输出详细程度，例如-vvvvv)
            --quiet                                     静默的方式执行

connection options:
    -h, --host=<hostname>                               连接的主机名 
            --port=<port>                               连接对应的端口
(连接副本集：-h <replsetName>/<host1:port>,<host2:port>)

ssl options:
            --ssl                                       连接到启用了TSL/SSL支持的mongod或mongos
            --sslCAFile=<filename>                      指定.pem根证书文件
            --sslPEMKeyFile=<filename>                  指定.pem包含的TSL/SSL证书和密钥文件
            --sslPEMKeyPassword=<password>              指定解密证书密钥文件的密码
            --sslCRLFile=<filename>                     指定.pem包含证书吊销列表的文件
            --sslAllowInvalidCertificates               绕过服务器的证书检查，并允许使用无效证书
            --sslAllowInvalidHostnames                  禁用TSL/SSL证书中主机名的验证
            --sslFIPSMode                               使用已安装的OPENSSL库的FIPS模式

authentication options:
    -u, --username=<username>                           登录用户名
    -p, --password=<password>                           登录密码
            --authenticationDatabase=<database-name>    用户对应的认证数据库
            --authenticationMechanism=<mechanism>       验证机制，默认为SCRAM-SHA-1

namespace options:
    -d, --db=<database-name>                            指定数据库名称
    -c, --collection=<collection-name>                  指定集合名称
uri options:
            --uri=mongodb-uri                           URI连接串(mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]])

input options:
    -f, --fields=<field>[,<field>]*                     在导入的CSV中第一行不存在字段名时，用逗号分割字段名
            --fieldFile=<filename>                      在导入的CSV中第一行不存在字段名时，以文件的形式每行设置一个字段名
            --file=<filename>                           要导入的数据文件位置及名称
            --headerline                                CSV中第一行表示字段名
            --jsonArray                                 将多个文档导入单个JSON数组，仅限不大于16M
            --parseGrace=<grace>                        处理类型错误的方式 : autoCast, skipField, skipRow, stop (defaults to 'stop')
            --type=<type>                               指定要导入的文件类型(default: json)
            --columnsHaveTypes                          指定fields中字段对应的类型（auto()，binary(<arg>)，boolean()，data(<arg>)，data_go(arg)，data_ms(<arg>)，data_oracle(<arg>)，double()，int32()，int64()，string()）

ingest options:
            --drop                                      导入前删除对应集合
            --ignoreBlanks                              忽略CSV中的空白字段
            --maintainInsertionOrder                    按照导入文件中的顺序插入，否则会任意顺序插入
    -j, --numInsertionWorkers=<number>                  并行度(default: 1)
            --stopOnError                               在遇到错误时停止操作
            --mode=[insert|upsert|merge]                指定导入过程中如何文档冲突(defaults:insert,upset,merge)
            --upsertFields=<field>[,<field>]*           当使用upset或者merge时，以逗号分割字段
            --writeConcern=<write-concern-specifier>    写入数据库时的写入参数，等价于mongo shell中的-w参数。Defaults:majority
            --bypassDocumentValidation                  导入时绕过文档验证
```

**示例**

简单导入

```
mongoimport --db users --collection contacts --file contacts.json
```

导入期间合并匹配文档

```
DB:
    {
      "_id" : ObjectId("580100f4da893943d393e909"),
      "name" : "Crystal Duncan",
      "region" : "United States",
      "email" : "crystal@example.com"
    }
    
FILE:
    {
      "_id" : ObjectId("580100f4da893943d393e909"),
      "username" : "crystal",
      "email": "crystal.duncan@example.com",
      "likes" : [ "running", "pandas", "software development" ]
    }
    
Merge:
    mongoimport -c people -d example --mode merge --file people-20160927.json
    
Request:
    {
      "_id" : ObjectId("580100f4da893943d393e909"),
      "name" : "Crystal Duncan",
      "region" : "United States",
      "email" : "crystal.duncan@example.com",
      "username" : "crystal",
      "likes" : [
            "running",
            "pandas",
            "software development"
        ]
    }
```

使用指定字段类型导入

```
mongoimport --db users --collection contacts --type csv --columnsHaveTypes --fields "name.string（），birthdate.date（2006-01-02），contacts.boolean（），followerCount.int32（），user thumbnail.binary（base64）" - file /example/file.csv
```

忽略空白字段

```
mongoimport --db users --collection contacts --type csv --file /example/data.csv --ignoreBlanks
```



## 2. mongodump/mongorestore

### 2.1 mongodump

Mongodump是一个用于创建数据库内容的二进制导出程序，mongodump可以从任意mongod或mongos实例导出数据。Mongodump将排除local数据库的内容，在3.4中添加了对只读视图的支持，默认情况仅捕获视图的原数据，要在视图中捕获文档，请使用–viewsAsCollections

| 选项                | 说明                                                         |
| :------------------ | :----------------------------------------------------------- |
| --host              | 参数的选择会影响连接副本集时的读取首选项，如果以replset_name/host1:port1,host2:port2的方式连接则从primary中读取，如果以host1:port1,host2:port2的方式读取则从nearest读取 |
| --uri               | 指定可解析的URI连接字符串连接到MongoDB，–uri “mongodb：// [username：password @] host1 [：port1] [，host2 [：port2]，… [，hostN [：portN]]] [/ [database] [？options]]” |
| --port              | 连接端口，不能与URI共用                                      |
| --username          | 指定连接用户，与–password和–authenticationDatabase一起使用-- |
| --db                | 指定要备份的数据库，未指定则表示所有数据库                   |
| --collection        | 指定要备份的集合，未指定则表示所有集合                       |
| --queryFile         | 指定包含JSON文档的文件路径，该文件作为查询过滤器，用于限制mongodb输出的文档 |
| --readPreference    | 指定读取首选项，当连接到一个mongos或者副本集默认primary优先，否则nearest |
| --gzip              | 压缩输出                                                     |
| --out               | 指定mongodump输出文件的保存路径                              |
| --oplog             | 导出oplog集合，包含mongodump过程中的oplog记录                |
| --excludeColleciton | 排除指定集合，也可以使用–excludeCollectionsWithPrefix排除具有指定前缀的所有集合 |
| –archive            | 将输出写入单个存档文件或标准输出                             |

**示例**

导出指定条件的数据

```
mongodump --host 192.168.10.10 --port 27017 -u root -p  --authenticationDatabase=admin --db=test --collection=coll1 -q '{update_time:{$gte:ISODate("2021-12-01T02:00:00Z"),$lt:ISODate("2021-12-01T05:30:00Z")}}' --type=json  --out=coll1.json
```

过滤指定集合

```
mongodump  --db test --excludeCollection=users --excludeCollection=salaries
```

将存档输出到标准输出

```
mongodump --archive --db test --port 27017  | mongorestore --archive --port 27018
```

### 2.2 mongorestore

mongorestore是插入的形式，可以创建新的数据库也可以将数据追加到现有数据库，如果将文档还原到现有数据库并且集合和现有文档_id相同，则mongorestore不会覆盖这些文档

| 选项                              | 说明                                                         |
| :-------------------------------- | :----------------------------------------------------------- |
| –dir                              | 指定转储目录，不能同时指定path参数和–archive选项             |
| –archive                          | 从归档文件或标准输入（stdin）恢复                            |
| path                              | mongorestore命令的最后一个参数是目录路径。此参数指定要从中还原的数据库转储的位置 |
| –gzip                             | 要从包含压缩文件的转储目录进行还原，请mongorestore使用新–gzip选项运行 |
| –numInsertionWorkersPerCollection | 指定每个集合并发运行的插入工作器数                           |
| –noIndexRestore                   | 阻止mongorestore恢复和构建相应mongodump输出中指定的索引      |
| –oplogReplay                      | 还原数据库转储后，从bson文件重放oplog条目                    |
| –oplogLimit                       | 阻止mongorestore应用时间戳大于或等于的oplog条目，与–oplogReplay选项一起使用 |
| –drop                             | 在从转储的备份还原集合之前，从目标数据库中删除集合           |
| –nsFrom                           | 用于在还原操作期间配合–nsTo选项重命名命名空间。–nsFrom指定转储文件中的集合，同时–nsTo指定应在已还原数据库中使用的名称 |
| –nsTo                             | 用于在还原操作期间配合–nsFrom选项重命名命名空间。–nsTo指定要在还原的数据库中使用的新集合名称，同时 –nsFrom指定转储文件中的名称 |
| –nsExclude                        | 从还原操作中排除指定的命名空间                               |
| –nsInclude                        | 仅包含还原操作中指定的命名空间。通过允许您指定要还原的多个集合， –nsInclude提供该–collection选项功能的超集 |

**示例**

恢复集合备份

```
mongorestore --host 192.168.10.10 --port 27017 -u root -p 'xxx' --authenticationDatabase=admin --db=target --collection=coll1 --file coll1.json
```

使用–nsInclude指定恢复transactions数据库的集合，同时使用–nsExclude排除名称以_dev结尾的集合

```
mongorestore --nsInclude 'transactions.*' --nsExclude 'transactions.*_dev' dump/
```

例如data数据库包含以下集合：

- Sales_customer1
- Sales_customer2
- Sales_customer3
- Users_customer1
- Users_customer2
- Users_customer3

现在要求将data数据库下的sales_<customerName>表恢复到customer数据库的sales集合，并将users_<customerName>恢复到customer数据库的users集合

```
mongorestore --nsInclude 'data.*' --nsFrom 'data.$prefix$_$customer$' --nsTo '$customer$.$prefix$'
```

从压缩归档数据恢复

```
mongorestore --gzip --archive = test.20150715.gz --db test
```

### 2.3 结合Oplog进行备份恢复

在利用mongodump导出数据时，如果不事先锁定，dump中的集合时间点将会不一致，比如A集合10点开始备份，B集合10点半才开始备份。这时可以设置–oplog选项来同步导出dump过程中产生的oplog信息。该选项将生成一个oplog.bson的文件，该文件记录了dump过程中生成的所有oplog，以实现基于dump结束时间点的一致性备份

插入测试数据

```
for(var i=0;i<10000;i++){db.resource.insert({a:i});};
```

同步进行dump备份

```
mongodump -h 10.0.139.162 --port 30000  --oplog  -o /service/mongodb/backup/
```

模拟删除

```
db.resource.remove({});
```

备份oplog

```
mongodump -h 10.0.139.162 --port 30000 -d local -c oplog.rs -o /service/mongodb/backup/oplog
```

恢复dump备份

```
mongorestore -h 10.0.139.162 --port 30000 --oplogReplay /service/mongodb/backup/
```

从oplog备份中查找故障时间点

```
[root@t-luhx02-v-szzb local]# bsondump oplog.rs.bson |egrep "\"op\":\"d\"\,\"ns\":\"test\.resource\"" | head -1
{"ts":{"$timestamp":{"t":1585894391,"i":1}},"t":{"$numberLong":"1"},"h":{"$numberLong":"-8046849749987056723"},"v":2,"op":"d","ns":"test.resource","ui":{"$binary":"U/2ma7QARICh+N4MDWjVdQ==","$type":"04"},"wall":{"$date":"2020-04-03T06:13:11.597Z"},"o":{"_id":{"$oid":"5e86d362d291bceb1eec05d9"}}}
```

恢复到故障之前

```
mongorestore -h 10.0.139.162 --port 30000 --oplogReplay --oplogLimit "1585894391:1" /service/mongodb/backup/
```

查询恢复数据

```
single:PRIMARY> db.resource.find({}).count();
10000
```



## 3. 基于LVM快照备份

在MongoDB 3.2之前，使用WiredTiger创建MongoDB实例的卷级备份要求数据文件和日志驻留在同一卷上

在MongoDB 3.2中添加了对MongoDB实例的数据文件和日志文件不在一个卷上的卷级备份支持，但是要创建一致的备份，需要锁定数据库，停止对数据库的写入

快照发生时，数据库必须有效。这意味着数据库接受的所有写入都要完全写入磁盘，如果备份发生时写入不在磁盘，则备份不会反映这些更改。对于WiredTiger存储引擎，数据文件反映了上一个检查点的状态。检查点在每2G数据或每分钟发生一次

快照会创建整个磁盘的镜像，除非你需要备份整个系统，否则请考虑将MongoDB的数据文件、日志文件、配置文件放在专用存储设备上。

**创建LVM快照**

```
lvcreate --size 100M --snapshot --name mdb-snap01 /dev/vg0/mongodb
```

其中size并不反应磁盘上数据的总量，而是反应当前状态和创建快照之间的差异数量。应当设置足够的大小来确保数据的增长，如果快照空间不足，则快照映像将变为不可用，需要丢弃该快照重新创建

**存档快照**

创建快照后将快照存入单独的存储设备中，并进行压缩

```
umount / dev / vg0 / mdb-snap01
dd if = / dev / vg0 / mdb-snap01 | gzip> mdb-snap01.gz
```

**恢复快照**

要还原使用LVM创建的快照，请发出以下命令序列：

```
lvcreate --size 1G --name mdb-new vg0
gzip -d -c mdb-snap01.gz | dd of=/dev/vg0/mdb-new
mount /dev/vg0/mdb-new /srv/mongodb
```

**直接从快照恢复**

要在不写入压缩gz文件的情况下还原备份，请使用以下命令序列：

```
umount /dev/vg0/mdb-snap01
lvcreate --size 1G --name mdb-new vg0
dd if=/dev/vg0/mdb-snap01 of=/dev/vg0/mdb-new
mount /dev/vg0/mdb-new /srv/mongodb
```

**远程备份存储**

您可以使用组合流程和SSH 实施系统外备份

```
umount /dev/vg0/mdb-snap01
dd if=/dev/vg0/mdb-snap01 | ssh username@example.com gzip > /opt/backup/mdb-snap01.gz
lvcreate --size 1G --name mdb-new vg0
ssh username@example.com gzip -d -c /opt/backup/mdb-snap01.gz | dd of=/dev/vg0/mdb-new
mount /dev/vg0/mdb-new /srv/mongodb
```

[backup sharded cluster with filesystem snapshots]:https://docs.mongodb.com/manual/tutorial/backup-sharded-cluster-with-filesystem-snapshots/



## 4. Percona Backup For MongoDB

### 4.1 概述

早前，在之前的文章中提到Percona版本的MongoDB支持热备份，但那时还是集成的。现在Percona已经将该功能独立出来，让你可以在社区版的MongoDB replset或集群上进行备份，支持MongoDB3.6或更高版本。

![Architecture](https://gitee.com/dba_one/wiki_images/raw/master/images/pbm-architecture.png)

Percona Backup For MongoDB的组件如下：

- pbm-agent：运行在集群或副本集每个节点上的进程，用于执行备份或恢复操作
- pbm：pbm是向pbm-agent下达操作指令的程序
- pbm control collections：控制集合用于存储配置数据和备份状态，pbm和pbm-agent都通过控制集合来检查备份状态并相互通信
- remote backup storage：远程备份存储是保存备份文件的位置，可以是S3存储，也可以是Filesystem

**pbm-agent**

备份需要本地连接一个pbm-agent实例到每个Mongod实例，这包括副本集的secondary节点和分片群集中的配置服务器副本节点。

当pbm-agent观察到pbm对控制集合的更新后，将触发备份和恢复操作，类似副本集选择primary节点的方法，pbm-agent在副本集选择一个节点执行备份或恢复的操作。

**pbm**

pbm可以通过一系列子命令来管理备份

```
$ pbm help
usage: pbm [<flags>] <command> [<args> ...]

Percona Backup for MongoDB

Flags:
  --help                     Show context-sensitive help (also try --help-long
                             and --help-man).
  --mongodb-uri=MONGODB-URI  MongoDB connection string. Default value read from
                             environment variable PBM_MONGODB_URI.

Commands:
  help [<command>...]
    Show help.

  config [<flags>]
    Set, change or list the config

  backup
    Make backup

  restore <backup_name>
    Restore backup

  delete-backup <backup_name> [<flags>]
    Delete backup(s)

  cancel-backup
    Cancel backup

  list [<flags>]
    Backup list

  version [<flags>]
    PBM version info
```

pbm将配置值保存在控制集合中，通过更新和读取操作、日志等控制集合来启动和监视备份或恢复操作

**pbm control collections**

备份和配置的状态保存在MongoDB集群的集合或复制集中，他们主要保存在副本集或分片配置服务的admin数据库中

- admin.pbmConfig
- admin.pbmCmd(Used to define and trigger operations)
- admin.pbmLock(pbm-agent synchronization-lock structure)
- admin.pbmBackup (Log / status of each backup)

**remote backup storage**

如果远程备份存储设置为filesystem类型，可以通过pbm list命令查看备份集。其中的备份文件名称都是以UTC备份开始时间作为前缀，每个备份都有一个元数据文件。对于备份中的每个副本集：

- 有一个mongodump格式的压缩归档文件，它是集合的转储
- 覆盖备份时间的oplog的BSON文件转储

oplog备份的结束时间是备份快照的数据一致时间点



### 4.2 安装配置

下载介质

[DownLoad Percona Backup For MongoDB](https://www.percona.com/downloads/percona-backup-mongodb/percona-backup-mongodb-1.3.4/binary/redhat/7/x86_64/percona-backup-mongodb-1.3.4-1.el7.x86_64.rpm)

安装介质

```
[root@t-luhx01-v-szzb ~]# rpm -ivh percona-backup-mongodb-1.3.4-1.el7.x86_64.rpm
```

创建登陆用户

```
db.getSiblingDB("admin").createRole({ "role": "pbmAnyAction",
      "privileges": [
         { "resource": { "anyResource": true },
           "actions": [ "anyAction" ]
         }
      ],
      "roles": []
   });
db.getSiblingDB("admin").createUser({user: "pbmuser",
       "pwd": "secretpwd",
       "roles" : [
          { "db" : "admin", "role" : "readWrite", "collection": "" },
          { "db" : "admin", "role" : "backup" },
          { "db" : "admin", "role" : "clusterMonitor" },
          { "db" : "admin", "role" : "restore" },
          { "db" : "admin", "role" : "pbmAnyAction" }
       ]
    });
```

编辑配置文件

```
[root@t-luhx01-v-szzb ~]# cat /etc/pbm-storage.conf 
storage:
  type: filesystem
  filesystem:
    path: /service/backup
```

初始化配置(分片选择Config节点执行)

```
[root@t-luhx01-v-szzb ~]# pbm config --file /etc/pbm-storage.conf --mongodb-uri "mongodb://pbmuser:secretpwd@10.0.139.161:20000,10.0.139.162:20000/?replicaSet=shard1"
```

> 如果想要修改配置可以执行pbm config –set修改单个配置值



### 4.3 备份恢复

**创建备份**

在完成前期配置后，就可以进行数据备份了，在执行备份之前需要先启动pbm-agent。分片集群中shard节点和config节点都需要启动，如果一台服务器有多个节点，应该启动多个pbm-agent一一对应。

```
[root@t-luhx01-v-szzb ~]# nohup pbm-agent --mongodb-uri "mongodb://pbmuser:secretpwd@10.0.139.161:20000/" > /service/backup/logs/pbm-agent.$(hostname -s).20000.log 2>&1 &
```

启动pbm-agent之后就可以执行pbm backup启动备份了，在分片集群中，每个分片节点和Config节点都有pbm-agent进程直接将备份快照和oplog写到远程存储中，Mongos不参与备份。

```
[root@t-luhx01-v-szzb ~]# pbm backup --mongodb-uri "mongodb://pbmuser:secretpwd@10.0.139.161:20000,10.0.139.162:20000/?replicaSet=shard1" --compression=gzip
```

![pbm-backup-shard](https://gitee.com/dba_one/wiki_images/raw/master/images/pbm-backup-shard.png)

*Tips：备份进度可以在pbm-agent日志中查看*

备份完成后，可以查看备份集

```
[root@t-luhx01-v-szzb ~]# pbm list --mongodb-uri "mongodb://pbmuser:secretpwd@localhost:20000"
Backup history:
  2020-11-25T14:44:33Z
```

如果备份过程中想要取消任务，可以执行`pbm cancel-backup`取消当前正在运行的任务

```
[root@t-luhx01-v-szzb ~]# pbm cancel-backup
Backup cancellation has started
```

鉴于备份存储资源有限，需要清理历史备份文件，可以删除指定备份集或超过指定时间的备份集

```
[root@t-luhx01-v-szzb ~]# pbm delete-backup 2020-04-20T13:45:59Z
```

要删除指定时间前的备份，可以指定–older-than参数，并传递以下格式的时间戳

- %Y-%M-%DT%H:%M:%S (e.g. 2020-04-20T13:13:20)
- %Y-%M-%D (e.g. 2020-04-20)

**恢复备份**

要还原pbm backup的备份集，可以通过pbm store命令并提供需要还原的备份时间戳。在还原之前还需要注意以下几点：

- 从1.x版本开始，Percona Backup For MongoDB复制了Mongodump的行为，还原时只清理备份中包含的集合，对于备份之后，还原之前创建的集合不进行清理，需要在还原前手动执行db.dropDatabase()清理

- 在恢复运行过程中，阻止客户端访问数据库

- 如果启用了Point-in-Time Recovery，需要提前禁用，因为Point-in-Time Recovery和还原是不兼容的操作

  ```
  $ pbm restore 2019-06-09T07:03:50Z
  ```

  为避免恢复期间pbm-agent内存消耗，V1.3.2可以针对恢复在配置文件设置下列参数。

  ```
  restore:
    batchSize: 500
    numInsertionWorkers: 10
  ```

在恢复分片集群前，需要进行下列配置：

1. 停止分片balance
2. 关闭所有mongos节点来阻止客户端连接
3. 禁用Point-in-Time Recovery

需要注意的是，分片备份只能还原到分片集群中，在恢复期间，pbm-agent将数据写入集群primary节点，下列是分片集群恢复流程

![pbm-restore-shard](https://www.percona.com/doc/percona-backup-mongodb/_images/pbm-restore-shard.png)

有时我们希望异机恢复，将备份还原到一个新的环境中，目标环境要注意以下要点：

- 目标集群中的副本集名称和备份的副本集名称必须相同
- 新环境中的PBM配置必须指定为原始环境定义的同一远程存储，一旦通过pbm list查看到了原始环境中的备份，就可以运行pbm restore命令恢复

### 4.4 Point-in-Time Recovery

Point-in-Time Recovery可以将数据库还原到指定时间点，期间会从备份快照中恢复数据库，并重放oplog到指定时间点。Point-in-Time Recovery是v1.3.0加入的，需要启用pitr.enabled参数

```
[root@t-luhx01-v-szzb ~]# pbm config --set pitr.enabled=true
```

在启用Point-in-Time Recovery之后，pbm-agent会定期保存oplog片，一个片包含10分钟跨度的oplog事件，如果禁用时间点恢复或因备份快照操作的开始而中断，则时间可能会更短。oplog保存在远程存储的pbmPitr子目录中，片的名称反映开始时间和结束时间。

pbm list包括备份快照和恢复的有效时间范围。它还显示时间点恢复状态。

```
[root@t-luhx01-v-szzb ~]# pbm list

Backup snapshots:
     2020-07-10T12:19:10Z
     2020-07-14T10:44:44Z
     2020-07-14T14:26:20Z
     2020-07-17T16:46:59Z
PITR <on>:
     2020-07-14T14:26:40 - 2020-07-16T17:27:26
     2020-07-17T16:47:20 - 2020-07-17T16:57:55
```

还原和时间点恢复增量备份是不能同时运行，在还原数据库前需要先禁用Point-in-Time Recovery，再运行pbm restore并指定有效时间范围内的时间戳

```
$ pbm config --set pitr.enabled=false
$ pbm restore --time="2020-07-14T14:27:04"
```

在我们执行完恢复后，会修改oplog的时间线。因此，在还原指定时间戳之后和上一次备份前的所有日志都无效，应该手动发起一次新的备份

```
$ pbm backup
$ pbm config --set pitr.enabled=true
```

当我们手动删除备份时，基于此备份对应的oplog片也会一起删除。当启用时间点恢复时，与之相关的最新备份快照和oplog片将不会被删除。

### 4.5 备份性能

Percona针对MongoDB备份程序提供了一个pbm-speed-test工具，可以测试压缩和备份上传速度，用于分析备份性能

```
$ /usr/bin/pbm-speed-test --help
usage: pbm-speed-test --mongodb-uri=MONGODB-URI [<flags>] <command> [<args> ...]

Percona Backup for MongoDB compression and upload speed test

Flags:
      --help                     Show context-sensitive help (also try
                                 --help-long and --help-man).
      --mongodb-uri=MONGODB-URI  MongoDB connection string
  -c, --sample-collection=SAMPLE-COLLECTION
                                 Set collection as the data source
  -s, --size-gb=SIZE-GB          Set data size in GB. Default 1
      --compression=s2           Compression type
                                 <none>/<gzip>/<snappy>/<lz4>/<s2>/<pgzip>

Commands:
  help [<command>...]
    Show help.

  compression
    Run compression test

  storage
    Run storage test

  version [<flags>]
    PBM version info
```

压缩测试

```
$ pbm-speed-test compression --compression=s2 --size-gb 10 --mongodb-uri "mongodb://root:Abcd123#@localhost:20000"
Test started ....
10.00GB sent in 8s.
Avg upload rate = 1217.13MB/s.
```

备份速度测试

```
$ pbm-speed-test storage --compression=s2 --mongodb-uri "mongodb://root:Abcd123#@localhost:20000"
Test started
1.00GB sent in 1s.
Avg upload rate = 1744.43MB/s.
```

pbm-speed-test默认采用半随机数据进行测试，如果要基于现有集合进行测试，请设置--sample-collection选项



[Percona Backup for MongoDB Documentation]:https://www.percona.com/doc/percona-backup-mongodb/index.html
[Back Up and Restore with Filesystem Snapshots]:https://docs.mongodb.com/manual/tutorial/backup-with-filesystem-snapshots/index.html
[Mongodump]:https://docs.mongodb.com/manual/reference/program/mongodump/index.html
[Mongorestore]:https://docs.mongodb.com/manual/reference/program/mongorestore/index.html


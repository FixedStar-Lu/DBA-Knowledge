---
title: 利用EBS Snapshot备份恢复MongoDB Cluster
date: 2021-10-18 15:30:30
updated: 2021-10-18 15:30:30
tags: aws,mongodb
categories: aws
keywords: ebs,snapshot,mongodb
description: 
top_img: 
comments: 
cover: https://mongodb-documentation.readthedocs.io/en/latest/_images/ec2-backup-and-restore-backup.jpg
toc: true
toc_number: 
copyright: 
copyright_author: luhengxing
copyright_author_href: 
copyright_url: 
copyright_info: 
mathjax: 
katex: 
aplayer: 
highlight_shrink: 
aside: true
---

[TOC]

# EBS Snapshot备份恢复MongoDB Cluster

## 1. EBS Snapshot概述

EBS Snapshot属于增量备份，也就是只保存设备在最新快照之后更改的数据块，由于不需要复制整体数据，将大大缩减快照的时间和增量存储的成本。例如：

- 在状态 1 中，该卷具有 `10 GiB` 数据。因为**快照 A** 是为该卷制作的首个快照，因此必须复制所有 `10 GiB` 数据。
- 在状态 2 中，该卷仍包含 `10 GiB` 数据，但是，`4 GiB` 数据已更改。**快照 B** 只需复制并存储制作**快照 A** 后更改的 `4 GiB` 数据。未更改的其他 `6 GiB` 数据（已复制并存储在**快照 A** 中）将由**快照 B**引用，而不是再次复制。这通过虚线箭头指示。
- 在状态 3 中，`2 GiB` 数据已添加到该卷中，共计 `12 GiB` 数据。**快照 C** 需要复制制作**快照 B** 之后添加的 `2 GiB` 数据。如虚线箭头所示，**快照 C** 还引用了存储在**快照 B** 中的 `4 GiB` 数据和存储在**快照 A** 中的 `6 GiB` 数据。
- 三个快照共需 `16 GiB` 存储空间。

![snapshot](https://gitee.com/dba_one/wiki_images/raw/master/images/snapshot_1a.png)

快照是基于时间点异步创建的，在完成之前，其状态为**pending**，内容为快照创建指令前已经写入磁盘的数据，不包含任何缓存数据。不受当前正在读写的操作影响，也就是获取EBS卷快照时间点的一致性数据。



## 2. 备份MongoDB Cluster

### 2.1 前提条件

- 开启journal log，确保每次写入都有对应的日志可用于数据恢复。如果未开启，则在备份前需要锁定数据库，禁止写入
- Journal log和dbpath处于同一个EBS卷，否则需要执行db.runCommand({fsync:1,lock:1})锁定数据库
- 如果dbpath包含多个EBS卷，也建议锁定数据库进行备份

### 2.2 全量备份

AWS控制台进入EC2 -> EBS -> 快照，点击创建快照

![image-20211018165514671](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20211018165514671.png)

> AWS API：
>
> aws ec2 create-snapshot --volume-id vol-1234567890abcdef0 --description “EBS Snapshot at 20211018”



### 2.3 增量备份

增量备份主要包含两种方式，一种是利用EBS Snapshot的增量特性，再次执行创建snapshot，基于全量快照做增量快照。另一种则是利用mongodb自身的oplog来实现增量备份。

```
$ mongodump -h 192.168.10.100 --port 21000 --authenticationDatabase=admin -d local -c oplog.rs -o  /data/backup/config/oplog
$ mongodump -h 192.168.10.101 --port 22001 --authenticationDatabase=admin -d local -c oplog.rs -o  /data/backup/shard1/oplog
$ mongodump -h 192.168.10.102 --port 22001 --authenticationDatabase=admin -d local -c oplog.rs -o  /data/backup/shard2/oplog
$ mongodump -h 192.168.10.103 --port 22001 --authenticationDatabase=admin -d local -c oplog.rs -o  /data/backup/shard3/oplog
```

## 3. 恢复MongoDB Cluster

### 3.1 快照恢复

快照恢复有两种方式，一种是将快照恢复成EBS卷，另一种是将快照恢复成AMI镜像。这里我们采用恢复成EBS卷的方式，再映射挂载到一台已经安装好mongodb环境的服务器上。当基于快照创建EBS卷时，复制的卷将在后台加载数据，立即就能开始使用数据，如果访问到未加载的数据，则卷将直接从S3下载请求数据，然后继续在后台加载剩余数据。

通过搜索定位到之前的快照，点击操作 -> 创建卷`(需要注意的是，EBS卷和EC2需要处于同一个region下)`

![image-20211018170301661](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20211018170301661.png)

> AWS API：
>
> aws ec2 create-volume --availability-zone eu-west-1c --snapshot snap-0dbb9f547f65ea87f



通过搜索定位到上一步创建的EBS卷，点击操作 -> 连接卷，设置对应的EC2信息

> AWS API：
>
> aws ec2 attach-volume --instance i-01ed8b12694765644 --device /dev/sdf



操作系统挂载卷到数据目录

```
$ yum install install xfsprogs
$ mount -t xfs /dev/xvdf /data
```



### 3.2 MongoDB配置

编辑config配置文件

```
dbpath=/data/mongodb/config/data
logpath=/data/mongodb/config/log/configsrv.log
pidfilepath=/data/mongodb/congsrv.pid 
directoryperdb=true
replSet= conf_repset
logappend=true 
configsvr=true 
bind_ip=192.168.10.200
port=21000 
oplogSize=10000 
fork=true 
noprealloc=true
auth=true
keyFile=/etc/keyFilers0.key
```

生成keyFile

```
$ openssl rand -base64 756 > /service/mongodb/keys/rsa_key
$ chmod 600 /service/mongodb/key/rsa_key
```

启动config replset

```
$ mongod --config /etc/config.conf
```

重置config成员

```
$ mongo mongodb://root:Abcd12345#@192.168.10.200:21000
> config = {_id:'conf_repset',"protocolVersion" : 1,configsvr:true,members: [
 {_id:0,host:"192.168.10.200:21000",priority:1}]
}
> rs.reconfig(config,{force:true})
```

编辑shard节点配置文件(注释参数replSet和shardsvr)

```
dbpath=/data/mongodb/shard1/data
logpath=/data/mongodb/shard1/log/shard1.log
pidfilepath=/data/mongodb/shard1.pid 
directoryperdb=true
bind_ip=192.168.10.200
logappend=true
logRotate=rename
#shardsvr=true 
#replSet=repset1
port=22001 
oplogSize=10000 
fork=true 
noprealloc=true
wiredTigerCacheSizeGB=5
auth=true
keyFile=/etc/keyFilers0.key
```

启动shard节点

```
$ mongod --config /etc/shard1.conf
```

创建临时用户

```
> use admin

> db.createUser({user:"backup_user","pwd":"Abcd12345#",roles:["__system"]})
> db.auth("backup_user","Abcd12345#")
```

删除shard节点上的local库

```
> use local
> db.dropDatabase()
```

从admin.system.version集合删除minOpTimeRecovery 

```
> use admin
> db.system.version.deleteOne( { _id: "minOpTimeRecovery" } )
```

更新shard元数据标识(config节点信息)

```
> db.system.version.find( {"_id" : "shardIdentity" } )
> db.system.version.update( {"_id" : "shardIdentity" },{$set:{configsvrConnectionString:"conf_repset/192.168.10.200:21000"}} )
```

取消参数注释并重启，然后执行下列初始化命令

```
> rs.initiate()
```

删除临时用户

```
> db.removeUser("backup_user")
```

启动mongos，并更新shard节点信息

```
$ mongos --config /etc/mongos.conf
> use config
> db.shards.update( { "_id" : "shard1" }, { $set: { "host": "repset1/192.168.10.200:22001" } }, { multi: true })
> db.shards.update( { "_id" : "shard2" }, { $set: { "host": "repset2/192.168.10.200:22002" } }, { multi: true })
> db.shards.update( { "_id" : "shard3" }, { $set: { "host": "repset3/192.168.10.200:22003" } }, { multi: true })
```



## 4. 附录

### 4.1 生命周期管理

EC2支持通过EBS生命周期管理器来创建EBS快照计划，定时对指定卷进行备份。

1. 配置基本信息

   ![image-20211026161616686](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20211026161616686.png)

2. 配置计划信息

   ![image-20211026161711839](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20211026161711839.png)



### 4.2 恢复oplog

在恢复前，可以根据实际需求查询要恢复的时间点

```
$ bsondump oplog.rs.bson |egrep "\"op\":\"d\"\,\"ns\":\"test\.users\"" |head -1 

```

对oplog进行恢复

```
$ mongorestore -h 192.168.10.200:22001 -u backup_user -p --oplogReplay --oplogLimit "1634551765:1" /data/backup/shard1/oplog
```

### 4.3 参考链接

[Restore a Sharded Cluster — MongoDB Manual](https://docs.mongodb.com/manual/tutorial/restore-sharded-cluster/)

[Amazon EBS 快照 - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/EBSSnapshots.html)


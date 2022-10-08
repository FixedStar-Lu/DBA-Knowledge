# MySQL8：replset cluster

**安装插件**

```
root@(none) 15:32:  INSTALL PLUGIN Rewriter SONAME 'rewriter.so';
root@(none) 15:07:  INSTALL PLUGIN clone SONAME 'mysql_clone.so';

root@(none) 15:32:  SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME LIKE 'clone';
+-------------+---------------+
| PLUGIN_NAME | PLUGIN_STATUS |
+-------------+---------------+
| clone       | ACTIVE        |
+-------------+---------------+

root@(none) 15:32:  SELECT PLUGIN_NAME, PLUGIN_STATUS FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME LIKE 'Rewriter';
+-------------+---------------+
| PLUGIN_NAME | PLUGIN_STATUS |
+-------------+---------------+
| Rewriter    | ACTIVE        |
+-------------+---------------+
```

**创建副本集**

```
[root@t-luhx03-v-szzb mysql]# mysqlsh
MySQL Shell 8.0.19

Copyright (c) 2016, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.

MySQL  JS > \connect root@10.0.139.161:33006
MySQL  10.0.139.161:33006 ssl  JS > var rs = dba.createReplicaSet("replset1")
A new replicaset with instance 't-luhx01-v-szzb:33006' will be created.

* Checking MySQL instance at t-luhx01-v-szzb:33006

This instance reports its own address as t-luhx01-v-szzb:33006
t-luhx01-v-szzb:33006: Instance configuration is suitable.

* Updating metadata...

ReplicaSet object successfully created for t-luhx01-v-szzb:33006.
Use rs.addInstance() to add more asynchronously replicated instances to this replicaset and rs.status() to check its status.

MySQL  10.0.139.161:33006 ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "replset1", 
        "primary": "t-luhx01-v-szzb:33006", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "t-luhx01-v-szzb:33006": {
                "address": "t-luhx01-v-szzb:33006", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}
```

**添加节点**

```
 MySQL  10.0.139.161:33006 ssl  JS > rs.addInstance('root@10.0.139.163:33006');
Adding instance to the replicaset...

* Performing validation checks

This instance reports its own address as t-luhx01-v-szzb:33006
t-luhx01-v-szzb:33006: Instance configuration is suitable.

* Checking async replication topology...

* Checking transaction state of the instance...

WARNING: A GTID set check of the MySQL instance at 't-luhx01-v-szzb:33006' determined that it contains transactions that do not originate from the replicaset, which must be discarded before it can join the replicaset.

t-luhx01-v-szzb:33006 has the following errant GTIDs that do not exist in the replicaset:
1fbe2d9e-1673-11ea-92f5-005056abc9c9:1-189125

WARNING: Discarding these extra GTID events can either be done manually or by completely overwriting the state of t-luhx01-v-szzb:33006 with a physical snapshot from an existing replicaset member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

Having extra GTID events is not expected, and it is recommended to investigate this further and ensure that the data can be removed prior to choosing the clone recovery method.

Please select a recovery method [C]lone/[A]bort (default Abort): C
* Updating topology
Waiting for clone process of the new member to complete. Press ^C to abort the operation.
* Waiting for clone to finish...
NOTE: t-luhx01-v-szzb:33006 is being cloned from t-luhx01-v-szzb:33006
** Stage DROP DATA: Completed
** Clone Transfer  
    FILE COPY  ############################################################  100%  Completed
    PAGE COPY  ############################################################  100%  Completed
    REDO COPY  ############################################################  100%  Completed

NOTE: t-luhx01-v-szzb:33006 is shutting down...

* Waiting for server restart... ready
* t-luhx01-v-szzb:33006 has restarted, waiting for clone to finish...
** Stage RESTART: Completed
* Clone process has finished: 1.66 GB transferred in 16 sec (103.59 MB/s)

** Configuring t-luhx01-v-szzb:33006 to replicate from t-luhx01-v-szzb:33006
** Waiting for new instance to synchronize with PRIMARY...

The instance 't-luhx01-v-szzb:33006' was added to the replicaset and is replicating from t-luhx01-v-szzb:33006.
```

**查看副本集状态**

```
 MySQL  10.0.139.161:33006 ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "replset1", 
        "primary": "t-luhx01-v-szzb:33006", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "t-luhx01-v-szzb:33006": {
                "address": "t-luhx01-v-szzb:33006", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }, 
            "t-luhx02-v-szzb:33006": {
                "address": "t-luhx02-v-szzb:33006", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for master to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }, 
            "t-luhx03-v-szzb:33006": {
                "address": "t-luhx03-v-szzb:33006", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for master to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}
```

**主从切换测试**

```
 MySQL  10.0.139.163:33006 ssl  JS > rs.setPrimaryInstance('root@10.0.139.163:33006')
t-luhx03-v-szzb:33006 will be promoted to PRIMARY of 'replset1'.
The current PRIMARY is t-luhx01-v-szzb:33006.

* Connecting to replicaset instances
** Connecting to t-luhx01-v-szzb:33006
** Connecting to t-luhx02-v-szzb:33006
** Connecting to t-luhx03-v-szzb:33006
** Connecting to t-luhx01-v-szzb:33006
** Connecting to t-luhx02-v-szzb:33006
** Connecting to t-luhx03-v-szzb:33006

* Performing validation checks
** Checking async replication topology...
** Checking transaction state of the instance...

* Synchronizing transaction backlog at t-luhx03-v-szzb:33006

* Updating metadata

* Acquiring locks in replicaset instances
** Pre-synchronizing SECONDARIES
** Acquiring global lock at PRIMARY
** Acquiring global lock at SECONDARIES

* Updating replication topology
** Configuring t-luhx01-v-szzb:33006 to replicate from t-luhx03-v-szzb:33006
** Changing replication source of t-luhx02-v-szzb:33006 to t-luhx03-v-szzb:33006

t-luhx03-v-szzb:33006 was promoted to PRIMARY.

 MySQL  10.0.139.163:33006 ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "replset1", 
        "primary": "t-luhx03-v-szzb:33006", 
        "status": "AVAILABLE", 
        "statusText": "All instances available.", 
        "topology": {
            "t-luhx01-v-szzb:33006": {
                "address": "t-luhx01-v-szzb:33006", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for master to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }, 
            "t-luhx02-v-szzb:33006": {
                "address": "t-luhx02-v-szzb:33006", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for master to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }, 
            "t-luhx03-v-szzb:33006": {
                "address": "t-luhx03-v-szzb:33006", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }
        }, 
        "type": "ASYNC"
    }
}
```

**故障转移测试**

Innodb Replset并不支持自动故障转移，如果主库宕机，Replset成为只读环境，可以通过forcePrimaryInstance()强制某个节点成为新的primary，需要注意的是仅在主库无法恢复的情况下使用该方式。

主库宕机后的状态

```
 MySQL  10.0.139.161:33006 ssl  JS > rs.status()
ERROR: Unable to connect to the PRIMARY of the replicaset replset1: t-luhx03-v-szzb:33006: Can't connect to MySQL server on 't-luhx03-v-szzb' (111)
Cluster change operations will not be possible unless the PRIMARY can be reached.
If the PRIMARY is unavailable, you must either repair it or perform a forced failover.
See \help forcePrimaryInstance for more information.
WARNING: MYSQLSH 51118: PRIMARY instance is unavailable
```

强制将10.0.139.161升级为primary

```
 MySQL  10.0.139.161:33006 ssl  JS > rs.forcePrimaryInstance('root@10.0.139.161:33006')
 MySQL  10.0.139.161:33006 ssl  JS > rs.status()
{
    "replicaSet": {
        "name": "replset1", 
        "primary": "t-luhx01-v-szzb:33006", 
        "status": "AVAILABLE_PARTIAL", 
        "statusText": "The PRIMARY instance is available, but one or more SECONDARY instances are not.", 
        "topology": {
            "t-luhx01-v-szzb:33006": {
                "address": "t-luhx01-v-szzb:33006", 
                "instanceRole": "PRIMARY", 
                "mode": "R/W", 
                "status": "ONLINE"
            }, 
            "t-luhx02-v-szzb:33006": {
                "address": "t-luhx02-v-szzb:33006", 
                "instanceRole": "SECONDARY", 
                "mode": "R/O", 
                "replication": {
                    "applierStatus": "APPLIED_ALL", 
                    "applierThreadState": "Waiting for an event from Coordinator", 
                    "applierWorkerThreads": 4, 
                    "receiverStatus": "ON", 
                    "receiverThreadState": "Waiting for master to send event", 
                    "replicationLag": null
                }, 
                "status": "ONLINE"
            }, 
            "t-luhx03-v-szzb:33006": {
                "address": "t-luhx03-v-szzb:33006", 
                "connectError": "t-luhx03-v-szzb:33006: Can't connect to MySQL server on 't-luhx03-v-szzb' (111)", 
                "fenced": null, 
                "instanceRole": null, 
                "mode": null, 
                "status": "INVALIDATED"
            }
        }, 
        "type": "ASYNC"
    }
}
```

当163恢复之后会自动强制成为主节点，但是其它从节点就纷纷报错了，期间的数据就会丢失，因此建议非极端情况下，不应执行force强制切换，如果已经牵制切换了，建议在主库mysql_innodb_cluster_metadata.instances表中删除该节点信息


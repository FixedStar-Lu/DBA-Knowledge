# MySQL8 : Asynchronous Replication Automatic failover

MySQL8.0支持在一个复制通道上配置多个可用复制源，当某个复制源不可用时，自动根据权重重新选择数据源进行同步。例如我们本地有一个跨机房的MGR高可用环境，计划在另一个地区的机房搭建一个异步复制节点做容灾。当MGR主节点发生切换时，就可以通过该特性在新主节点继续进行复制。

**开启source_connection_auto_failover选项**

```
change master to master_user='msandbox',
master_password='msandbox',
master_host='127.0.0.1',
master_auto_position=1,
source_connection_auto_failover=1,
master_port=23223,
master_retry_count=6,
master_connect_retry=10
for channel 'mgr';
```

> master_retry_count和master_connect_retry表示重试多久切换复制源

**配置asynchronous connection auto failover**

参数语法

```
asynchronous_connection_failover_add_source(channel-name,host,port,network-namespace,weight)
```

- channel-name：通道名称
- host：节点地址
- port：节点端口
- network-namespace：网络区
- weight：权重，权重高的优先作为复制源，可配合MGR权重配置

添加复制源

```
SELECT asynchronous_connection_failover_add_source('mgr','127.0.0.1',23223,null,100);
```

检查复制

```
mysql> SELECT CHANNEL_NAME, SOURCE_CONNECTION_AUTO_FAILOVER FROM
performance_schema.replication_connection_configuration;
+--------------+---------------------------------+
| CHANNEL_NAME | SOURCE_CONNECTION_AUTO_FAILOVER |
+--------------+---------------------------------+
| mgr          | 1                               |
+--------------+---------------------------------+
```
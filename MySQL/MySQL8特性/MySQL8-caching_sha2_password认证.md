# MySQL8: caching_sha2_password认证

在MySQL8.0中，用户认证插件默认值为caching_sha2_password，可以实现SHA-256加密认证。

```
msandbox@(none) 15:39:  show variables like 'default_authentication_plugin';
+-------------------------------+-----------------------+
| Variable_name                 | Value                 |
+-------------------------------+-----------------------+
| default_authentication_plugin | caching_sha2_password |
+-------------------------------+-----------------------+
```

在创建用户时，也可以指定认证插件

```
CREATE USER 'sha2user'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password';
```

caching_sha2_password会将用户名和密码通过哈希进行缓存当用户访问时，会跟缓存条目进行匹配，匹配则认证成功，否则会对mysql.user进行校验，校验通过再进行缓存，提升认证效率。当删除用户、修改用户、修改认证方式都会清空认证缓存，flush privileges同样也会清空认证缓存。

caching_sha2_password只能通过加密的安全连接或RSA密钥进行密码交换的非加密连接进行访问，caching_sha2_password_auto_generate_rsa_keys参数默认会在数据库启动时生成对应的公钥和私钥。我们可以通过Caching_sha2_password_rsa_public_key查看公钥值

```
msandbox@(none) 15:28:  show status like 'Caching_sha2_password_rsa_public_key';
+--------------------------------------+-------------------------------------+
| Variable_name                        | Value                               |         
+--------------------------------------+-------------------------------------+
| Caching_sha2_password_rsa_public_key | -----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0N+rEZ/4F4CIsVdi+kif
M7YZgpyNv/qyJJv/hWf2Cwc/edS0yrQNUlsQzb6QRqlwCRRehgRiqu5ZPHLuxs+J
osCHkPZZkIVJO8j1GfmP1Pk8um7WVeAY7JEqmW6lzdrN3Ar2G74n+0C7HvchRI2u
aH5dK1FNtauz6f7owRo5xI4DdWG0MWCRq+xWeqlzdVUk+qDWlIwud6qPxAQO6azU
L2lztHKjBuWjXEoFSvZCC19wMKeDRxrk/eLSWCzFmquJRXB8oFlRnxl0OEnnYze1
qxAdSirG0Z+eRNmpMdqhIsz0Qrekznl/xX1yOeBkJVFZCbpn7SaKxBE2gKWoHkem
ewIDAQAB
-----END PUBLIC KEY-----
```

对于命令行客户端可以与服务器进行基于RSA密钥对进行认证连接，例如：mysqldump，mysqlslow等。但默认情况下，MySQL服务端并不会将RSA公钥发送给客户端，因此需要指定下列选项之一来获取RSA公钥

- –get-server-public-key：通过请求访问获取RSA公钥
- –server-public-key-path：客户端从服务端拷贝RSA公钥，相对更安全一点

对于MySQL主从复制来说change master to命令需要通过MASTER_PUBLIC_KEY_PATH选项指定RSA公钥文件，或通过GET_MASTER_PUBLIC_KEY获取源端的公钥

对于MGR来说，节点相互通信认证需要开启参数group_replication_recovery_get_public_key，否则可能出现如下情况

```
root@(none) 01:38:  select * from performance_schema.replication_connection_status\G
*************************** 1. row ***************************
                                      CHANNEL_NAME: group_replication_applier
                                        GROUP_NAME: e91c4508-d45a-11e9-911e-005056ab71f1
                                       SOURCE_UUID: e91c4508-d45a-11e9-911e-005056ab71f1
                                         THREAD_ID: NULL
                                     SERVICE_STATE: OFF
                         COUNT_RECEIVED_HEARTBEATS: 0
                          LAST_HEARTBEAT_TIMESTAMP: 0000-00-00 00:00:00.000000
                          RECEIVED_TRANSACTION_SET: 
                                 LAST_ERROR_NUMBER: 0
                                LAST_ERROR_MESSAGE: 
                              LAST_ERROR_TIMESTAMP: 0000-00-00 00:00:00.000000
                           LAST_QUEUED_TRANSACTION: 
 LAST_QUEUED_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
LAST_QUEUED_TRANSACTION_IMMEDIATE_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
     LAST_QUEUED_TRANSACTION_START_QUEUE_TIMESTAMP: 0000-00-00 00:00:00.000000
       LAST_QUEUED_TRANSACTION_END_QUEUE_TIMESTAMP: 0000-00-00 00:00:00.000000
                              QUEUEING_TRANSACTION: 
    QUEUEING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
   QUEUEING_TRANSACTION_IMMEDIATE_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
        QUEUEING_TRANSACTION_START_QUEUE_TIMESTAMP: 0000-00-00 00:00:00.000000
*************************** 2. row ***************************
                                      CHANNEL_NAME: group_replication_recovery
                                        GROUP_NAME: 
                                       SOURCE_UUID: 
                                         THREAD_ID: NULL
                                     SERVICE_STATE: OFF
                         COUNT_RECEIVED_HEARTBEATS: 0
                          LAST_HEARTBEAT_TIMESTAMP: 0000-00-00 00:00:00.000000
                          RECEIVED_TRANSACTION_SET: 
                                 LAST_ERROR_NUMBER: 2061
                                LAST_ERROR_MESSAGE: error connecting to master 'repl@10.255.210.9:3306' - retry-time: 60 retries: 1 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
                              LAST_ERROR_TIMESTAMP: 2021-01-25 01:26:54.897330
                           LAST_QUEUED_TRANSACTION: 
 LAST_QUEUED_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
LAST_QUEUED_TRANSACTION_IMMEDIATE_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
     LAST_QUEUED_TRANSACTION_START_QUEUE_TIMESTAMP: 0000-00-00 00:00:00.000000
       LAST_QUEUED_TRANSACTION_END_QUEUE_TIMESTAMP: 0000-00-00 00:00:00.000000
                              QUEUEING_TRANSACTION: 
    QUEUEING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
   QUEUEING_TRANSACTION_IMMEDIATE_COMMIT_TIMESTAMP: 0000-00-00 00:00:00.000000
        QUEUEING_TRANSACTION_START_QUEUE_TIMESTAMP: 0000-00-00 00:00:00.000000
```

[Caching SHA-2 Pluggable Authentication]:https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html


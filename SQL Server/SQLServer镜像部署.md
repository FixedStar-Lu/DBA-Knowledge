[TOC]

# SQLServer镜像部署

## 概述

![img](https://gitee.com/dba_one/wiki_images/raw/master/images/dbm-3-way-session-intro-ov.gif)

**镜像角色**

- 主节点(principal)：具有完整的数据副本，对外提供读写服务
- 镜像节点(mirror)：具有完整的数据副本，不提供读写服务，通过接收principal节点的日志实现数据同步，允许创建数据库快照
- 见证节点(witness)：本身不存储数据，只在高安全允许模式下提供自动故障转移的功能，确保只有一个节点对外提供服务

在镜像群集中，principal和mirror的数据同步依靠事务日志来实现的，与Oracle和Mysql不同，SQL Server的事务日志是DataBase级别的，每个数据库都有单独的事务日志，因此镜像服务是可以基于数据库的。一个数据库只能有一个mirror节点，与Mysql的级联复制不太一样。

SQL Server的事务日志是物理级别的，记录对数据页的操作。principal创建镜像后，会启动一个日志发送线程，维护一个虚拟的发送队列，然后读取事务日志，进行压缩再发送到Mirror节点，Mirror节点接收到后，将其写入本地磁盘的重做队列中，通过异步线程从重做队列中获取事务日志分发给应用线程回放。

**镜像运行模式**

- 高性能模式：principal与mirror之间异步传输，无需等待mirror的确认，在principal宕机后可能会造成数据丢失，不支持自动故障转移，适合对数据可靠性要求不高，性能要求比较高的场景，类似与Oracle DataGurad中的最大性能模式
- 不带故障转移的高安全模式：principal上所有的事务提交，都必须确认该事务已经应用到mirror中，才可提交。可实现principal宕机数据零丢失，不支持故障转移，可手动转移。
- 带故障转移的高安全模式：与不带故障转移的高安全模式，增加了witness(见证服务器)，可以实现故障自动转移，确保只有一个节点成为principal对外提供数据库服务

带故障转移的高安全模式下自动故障转移服务有以下情况：

1. principal与witness连接断开
   此时witness与mirror连接正常，触发自动故障转移，principal状态标记为disconnected，切断客户端连接，停止读写服务，等待故障切换。为避免网络抖动的原因造成不必要的切换，连接超时时间为10S。等待mirror上的重做队列日志回放完成后，mirror成为新的principal对外提供服务。
2. mirror和witness连接断开
   此时principal与witness连接正常，principal状态变成disconnected，与mirror断开连接，mirror状态为suspend，不再向mirror发送日志，等待mirror重新连接witness才会恢复与mirror之间的同步。
3. principal和mirror连接断开
   principal 与 mirror 同时保持 witness 的连接会话，但是 principal 与 mirror 之间会话中断，witness 会通知 mirror，principal 依然保持连接状态，不会触发故障切换；此时 principal 由于保持有 witness 的连接会话，服务正常
4. principal 与所有节点会话中断
   只要 mirror 与 witness 会话正常，即可完成正常的故障转移；如果 mirror 与 witness 连接也中断，则无法完成，即便是后来 mirror 与 witness 的会话优先恢复，则也无法故障切换，因为已然不确定 mirror 是否拥有全部 principal 的数据，此时即便 principal 处于运行状态，也无法提供服务，等待 principal 与任意节点会话恢复正常，即可恢复读写服务
5. mirror 与 所有节点会话中断
   不会触发故障切换，principal 切入公开运行模式（异步），即不会再向 mirror 发送事务日志，也不再需要等待 mirror 的响应，直到 mirror 重新恢复会话
6. witness 与所有节点会话中断
   不会触发故障切换，principal 继续提供读写服务，与 mirror 数据继续同步，镜像集群丧失自动故障转移能力，退化为不带故障转移的高安全模式


## 镜像群集配置(AD域)

**先决条件**

数据库必须处于完全恢复模式，如果不是请用下列语句修改

```
USE master;  
GO  
ALTER DATABASE AdventureWorks   
SET RECOVERY FULL;  
GO
```

**主节点备份数据库**

```
BACKUP DATABASE TEST 
TO DISK = N'D:\backup\test.bak' 
   WITH NOFORMAT, NOINIT,  NAME = N'TEST-完整 数据库 备份', SKIP, NOREWIND, NOUNLOAD, COMPRESSION,  STATS = 10
GO
```

**镜像节点恢复数据库(NORECOVERY)**

```
RESTORE DATABASE test  
   FROM disk='D:\backup\test.bak'  
   WITH NORECOVERY,  
   MOVE 'test_Data' TO 'D:\data\test_Data.mdf',   
   MOVE 'test_Log' TO 'F:\log\test_Log.ldf';  
GO
```

**主节点创建端点**

```
CREATE ENDPOINT Endpoint_Mirroring  
    STATE=STARTED   
    AS TCP (LISTENER_PORT=5022)   
    FOR DATABASE_MIRRORING (ROLE=PARTNER)  
GO  
--Partners under same domain user; login already exists in master.  
--Create a login for the witness server instance,  
--which is running as Somedomain\witnessuser:  
USE master ;  
GO  
CREATE LOGIN [Somedomain\witnessuser] FROM WINDOWS ;  
GO  
-- Grant connect permissions on endpoint to login account of witness.  
GRANT CONNECT ON ENDPOINT::Endpoint_Mirroring TO [Somedomain\witnessuser];  
--Grant connect permissions on endpoint to login account of partners.  
GRANT CONNECT ON ENDPOINT::Endpoint_Mirroring TO [Mydomain\dbousername];  
GO
```

**镜像节点创建端点**

```
CREATE ENDPOINT Endpoint_Mirroring  
    STATE=STARTED   
    AS TCP (LISTENER_PORT=5022)   
    FOR DATABASE_MIRRORING (ROLE=ALL)  
GO  
--Partners under same domain user; login already exists in master.  
--Create a login for the witness server instance,  
--which is running as Somedomain\witnessuser:  
USE master ;  
GO  
CREATE LOGIN [Somedomain\witnessuser] FROM WINDOWS ;  
GO  
--Grant connect permissions on endpoint to login account of witness.  
GRANT CONNECT ON ENDPOINT::Endpoint_Mirroring TO [Somedomain\witnessuser];  
--Grant connect permissions on endpoint to login account of partners.  
GRANT CONNECT ON ENDPOINT::Endpoint_Mirroring TO [Mydomain\dbousername];  
GO
```

**见证节点创建端点**

```
CREATE ENDPOINT Endpoint_Mirroring  
    STATE=STARTED   
    AS TCP (LISTENER_PORT=5022)   
    FOR DATABASE_MIRRORING (ROLE=WITNESS)  
GO  
--Create a login for the partner server instances,  
--which are both running as Mydomain\dbousername:  
USE master ;  
GO  
CREATE LOGIN [Mydomain\dbousername] FROM WINDOWS ;  
GO  
--Grant connect permissions on endpoint to login account of partners.  
GRANT CONNECT ON ENDPOINT::Endpoint_Mirroring TO [Mydomain\dbousername];  
GO
```

**在镜像节点设置主节点成为伙伴**

```
ALTER DATABASE AdventureWorks   
    SET PARTNER =   
    'TCP://192.168.25.10:5022'  
GO
```

**在主节点设置镜像节点成为伙伴**

```
ALTER DATABASE AdventureWorks   
    SET PARTNER = 'TTCP://192.168.25.11:5022'  
GO
```

**在主节点设置见证节点**

```
ALTER DATABASE AdventureWorks   
    SET WITNESS =   
    'TCP://192.168.25.12:5022'  
GO
```

## 镜像群集配置(非域)

**先决条件**

数据库必须处于完全恢复模式，如果不是请用下列语句修改

```
USE master;  
GO  
ALTER DATABASE AdventureWorks   
SET RECOVERY FULL;  
GO
```

**主节点备份数据库**

```
BACKUP DATABASE TEST 
TO DISK = N'D:\backup\test.bak' 
   WITH NOFORMAT, NOINIT,  NAME = N'TEST-完整 数据库 备份', SKIP, NOREWIND, NOUNLOAD, COMPRESSION,  STATS = 10
GO
```

**镜像节点恢复数据库(NORECOVERY)**

```
RESTORE DATABASE test  
   FROM disk='D:\backup\test.bak'  
   WITH NORECOVERY,  
   MOVE 'test_Data' TO 'D:\data\test_Data.mdf',   
   MOVE 'test_Log' TO 'F:\log\test_Log.ldf';  
GO
```

**主节点创建证书**

```
IF NOT EXISTS(  
SELECT * FROM sys.symmetric_keys
WHERE name = N'##MS_DatabaseMasterKey##')
CREATE MASTER KEY
ENCRYPTION BY PASSWORD = N'Roy@123';    --KEY密码
GO
CREATE CERTIFICATE CA_MIRROR_SQL01     --证书名称
WITH
SUBJECT = N'certificate for database mirror',
START_DATE = '19990101',
EXPIRY_DATE = '99991231';
GO
```

**镜像节点创建证书**

```
IF NOT EXISTS(  
SELECT * FROM sys.symmetric_keys
WHERE name = N'##MS_DatabaseMasterKey##')
CREATE MASTER KEY
ENCRYPTION BY PASSWORD = N'Roy@123';    --KEY密码
GO
CREATE CERTIFICATE CA_MIRROR_SQL01     --证书名称
WITH
SUBJECT = N'certificate for database mirror',
START_DATE = '19990101',
EXPIRY_DATE = '99991231';
GO
```

**见证节点创建证书**

```
IF NOT EXISTS(  
SELECT * FROM sys.symmetric_keys
WHERE name = N'##MS_DatabaseMasterKey##')
CREATE MASTER KEY
ENCRYPTION BY PASSWORD = N'Roy@123';    --KEY密码
GO
CREATE CERTIFICATE CA_MIRROR_SQL01     --证书名称
WITH
SUBJECT = N'certificate for database mirror',
START_DATE = '19990101',
EXPIRY_DATE = '99991231';
GO
```

**备份主节点证书到其它节点**

```
BACKUP CERTIFICATE CA_MIRROR_SQL01
TO FILE ='C:\Program Files\Microsoft SQL Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_SQL01.cer'
```

**备份镜像节点证书到其它节点**

```
BACKUP CERTIFICATE CA_MIRROR_SQL01
TO FILE ='C:\Program Files\Microsoft SQL Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_SQL01.cer'
```

**备份见证节点证书到其它节点**

```
BACKUP CERTIFICATE CA_MIRROR_SQL01
TO FILE ='C:\Program Files\Microsoft SQL Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_SQL01.cer'
```

**主节点创建端点**

```
CREATE ENDPOINT EDP_Mirror
STATE = STARTED
AS TCP(
LISTENER_PORT = 5022,  -- 镜像端点使用的通信端口
LISTENER_IP = ALL)     -- 侦听的IP地址
        FOR DATABASE_MIRRORING(
AUTHENTICATION = CERTIFICATE CA_MIRROR_SQL01, -- 证书身份验证
ENCRYPTION = DISABLED,         -- 不对传输的数据加密,如果需要加密,可以配置为  SUPPORTED 或 REQUIRED, 并可选择加密算法
ROLE = ALL)
```

**镜像节点创建端点**

```
CREATE ENDPOINT EDP_Mirror
STATE = STARTED
AS TCP(
LISTENER_PORT = 5022,  -- 镜像端点使用的通信端口
LISTENER_IP = ALL)     -- 侦听的IP地址
        FOR DATABASE_MIRRORING(
AUTHENTICATION = CERTIFICATE CA_MIRROR_SQL02, -- 证书身份验证
ENCRYPTION = DISABLED,           -- 不对传输的数据加密,如果需要加密,可以配置为  SUPPORTED 或 REQUIRED, 并可选择加密算法
ROLE = ALL)
```

**见证节点创建端点**

```
CREATE ENDPOINT EDP_Mirror
STATE = STARTED
AS TCP(
LISTENER_PORT = 5022,  -- 镜像端点使用的通信端口
LISTENER_IP = ALL)     -- 侦听的IP地址
        FOR DATABASE_MIRRORING(
AUTHENTICATION = CERTIFICATE CA_Mirror_WITNESS, -- 证书身份验证
ENCRYPTION = DISABLED,    -- 不对传输的数据加密,如果需要加密,可以配置为  SUPPORTED 或 REQUIRED, 并可选择加密算法
ROLE = ALL)
```

**主节点创建镜像节点证书**

```
CREATE LOGIN CA_MIRROR_SQL02 FROM FILE='C:\Program Files\Microsoft SQL  Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_SQL02.cer';
```

**主节点创建见证节点证书**

```
create CERTIFICATE CA_MIRROR_WITNESS
FROM FILE = 'C:\Program Files\Microsoft SQL  Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_WITNESS.cer';
```

**镜像节点创建主节点证书**

```
create CERTIFICATE CA_MIRROR_WITNESS
FROM FILE = 'C:\Program Files\Microsoft SQL  Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_WITNESS.cer';
```

**镜像节点创建见证节点证书**

```
create CERTIFICATE CA_MIRROR_WITNESS
FROM FILE = 'C:\Program Files\Microsoft SQL  Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_WITNESS.cer';
```

**见证节点创建主节点证书**

```
create CERTIFICATE CA_MIRROR_WITNESS
FROM FILE = 'C:\Program Files\Microsoft SQL  Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_WITNESS.cer';
```

**见证节点创建镜像节点证书**

```
CREATE CERTIFICATE CA_MIRROR_SQL02
FROM FILE = 'C:\Program Files\Microsoft SQL  Server\MSSQL12.MSSQLSERVER\MSSQL\Backup\CA_MIRROR_SQL02.cer';
```

**主节点创建登陆用户**

```
CREATE LOGIN LOGIN_TO_SQL02 FROM CERTIFICATE CA_MORROR_SQL02;
GRANT CONNECT ON ENDPOINT::EDP_Mirror TO LOGIN_TO_SQL02;
CREATE LOGIN LOGIN_TO_WITNESS FROM CERTIFICATE CA_MIRROR_WITNESS;
GRANT CONNECT ON ENDPOINT::EDP_Mirror TO LOGIN_TO_WITNESS;
```

**镜像节点创建登陆用户**

```
CREATE LOGIN LOGIN_TO_SQL02 FROM CERTIFICATE CA_MORROR_SQL02;
GRANT CONNECT ON ENDPOINT::EDP_Mirror TO LOGIN_TO_SQL02;
CREATE LOGIN LOGIN_TO_WITNESS FROM CERTIFICATE CA_MIRROR_WITNESS;
GRANT CONNECT ON ENDPOINT::EDP_Mirror TO LOGIN_TO_WITNESS;
```

**见证节点创建登陆用户**

```
CREATE LOGIN LOGIN_TO_SQL02 FROM CERTIFICATE CA_MORROR_SQL02;
GRANT CONNECT ON ENDPOINT::EDP_Mirror TO LOGIN_TO_SQL02;
CREATE LOGIN LOGIN_TO_WITNESS FROM CERTIFICATE CA_MIRROR_WITNESS;
GRANT CONNECT ON ENDPOINT::EDP_Mirror TO LOGIN_TO_WITNESS;
```

**镜像节点设置主节点为伙伴**

```
alter database mirror set PARTNER='TCP://192.168.25.10:5022';
```

**主节点设置镜像节点为伙伴**

```
alter database mirror set PARTNER='TCP://192.168.25.11:5022';
```

**主节点设置见证节点为伙伴**

```
ALTER DATABASE mirror SET WITNESS = 'TCP://192.168.25.13:5022'
```

## 日常运维

**查看镜像服务状态**

```
SELECT 
mirroring_role_desc,           -- 数据库在镜像会话中当前的角色
mirroring_state_desc,          -- 镜像当前状态
mirroring_safety_level_desc,   -- 镜像运行模式
mirroring_witness_state_desc   -- 与见证服务器的连接情况
FROM sys.database_mirroring
WHERE database_id = DB_ID(N'mirror');
```

**镜像暂停与恢复**

```
--暂停
ALTER DATABASE AdventureWorks2012 SET PARTNER SUSPEND;  
--恢复
ALTER DATABASE AdventureWorks2012 SET PARTNER RESUME;
```

**镜像切换**

```
ALTER DATABASE [MirrorDB] SET PARTNER FAILOVER;
Go
```

*Tips：应用程序在连接镜像时需要指定Failover节点的信息，例如：Server=IP,端口; Failover_Partner=IP,端口;*

[数据库镜像]:https://docs.microsoft.com/zh-cn/sql/database-engine/database-mirroring/database-mirroring-sql-server?redirectedfrom=MSDN&view=sql-server-ve
[使用 Windows 身份验证配置数据库镜像]:https://docs.microsoft.com/zh-cn/sql/database-engine/database-mirroring/example-setting-up-database-mirroring-using-windows-authentication-transact-sql?view=sql-server-ver15
[SQL Server Mirroring 各种模式下发生异常的情景演练]:https://blogs.msdn.microsoft.com/apgcdsd/2012/10/19/sql-server-mirroring/


#MySQL #Shell

# MySQL Shell
MySQL Shell是MySQL Server的高级客户端和代码编辑器。除了提供SQL功能外，MySQL Shell还为Javascript和python提供脚本化功能，并包括用于使用 MySQL 的 API。

**安装Shell**
```
[root@t-luhx02-v-szzb mysql]# tar -xvf mysql-shell-8.0.19-linux-glibc2.12-x86-64bit.tar.gz
[root@t-luhx02-v-szzb mysql]# mv mysql-shell-8.0.19-linux-glibc2.12-x86-64bit /usr/local/mysql-shell
[root@t-luhx02-v-szzb mysql]# echo 'export PATH=$PATH:/usr/local/mysql-shell/bin' >> /etc/profile
[root@t-luhx02-v-szzb mysql]# source /etc/profile
```

**MySQL Shell命令**

命令 | 快捷方式 | 描述
-- | -- | --
\help | \h或\? | 查看Shell帮助
\quit | \q或\exit | 退出MySQL Shell
\ | | 在SQL模式下，开启多行模式
\status | \s | 显示当前Shell状态
\js | | 将执行模式切换到javascript
\py | | 将执行模式切换到python
\sql | | 将执行模式切换到SQL
\connect | \c | 连接到MySQL服务器
\reconnect | | 重新连接到统一MySQL服务器
\use | \u | 指定schema
\source | \.或source | 执行脚本文件
\warnings | \W | 显示语句的任何告警
\nowarning | \w | 不显示语句生成的任何告警
\history | | 查看和编辑命令行历史记录
\rehash | | 手动更新自动完成名称缓存
\option | | 查询并更改配置选项
\show | | 使用提供的选项和参数运行指定的报表
\watch | | 使用提供的选项和参数运行指定的报表，并定期刷新结果
\edit | \e | 在默认系统编辑器中打开命令
\system | \! | 运行指定的操作系统命令

**连接MySQL Server**

在MySQL Shell安装完成后，我们可以通过mysqlsh命令进入Shell窗口
```
[root@t-luhx02-v-szzb ~]# mysqlsh
MySQL Shell 8.0.19

Copyright (c) 2016, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
 MySQL  JS >
```
在连接MySQL Server时，我们可以使用两种连接类型：
- session：需要安装并启用x插件，否则会出现如下错误，x插件默认监听端口33060，
```
[root@t-luhx02-v-szzb ~]# mysqlsh --mysqlx -u dba -P 3307
Please provide the password for 'dba@localhost:3307': ********
MySQL Shell 8.0.19

Copyright (c) 2016, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating an X protocol session to 'dba@localhost:3307'
MySQL Error 2027: Requested session assumes MySQL X Protocol but 'localhost:3307' seems to speak the classic MySQL protocol (Unexpected response received from server, msg-id:10)
```
- classicsession：不使用x插件进行交互，利用MySQL协议对服务器运行SQL，可用于此类会话的开发API非常有限。例如，没有集合处理，并且不支持绑定。对于开发而言，尽可能使用session
```
[root@t-luhx02-v-szzb ~]# mysqlsh --mysql -u dba -P 3307
Please provide the password for 'dba@localhost:3307': ********
Save password for 'dba@localhost:3307'? [Y]es/[N]o/Ne[v]er (default No): 
MySQL Shell 8.0.19

Copyright (c) 2016, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating a Classic session to 'dba@localhost:3307'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 137
Server version: 5.7.17-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
 MySQL  localhost:3307  JS >
```

MySQL Shell还支持URI的方式连接MySQL Server
```
[root@t-luhx02-v-szzb ~]# mysqlsh --uri mysql://dba@localhost:3307
[root@t-luhx02-v-szzb ~]# mysqlsh mysql://user@localhost:3306
```
在已经启动Shell之后，如果需要连接数据库可使用\connect选项进行连接
```
 MySQL  JS > \connect --mysql dba@localhost:3307
Creating a Classic session to 'dba@localhost:3307'
Please provide the password for 'dba@localhost:3307': ********
Save password for 'dba@localhost:3307'? [Y]es/[N]o/Ne[v]er (default No): 
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 141
Server version: 5.7.17-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
 MySQL  localhost:3307  JS > 
```

在python和js模式下可用函数来所选类型的多个会话对象，并将其分配给变量。通过会话对象可以建立和管理连接
```
 MySQL  JS > var s1 = mysql.getClassicSession('dba@10.0.139.162:3307','Abcd123#');
 MySQL  JS > s1
<ClassicSession:dba@10.0.139.162:3307>
 MySQL  JS > shell.setSession(s1)
<ClassicSession:dba@10.0.139.162:3307>
 MySQL  10.0.139.162:3307 ssl  JS > session
<ClassicSession:dba@10.0.139.162:3307>
 MySQL  10.0.139.162:3307 ssl  JS > shell.status();
MySQL Shell version 8.0.19

Session type:                 Classic
Connection Id:                5
Current schema:               
Current user:                 dba@10.0.139.162
SSL:                          Cipher in use: DHE-RSA-AES256-SHA TLSv1.1
Using delimiter:              ;
Server version:               5.7.17-log MySQL Community Server (GPL)
Protocol version:             classic 10
Client library:               8.0.19
Connection:                   10.0.139.162 via TCP/IP
TCP port:                     3307
Server characterset:          utf8
Schema characterset:          utf8
Client characterset:          utf8
Conn. characterset:           utf8
Compression:                  Disabled
Uptime:                       1 min 24.0000 sec

Threads: 2  Questions: 12  Slow queries: 0  Opens: 112  Flush tables: 1  Open tables: 66  Queries per second avg: 0.142
```
*注：该方式要求MySQL Server启动了SSL认证，如果未启用需通过mysql_ssl_rsa_setup生成证书*


Tips：[MySQL Shell 8.0](https://dev.mysql.com/doc/mysql-shell/8.0/en/)
[TOC]

---

## 升级检查

- **mysqlcheck**

```
[root@centos7-4 service]# mysqlcheck -uroot -p --all-databases --check-upgrade
mysqlcheck: [Warning] Using a password on the command line interface can be insecure.
confluence.ao_187ccc_sidebar_link                  OK
confluence.ao_21d670_whitelist_rules               OK
confluence.ao_26db7f_entities_to_room_cfg          OK
confluence.ao_26db7f_entities_to_rooms             OK
confluence.ao_38321b_custom_content_link           OK
confluence.ao_42e351_health_check_entity           OK
confluence.ao_54c900_c_template_ref                OK
confluence.ao_54c900_content_blueprint_ao          OK
confluence.ao_54c900_space_blueprint_ao            OK
confluence.ao_5f3884_feature_discovery             OK
confluence.ao_5fb9d7_aohip_chat_link               OK
confluence.ao_5fb9d7_aohip_chat_user               OK
confluence.ao_6384ab_discovered                    OK
confluence.ao_6384ab_feature_metadata_ao           OK
confluence.ao_7cde43_event                         OK
confluence.ao_7cde43_filter_param                  OK
confluence.ao_7cde43_notification                  OK
confluence.ao_7cde43_notification_scheme           OK
confluence.ao_7cde43_recipient                     OK
confluence.ao_7cde43_server_config                 OK
confluence.ao_7cde43_server_param                  OK
confluence.ao_88263f_health_check_status           OK
```

- **非innodb或NDB引擎的分区表**

MySQL8.0 Server服务不再提供通用分区支持，InnoDB和NDB是唯一提供MySQL8.0支持的本地分区处理程序的存储引擎。如果分区表用的是别的存储引擎，存储引擎必须进行修改为innodb或ndb，要么就删除分区。
```
SELECT TABLE_SCHEMA, TABLE_NAME
  FROM INFORMATION_SCHEMA.TABLES
WHERE ENGINE NOT IN ('innodb', 'ndbcluster')
  AND CREATE_OPTIONS LIKE '%partitioned%';
```

- **加密认证插件**

5.7用的MySQL_native_password，8.0默认已经为caching_sha2_password，如果客户端版本过低可能识别不了caching_sha2_password，导致链接失败。

- **取消过时的SQL_MODE**

配置文件的SQL_MODE参数中不能包含NO_AUTO_CREATE_USER，并且删除了部分SQL_MODE：DB2，MAXDB，MSSQL，MySQL323，MySQL40，ORACLE，POSTGRESQL，NO_FIELD_OPTIONS，NO_KEY_OPTIONS，NO_TABLE_OPTIONS

- **过时参数**

MySQL8.0取消了查询缓存功能，建议将参数从配置文件取消。innodb_undo_logs参数也即将被移除，建议修改为innodb_rollback_segments参数

- **外键约束名称不能超过64个字符**
```
SELECT TABLE_SCHEMA, TABLE_NAME
     FROM INFORMATION_SCHEMA.TABLES
     WHERE TABLE_NAME IN
       (SELECT LEFT(SUBSTR(ID,INSTR(ID,'/')+1),
                    INSTR(SUBSTR(ID,INSTR(ID,'/')+1),'_ibfk_')-1)
        FROM INFORMATION_SCHEMA.INNODB_SYS_FOREIGN
        WHERE LENGTH(SUBSTR(ID,INSTR(ID,'/')+1))>64);
```

- **查询重写**

经测试，在开启查询重写功能后，升级会报错。因为存储过程flush_rewrite_rules会调用RESET QUERY CACHE，但8.0已经移除了QUERY CACHE功能。

- **Upgrade Checker**

MySQL Shell8.0引入了Upgrade Checker升级检查功能，在升级之前我们可以运行它来查看相关升级配置是否符合升级要求。
```
mysqlsh -- util check-for-server-upgrade { --user=root --host=localhost --port=33006 } --target-version=8.0.15 --output-format=JSON --config-path=/etc/my.cnf
```
> 也可以执行 mysqlsh -- util checkForServerUpgrade root@localhost:33006 --target-version=8.0.19 --config-path=/etc/my.cnf查看老式报告

**报告说明**

分类 | 选项 | 说明
-- | -- | --
serverAddress | | MySQL Shell与已检查的MySQL服务器实例的连接的主机名和端口号
serverVersion | | 检测到已检查的服务器实例的MySQL版本
targetVersion | | 目标MySQL版本进行升级检查
errorCount | | 发现的错误数
warningCount | | 发现的警告数
noticeCount | | 发现的通知数
summary | | 汇总信息
checksPerformed | | 一组JSON对象，用于自动检查的每个单独的升级问题
checksPerformed | id | 唯一ID
checksPerformed | title |  标题说明
checksPerformed | status | 如果检查成功运行，则为“ OK”，否则为“ ERROR”
checksPerformed | description | 包含建议的检查的详细描述（如果有），如果检查无法运行，则显示错误消息
checksPerformed | documentationLink | 如果有的话，指向包含更多信息或建议的文档的链接。
checksPerformed | detectedProblems | 检测到的问题
manualChecks | | 一组JSON对象，每个与升级路径相关的单个升级问题，需要手动检查
manualChecks | id | 唯一ID
manualChecks | title | 标题说明
manualChecks | description | 有关手动检查的详细说明，包括信息和建议
manualChecks | documentationLink | 如果有的话，指向包含更多信息或建议的文档的链接

更多关于Upgrade Checker的信息请参阅：[Upgrade Checker Utility](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-upgrade.html)

## 备份数据库

如果check未出现需要修复的错误，就可以采用inplace的方式升级到8.0。但在停止数据库并升级之前，需要对数据库进行备份，可以选择mysqldump或mysqlpump工具，备份应该包含mysql系统数据库和日志文件。

## 升级数据库

下载MySQL8.0的二进制文件，并以8.0的程序重新启动数据库(关闭数据库前需要将参数innodb_fast_shutdown为0)
```
[root@t-luhx03-v-szzb support-files]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 16
Server version: 8.0.19 MySQL Community Server - GPL

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

root@(none) 14:57: 
```

在mysql8.0.16中mysql_upgrade已经被废弃，已经集成到mysqld中，可以通过mysqld的upgrade选项升级系统数据库。

8.0.16之前的升级方式
![before](https://mysqlserverteam.com/wp-content/uploads/2019/04/img-1.png)

8.0.16之后的升级方式
![after](https://mysqlserverteam.com/wp-content/uploads/2019/04/img-2.png)

mysqld --upgrade提供了4个选项：
- NONE：不尝试进行升级
- AUTO：默认值。可使服务器尝试数据字典升级和Server升级，sever升级包含升级用户schema
- MINIMAL：仅升级数据字典
- force：强制升级

>除非指定了--upgrade = NONE或--upgrade = MINIMAL，否则每次启动服务器时都将尝试进行升级

## 附录

**1、认证连接失败**

升级完成后，连接数据库出现下列错误
```
Authentication plugin 'caching_sha2_password' cannot be loaded: dlopen(/usr/local/mysql/lib/plugin/caching_sha2_password.so, 2): image not found
```
经查发现8.0中的认证插件已经变成caching_sha2_password，因此需要将我们的用户修改为之前的mysql_native_password插件
```
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'Abcd123#';
mysql> FLUSH PRIVILEGES;
```

更多详细信息请参考[MySQL 8.0.16: mysql_upgrade is going away](https://mysqlserverteam.com/mysql-8-0-16-mysql_upgrade-is-going-away/)

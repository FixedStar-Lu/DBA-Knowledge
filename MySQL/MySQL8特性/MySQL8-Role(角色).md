[TOC]

---

# MySQL8 - Roles

## MySQL5.7实现

在5.7中想要实现角色功能的话可以借助proxies_priv来简单实现，要想使用proxies_priv需要先开启相关参数
```
root@(none) 17:28:  set global check_proxy_users =on;
Query OK, 0 rows affected (0.00 sec)

root@(none) 17:29:  set global mysql_native_password_proxy_users = on;
Query OK, 0 rows affected (0.00 sec)

root@(none) 17:29:  show variables like "%proxy%"; 
+-----------------------------------+-------+
| Variable_name                     | Value |
+-----------------------------------+-------+
| check_proxy_users                 | ON    |
| mysql_native_password_proxy_users | ON    |
| proxy_user                        |       |
+-----------------------------------+-------+
```

创建dba组
```
root@(none) 17:29:  create user dba;
Query OK, 0 rows affected (0.00 sec)
```

创建成员
```
root@(none) 17:31:  create user tom@'localhost' identified by 'Abcd123#';
Query OK, 0 rows affected (0.01 sec)

root@(none) 17:31:  create user jey@'localhost' identified by 'Abcd123#';
Query OK, 0 rows affected (0.01 sec)
```

将DBA权限映射到成员中
```
root@(none) 17:32:  grant proxy on dba to tom@'localhost';
Query OK, 0 rows affected (0.00 sec)

root@(none) 17:33:  grant proxy on dba to jey@'localhost';
Query OK, 0 rows affected (0.00 sec)
```

授予权限给DBA
```
root@(none) 17:35:  grant select on test.* to dba;
Query OK, 0 rows affected (0.00 sec)
```

查看权限
```
root@(none) 17:36:  select * from mysql.proxies_priv;
+-----------+------+--------------+--------------+------------+----------------------+---------------------+
| Host      | User | Proxied_host | Proxied_user | With_grant | Grantor              | Timestamp           |
+-----------+------+--------------+--------------+------------+----------------------+---------------------+
| localhost | root |              |              |          1 | boot@connecting host | 0000-00-00 00:00:00 |
| localhost | tom  | %            | dba          |          0 | root@localhost       | 0000-00-00 00:00:00 |
| localhost | jey  | %            | dba          |          0 | root@localhost       | 0000-00-00 00:00:00 |
+-----------+------+--------------+--------------+------------+----------------------+---------------------+
```

## MySQL8.0实现

MySQL8.0已经正式提供了角色功能，我们可以通过create role来创建角色
```
root@(none) 17:27:  create role db_owner,db_reader,db_writer;
Query OK, 0 rows affected (0.01 sec)
```

给不同的角色添加不同的权限
```
root@(none) 17:39:  grant all on test.* to db_owner;
Query OK, 0 rows affected (0.02 sec)

root@(none) 17:40:  grant select on test.* to db_reader;
Query OK, 0 rows affected (0.01 sec)

root@(none) 17:41:  grant delete,update,insert on test.* to db_reader;
Query OK, 0 rows affected (0.01 sec)

root@(none) 17:41:  grant delete,update,insert on test.* to db_writer;
Query OK, 0 rows affected (0.01 sec)
```

创建用户并指定对应角色，一个用户可以对应多个角色，必须设置一个默认角色
```
root@(none) 17:41:  create user test@'%' identified by 'Abcd123#';
Query OK, 0 rows affected (0.01 sec)

root@(none) 17:43:  grant db_owner,db_reader,db_writer to test;
Query OK, 0 rows affected (0.02 sec)

root@(none) 17:44:  set default role all to test;
Query OK, 0 rows affected (0.00 sec)
```

查看当前用户对应的角色
```
test@(none) 17:47:  select current_role();
+------------------------------------------------+
| current_role()                                 |
+------------------------------------------------+
| `db_owner`@`%`,`db_reader`@`%`,`db_writer`@`%` |
+------------------------------------------------+
```

切换角色
```
test@(none) 17:49:  set role db_reader;
Query OK, 0 rows affected (0.00 sec)

test@(none) 17:49:  create table test.f(id int);
ERROR 1142 (42000): CREATE command denied to user 'test'@'localhost' for table 'f'

test@(none) 17:49:  set role db_owner;
Query OK, 0 rows affected (0.00 sec)

test@(none) 17:52:  create table test.f(id int);
Query OK, 0 rows affected (0.03 sec)
```

针对角色还有两个参数：activate_all_roles_on_login和mandatory_roles。activate_all_roles_on_login表示是否在连接mysql时自动激活角色，mandatory_roles表示强制用户的默认角色
```
root@(none) 18:05:  set global activate_all_roles_on_login=on;
Query OK, 0 rows affected (0.00 sec)

root@(none) 18:05:  set global mandatory_roles='db_reader';
Query OK, 0 rows affected (0.00 sec)
```

实际上MySQL用户也能作为角色将权限授予给其它用户
```

root@(none) 18:06:  select user,host from mysql.user;
+------------------------+------------+
| user                   | host       |
+------------------------+------------+
| cachecloud             | %          |
| db_owner               | %          |
| db_reader              | %          |
| db_writer              | %          |
| dba                    | %          |
+------------------------+------------+

root@(none) 18:08:  grant dba to test;
Query OK, 0 rows affected (0.01 sec)
```

回收用户角色权限
```
root@(none) 18:09:  revoke dba from test;
Query OK, 0 rows affected (0.00 sec)
```

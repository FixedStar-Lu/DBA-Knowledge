#MySQL #QueryRewrite
# 查询重写(Query_rewrite)

MySQL从5.7.6开始支持Rewrite plugin插件，可以将符合条件的SQL语句进行改写

**安装插件**
```
[root@t-luhx03-v-szzb share]# mysql -uroot -p < /usr/local/mysql/share/install_rewriter.sql
Enter password: 

root@test 09:45:  SHOW GLOBAL VARIABLES LIKE 'rewriter_enabled';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| rewriter_enabled | ON    |
+------------------+-------+    
```

脚本会创建一个query_rewrite的数据库，其中包含一张rewrite_rules的表，用于定义并管理重写的规则

**示例**

插入规则
```
root@(none) 09:55:  insert into query_rewrite.rewrite_rules(pattern,replacement,pattern_database) values ("select * from tab1 where id=?","select * from tab1 where name='lu'","test");
Query OK, 1 row affected (0.01 sec)

root@(none) 10:02:  select * from query_rewrite.rewrite_rules\G
*************************** 1. row ***************************
                id: 1
           pattern: select * from tab1 where id=?
  pattern_database: test
       replacement: select * from tab1 where name='lu'
           enabled: YES
           message: NULL
    pattern_digest: NULL
normalized_pattern: NULL
```

重新加载
```
root@(none) 10:02:  call query_rewrite.flush_rewrite_rules();
```

验证效果
```
root@test 10:04:  select * from tab1;
+----+------------+------+
| id | name       | comm |
+----+------------+------+
|  1 | lu         | NULL |
|  2 | heng       | NULL |
|  3 | xing       | NULL |
|  4 | luhengxing | NULL |
+----+------------+------+
4 rows in set (0.00 sec)

root@test 10:04:  select * from tab1 where id=2;
+----+------+------+
| id | name | comm |
+----+------+------+
|  1 | lu   | NULL |
+----+------+------+
1 row in set, 1 warning (0.00 sec)

root@test 10:04:  show warnings\G
*************************** 1. row ***************************
  Level: Note
   Code: 1105
Message: Query 'select * from tab1 where id=2' rewritten to 'select * from tab1 where name='lu'' by a query rewrite plugin
1 row in set (0.01 sec)
```

禁用规则
```
root@test 10:08:  UPDATE query_rewrite.rewrite_rules SET enabled = 'NO' WHERE id = 1;
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0

root@test 10:11:  CALL query_rewrite.flush_rewrite_rules();
Query OK, 0 rows affected (0.01 sec)
```

**Reference**

[Using the Rewriter Query Rewrite Plugin](https://dev.mysql.com/doc/refman/5.7/en/rewriter-query-rewrite-plugin-usage.html)
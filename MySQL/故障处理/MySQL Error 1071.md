日前应用在MySQL5.6上执行ALTER修改字段长度时出现"ERROR 1071 (42000): Specified key was too long; max key length is 767 bytes"的错误。这里的长度是指的索引长度，并不是字段长度限制。

MySQL5.6文档内容记录如下

> By default, the index key prefix length limit is 767 bytes. See Section 13.1.13, “CREATE INDEX Syntax”. For example, you might hit this limit with a column prefix index of more than 255 characters on a TEXT or VARCHAR column, assuming a utf8mb3 character set and the maximum of 3 bytes for each character. When the innodb_large_prefix configuration option is enabled, the index key prefix length limit is raised to 3072 bytes for InnoDB tables that use DYNAMIC or COMPRESSED row format.
Attempting to use an index key prefix length that exceeds the limit returns an error. To avoid such errors in replication configurations, avoid enablinginnodb_large_prefix on the master if it cannot also be enabled on slaves.
The limits that apply to index key prefixes also apply to full-column index keys.


大意就是默认情况下，索引键前缀长度限制为767个字节，在启用innodb_large_prefix并且对于ROW_FORMAT为DYNAMIC或者COMPRESSED格式的表，索引前缀长度将提高的3072字节

开启innodb_large_prefix(5.6默认关闭)
```
root@test 10:03:  set global innodb_large_prefix=on;
Query OK, 0 rows affected (0.00 sec)
root@test 10:12:  ALTER TABLE TEST MODIFY CODE_VALUE1 VARCHAR(350);
ERROR 1709 (HY000): Index column size too large. The maximum column size is 767 bytes.
```

修改innodb_file_format
```
root@test 10:12:  show variables like '%innodb_file_format%';
+--------------------------+----------+
| Variable_name            | Value    |
+--------------------------+----------+
| innodb_file_format       | Antelope |
| innodb_file_format_check | ON       |
| innodb_file_format_max   | Antelope |
+--------------------------+----------+
3 rows in set (0.00 sec)

root@test 10:13:  set global innodb_file_format=Barracuda;
Query OK, 0 rows affected (0.00 sec)

root@test 10:14:  set global innodb_file_format_max=Barracuda;
Query OK, 0 rows affected (0.01 sec)

root@test 10:14:  ALTER TABLE TEST MODIFY CODE_VALUE1 VARCHAR(350);
ERROR 1709 (HY000): Index column size too large. The maximum column size is 767 bytes.
```

修改表的ROW_FORMAT
```
root@test 10:14:  ALTER TABLE TEST ROW_FORMAT=DYNAMIC;
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

root@test 10:16:  ALTER TABLE TEST MODIFY CODE_VALUE1 VARCHAR(350);
Query OK, 0 rows affected (0.03 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

当然我们也可以通过使用前缀索引的方式来避免这个错误，而且不需要修改任何参数，但前缀索引无法使用无法在order by或group by中使用，也无法使用前缀索引做覆盖扫描。
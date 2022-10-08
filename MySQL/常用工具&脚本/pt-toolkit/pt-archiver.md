[TOC]

---

# pt-archiver

随着业务数据不断增长，很多表都到了千万级别，这时可以考虑根据业务来对历史数据进行归档清理，生产环境只保留固定周期。而pt-archiver正好可以很好的协助完成这项工作，它能将归档数据保存到其它表或一个平面文件中，对数据库的影响也较小

## 常用参数

  参数 | 描述 | 可选值
-- | -- | --
--analyze | 指定工具完成数据归档后对表执行analyze table | d,s,ds(d为目标端，s为源端)
--ask-pass | 连接MySQL时提示输入密码 | 
--buffer | 缓冲区输出到file并在提交时刷新，能够提升一定的性能，崩溃可能造成数据丢失 | 
--bulk-delete | 用单个语句删除每个chunk | 
--[no]bulk-delete-limit | 针对bulk-delete添加limit | 默认yes
--bulk-insert | 用LOAD DATA INFILE写入每个chunk | 
--channel | 使用复制通道连接到服务器时使用的channel名称，适用于多源复制 | 
--charset | 默认的字符集，连接到MySQL后会执行SET NAMES | utf8
--[no]check-charset | 检查连接字符集和表字符集是否相同 | 默认yes
--[no]check-columns | 检查源端和目标端的列是否相同 | 默认yes
--check-slave-lag | 当主从延迟大于--max-lag延迟时暂停归档 | 
--check-interval | 当指定了--check-slave-lag，该选项指定的时间为工具发现主从复制延迟时暂停的时间 | 默认1s
--columns | 要归档的列，逗号分隔 | 
--commit-each | 指定按每次获取和归档的行数进行提交据，会禁用选项'--txn-size' | 
--config | 读取配置文件列表，需要指定为第一个选项 | 
--database | 连接的数据库 | 
--delayed-insert | 将DELAYED修饰符写入insert语句 | 
--source | 源端的DSN | --source F=host1.cnf,D=db,t=tbl
--dest | 目标端的DSN |--source F=host1.cnf,D=db,t=tbl --dest h=host2
--dry-run | 打印查询并退出，不执行任何操作 | 
--file | 归档数据的平面文件 | --file '/var/log/archive/%Y-%m-%d-%D.%t'
--for-update | 添加FOR UPDATE到查询中 | 
--header | 包含列标题 | 
--high-priority-select | 添加HIGH_PRIORITY到查询语句 | 
--host | 连接的MySQL Host | 
--ignore | 添加IGNORE到INSERT语句 | 
--limit | 每个语句要获取和存档的行数 | 默认为1
--local | OPTIMIZE或ANALYZE操作不写入binlog | 
--low-priority-delete | 添加LOW_PRIORITY修饰符到delete语句 | 
--max-lag | 最大的延迟时间，超过则由--check-slave-lag暂停归档 | 默认1s
--no-delete | 不删除归档数据 | 
--optimize | 在源端和目标端运行OPTIMIZE TABLE | 
--output-format | 归档文件格式 | dump/csv
--password | 连接数据库密码 | 
--port | 连接数据库端口 | 
--primary-key-only | 通过主键来完成数据删除 | 
--progress | 每N行打印进度信息 | 
--purge | 清理数据，可以不进行归档 | 
--where | 指定通过WHERE条件语句指定需要归档的数据，必选项 | 
--set-vars | 设置一些临时变量 | wait_timeout=10000

**DSN配置**

除了指定HOST,DATABASE,PORT等连接选项，也可以直接通过DSN的方式来连接数据，采用key-value的格式，分为source和dest
```
--source h=127.0.0.1,P=19529,u=root,D=employees,t=test_titles,A=utf8
```
- a：指定归档在哪个数据库下进行
- A：指定字符集
- D：归档表所在的数据库
- h：MySQL Host
- u：连接的用户
- p：连接的用户密码
- P：连接的端口
- S：连接的socket文件
- t：需要归档的表
- i：使用的索引

## 示例

**归档到表**

创建目标表
```
CREATE TABLE `test_titles` (
  `emp_no` int(11) NOT NULL,
  `title` varchar(50) NOT NULL,
  `from_date` date NOT NULL,
  `to_date` date DEFAULT NULL,
  PRIMARY KEY (`emp_no`,`title`,`from_date`),
  KEY `idx_to_date` (`to_date`),
  KEY `idx_test` (`emp_no`,`title`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

批量归档
```
$ pt-archiver --source h=127.0.0.1,P=3306,u=root,D=employees,t=test_titles,A=utf8 --dest h=127.0.0.1,P=3308,u=admin,D=test,t=test_titles,A=utf8 --charset=utf8 --where "title = 'Engineer'" --progress=1000 --txn-size=1000 --limit=100 --no-delete --statistics  --bulk-insert --ask-pass
```

**归档到文件**
```
$ pt-archiver --source h=127.0.0.1,P=19529,u=root,D=employees,t=test_titles --file='/root/backup/test.txt' --output-format=dump --progress=1000 --where "to_date is null"  --txn-size=1000 --limit=100 --no-delete --statistics --ask-pass
```

==如果希望归档后删除需要去掉--no-delete选项==

**清理数据**

先执行--dry-run查看脚本
```
$ pt-archiver --source h=127.0.0.1,P=3306,u=admin,D=employees,t=employees,A=utf8 --purge --charset=utf8 --where "first_name = 'Anneke'" --progress=1000 --txn-size=1000 --limit=100 --statistics --ask-pass --dry-run
```

确认之后再清理
```
$ pt-archiver --source h=127.0.0.1,P=3306,u=admin,D=employees,t=employees,A=utf8 --purge --charset=utf8 --where "first_name = 'Anneke'" --progress=1000 --txn-size=1000 --limit=100 --bulk-delete --statistics --ask-pass
```


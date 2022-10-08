#MySQL #Table #Partition

# MySQL分区表
## 分区概述

如Oracle一样，MySQL也支持根据规则对表进行分区，分区后还是一张完整的表，只是将分区数据分散到不同的数据文件中。表分区带来的优化在于当查询条件包含分区键时，性能得到提升。并且我们也可以单独对指定分区进行操作，提升了分区表的可操作性，例如单独清理往年历史分区数据。
```
root@test 15:44:  CREATE TABLE members (
    ->     firstname VARCHAR(25) NOT NULL,
    ->     lastname VARCHAR(25) NOT NULL,
    ->     username VARCHAR(16) NOT NULL,
    ->     email VARCHAR(35),
    ->     joined DATE NOT NULL
    -> )
    -> PARTITION BY RANGE( YEAR(joined) ) (
    ->     PARTITION p0 VALUES LESS THAN (1960),
    ->     PARTITION p1 VALUES LESS THAN (1970),
    ->     PARTITION p2 VALUES LESS THAN (1980),
    ->     PARTITION p3 VALUES LESS THAN (1990),
    ->     PARTITION p4 VALUES LESS THAN MAXVALUE
    -> )

[root@t-luhx01-v-szzb test]# ls -lrt
-rw-r----- 1 mysql mysql  8712 Oct 12 15:44 members.frm
-rw-r----- 1 mysql mysql 98304 Oct 12 15:44 members#P#p1.ibd
-rw-r----- 1 mysql mysql 98304 Oct 12 15:44 members#P#p2.ibd
-rw-r----- 1 mysql mysql 98304 Oct 12 15:44 members#P#p3.ibd
-rw-r----- 1 mysql mysql 98304 Oct 12 15:44 members#P#p4.ibd
-rw-r----- 1 mysql mysql 98304 Oct 12 15:44 members#P#p0.ibd
```
我们可以通过show plugins查看插件的方式，查看当前是否支持分区表
```
root@test 15:44:  show plugins;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
| partition                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+
```

分区注意事项：
- 存储引擎应当选择INNODB，我们遇到过大量MyISAM分区表导致文件描述符耗尽的问题，因为MyISAM无论是否需要访问的分区都需要占用2个文件描述符，可以通过ALTER TABLE ENGINE=INNODB重建分区表
- 分区表的分区表达式中使用的所有列必须是该表可能具有的每个唯一键的一部分(包括主键)。如果表上没有唯一键，则不受限制
- InnoDB外键和MySQL分区不兼容。 分区的InnoDB表不能具有外键引用，也不能具有由外键引用的列。具有或由外键引用的InnoDB表无法分区


## 分区类型

MySQL支持以下分区类型：
- 范围分区：根据给定范围的列值进行分区
- LIST分区：根据给定的一组离散值进行分区
- HASH分区：指定分区数，根据表达式返回的值进行分区，分区键类型需要为整数
- KEY分区：与HASH分区类型，不同之处在于KEY分区允许多个分区列，列可以包含非整数值
- 子分区：在分区表的基础上，进一步细化分区

### 范围分区

每个分区都包含分区表达式值的给定范围内的数据行，范围应该是连续不重复的，通过VALUES LESS THAN来界定分区范围。下面的示例中，我们分成了4个分区
```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL
)
PARTITION BY RANGE (store_id) (
    PARTITION p0 VALUES LESS THAN (6),
    PARTITION p1 VALUES LESS THAN (11),
    PARTITION p2 VALUES LESS THAN (16),
    PARTITION p3 VALUES LESS THAN (21)
);
```
store_id中1-5的记录保存在P0分区，6-10的记录保存在P1分区，以此类推。但是如果此时插入store_id大于20的记录则会发生错误，这是因为我们并未定义大于20的分区，MySQL不知道该如何保存。这时可以通过指定MAXVALUE分区来避免该错误，MAXVALUE总是大于可能的最大整数值
```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT NOT NULL,
    store_id INT NOT NULL
)
PARTITION BY RANGE (store_id) (
    PARTITION p0 VALUES LESS THAN (6),
    PARTITION p1 VALUES LESS THAN (11),
    PARTITION p2 VALUES LESS THAN (16),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```
范围分区也可以基于表达式，但是MySQL必须能够计算表达式的范围值用于LESS THAN的比较。例如我可以按照日期里的年份来做分区
```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY RANGE ( YEAR(separated) ) (
    PARTITION p0 VALUES LESS THAN (1991),
    PARTITION p1 VALUES LESS THAN (1996),
    PARTITION p2 VALUES LESS THAN (2001),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```
> 对于TIMESTAMP类型，我们可以基于UNIX_TIMESTAMP()函数对时间戳进行分区

**范围列分区**

范围列分区是范围分区的一种变体，其与范围分区的区别如下：
- 范围列分区不支持表达式
- 范围列分区支持一个或多个列
- 除了整数列外，还支持日期、时间和字符串作为分区列
```
CREATE TABLE table_name
PARTITIONED BY RANGE COLUMNS(column_list) (
    PARTITION partition_name VALUES LESS THAN (value_list)[,
    PARTITION partition_name VALUES LESS THAN (value_list)][,
    ...]
)

column_list:
    column_name[, column_name][, ...]

value_list:
    value[, value][, ...]
```
column_list可以指定一个或多个分区列，value_list则是值列表，column_list和value_list的数量及顺序需要保持一致，并且数据类型也要一致。

在范围分区中，永远不会将分区范围的极值插入当前分区中，例如LESS THAN(10)，则10这条记录不会放到当前分区中。但在范围列分区中，可能会将值列表的第一个元素值与less than列表中的第一个元素相等的记录存放在当前分区。

例如我们现在定义下列分区
```
PARTITION BY RANGE COLUMNS(a, b) (
    PARTITION p0 VALUES LESS THAN (5, 12),
    PARTITION p1 VALUES LESS THAN (MAXVALUE, MAXVALUE)
)
```
现在插入下列数据，其中前两条数据会记录在p0中，最后这条记录会保存在p1中
```
INSERT INTO rc1 VALUES (5,10), (5,11), (5,12);
```
范围列分区中，其中一个列或多个列的限制值可以在分区定义中重复出现，只要用于定义分区的列值元组严格进行增加，就不会产生错误
```
CREATE TABLE tab1 (
    a INT,
    b INT,
    c INT
)
PARTITION BY RANGE COLUMNS(a,b,c) (
    PARTITION p0 VALUES LESS THAN (0,25,50),
    PARTITION p1 VALUES LESS THAN (10,20,100),
    PARTITION p2 VALUES LESS THAN (10,30,50)
    PARTITION p3 VALUES LESS THAN (MAXVALUE,MAXVALUE,MAXVALUE)
 );
```
在上诉示例中，并不会出现错误，虽然看起来很怪异，我们可以通过下列方式验证一下：
```
SELECT (0,25,50) < (10,20,100), (10,20,100) < (10,30,50);
+-------------------------+--------------------------+
| (0,25,50) < (10,20,100) | (10,20,100) < (10,30,50) |
+-------------------------+--------------------------+
|                       1 |                        1 |
+-------------------------+--------------------------+
```

**QUESTION：何时采用范围分区？**

1. 希望定期清理历史数据。例如希望仅保留一年的数据，我们可以通过ALTER TABLE DROP PARTITION来删除一年前的分区
2. 分区键包含日期或时间值的列，或其它一些序列值
3. 经常通过分区列查询分区数据，MySQL能够快速定位分区，提升查询性能

### LIST分区

与范围分区类似，必须显示定义每个分区，其区别在于LIST分区，每个分区都是根据一组值列表的成员关系来定义和选择的，而不是一组连续的范围值。LIST分区通过PARTITION BY LIST(expr)来完成的，其中expr是一个列值或基于列值返回一个整数值的表达式，然后通过value list中的值定义每个分区，value list即一个逗号分隔的整数列表
```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY LIST(store_id) (
    PARTITION pNorth VALUES IN (3,5,6,9,17),
    PARTITION pEast VALUES IN (1,2,10,11,19,20),
    PARTITION pWest VALUES IN (4,12,13,14,18),
    PARTITION pCentral VALUES IN (7,8,15,16)
);
```
与范围分区不同，LIST分区没有MAXVALUE，LIST分区所有可能产生的值都应该包含在分区中，如果插入分区不存在的值，则会插入报错

**LIST列分区**

LIST列分区是LIST分区的一种变体，它允许多个列作为分区键，并且支持整数类型以外的数据类型，例如字符串，日期时间
```
CREATE TABLE customers_1 (
    first_name VARCHAR(25),
    last_name VARCHAR(25),
    street_1 VARCHAR(30),
    street_2 VARCHAR(30),
    city VARCHAR(15),
    renewal DATE
)
PARTITION BY LIST COLUMNS(city) (
    PARTITION pRegion_1 VALUES IN('Oskarshamn', 'Högsby', 'Mönsterås'),
    PARTITION pRegion_2 VALUES IN('Vimmerby', 'Hultsfred', 'Västervik'),
    PARTITION pRegion_3 VALUES IN('Nässjö', 'Eksjö', 'Vetlanda'),
    PARTITION pRegion_4 VALUES IN('Uppvidinge', 'Alvesta', 'Växjo')
);
```

### HASH分区

HASH分区可以确保分区之间均匀分布数据，并且无需指定每个分区的规则，只需要指定分区列以及分区数量。
```
CREATE TABLE tablename (
    columns...
)
PARTITION BY HASH(expr)
PARTITIONS number;
```
其中expr为整数类型的字段或者返回整数类型的表达式，表达式要求返回具有确定性的整数值，每次插入或更新时，都会计算该表达式，因此太过复杂的表达式可能导致性能问题；number为分区数量，如果不指定PARTITIONS，则默认为1个分区，其实通过MOD求余来确定分区

最高效的hash分区，是对单个表列进行操作，其值随列值一致增加或减少。也就是说，列值相对于表达式的值的图越接近直线，就越适合于哈希。例如to_days(date)函数，它总是随着date列的修改以一致的方式进行修改。相反，POW（5-int_col，3）+6就是一个较差的表达式。

**线性HASH分区**

MySQL还支持线性HASH，这与HASH的不同之处在于，线性HASH使用线性二乘幂算法，而HASH则使用HASH函数值的模数
```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY LINEAR HASH( YEAR(hired) )
PARTITIONS 4;
```
其中的分区号N根据下列算法计算：

1. 查找下一个大于2的幂，记录为V。V=POWER(2,CEILING(LOG(2,NUMBER)))
2. N=F(column_list) & (v-1)
3. 当N大于等于number，V=V/2 ,N=N & (V-1)

假设NUMBER为6，则V=8，N=YEAR('2003-04-14') & (8 - 1)=3，则保存在第三个分区中；N=YEAR('1998-10-19') & (8 - 1)=6，大于等于number需要进一步计算，N=6 & ((8 / 2) - 1)=2，则保存在2号分区中

使用线性哈希进行分区的好处是，分区的添加、删除、合并和分割速度更快，这在处理包含大量(TB)数据的表时非常有用。其缺点是，与使用常规哈希分区获得的分布相比，数据不太可能均匀地分布在分区之间

### KEY分区

KEY分区类似于HASH分区，不同之处在于HASH分区采用用户定义的表达式，而KEY分区使用的HASH函数由MySQL提供，NDB Cluster使用MD5()，其它存储引擎使用了与PASSWORD()相同的算法。

KEY接收零个或多个列，用作分区键的列必须包含部分或全部主键列，如果没有指定列作为分区表的话，则默认使用主键，如果不存在主键则使用唯一索引列，但必须设置为not null
```
CREATE TABLE k1 (
    id INT NOT NULL PRIMARY KEY,
    name VARCHAR(20)
)
PARTITION BY KEY()
PARTITIONS 2;
```

分区键除了支持整数类型和NULL，同样也支持字符等其它类型。但KEY分区不支持带索引前缀的列，因此text和blob类型并不支持。

KEY分区也支持线性KEY分区，使用LINEAR对KEY分区的影响与对HASH分区的影响相同，分区号是使用二乘幂算法而不是取模算法得出的

### 子分区

子分区也称为复合分区，是对分区表进一步进行分区。在MySQL 5.7中，可以对按范围或列表分区的表进行子分区，子分区可以使用HASH或KEY分区
```
CREATE TABLE ts (id INT, purchased DATE)
    PARTITION BY RANGE( YEAR(purchased) )
    SUBPARTITION BY HASH( TO_DAYS(purchased) )
    SUBPARTITIONS 2 (
        PARTITION p0 VALUES LESS THAN (1990),
        PARTITION p1 VALUES LESS THAN (2000),
        PARTITION p2 VALUES LESS THAN MAXVALUE
    );
```
- 每个分区都必须拥有相同数量的子分区
- 每个子分区子句必须(至少)包含子分区的名称
- 整个表中的子分区名称必须是惟一的
- MyISAM的分区表子分区可以单独设置DATA DIRECTORY和INDEX DIRECTORY将文件存放在不同磁盘上

### 分区与NULL

在MySQL中进行分区并不会禁止将NULL作为分区表达式的值，无论它是列值还是用户定义的表达式，但NULL并不是数字，MySQL将其视作小于如何非NULL的值，与ORDER BY中的行为一致

不同类型的针对NULL的处理方式也是不同的：
- 对于范围分区来说，它会将为NULL的数据行保存到最小的分区中
- 对于LIST分区来说，如果有分区显示定义了NULL，则LIST分区允许NULL值；否则，就拒绝插入
- 对于HASH分区和LIST分区来说，会将生成NULL值的分区表达式都看作其返回值为0

## 管理分区表

**删除分区**

范围分区和LIST分区采用以下方式删除，分区删除后，分区对应的数据也一并被删除
```
ALTER TABLE t1 DROP PARTITION p2;
```
HASH分区和KEY分区采用以下方式删除
```
ALTER TABLE t1 COALESCE PARTITION 4;
```

**截断分区**
```
ALTER TABLE t1 TRUNCATE PARTITION p1;
```

**移除分区**
```
ALTER TABLE t1 REMOVE PARTITIONING;
```
>移除分区仅修改表分区定义，并不清理数据

**添加分区**

范围分区和LIST分区采用下列方式添加分区
```
ALTER TABLE members ADD PARTITION (PARTITION p3 VALUES LESS THAN (2010));
```
HASH分区和KEY分区采用下列方式添加，number为要增加的分区数量
```
ALTER TABLE clients ADD PARTITION PARTITIONS number;
```

**重整分区**

无法在现有分区中间或之前添加分区，可以通过REORGANIZE PARTITION来重新组织分区
```
ALTER TABLE members REORGANIZE PARTITION p1 into(PARTITION p1 VALUES LESS THAN (1985),PARTITION p4 VALUES LESS THAN (1990));
```
REORGANIZE PARTITION也可以用于合并分区
```
ALTER TABLE members REORGANIZE PARTITION p1,p4 INTO (
    PARTITION p1 VALUES LESS THAN (1990)
)
```

**交换分区**

将pt分区表的分区p转换为普通表nt，需要满足下列要求：
- nt表未分区
- nt表不能是临时表
- pt分区表与nt表结构一致
- nt表不能含有外键或被引用的外键列
- 对于INNODB，需要使用相同的行格式

```
ALTER TABLE pt
    EXCHANGE PARTITION p
    WITH TABLE nt;
```

如果nt表中已经包含数据了，当exchange执行后，p分区与nt表数据发生交换，需要注意的是nt表的数据不能超过p分区的边界，否则会出现错误(设置 WITHOUT VALIDATION选项不检查行的情况下可以执行)

**重建分区**
```
ALTER TABLE t1 REBUILD PARTITION p0, p1;
```

**分析分区**
```
ALTER TABLE t1 ANALYZE PARTITION p3;
```

**修复分区**
```
ALTER TABLE t1 REPAIR PARTITION p0,p1;
```

**检查分区**
```
ALTER TABLE t1 CHECK PARTITION p1;
```
>也可以使用ALL关键字替换具体分区，将作用于整张表

**优化分区**

当从分区中删除了大量数据，可以通过OPTIMIZE回收未使用的空间，并且整理数据文件碎片。
```
ALTER TABLE t1 OPTIMIZE PARTITION p0, p1;
```
OPTIMIZE PARTITION包括CHECK PARTITION、ANALYZE PARTITION和REPAIR PARTITION

INNODB并不支持按分区OPTIMIZE，这种情况下将分析并重建整张表，我们可以通过REBUILD PARTITION和ANALYZE PARTITION来避免

**查看分区信息**

通过`show create table`我们可以查看建表语句中分区的定义，当然我们也可以采用show table status的方式查看
```
root@test 14:06:  show create table members;
+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
+---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| members | CREATE TABLE `members` (
  `id` int(11) DEFAULT NULL,
  `fname` varchar(25) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `lname` varchar(25) COLLATE utf8mb4_unicode_ci DEFAULT NULL,
  `dob` date DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
/*!50100 PARTITION BY RANGE ( YEAR(dob))
(PARTITION p0 VALUES LESS THAN (1980) ENGINE = InnoDB,
 PARTITION p1 VALUES LESS THAN (1990) ENGINE = InnoDB,
 PARTITION p2 VALUES LESS THAN (2000) ENGINE = InnoDB) */ |
```

更多分区表内容请查阅[Partitioning](https://dev.mysql.com/doc/refman/5.7/en/partitioning.html)

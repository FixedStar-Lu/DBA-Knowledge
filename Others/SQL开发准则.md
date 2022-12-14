- ## 1. 基本概述

  ### 1.1 术语说明

  - 强制：需严格遵守，如有违反需业务负责人确认
  - 推荐：最佳实践，建议遵守。如果有更好的方案，与规范负责人确认，更新文档
  - 参考：可根据实际情况进行选择

  ### 1.2 撰写目的

  为了避免数据库对象杂乱，SQL性能下降，管理成本升高等因素，特撰写下列规范内容。旨在形成一个有效的，具有指导意义的共识，能够帮我们在日常开发或运维过程中避免一些不必要的问题，建立一个稳定，高效的业务环境。

  ## 2. 表设计规范

  **[强制]** 每张表都必须有主键，在使用自增值做主键时，一定要设置为bigint类型

  **[强制]** 命名采用字母和'_'(下划线)组成，并且全部为小写字母。禁止使用中文命名

  **[强制]** 命名不应该使用数据库保留字。例如group，desc，order等。MySQL保留字可查看：https://dev.mysql.com/doc/refman/5.7/en/keywords.html

  **[强制]** 命名要见名知意，突出其功能模块及作用，长度保持在30个字符以内

  **[强制]** 表达是与否的概念，应该以is_xxx的方式命名

  **[强制]** 禁止使用主外键约束

  **[强制]** 字段类型的选择参考下列方案：

  - 小数采用DECIMAL高精度类型，禁止使用float和double
  - 金额字段使用INT类型。如保留两位小数，可以乘以100再存入数据表
  - varchar(N)中的N表示字符数而非字节数，应当明确长度上限避免空间浪费。
  - IP采用Int存储
  - 使用tinyint代替enum和boolean

  **[强制]** 表引擎使用InnoDB，禁止使用MyISAM

  **[强制]** 字符集统一采用utf8mb4，支持汉字及emoj表情

  **[强制]** 每张表及每个字段都需要有明确的注释

  **[强制]** 不同国家之间相同的库中表和字段需保持一致

  **[强制]** 禁止使用blob大对象来存放图片，文件内容等信息，最佳实践是保存S3或其它存储介质的访问路径

  **[推荐]** 禁止密码等隐私字段明文存储，应该考虑通过盐值+动态加密算法的方式，确保账户安全

  **[推荐]** 建议每张表都设计create_time和update_time字段，保存每条记录的创建时间和最后更新时间

  **[推荐]** 临时表或备份表应存放在tmp库内，并指定为tmp_表名_日期或bak_表名_日期

  **[推荐]** 字段允许适当的冗余，以提高查询性能，但必须考虑数据一致，冗余字段应遵循

  - 不是频繁修改的字段
  - 不是唯一索引字段
  - 不是varchar字段或text字段

  **[参考]** 对需要参与数值计算的整数列，不推荐使用Unsigned属性，如需使用应在sql_mode加上NO_UNSIGNED_SUBTRACTION选项，否则可能出现如下错误

  ```
  ERROR 1690 (22003): ``BIGINT` `UNSIGNED value ``is` `out` `of` `range ``in` `'(`test`.`s2`.`sale_count` - `test`.`s1`.`sale_count`)'
  ```

  **[参考]** 合适的字符存储长度，不但节约数据库表空间、索引空间等，应该贴近实际需求，不宜设置过长。满足要求的前提下，类型越小越好，能用tinyint就不用int，能用Int就不用varchar
  

  **建表示例**

  ```
  CREATE` `TABLE` ``tab1```(``  ```id`       ``bigint` `NOT` `NULL` `AUTO_INCREMENT COMMENT ``'主键ID'``,``  ```member_id`    ``varchar``(50) ``NOT` `NULL` `COMMENT ``'会员id'``,``  ```gaid`      ``varchar``(50) ``NULL` `COMMENT ``'广告id'``,``  ```bvn`       ``varchar``(255) ``NULL` `COMMENT ``'bvn'``,``  ```is_delete`    tinyint ``NULL` `COMMENT ``'是否删除：1为YES，0为NO'``,``  ```register_time`  ``timestamp`  `DEFAULT` `NULL` `COMMENT ``'注册时间'``,``  ```create_time`   ``timestamp`  `NOT` `NULL` `DEFAULT` `CURRENT_TIMESTAMP` `COMMENT ``'创建时间'``,``  ```update_time`   ``timestamp`  `DEFAULT` `NULL` `ON` `UPDATE` `CURRENT_TIMESTAMP` `COMMENT ``'更新时间'``,``  ``PRIMARY` `KEY` `(`id`),``  ``UNIQUE` `KEY` ``uk_member` (`member_id`),``  ``KEY` ``idx_bvn` (`bvn`)``) ENGINE = InnoDB COMMENT =``'表注释'``;
  ```

  ## 3. 索引规范

  **[强制]** 主键索引命名以PK_开头，唯一索引以uk_开头，二级索引以idx_开头

  **[强制]** 设计复合索引时应遵循以下规则：

  - 根据最左前缀原则，将复用率高且索引基数高的字段放在最左边
  - 复合索引的最左字段应该采用等值条件，否则后续的索引字段将无法被利用
  - 复合索引的字段数不宜过多，否则索引过大
  - 避免创建冗余索引，例如建立了(a,b)组合索引，(a)索引就不需要了

  **[强制]** 针对布尔类型或枚举等可选值较少的情况下不应创建独立索引

  **[强制]** 用于Join关联的字段必须存在索引

  **[强制]** 业务上具有唯一特性的字段，即使是组合字段也应该创建成一索引

  **[推荐]** 尽可能利用覆盖索引的特性，避免回表

  **[推荐]** 针对较长的varchar字段，在满足区分度要求的情况下，应采用前缀索引来避免索引过长，原则上不超过128。可以根据select count(distinct column)/count(*)的结果要评估

  **[参考]** 创建索引时应避免如下极端误解：

  - 索引越多越好，认为一个查询就需要一个对应的索引
  - 认为索引会占用大量空间，影响数据更新以及插入性能
  - 抵制唯一索引
  - 查询用到的条件都必须加入组合索引，实际上好的过滤条件通过索引能过滤绝大部分数据，因此我们更应该关注如何设计出高效的索引

  ## 4. SQL规范

  **[强制]** 查询数据时，禁止采用select *语法，而是根据实际需求声明必要的返回字段

  **[强制]** 禁止对WHERE条件字段进行函数转换和计算，会导致无法利用索引。具体可看[附录一]

  **[强制]** 禁止任何不带WHERE条件的SQL

  **[强制]** 禁止使用count(字段)或count(常量)，应采用count(*)或count(1)

  **[强制]** 禁止超过3张表的Join操作

  **[强制]** INSERT语句必须明确声明插入的字段

  **[强制]** 禁止采用limit offset进行分页查询，具体优化措施可参考[附录二]

  **[强制]** 严禁使用like '%aaa' 或like ‘%aaa%'的模糊匹配查询

  **[强制]** 在不确定SQL性能情况下，需要先通过explain查看执行计划，并优化可能存在的问题。执行计划分析可参考：[附录三]

  **[强制]** 在DDL语句中，应当显示添加ALGORITHM=inplace,LOCK=NONE来确定是否为ONLINE DDL，如果SQL不支持inplace，应当考虑通过工具来实现，避免锁表。

  **[推荐]** 尽可能不使用存储过程，存储过程难以调试和扩展，不具备移植性

  **[推荐]** 尽量避免使用in，即使使用也需要限制元素数量在1000以内

  **[推荐]** 修正数据时(DML)，要先select确认数据，再执行更新

  **[推荐]** 判断非空时应采用ISNULL()函数

  **[推荐]** 为了减少与数据库的交互次数，提升执行效率，应采用批量式的操作。例如针对单张表的多个insert语句或添加多个字段的DDL操作

  **[推荐]** 针对order by的字段，尽量利用索引有序的特性来避免额外的排序

  **[参考]** group by如果对结果没有排序要求，可指定`order by null`来消除group by默认的排序消耗

  ## 5. 事务设计

  **[强制]** 禁止设计过大的事务，应该从业务逻辑上进行拆分

  **[强制]** 事务中的操作必须具有相同的维度，避免不相关的操作进入事务

  **[强制]** 事务与事务之间应该做到无关联或弱关联，避免因为事务互相等待引起的锁等待或死锁

  **[推荐]** 事务中耗时较长的操作应该放在事务的最后来执行，避免长时间占有锁

  ## 6. 附录

  ### 6.1 无法利用索引的几种情况

  **1）条件字段函数操作**

  假设数据库存在如下表

  ```
  CREATE` `TABLE` ``tradelog` (`` ```id` ``int``(11) ``NOT` `NULL``,`` ```tradeid` ``varchar``(32) ``DEFAULT` `NULL``,`` ```operator` ``int``(11) ``DEFAULT` `NULL``,`` ```t_modified` datetime ``DEFAULT` `NULL``,`` ``PRIMARY` `KEY` `(`id`),`` ``KEY` ``tradeid` (`tradeid`),`` ``KEY` ``t_modified` (`t_modified`)``) ENGINE=InnoDB ``DEFAULT` `CHARSET=utf8mb4;
  ```

  现在通过下列方式查询7月份的数据，会发现SQL并不走tradeid索引而是走PRIMARY，原因正是因为对条件字段做了函数操作

  **2）隐式类型转换**

  现通过下列方式查询tradeid=110717的数据

  ```
  select` `* ``from` `tradelog ``where` `tradeid=110717;
  ```

  tradeid上存在索引，但SQL实际会进行全表扫描，这是因为tradeid定义为varchar，但传入的是INT，会自动进行转换，相当于下列语句

  ```
  select` `* ``from` `tradelog ``where` `CAST``(tradid ``AS` `signed ``int``) = 110717;
  ```

  索引本质上还是对条件字段做了函数操作。除此之外字符集格式不同也可能导致隐式转换从而无法利用索引

  ### 6.2 分页优化

  分页查询可以通过LIMIT M,N的方式，通过不断移动M的偏移量来实现翻页。但实际运行过程中，往往分页性能会随着M的增加而变得缓慢，甚至最后直接通过PRIMARY扫描来进行查询。例如下面SQL

  ```
  mysql> explain ``select` `emp_no,from_date ``from` `salaries limit 1000000,10;``+``----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+``| id | select_type | ``table`  `| partitions | type | possible_keys | ``key`   `| key_len | ref | ``rows`  `| filtered | Extra    |``+``----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+``| 1 | SIMPLE   | salaries | ``NULL`    `| ``index` `| ``NULL`     `| ``PRIMARY` `| 7    | ``NULL` `| 2838426 |  100.00 | Using ``index` `|``+``----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
  ```

  **深翻页**

  深翻页即上下翻页，用户通过点击上一页或下一页来获取临近页的数据，这种情况如果表中存在自增主键或具有唯一性、递增性的字段，例如主键自增ID，就可以通过获取当前页的最小ID和最大ID，来确定下一页的边界值，这样通过ID辅助完成分页，能够大大的提升性能。

  假设目前对salaries表进行分页，100条记录为一页，当前第10页的最小ID为900，最大ID为1000，那第11页则为

  ```
  mysql> ``select` `* ``from` `salaries ``where` `id> 1000 ``and` `to_date=``'1985-04-09'` `order` `by` `id ``asc` `limit 0,100;
  ```

  查询第9页则为

  ```
  mysql> ``select` `* ``from` `salaries ``where` `id<900 ``and` `to_date=``'1985-04-09'` `order` `by` `id ``desc` `limit 0,100;
  ```

  **跳页查询**

  跳页查询即提供页码查询，可以自由选择查询第2页或第10页，不用再顺序点击下一页或上一页，这样情况用户点击的页码是不确定的，因此还需要针对这种情况进行优化。具体的优化思想就是在深翻页的基础上将每5页作为一个分页区，计算当前页的最小ID和最大ID，然后通过增加LIMIT M,N中的M值在分页区内进行查询。由于分页区只包含5页，因此M的量整体不会很大

  假设目前对salaries表进行分页，100条记录为一页，5页为一个分页区，当前第10页的最小ID为900，最大ID为1000，那第8页为

  ```
  mysql> ``select` `* ``from` `salaries ``where` `id<900 ``and` `to_date=``'1985-04-09'` `order` `by` `id ``desc` `limit 100,100;
  ```

  第12页为

  ```
  mysql> ``select` `* ``from` `salaries ``where` `id> 1000 ``and` `to_date=``'1985-04-09'` `order` `by` `id ``asc` `limit 0,100;
  ```

  ### 6.3 执行计划分析

  执行计划用于分析SQL执行过程，包括连接方式、索引、扫描行数、临时表等信息。因此，执行计划作为SQL性能的重要参考依据

  ```
  mysql> explain ``select` `emp_no,from_date ``from` `salaries limit 1000000,10;``+``----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+``| id | select_type | ``table`  `| partitions | type | possible_keys | ``key`   `| key_len | ref | ``rows`  `| filtered | Extra    |``+``----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+``| 1 | SIMPLE   | salaries | ``NULL`    `| ``index` `| ``NULL`     `| ``PRIMARY` `| 7    | ``NULL` `| 2838426 |  100.00 | Using ``index` `|``+``----+-------------+----------+------------+-------+---------------+---------+---------+------+---------+----------+-------------+
  ```

  **ID**

  - ID列中的如果数据为一组数字，表示执行SELECT语句的顺序；如果为NULL，则说明这一行数据是由另外两个SQL语句进行 UNION操作后产生的结果集
  - ID值相同时，说明SQL执行顺序是按照显示的从上至下执行的
  - ID值不同时，ID值越大代表优先级越高，则越先被执行

  **SELECT_TYPE**

  | value              | 说明                                                        |
  | :----------------- | :---------------------------------------------------------- |
  | SIMPLE             | 不包含子查询或是UNION操作的查询                             |
  | PRIMARY            | 查询中如果包含任何子查询，那么最外层的查询则被标记为PRIMARY |
  | SUBQUERY           | SELECT 列表中的子查询                                       |
  | DEPENDENT SUBQUERY | 依赖外部结果的子查询，效率非常慢                            |
  | UNION              | Union操作的第二个或是之后的查询的值为union                  |
  | DEPENDENT UNION    | 当UNION作为子查询时，第二或是第二个后的查询的select_type值  |
  | UNION RESULT       | UNION产生的结果集                                           |
  | DERIVED            | 出现在FROM子句中的子查询                                    |

  **TYPE**

  访问类型，SQL 查询优化中一个很重要的指标，结果值从好到坏依次是：system > const > eq_ref > ref > range > index > ALL

  | value       | 说明                                                         |
  | :---------- | :----------------------------------------------------------- |
  | system      | 这是const联接类型的一个特例，当查询的表只有一行时使用        |
  | const       | 表中有且只有一个匹配的行时使用，如对主键或是唯一索引的查询，这是效率最高的联接方式 |
  | eq_ref      | 唯一索引或主键索引查询，对应每个索引键，表中只有一条记录与之匹配 |
  | ref         | 非唯一索引查找，返回匹配某个单独值的所有行                   |
  | ref_or_null | 类似于ref类型的查询，但是附加了对NULL值列的查询              |
  | index_merge | 该联接类型表示使用了索引合并优化方法                         |
  | range       | 索引范围扫描，常见于between、>、<这样的查询条件              |
  | index       | FULL index Scan 全索引扫描，同ALL的区别是，遍历的是二级索引树 |
  | ALL         | FULL TABLE Scan 全表扫描，这是效率最差的联接方式             |

  **KEY**

  执行计划选择的索引，未被选中的索引可以查看possible_keys

  **ROWS**

  预估的扫描行数，不一定准确，但对优化器的选择有一定影响

  **Filetered**

  - 表示返回结果的行数占需读取行数的百分比
  - Filtered列的值越大越好（值越大，表明实际读取的行数与所需要返回的行数越接近）
  - Filtered列的值依赖统计信息，所以同样也不是十分准确，只是一个参考值

  **Extra**

  | value                        | 说明                                                         |
  | :--------------------------- | :----------------------------------------------------------- |
  | Distinct                     | 优化distinct操作，在找到第一个匹配的元素后即停止查找         |
  | Not exists                   | 使用not exists来优化查询                                     |
  | Using filesort               | 使用额外操作进行排序，通常会出现在order by或group by查询中   |
  | Using index                  | 使用了覆盖索引进行查询                                       |
  | Using index condition        | 使用了索引下推优化                                           |
  | Using temporary              | MySQL需要使用临时表来处理查询，常见于排序，子查询，和分组查询 |
  | Using where                  | 需要在MySQL服务器层使用WHERE条件来过滤数据                   |
  | select tables optimized away | 直接通过索引来获得数据，不用访问表，这种情况通常效率是最高的 |
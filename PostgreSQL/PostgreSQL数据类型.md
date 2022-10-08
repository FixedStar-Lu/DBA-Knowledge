[TOC]

# PostgreSQL数据类型

### 数值类型

数值类型由两字节、四字节、八字节整数、四字节和八字节浮点数以及可选精度小数组成。下表列出了可用的类型

| 类型             | 字节数 | 范围                                        |
| :--------------- | :----- | :------------------------------------------ |
| smallint         | 2bytes | -32768 to +32767                            |
| integer          | 4bytes | -2147483648 to +2147483647                  |
| bigint           | 8bytes | -9223372036854775808 to 9223372036854775807 |
| decimal          | 可变   | 小数点前131072位;小数点后的16383位          |
| numeric          | 可变   | 小数点前131072位;小数点后的16383位          |
| real             | 4bytes | 6位小数精度                                 |
| double precision | 8bytes | 15位小数精度                                |
| smallserial      | 2bytes | 1 to 32767                                  |
| serial           | 4bytes | 1 to 2147483647                             |
| bigserial        | 8bytes | 1 to 9223372036854775807                    |



- smallint存储2个字节的整数，也可以定义为int2；integer存储4个字节的整数，定义可以写成int4，bigint存储8个字节整数，定义可以写成int8，一般情况下integer就足够了
- decimal和numeric是等效的，可以存储指定精度的小数。numeric(precision,scale)中precision表示总的数值位数，scale指小数部分的位数
- real和double precision是浮点数据类型，real支持4字节，double precision支持8字节
- smallserial、serial和bigserial为自增序列类型

> 货币类型以固定的精度存储货币数量。numeric、int和bigint数据类型的值可以转换为money。不建议使用浮点数处理金钱，因为可能会出现舍入错误

**数值运算**

PostgreSQL中数值类型支持多种运算，例如加、减、乘、除、取余等

```
test@[local]:3433=#select 1+2,2*2,4/2,5-1;
 ?column? | ?column? | ?column? | ?column? 
----------+----------+----------+----------
        3 |        4 |        2 |        4
```

取余

```
test@[local]:3433=#select mod(8,3);
 mod 
-----
   2
```

四舍五入函数

```
test@[local]:3433=#select round(7.8),round(8.3);
 round | round 
-------+-------
     8 |     8
```

返回大于或等于指定值的最小整数

```
test@[local]:3433=#select ceil(2.4),ceil(-2.4);
 ceil | ceil 
------+------
    3 |   -2
```

返回小于等于指定值的最大整数

```
test@[local]:3433=#select floor(2.4),floor(-2.4);
 floor | floor 
-------+-------
     2 |    -3
```

### 字符类型

下表列出了PostgreSQL中可用的通用字符类型

| 类型                             | 字节数               |
| :------------------------------- | :------------------- |
| character varying(n), varchar(n) | 可变长度             |
| character(n), char(n)            | 固定长度，以空格填充 |
| text                             | 无限制长度           |

varchar(n)存储的变长字符类型，N是一个整数，当存储的字符串长度超过N则会报错，比N小时则存储实际的长度；char(n)存储定长字符，当存储的字符串长度超过N则会报错，小于N则用空白填充。

```
test@[local]:3433=#select char_length(a),char_length(b),octet_length(a),octet_length(b) from t2;
 char_length | char_length | octet_length | octet_length 
-------------+-------------+--------------+--------------
           1 |           1 |            1 |            5
```

> varchar(n)不声明N时，将存储任意长度的字符串，与text类型相似；char(n)不声明n时就相当于char(1)

**字符函数**

查询字符在字符串中的位置

```
test@[local]:3433=#select position('a' in 'abcde');
 position 
----------
        1
```

截取字符串的部分内容

```
test@[local]:3433=#select substring('luhengxing' from 3 for 4);
 substring 
-----------
 heng
```

根据指定分隔符拆分字符串，并返回指定字段

```
test@[local]:3433=#select split_part('heng@xing@qq.com','@',2);
 split_part 
------------
 xing
```

### 时间类型

PostgreSQL支持一组完整的SQL日期和时间类型，如下表所示。日期按照公历计算

| 类型                                 | 字节数  | 描述                   | 最小值           | 最大值          |
| :----------------------------------- | :------ | :--------------------- | :--------------- | :-------------- |
| timestamp [(p)] [without time zone ] | 8bytes  | 日期和时间(无时区)     | 公元前4713年     | 公元294276年    |
| timestamp [(p)] [with time zone]     | 8bytes  | 日期和时间，带时区     | 公元前4713年     | 公元294276年    |
| date                                 | 4bytes  | 日期（无时间）         | 公元前4713年     | 公元5874897年   |
| time [ (p)] [ without time zone ]    | 8bytes  | 一天中的时间（无日期） | 00:00:00         | 24:00:00        |
| time [ (p)] with time zone           | 12bytes | 一天中的时间，带时区   | 00:00:00+1459    | 24:00:00-1459   |
| interval [fields ] [(p) ]            | 12bytes | 时间间隔               | -178000000 years | 178000000 years |

timestamp without time zone

```
test@[local]:3433=#select now()::timestamp with time zone;
              now              
-------------------------------
 2020-09-16 15:02:43.515159+08
```

timestamp without time zone

```
test@[local:/service/pgsql/data]:3433=#select now()::timestamp without time zone;
            now            
---------------------------
 2020-09-16 15:03:15.96061
```

date

```
test@[local:/service/pgsql/data]:3433=#select now()::date;
    now     
------------
 2020-09-16
```

time with time zone

```
test@[local]:3433=#select now()::time with time zone;
        now         
--------------------
 15:04:23.212294+08
```

time without time zone

```
test@[local]:3433=#select now()::time without time zone;
       now       
-----------------
 15:04:51.824421
```

其中p是指时间精度，默认精度为6，我们可以设置为0

```
test@[local]:3433=#select now()::timestamp(0);
         now         
---------------------
 2020-09-16 15:07:41
```

interval是指时间间隔，其单位可以选择hour、day、month、year等

```
test@[local]:3433=#select now(),now()+interval'1 hour';
              now              |           ?column?            
-------------------------------+-------------------------------
 2020-09-16 15:09:22.088006+08 | 2020-09-16 16:09:22.088006+08
```

**日期运算**

增加日期

```
test@[local]:3433=#select now()+interval'1 day';
           ?column?            
-------------------------------
 2020-09-17 15:11:26.965942+08
```

减少日期

```
test@[local]:3433=#select now()-interval'1 day';
           ?column?            
-------------------------------
 2020-09-15 15:12:13.633149+08
```

日期相乘

```
test@[local]:3433=#select 100 * interval'2 second';
 ?column? 
----------
 00:03:20
```

日期相除

```
test@[local]:3433=#select interval'2 hour' / double precision '3';
 ?column? 
----------
 00:40:00
```

查看当前时间

```
test@[local]:3433=#select current_date,current_time;
 current_date |    current_time    
--------------+--------------------
 2020-09-16   | 15:16:04.220121+08
```

extract可以从日期时间中抽取年、月、日、时、分、秒信息

```
test@[local]:3433=#select extract( year from now());
 date_part 
-----------
      2020
```

当前日期在一年中的第几周

```
test@[local]:3433=#select extract(week from now());
 date_part 
-----------
        38
```

当天属于一年中的第几天

```
test@[local]:3433=#select extract(doy from now());
 date_part 
-----------
       260
```

### 网络地址类型

PostgreSQL提供了用于存储IPv4，IPv6和MAC地址的数据类型。 最好使用这些类型而不是纯文本类型来存储网络地址，因为这些类型提供输入错误检查以及专门的运算符和功能

| 类型    | 字节数       | 描述                 |
| :------ | :----------- | :------------------- |
| cidr    | 7 or 19bytes | IPv4和IPv6网络       |
| inet    | 7 or 19bytes | IPv4和IPv6主机和网络 |
| macaddr | 6bytes       | MAC地址              |

inet和cidr类型存储的网络格式为address/n，其中address表示IPv4或IPv6地址，n为网络掩码位数，默认IPv4为32，IPv6为128。inet和cidr更推荐使用cidr

### JSON/JSONB类型

json数据类型可用于存储json数据。这些数据也可以存储为文本，但是json数据类型的优点是检查每个存储的值是否是一个有效的json值。

创建JSON类型的表

```
test@[local]:3433=#create table t3 (id serial primary key,address json)
CREATE TABLE
test@[local]:3433=#insert into t3(address) values('{"country":"china","province":"guangdong"}');
INSERT 0 1
test@[local]:3433=#insert into t3(address) values('{"country":"china","province":"jiangxi"}');
INSERT 0 1
```

查询JSON数据

```
test@[local]:3433=#select address->'province' from t3;
  ?column?   
-------------
 "jiangxi"
 "guangdong"
```

PostgreSQL支持两种JSON格式：jsonb和json，两种类型主要的区别在于json存储格式为文本而jsonb存储格式为二进制，jsonl类型以文本存储并且存储的内容和输入数据一样，当解析JSON数据时需要重新解析，而jsonb以二进制形式存储已经解析好的数据。因此json插入更快，jsonb检索更快。

除此之外，jsonb输出的键的顺序和输入不同，而json是一致的

```
test@[local]:3433=#select '{"id":1,"name":"luhengxing","address":"jiangxi","age":18}'::jsonb;
                              jsonb                               
------------------------------------------------------------------
 {"id": 1, "age": 18, "name": "luhengxing", "address": "jiangxi"}
(1 row)

test@[local]:3433=#select '{"id":1,"name":"luhengxing","address":"jiangxi","age":18}'::json;
                           json                            
-----------------------------------------------------------
 {"id":1,"name":"luhengxing","address":"jiangxi","age":18}
```

jsonb也会删除输入数据中键值的空格，并且也会删除重复的键，仅保留最后一个

```
test@[local]:3433=#select '{"id":1,"age":20,"age":18}'::json;
            json            
----------------------------
 {"id":1,"age":20,"age":18}
(1 row)

test@[local]:3433=#select '{"id":1,"age":20,"age":18}'::jsonb;
        jsonb         
----------------------
 {"id": 1, "age": 18}
```

**JSON操作**

以文本格式返回JSON字段

```
test@[local]:3433=#select address->>'province' from t3;
 ?column?  
-----------
 jiangxi
 guangdong
```

判断字符串是否为顶层键值

```
test@[local]:3433=#select '{"a":1,"b":2}'::jsonb ? 'b';
 ?column? 
----------
 t
```

删除JSON键

```
test@[local]:3433=#select '{"a":1,"b":2}'::jsonb- 'b';
 ?column? 
----------
 {"a": 1}
```

扩展最外层JSON成为一组常规的结果集

```
test@[local]:3433=#select * from json_each('{"a":1,"b":2}');
 key | value 
-----+-------
 a   | 1
 b   | 2
```

将行数据转化为JSON格式输出

```
test@[local:/service/pgsql/data]:3433=#select row_to_json(t3) from t3;
                          row_to_json                          
---------------------------------------------------------------
 {"id":2,"address":{"country":"china","province":"jiangxi"}}
 {"id":1,"address":{"country":"china","province":"guangdong"}}
```

返回最外层键

```
test@[local]:3433=#select * from json_object_keys('{"a":1,"b":2}');
 json_object_keys 
------------------
 a
 b
```

JSONB添加键值，如果键值存在则为覆盖更新

```
test@[local]:3433=#select '{"name":"luhengxing","age":18}'::jsonb || '{"country":"china"}'::jsonb;
                       ?column?                        
-------------------------------------------------------
 {"age": 18, "name": "luhengxing", "country": "china"}
```

JSONB删除键值

```
test@[local]:3433=#select '{"name":"luhengxing","age":18}'::jsonb - 'age';
        ?column?        
------------------------
 {"name": "luhengxing"}
```

嵌套JSON删除键值

```
test@[local:/service/pgsql/data]:3433=#select '{"id":1,"address":{"country":"china","province":"guangdong"}}'::jsonb #- '{address,province}' ::text[];
                  ?column?                  
--------------------------------------------
 {"id": 1, "address": {"country": "china"}}
```

### 数组类型

PostgreSQL提供了将表中的列定义为可变长度多维数组的机会。可以创建任何内置或用户定义的基础类型、枚举类型或复合类型的数组

声明数组

```
CREATE TABLE monthly_savings (
   name text,
   saving_per_quarter integer ARRAY[4],
   scheme text[][]
);
```

插入数据

```
INSERT INTO monthly_savings
VALUES ('Manisha',
'{20000, 14600, 23500, 13250}',
'{{"FD", "MF"}, {"FD", "Property"}}');
```

访问数组

```
test@[local]:3433=#SELECT name FROM monthly_savings WHERE saving_per_quarter[2] > saving_per_quarter[4];
  name   
---------
 Manisha
```

修改数组

```
UPDATE monthly_savings SET saving_per_quarter = ARRAY[25000,25000,27000,27000]
WHERE name = 'Manisha';
```

搜索数组

```
SELECT * FROM monthly_savings WHERE saving_per_quarter[1] = 10000 OR
saving_per_quarter[2] = 10000 OR
saving_per_quarter[3] = 10000 OR
saving_per_quarter[4] = 10000;
```

如果已知数组的大小，可以使用上面给出的搜索方法。否则，下面的示例将展示如何在大小未知时进行搜索。

```
SELECT * FROM monthly_savings WHERE 10000 = ANY (saving_per_quarter);
```

**数组函数**

添加元素或删除元素

```
test@[local]:3433=#select array_append(array['a','b','c'],'d'),array_remove(array['a','b','c'],'c');
 array_append | array_remove 
--------------+--------------
 {a,b,c,d}    | {a,b}
```

获取数组维度

```
test@[local]:3433=#select array_ndims(array['a','b','c']);
 array_ndims 
-------------
           1
```

获取数组长度

```
test@[local]:3433=#select array_length(array['a','b','c'],1);
 array_length 
--------------
            3
```

查找元素第一次出现的位置

```
test@[local]:3433=#select array_position(array['a','b','c','c'],'c');
 array_position 
----------------
              3
```

元素替换

```
test@[local]:3433=#select array_replace(array['a','b','c','c'],'c','d');
 array_replace 
---------------
 {a,b,d,d}
```

数组转字符串

```
test@[local]:3433=#select array_to_string(array['a','b',null,'d'],',','c');
 array_to_string 
-----------------
 a,b,c,d
```

### 复合类型

声明复合类型

```
CREATE TYPE inventory_item AS (
   name text,
   supplier_id integer,
   price numeric
);
```

这个数据类型可以在创建表中使用，如下所示

```
CREATE TABLE on_hand (
   item inventory_item,
   count integer
);
```

插入数据

```
INSERT INTO on_hand VALUES (ROW('fuzzy dice', 42, 1.99), 1000);
```

访问复合类型

```
SELECT (item).name FROM on_hand WHERE (item).price > 9.99;
```

### 范围类型

范围类型表示使用数据范围的数据类型。范围类型可以是离散范围(例如，所有的整数值1到10)或连续范围(例如，在上午10:00到11:00之间的任何时间点)。可用的内置范围类型包括以下范围

- int4range：int范围
- int8range：bigint范围
- numrange：numeric范围
- tsrange：timestamp范围(不包含时区)
- tstzrange：timestamp范围(包含时区)
- daterange：date范围

可以创建自定义范围类型以使新的范围类型可用，例如使用inet类型作为基础的IP地址范围范围。

类型分别使用[]和()字符支持包含和独占范围边界。例如，’[4,9)’表示从4开始并包括4到不包括9的所有整数

查询范围下界

```
test@[local]:3433=#select lower(int4range(4,9));
 lower 
-------
     4
```

查询范围上界

```
test@[local]:3433=#select upper(int4range(4,9));
 upper 
-------
     9
```

### 伪类型

PostgreSQL类型系统包含许多特殊用途的条目，它们统称为伪类型。伪类型不能用作列数据类型，但可以用于声明函数的参数或结果类型

| 类型             | 描述                                             |
| :--------------- | :----------------------------------------------- |
| any              | 指示函数接受任何输入数据类型                     |
| anyelement       | 指示函数接受任何数据类型                         |
| anyarray         | 指示函数接受任何数组数据类型                     |
| anynonarray      | 指示函数接受任何非数组数据类型                   |
| anyenum          | 指示函数接受任意枚举数据类型                     |
| anyrange         | 指示函数接受任何范围数据类型                     |
| cstring          | 指示函数接受或返回以null结尾的C字符串            |
| internal         | 指示函数接受或返回服务器内部数据类型             |
| language_handler | 程序语言调用处理程序被声明为返回language_handler |
| fdw_handler      | 外数据包装处理程序被声明为返回fdw_handler        |
| record           | 标识返回未指定行类型的函数                       |
| trigger          | 触发器函数被声明为返回触发器                     |
| void             | 指示函数不返回值                                 |

### 文本搜索类型

这种类型支持全文搜索，这是一种搜索自然语言文档集合以找到与查询最匹配的文档的活动。这里有两种数据类型

| 类型     | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| tsvector | 这是一个由不同单词组成的排序列表，这些单词已经被规范化，以合并同一个单词的不同变体，称为“词汇” |
| tsquery  | 它存储要搜索的词元，并按照布尔运算符组合它们，括号可用于强制操作符的分组 |

### XML类型

XML数据类型可用于存储XML数据。为了存储XML数据，首先必须使用函数xmlparse创建XML值，如下所示

```
XMLPARSE (DOCUMENT '<?xml version="1.0"?>
<tutorial>
<title>PostgreSQL Tutorial </title>
   <topics>...</topics>
</tutorial>')

XMLPARSE (CONTENT 'xyz<foo>bar</foo><bar>foo</bar>')
```

### 布尔类型

PostgreSQL提供了标准的布尔型SQL。布尔数据类型可以有true、false和第三种状态unknown，它由SQL null值表示。插入时true可以用t、yes、t、on、1来表示，false可以用f、n、no、off、0来表示，但最终查询出来的结果只有t或者f

| 类型    | 字节数 |
| :------ | :----- |
| boolean | 1bytes |

### 枚举类型

枚举类型是包含静态有序值集的数据类型，与其他类型不同，需要使用CREATE TYPE命令创建枚举类型，创建完成后就可以和其它类型一样使用

```
CREATE TYPE week AS ENUM ('Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun');
```

### 几何类型

几何数据类型表示二维空间对象。最基本的类型point构成了所有其他类型的基础

| 类型    | 字节数       | 表现                     | 描述                         |
| :------ | :----------- | :----------------------- | :--------------------------- |
| point   | 16bytes      | 平面上的点               | (x,y)                        |
| line    | 32bytes      | 无限行                   | ((x1,y1),(x2,y2))            |
| lseg    | 32bytes      | 有限的线段               | ((x1,y1),(x2,y2))            |
| box     | 32bytes      | 矩形                     | ((x1,y1),(x2,y2))            |
| path    | 16+16n bytes | 封闭路径(类似于多边形)   | ((x1,y1),…)                  |
| path    | 16+16n bytes | 开放路径                 | [(x1,y1),…]                  |
| polygon | 40+16n bytes | 多边形（类似于封闭路径） | ((x1,y1),…)                  |
| circle  | 24bytes      | 圆                       | <(x,y)，r>(圆的中心点和半径) |

### 数据类型转换

通过CAST函数可以进行数据转换，例如将int转换为text

```
test@[local:/service/pgsql/data]:3433=#select CAST(int'10' as text);
 text 
------
 10
```

正如前面的示例，我们也可以通过`::`来进行转换

```
test@[local:/service/pgsql/data]:3433=#select 3/2::numeric;
      ?column?      
--------------------
 1.5000000000000000
```

也可以通过下列表格中的格式化函数进行转换

| 函数                    | 返回类型                 | 描述                   |
| :---------------------- | :----------------------- | :--------------------- |
| to_char(timestamp,text) | text                     | 将时间戳转换为字符串   |
| to_char(interval,text)  | text                     | 将时间间隔转换为字符串 |
| to_char(int,text)       | text                     | 将整数转换为字符串     |
| to_char(number,text)    | text                     | 将数字转换为字符串     |
| to_date(text,text)      | date                     | 将字符串转换为日期     |
| to_number(text,text)    | numeric                  | 将字符串转换为数字     |
| to_timestamp(text,text) | timestamp with time zone | 将字符串转换为时间戳   |

[Data Type]:https://www.tutorialspoint.com/postgresql/postgresql_data_types.htm
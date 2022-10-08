[TOC]

# MongoDB基础概念

## 1. 数据库

MongoDB能够承载多个数据库，一个数据库可以包含0个或多个集合。每个数据库都有独立的权限，每个数据库也存放在不同文件上。MongoDB数据库通过名称来标识，数据库名可以是满足以下条件的任意UTF-8字符串：
- 不能是空字符串
- 不能含有/、\、.、"、*、<、>、:、|、?、$、空格、\0。基本上只能使用ASCII中的字母和数字
- 数据库名称区分大小写
- 数据库名称最多为64字节

另外MongoDB中还包含几个系统数据库，需要了解它的作用
- admin：添加到admin数据库的用户自动获得所有数据库的权限。同时也可以运行一些特定命令，如列出所有数据库、关闭服务器。
- local：本地集合数据库，在副本集或分配环境无法被复制
- config：用于分片设置，分片信息会存在config数据库中

查看当前数据库
```
> db
test
```

切换数据库
```
> use admin
```

创建数据库
```
>db.createCollection(database)
```
>当我们use切换到一个不存在的数据库中，并创建集合，数据库会一同创建



## 2. 文档

文档即一组键值对的有序集，文档中的值可以是多种不同类型的数据类型，甚至是一个内嵌文档。文档的键是字符串，键通常可以是任意UTF-8字符，但需要注意以下两点：
- 键不能含有\o(空字符)，这个字符表示键的结尾
- . 和 $具有特殊意义，只能在特定环境下使用，通常只作为保留

MongoDB不但区分类型还区分大小写。例如，下面两个文档是不同的
```
{"foo":3}
{"foo":"3"}
```
文档中的键值对是有序的，{"X":1,"Y":2}和{"Y":2,"X":1}两个是不同的。MongoDB文档不能有重复的键



## 3. 集合

集合就是一组文档，集合是动态模式的，这意味着集合内的文档可以是各式各样的。集合名称也存在一定的要求
- 集合名不能为空字符串
- 集合名不能包含\0字符
- 集合名不能以"system."开头，这是为系统集合保留的前缀。
- 用户创建的集合不能包含保留字符"$"

组织集合常用的惯例就是通过"."来分隔不同命名空间的子集合。例如博客系统的blog.posts和blog.categories。使得组织结构更清晰，系统中GridFS也采用了子集合的形式。

db.collectionName可以获取一个集合的内容，如果集合名称中包含保留字或者无效的javascript属性名称，db.collectionName就不能正常工作了，这时可以采用db.getCollection("collectionName")来获取，也可以使用数组访问法，x.y等价于x['y']。

**固定集合**

固定集合，顾名思就是固定大小的集合，向一个插满的固定集合插入新数据，将会将最老的数据先给删除，如此循环往复。固定集合数据顺序写入，因此写入速度很快，oplog日志则就是采用固定集合。
```
db.createCollection("mycoll",{"capped":true,"size":10000});
```
size为固定集合的大小，单位为字节，除了大小，还可以通过max选项指定集合中的文档数量。另外，我们也可以将普通集合转换为固定集合，通过convertToCapped实现。
```
db.runCommand({"convertToCapped":"test","size":10000});
```
> 无法将固定集合转换为普通集合



## 4. 数据类型

MongoDB文档与Javascript对象相近，采用了类似于JSON格式的BSON格式，其支持下列数据类型
- null：null用于表示空值或者不存在的字段
- Boolean：布尔类型包含true和false两个值
- 数值：shell默认使用64为浮点数数值
- 字符串：UTF-8字符串都可表示为字符串类型的数据
- 日期：日期被存储为自新纪元以来经过的毫秒数，不存储时区
- 正则表达式：查询时，使用正则表达式作为限定条件，语法与JavaScript相同
- 数组：数据列表或数据集可以表示为数组，数组可以包含不同数据类型的元素
- 内嵌文档：文档可嵌套其它文档，被嵌套的文档作为父文档的值
- 对象id：文档必须有一个_id键作为唯一标识，这个键的值可以是任何类型的，默认是个ObjectId对象，它使用12个字节的存储空间，是一个24个十六进制数字组成的字符串，前四个字节为时间戳，单位为秒，后三位为机器的唯一标识，由主机名散列值生成，再后两位为PID进程标识符，最后三个字节是一个自动增加的计数器
- 二进制数据：二进制数据是一个任意字节的字符串，它不能直接在shell中使用，如果要将非UTF-8字符保存到数据库中，二进制数据是唯一的方式。



## 5. MongoDB Shell

MongoDB Shell是MongoDB自带的Javascript Shell，支持完整的javascript，同时可作为MongoDB的客户端。可以通过mongo命令进入MongoDB Shell，如果对Shell中的命令不太熟悉，可以通过db.help()查看数据库帮助信息或者通过db.[collection].help()查看集合的帮助信息。如果要查看函数的实现代码或参数顺序可以通过不加括号来获取，例如想查看insert函数，就可以输入db.[collection].insert来获取

在交互方式与脚本化之间部分命令存在差异，例如脚本中不能采用use db或show collections命令，具体如下：

Shell Helpers   | Javascript |
-- | -- |
show dbs,show databases | db.adminCommand('listDatabases') |
use <db>        | db=db.getSiblingDB('<db>') |
show collections        | db.getCollectionNames() |
show users      | db.getUsers() |
show roles      | db.getRoles(showBuiltinRoles:true) |
show log <logname>      | db.adminCommand({'getLog':'<logname>'}) |
show logs       | db.adminCommand({'getLog':'*'})
it      | cursor=db.collection.find() if (cursor.hasNext()){ cursor.next();} |

我们可以通过--eval参数直接返回命令结果：
```
mongo --eval "db.getCollectionNames()"
```

我们也可以直接执行js脚本:
```
mongo --quiet scripts1.js scripts2.js
```
或者直接在shell中加载js脚本，当前的查找目录可以通过run("pwd")调用系统命令查看
```
load(scripts1.js)
```

我们还可以将脚本变量注入到shell中。例如，可以在脚本中简单创建一些常用的辅助函数
，下面定义了一个connctTo()的函数，它连接到指定端口的本地数据库
```
var connectTo = new function(port,dbname) {
  if (!port) {
    port=27017;
  }
  if(!dbname) {
    dbname="test";
  }
  db = connect("localhost:"+port+"/"+dbname);
  return db;
}
```
通过load('connectTo.js')来加载辅助函数

**.mongorc.js**

.mongorc.js是在Shell启动时自动加载的一个脚本，可以用于一些提示或者危险命令重定向的用途。例如禁用dropDatabase或者deleteIndexes命令
```
var no = function() {
  print("Not on my watch");
}

db.dropDatabase = DB.prototype.dropDatabase = no;
DBCollection.prototype.drop = no;
DBCollection.prototype.dropIndex = no;
```

定制shell提示格式
```
prompt = function() {
  if (typeof db == 'undefined') {
     return '(nodb)> ';
  }

  try {
     db.runCommand({getLastError:1});
  }
  catch (e) {
     print(e);
  }
  return db+"> ";
};
```

在shell中编辑多行时无法编辑之前的行，可以配置shell采用外部编辑器。在.mongorc.js最后添加：
```
EDITOR = /usr/bin/vi
```
例如编辑一个变量，可以使用edit命令
```
var hello = db.code.findOne({title:"hello world"})
edit hello
```
edit编辑完保存退出，变量就会被重新解析加载到shell中



## 6. 连接MongoDB


MongoDB Shell使用mongo提供两种格式来连接数据库，一种是URI的方式，一种是参数选项

**URI**
```
mongo mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]
```
- username:password：如果MongoDB启用了认证登陆，则需要输入相应的用户名和密码
- host:port ：指定要连接的IP和端口，如果是副本集则需要输入多个节点
- database：连接并验证登陆指定的数据库，如果不指定默认打开test数据库
- ?options：连接选项，key-value形式，以&分隔

**参数选项**
```
mongo host:port -u username -p password --authenticationDatabase admin
```

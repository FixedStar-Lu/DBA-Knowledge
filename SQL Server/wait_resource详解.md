[TOC]

# wait_resource详解

当我们通过下列SQL查询数据库当前正在执行的SQL时，我们会发现wait_resource时常会出现等待，今天就来对常见的等待对象类型进行介绍

```
SELECT 
  [session_id],
 [request_id],
  [start_time] AS '开始时间',
  [status] AS '状态',
  [command] AS '命令',
  dest.[text] AS 'sql语句', 
  DB_NAME([database_id]) AS '数据库名',
  [blocking_session_id] AS '正在阻塞其他会话的会话ID',
 [wait_type] AS '等待资源类型',
 [wait_time] AS '等待时间',
  [wait_resource] AS '等待的资源',
 [reads] AS '物理读次数',
 [writes] AS '写次数',
 [logical_reads] AS '逻辑读次数',
 [row_count] AS '返回结果行数'
 FROM sys.[dm_exec_requests] AS der 
 CROSS APPLY 
 sys.[dm_exec_sql_text](der.[sql_handle]) AS dest 
 ORDER BY [cpu_time] DESC
```



### Page

对于page类型的等待，page字符往往会隐藏，其格式如下

```
waitresource="6:3:5335455" = Database_Id : FileId : PageNumber
```

通过DBCC PAGE查看对应页内容，必须开启3604 trace

```
DBCC TRACEON (3604);
DBCC PAGE (6,1,5335455,2);
```

其输出结果如下，其中Metadata: ObjectId为对应ID，我们可以通过select object_name(5335455)查看对应对象

```
PAGE: (1:5335455)


BUFFER:


BUF @0x0000000494C73840

bpage = 0x0000000C5BF28000          bhash = 0x0000000000000000          bpageno = (1:5335455)
bdbid = 6                           breferences = 0                     bcputicks = 0
bsampleCount = 0                    bUse1 = 31633                       bstat = 0x9
blog = 0x5215215a                   bnext = 0x0000000000000000          

PAGE HEADER:


Page @0x0000000C5BF28000

m_pageId = (1:5335455)              m_headerVersion = 1                 m_type = 1
m_typeFlagBits = 0x0                m_level = 0                         m_flagBits = 0x200
m_objId (AllocUnitId.idObj) = 105172m_indexId (AllocUnitId.idInd) = 256 
Metadata: AllocUnitId = 72057600930480128                                
Metadata: PartitionId = 72057600417857536                                Metadata: IndexId = 1
Metadata: ObjectId = 498868894      m_prevPage = (1:5335454)            m_nextPage = (1:5335800)
pminlen = 48                        m_slotCnt = 49                      m_freeCnt = 3980
m_freeData = 7968                   m_reservedCnt = 0                   m_lsn = (1172491:239610:691)
m_xactReserved = 0                  m_xdesId = (1:861477284)            m_ghostRecCnt = 0
m_tornBits = 352880690              DB Frag ID = 1                      

Allocation Status

GAM (1:5112320) = ALLOCATED         SGAM (1:5112321) = NOT ALLOCATED    
PFS (1:5329992) = 0x40 ALLOCATED   0_PCT_FULL                            DIFF (1:5112326) = NOT CHANGED
ML (1:5112327) = NOT MIN_LOGGED
```

根据DBCC PAGE返回的内容，可以使用 %%physloc%% 来查看Page上数据行的定位器。在Page上，每一个数据行都可以通过一个索引来寻址，该索引就是数据行的定位器

```
SELECT 
    sys.fn_PhysLocFormatter (%%physloc%%) AS PhysLoc,
    *
FROM [table_name] WITH(NOLOCK)
WHERE sys.fn_PhysLocFormatter (%%physloc%%) like '(1:5335455%'
```

### key

对于等待的资源是Key的情况，该阻塞发生在聚集索引上，Key资源的格式是 database_id, hobt_id (Magic Hash)，其中 Magic Hash的某一个数据行的哈希值

```
waitresource=“KEY: 6:72057594041991168 (ce52f92a058c)” = Database_Id, HOBT_Id ( Magic Hash )
```

定位等待的对象

```
select * from sys.partitions where hobt_id=72057600638910464
select object_name([object_id])
```

定位具体的行

```
SELECT *
FROM [table_name] WITH(NOLOCK)
WHERE %%lockres%% = '(ce52f92a058c)';
```

### RID

对于等待资源是RID的情况，阻塞发生在heap上，也就是说，RID等待只会发生在没有创建聚集索引的表上

```
waitresource="RID: 6:15:11695844:3"= Databae_Id:File_Id:Page_Id:Slot_No
```

该描述符表示等待的资源某一个特定的行，数据行存储在Page上，Page的底部是行偏移（Row Offsets），每一行的偏移量连续排列在Page的末尾，称作槽数组（Slot Array），每一个slot中存储的都是行在Page中的偏移量
![img](https://gitee.com/dba_one/wiki_images/raw/master/images/628084-20191202121552736-415536620.png)

### OBJECT和TABLE

```
waitresource="OBJECT: 6:12347015633:1 "=Database_Id:Object_Id:PageNumber
```

我们可以通过PageNumber来还原阻塞发生的情况

```
waitresource="TAB: 5:261575970:1"=DatabaseID:ObjectID:IndexID
```

阻塞过程中，由于全表扫描导致整张表都锁定了

[PAGELATCH_EX争用]:https://support.microsoft.com/zh-cn/help/4460004/how-to-resolve-last-page-insert-pagelatch-ex-contention-in-sql-server
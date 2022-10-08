[TOC]

# MongoDB GridFS

GridFS用于存储和检索超过16M的bson文档大小限制的文件，GridFS不会将文件存储在单个文档中，而是将文件分为多个部分或chunks，并将每个chunks存储为单独的文档。默认情况下，GridFS使用的默认块大小为255KB

## 1. mongofiles

我们可以通过自带的mongofiles工具上传文件、下载文件、查看文件列表、搜索文件以及删除文件。

上传文件

```
mongofiles put filename
```

下载文件

```
mongofiles get filename
```

查看文件列表

```
mongofiles list
```

>  更多的命令选项可以查看`mongofiles –help`

在驱动程序中使用GridFS，例如python可以采用PyMongo执行上面mongofiles操作

```
from pymongo import Connection
import gridfs
db = Connection().test
fs = gridfs.GridFS(db)
file_id = fs.put("hello,world",filename="test.txt")
fs.list()
fs.get(file_id).read()
```



## 2. GridFS集合

GridFS使用两个集合来存储文件，一个存储文件chunks，另一个存储文件元数据。fs.files集合结构如下：

```
{
  "_id" : <ObjectId>,
  "length" : <num>,
  "chunkSize" : <num>,
  "uploadDate" : <timestamp>,
  "md5" : <hash>,
  "filename" : <string>,
  "contentType" : <string>,
  "aliases" : <string array>,
  "metadata" : <any>,
}
```

- files._id：文档的唯一标识
- files.length：文档的大小，单位为字节
- files.chunkSize：每个块的大小，单位为字节
- files.uploadDate：存储文档的时间
- files.md5：不推荐使用
- files.filename：可选。GridFS文件的名称
- files.contentType：不推荐使用
- files.aliases：不推荐使用
- files.metadata：元数据

fs.chunks

```
{
  "_id" : <ObjectId>,
  "files_id" : <ObjectId>,
  "n" : <num>,
  "data" : <binary>
}
```

- chunks._id：chunk唯一标识
- chunks.files_id：与fs.files中的_id相对应
- chunks.n：块在文件中的相对位置
- chunks.data： 块所包含的二进制数据

GridFS使用每个chunks和files集合上的索引来提高效率。

- fs.chunks索引

  ```
  db.fs.chunks.createIndex( { files_id: 1, n: 1 }, { unique: true } );
  ```

- fs.files索引

  ```
  db.fs.files.createIndex( { filename: 1, uploadDate: 1 } );
  ```



## 3. GridFS分片

在分片环境中，对chunks进行分片请使用{ files_id : 1, n : 1 } or { files_id : 1 }作为分片索引。files集合很小，仅保留元数据，可以考虑仅存放在主分片上，如果要对files进行分片可以考虑利用_id作为片键。
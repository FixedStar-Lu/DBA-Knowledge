[TOC]

---
# 通过COPY加载S3数据

从 Amazon S3 加载数据，一般遵循以下流程：
1. 将数据拆分为多个文件
2. 将文件上传到S3
3. 运行COPY命令加载表
4. 对表执行VACUUM和ANALYZE

**数据拆分**

将数据文件拆分为多个文件有助于利用并行处理，加快加载速度，文件数应该为集群切片数的倍数，例如集群计算节点有两个切片，就可以分为4个文件，拆分后的大小应该介于1M到1G之间。拆分工具可以考虑利用split命令或在数据导出时就进行拆分

文件之间应该指定相同的前缀，加载过程中通过manifest来匹配前缀加载所有数据文件。

## 1. 加载数据

COPY命令的基础语法
```
COPY table_name [ column_list ] FROM data_source CREDENTIALS access_credentials [options] 
```
- table_name为将数据加载到指定表
- column_list为可选项，指定列顺序进行映射，逗号分隔
- 指定访问密钥时，我们可以使用CREDENTIALS提供用户证书，也可以使用iam_role提供角色权限
- data_source为数据源，从S3加载数据时需要指定文件S3路径，多文件加载的方式有键前缀和清单文件两种方式：键前缀是指一组具有相同前缀的对象，COPY命令可以加载指定前缀的对象；清单文件则是将需要加载的所有对象都记录在清单，读取清单内容进行加载

**常用选项**

- **CSV**：指定加载CSV格式的文件
- **REGION**：当S3和集群不在同一个区域内，通过REGION指定AWS资源区域
- **MANIFEST**：避免通过前缀加载到不需要的表，可以采用清单文件的方式，清单采用JSON格式，需要列出每个加载对象，对象可以位于不同的桶内，但需要在同一区域
```
{
  "entries": [
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-000"},
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-001"},
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-002"},
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-003"},
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-004"},    
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-005"},
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-006"}, 
    {"url":"s3://<your-bucket-name>/load/customer-fw.tbl-007"} 
    ]
}
```
- **DELIMITER**：指定分隔符，CSV默认为逗号
- **MAXERROR**：默认情况下，COPY遇到错误就会失败并返回错误消息，可以指定MAXERROR指定COPY在跳过指定数量的错误后失败
- **FIXEDWIDTH**：固定宽度格式将每个字段定义为固定数量的字符，而不是使用分隔符隔开的字段
- **ACCEPTINVCHARS**：指示 COPY 将每个无效字符替换为指定的有效字符，然后继续加载操作，仅对VARCHAR有效
- **DATEFORMAT**：加载 DATE 和 TIMESTAMP 列时，COPY 需要默认格式，如果不使用默认格式则使用DATEFORMAT或TIMEFORMAT来指定格式，通常选择auto即可
- **COMPUPDATE**：当 COPY 加载无压缩编码的空表时，它会分析加载数据以确定最佳编码，过程比较耗时，可以指定COMPUPDATE跳过
- **GZIP、LZOP、ZSTD 和 BZIP2**：在从压缩文件加载时，COPY 会在加载过程中解压缩这些文件
- **NOLOAD**：分析数据文件，而不进行加载

**转换参数**

有时数据文件格式无法简单的完成加载，需要对数据进行转换再加载，COPY命令支持下列转换参数：
- **ACCEPTANYDATE**：允许加载包括无效格式（如 00/00/00 00:00:00）在内的任何日期格式，而不生成错误
- **ACCEPTINVCHARS [AS] ['replacement_char']**：允许将数据加载到 VARCHAR 列中，即使数据包含无效的 UTF-8 字符
- **BLANKSASNULL**：将仅包含空格字符的空白字段作为 NULL 加载
- **EMPTYASNULL**：指示Redshift 应将空 CHAR 和 VARCHAR 字段作为 NULL 加载
- **ENCODING [AS] file_encoding**：指定加载数据的编码类型，可通过linux file命令查看文件编码
- **ESCAPE**：指定此参数后，输入数据中的反斜杠字符 (\) 将被视为转义字符，与CSV参数冲突
- **EXPLICIT_IDS**：如果要将表中自动生成的值替换为源数据文件中的显式值，请对具有 IDENTITY 列的表使用 EXPLICIT_IDS
- **FILLRECORD**：当一些记录的末尾缺少相邻列时，允许加载数据文件
- **IGNOREBLANKLINES**：忽略数据文件中仅包含换行符的空行并且不尝试加载它们
- **IGNOREHEADER [ AS ] number_rows**：将指定的 number_rows 视为文件标题并且不加载它们
- **NULL AS 'null_string'**：加载将 null_string 匹配为 NULL 的字段，其中 null_string 可以是任何字符串
- **REMOVEQUOTES**：删除传入数据中的字符串周围的引号。将保留引号中的所有字符（包括分隔符），与CSV参数冲突
- **ROUNDEC**：当输入值的小数位数超出列的小数位数时，会将数值向上取整
- **TRIMBLANKS**：删除 VARCHAR 字符串的尾部空格字符
- **TRUNCATECOLUMNS**：将列中的数据截断为合适的字符数以符合列规范

```
COPY test_titles from 's3://transsnet-db-backup/mysql_archiver/test.csv'
iam_role 'arn:aws:iam::502076313356:role/mySpectrumRole'
DELIMITER ','
EMPTYASNULL
REMOVEQUOTES
dateformat as 'auto'
ESCAPE
```
## 2. 执行VACUUM和ANALYZE



## 3. 参考链接

1. [教程：从 Amazon S3 加载数据](https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/tutorial-loading-data.html)

2. [COPY命令](https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/r_COPY.html#r_COPY-syntax-overview-data-conversion)




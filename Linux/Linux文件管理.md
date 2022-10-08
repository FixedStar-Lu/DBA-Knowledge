[TOC]

# Linux文件管理

## 1. 文件/目录基本操作

**创建目录**
```
$ mkdir /etc/mysql
```
当上一层目录不存在时，我们可以用`-p`选项进行级联创建。

**创建文件**
```
$ touch mytest.txt
```
创建文件时，我们可以通过touch命令先生成一个空白文件，也可以利用vi或vim编辑保存生成一个文件

**删除文件/目录**
```
$ rm -fr mytest.txt
```
在进行批量删除时，可能会出现参数溢出的情况，我们可以结合`xargs`分批获取并执行

**移动/拷贝文件目录**
```
$ mv mytest.txt test.txt
$ cp -r mytest.txt test.txt
```
mv可用来移动或重命名文件/目录，cp则为生成复制副本，针对目录需要添加`-r`选项

**查看文件/目录**
```
$ ls -lrt
```
部分目录下存在隐藏文件，需要添加`-a`选项进行查看

**查看文件内容**
```
$ cat mytest.txt
```
文件内容可能过长，cat不利于查看，可以选择`more`命令分页查看或者`tail`命令查看

**查找文件**
```
$ find path -option [-print] [-exec command] {} \;
```
- path为指定的查找路径
- -option包含一些常用的选项

选项 | 描述
-- | --
-admin n | 在过去N分钟内被读取过的文件或目录
-anewer file | 比文件file更晚被读取的文件或目录
-atime n | 在过去N天内被读取过的文件或目录
-cmin n | 在过去N分钟内被更改过的文件或目录
-cnewer file | 比文件file更晚被修改的文件或目录
-ctime n | 在过去N天内被更改过的文件或目录
-mtime n | 在过去N天内被修改过的文件或目录
-expty | 查找文件大小为0 byte或没有任何子目录的目录
-user | 查找指定文件所有者的文件
-gid / -group | 查找组id或组名称对应的文件和目录
-path / -ipath | 指定字符串为寻找目录的样式，-ipath忽略大小写
-name / -iname | 指定字符串为文件名称的样式，iname忽略大小写
-size | 指定文件大小阈值
-type | 指定查找对象类型
-mount,-xdev | 只检查和指定path在同一个文件系统下的文件

在通过时间选项检索时，我们可以通过stat查看文件时间信息，访问时间，修改时间，更改时间有以下对应关系
- 访问时间        | access  | atime
- 修改时间        | Modify  | mtime
- 状态改动时间 | change | ctime
```
$ find /var/log -mtime +30 -a -size +100k -exec ls -l {} \;
```
- +30代表大于等于30天前的文件
- -30代表小于等于30天以内的文件
- 30则代表30-31那一天的文件

find命令支持and(-a)，or(-o)，-not的条件
```
$ [dba@t-luhx01-v-szzb ~]$ find ./ -user dba -o -user root
```
对于type选项的类型选择，find支持以下类型：
- d：目录
- c：字型装置文件
- b：区块装置文件
- p：具名伫列
- f：一般文件
- l：link链接
- s：socket文件

**软链接或硬链接**
```
/*软连接*/
$ ln -s source.txt target.txt
/*硬链接*/
ln source.txt target.txt
```
硬链接与软链接的区别在于：硬链接与源文件的inode都指向同一磁盘中的区块，不受源文件删除的影响，而软链接只是一种符号链接，访问时会访问源文件路径，源文件删除则软链接也失效。



## 2. 文件目录权限

![file-detail.png](https://upload-images.jianshu.io/upload_images/26125409-01e6551656ef568c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图我们可以发现，Linux的文件权限分为三部分：所属用户的权限，所属组的权限，其它用户的权限。分别用U，G，O表示，例如我们现在要给所属用户添加执行权限
```
$ chmod u+x test.sh
```
相反，我们也可以回收该权限
```
$ chmod u-x test.sh
```
除此之外，权限中的r(读)、w(写)、x(执行)也可以用数字1，2，4来对应代替，例如我们现在添加655的权限
```
$ chmod 655 test.sh
$ ls -l
-rw-r-xr-x 1 dba dba 0 Mar 31 14:07 test.sh
```
>如果要级联修改目录的所有文件权限，需要指定`-R`选项

如果要改变文件的用户和属组，可以通过chown命令修改，修改后文件所属用户和组发生变化，权限对象也发生变化
```
$ chown mysql:mysql test.sh
```



## 3. 文件目录打包压缩

压缩成tar.gz格式
```
$ tar -zcvf target.tar.gz ./source
```
压缩成zip格式
```
$ zip -q -r target.zip ./source
```
解压tar.gz
```
$ tar -xvf target.tar.gz
```
加压zip
```
$ unzip target.zip
```

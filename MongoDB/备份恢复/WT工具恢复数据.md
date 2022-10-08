#MongoDB #WireTiger #Restore
# WT工具恢复数据

MongoDB底层存储引擎采用WiredTiger，数据文件格式为`.wt`。当磁盘文件损坏等因素导致实例无法启动，我们可以利用WT工具来解析恢复数据文件中的数据，仅作为最后的应急方案。当然，还是强烈建议采用副本集高可用架构来避免这种情况

## 安装WireTiger源码

安装依赖包
```
sudo yum install -y epel-release libtool automake snappy snappy-devel lz4 lz4-devel zstd zstd-devel libzstd-devel zlib zlib-devel git make vim-common
```

WT版本应该与MongoDB的版本相同
```
> git clone https://github.com/wiredtiger/wiredtiger.git
> git tag | grep 4.4.1
mongodb-4.4.1
mongodb-4.4.1-rc0
mongodb-4.4.1-rc1
mongodb-4.4.1-rc2
mongodb-4.4.1-rc3
mongodb-4.4.10
mongodb-4.4.10-rc0
mongodb-4.4.11-rc0
> git checkout tags/mongodb-4.4.1 -b v4.4.1
```

编译源码
```
sh autogen.sh
./configure --disable-shared --with-builtins=lz4,snappy,zlib,zstd
make -j $(nproc)
make install
```
> `--disable-shared`来将动态链接库直接打包到执行文件中，因此生成的可执行文件较大

## 测试恢复数据

现在有下列集合包含50000条数据
```
> db.test_station4.count()db.test_station4.count()
50000
```
查看数据文件路径
```
> db.test_station4.stats().wiredTiger.uri
statistics:table:test2/collection-2-6903356008510035750
```
查看所有对象
```
$ wt -v -h /service/mongodb/data/ list

```
从数据文件导出数据
```
$ wt -v -h /service/mongodb/data dump  -f /root/data.dump test2/collection-2-6903356008510035750
```

在临时实例中创建一个集合
```
> use test4
> db.dump.insert({})
> db.dump.stats().wiredTiger.uri
statistics:table:test4/collection-0-6404410910276424166
```
关闭临时实例，导入数据到临时集合
```
$ mongod --config /etc/mongodb.conf --shutdown
$ wt -v -h /service/mongodb/data/ -R load -f /root/data.dump  -r test4/collection-0-6404410910276424166
        table:test4/collection-0-6404410910276424166: 50000
```
> 也可以直接用数据文件替换临时集合的数据文件，并执行`wt salvage file:test4/collection-0-6404410910276424166.wt`修复数据文件

启动实例，修复集合
```
$ mongod --config /etc/mongodb.conf
$ mongo mongodb://root:abcd123#@10.0.4.16:2701
> db.dump.validate({full: true});
```
查看集合数据
```
> use test4
> db.dump.count()
50000
```

**`注意：该方案仅在测试效果下实现，需要保证数据文件和WiredTiger元数据文件存在，存在不确定因素只可作为最后的解决方案，正确的做法还是应当建立数据库高可用容灾以及备份策略`**
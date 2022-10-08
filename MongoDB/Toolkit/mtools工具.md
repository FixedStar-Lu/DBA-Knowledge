[TOC]

## mtools工具

mtools是一个帮助脚本的集合，用于解析、过滤和可视化MongoDB日志文件(mongod, mongos)。mtools还包括mlaunch，一个用于在本地机器上快速设置复杂MongoDB测试环境的实用程序，以及mtransfer，一个用于在MongoDB实例之间传输数据库的工具。项目地址：[mtools](https://github.com/rueckstiess/mtools)
- mlogfilter：按时间切片日志文件，合并日志文件，过滤慢查询，查找表扫描，缩短日志行，根据其他属性过滤，转换为JSON
- mloginfo：返回关于日志文件的信息，如开始和结束时间，版本，二进制，特殊部分，如重启，连接，独特的视图
- mplotqueries：visualize log files with different types of plots
- mlaunch：快速构建副本集或分片集群架构的测试环境
- mtransfer：通过复制wiredTiger数据文件在mongodb之间传输数据(实验性质)

### 安装mtools

> 环境要求：python3+pip3
>
> 依赖包：psutil+pymongo+matplotlib+numpy

下载介质

[DownLoad mtools](https://github.com/rueckstiess/mtools/archive/refs/tags/v1.6.4.tar.gz)

解压介质

```
$ tar -xvf mtools-1.6.4.tar.gz
```

安装介质

```
python3 setup.py install
```






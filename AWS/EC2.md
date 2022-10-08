[TOC]

# 1. 磁盘管理

## 1.1 磁盘扩容

EC2控制台进入对应实例，查询对应卷存储，并进入卷管理页面，选择修改容量大小

![image-20211020111529037](https://gitee.com/dba_one/wiki_images/raw/master/images/image-20211020111529037.png)

这时候在系统层面还无法看到扩容后的容量，需要手动刷新

```
$ xfs_growfs /data
meta-data=/dev/nvme1n1           isize=512    agcount=50, agsize=5570560 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=274726912, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=10880, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 274726912 to 327155712
```


[TOC]

# Linux磁盘管理

## 1. 磁盘整列(raid)

RAID分为软件RAID和硬件RAID，软件RAID依赖于操作系统以及RAID软件，Linux下通常使用`mdadm`包；硬件RAID主要通过RAID卡来实现，它提供了专用的RAID控制器，目前大多数都采用硬件RAID。

RAID不同级别下，其架构与特性也所不同，主要包含以下级别：
- RAID0：条带化
- RAID1：镜像
- RAID5：单磁盘分布式奇偶校验
- RAID6：双磁盘分布式奇偶校验
- RAID10：镜像+条带化

**RAID0**

![raid0](https://upload-images.jianshu.io/upload_images/26125409-0fbb7b89454a269e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RAID0将数据分布在不同的磁盘上，以获取最大的读写性能，但也因此损失了数据的安全性，当其中一块磁盘发生故障无法恢复，那我们将丢失部分数据。仅用于追求性能而不考虑数据安全的场景之下。

**RAID1**

![raid1](https://upload-images.jianshu.io/upload_images/26125409-804d5ff85e22261b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RAID1通过镜像复制的方式复制出一个数据副本，以此来提升数据的安全性，当一份数据损坏后，可以利用数据副本来恢复。但我们也能发现，RAID1之后的容量仅有总容量的50%，并且写入速度也有所下降。

**RAID5**

![raid5](https://upload-images.jianshu.io/upload_images/26125409-8ad9f6bbcb906ff1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RAID5以分布式奇偶校验的方式工作，其至少需要三个disk，当一个节点发生故障进行更换时，它通过其它正常的驱动器上的校验信息进行数据重建，确保数据不会丢失，如果故障数量超过1个，将也会导致数据丢失。常用于生产WEB服务器，文件服务器等场景。

**RAID6**

![raid6](https://upload-images.jianshu.io/upload_images/26125409-5fb4da849a6e1cf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RAID6相对于RAID5采用了两个分布式奇偶校验，其至少需要4个disk，如果有2个disk发生故障也不影响数据重建。但成本较高，其中2个disk用于保存校验信息，并且如果不适用硬件RAID驱动器，性能将比较差

**RAID10**

![raid1+0](https://upload-images.jianshu.io/upload_images/26125409-0e40938abb8c65f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![raid0+1](https://upload-images.jianshu.io/upload_images/26125409-00ca8144d25e6fcd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

RAID10也就是RAID0和RAID1嵌套组成的，RAID1+0即先做镜像再做条带化，RAID0+1则是先做条带再做镜像，RAID10有着良好的性能，但其可用空间占总容量的50%，是性能与安全性权衡后的方案，常用于数据库存储环境。

更多关于RAID的信息，请参考[在 Linux 下使用 RAID](https://linux.cn/article-6085-1.html)



## 2. 磁盘管理

**磁盘分区**

磁盘分区能够将磁盘划分成多个分区，不同的分区可以作用于不同的用途，例如系统盘，数据盘，日志盘等。

查看磁盘分区
```
[root@t-luhx01-v-szzb ~]# fdisk -l

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000c4e35

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      821247      409600   83  Linux
/dev/sda2          821248    83886079    41532416   8e  Linux LVM

Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xb42fafac

   Device Boot      Start         End      Blocks   Id  System
```
> 其中Device为磁盘设备名；boot表示用于引导系统进行启动；start和end表示分区开始扇区和结束扇区，每个扇区大小为512B；blocks为扇区数量；ID表示磁盘编号；system表示系统分区信息

对磁盘进行分区
```
[root@t-luhx01-v-szzb ~]# fdisk /dev/sdb
Command (m for help)：
```
其中最主要的一些操作选项如下：

选项 | 描述
-- | --
n | 添加新分区，选择主分区或扩展分区并指定起始扇区和结束扇区
d | 删除一个分区
p | 打印当前分区
m | 查看帮助
q | 不保存退出
w | 保存退出

当我们添加新分区后，可以执行partprobe命令使内核重载分区信息

**格式化磁盘**

创建分区后，我们并不能直接对磁盘进行数据存储，需要先进行格式化并指定文件系统格式，常用的文件系统有ext4，xfs等。格式化命令如下：
```
[root@t-luhx01-v-szzb ~]# mkfs.ext4 /dev/sdb1
```

**磁盘挂载**

磁盘格式化完成之后，我们还需要将磁盘挂载到某个具体的目录，才能对其进行读写。

创建目录
```
[root@t-luhx01-v-szzb ~]# mkdir /service
```
挂载分区
```
[root@t-luhx01-v-szzb ~]# mount /dev/sdb1 /service
```
查看挂载信息
```
[root@t-luhx01-v-szzb ~]# df -h
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/vg01-root   35G   28G  5.4G  84% /
devtmpfs               3.9G     0  3.9G   0% /dev
tmpfs                  3.9G     0  3.9G   0% /dev/shm
tmpfs                  3.9G  418M  3.5G  11% /run
tmpfs                  3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/sda1              380M  111M  245M  32% /boot
/dev/mapper/dbvg-dblv   99G   50G   44G  53% /service
tmpfs                  781M     0  781M   0% /run/user/0
```
到这里就结束了么？并没有，这只是临时挂载，当系统重新启动后就失效了。我们还需要将挂载信息写入`/etc/fstab`中永久挂载
```
[root@t-luhx01-v-szzb ~]# blkid
/dev/sda1: UUID="cb450161-dbd3-49e7-9c06-2c5a482a5329" TYPE="ext4" 
/dev/sda2: UUID="Fjcal0-Yu1z-ks7S-BytS-GHAU-IUv2-iVu5RG" TYPE="LVM2_member" 
/dev/sdb1: UUID="hqFtae-7zkq-9uZr-aOBJ-kh27-BDeX-cjjV3t" TYPE="LVM2_member" 
[root@t-luhx01-v-szzb ~]# vi /etc/fstab
UUID="hqFtae-7zkq-9uZr-aOBJ-kh27-BDeX-cjjV3t"   /service   ext4  defaults 0 0
```



## 3. LVM逻辑卷

LVM逻辑卷是对Linux磁盘分区进行管理的一种机制，它将多个分区在逻辑上集合在一起，能够更简单有效的管理磁盘分区空间

**创建LVM逻辑卷**

创建磁盘分区
```
[root@t-luhx01-v-szzb ~]# fdisk /dev/sdb
[root@t-luhx01-v-szzb ~]# fdisk -l
Disk /dev/sdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0xb42fafac

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   209715199   104856576   8e  Linux LVM
```
创建物理卷(PV)
```
[root@t-luhx01-v-szzb ~]# pvcreate /dev/sdb1
[root@t-luhx01-v-szzb ~]# pvdisplay  
  --- Physical volume ---
  PV Name               /dev/sdb1
  VG Name               dbvg
  PV Size               <100.00 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              25599
  Free PE               0
  Allocated PE          25599
  PV UUID               hqFtae-7zkq-9uZr-aOBJ-kh27-BDeX-cjjV3t
```
创建卷组(VG)
```
[root@t-luhx01-v-szzb ~]# vgcreate dbvg /dev/sdb1
[root@t-luhx01-v-szzb ~]# vgs
  VG   #PV #LV #SN Attr   VSize    VFree  
  dbvg   1   1   0 wz--n- <100.00g      0 
  vg01   1   2   0 wz--n-  <39.61g 532.00m
[root@t-luhx01-v-szzb ~]# vgdisplay
  --- Volume group ---
  VG Name               dbvg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <100.00 GiB
  PE Size               4.00 MiB
  Total PE              25599
  Alloc PE / Size       25599 / <100.00 GiB
  Free  PE / Size       0 / 0   
  VG UUID               USwHx4-2obH-FYH7-9fp2-3b26-6d7B-8jP0Oh
```
创建逻辑卷(LV)
```
[root@t-luhx01-v-szzb ~]# lvcreate -l 100%FREE -n dblv dbvg
[root@t-luhx01-v-szzb ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/dbvg/dblv
  LV Name                dblv
  VG Name                dbvg
  LV UUID                67gTkY-Xp6O-Qa08-ASeQ-IvlK-rhrs-3wSbzt
  LV Write Access        read/write
  LV Creation host, time t-luhx01-v-szzb, 2019-09-06 10:18:20 +0800
  LV Status              available
  # open                 1
  LV Size                <100.00 GiB
  Current LE             25599
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:2
```
格式化lv
```
[root@t-luhx01-v-szzb ~]# mkfs.ext4 /dev/dbvg/dblv
```
挂载lv
```
[root@t-luhx01-v-szzb ~]# mkdir /service
[root@t-luhx01-v-szzb ~]# mount /dev/dbvg/dblv /service
```
>不要忘记写入`/etc/fstab`里面

**逻辑卷在线扩容**

创建pv
```
[root@t-luhx01-v-szzb ~]# pvcreate /dev/sdc
```
添加到VG中
```
[root@t-luhx01-v-szzb ~]# vgextend dbvg /dev/sdc
```
扩展LV
```
[root@t-luhx01-v-szzb ~]# lvextend -l +100%FREE -n /dev/dbvg/dblv
```
扩容文件系统
```
[root@t-luhx01-v-szzb ~]# resize2fs /dev/dbvg/dblv
```

**删除LVM逻辑卷**

取消挂载，包括/etc/fstab中的挂载信息
```
[root@t-luhx01-v-szzb ~]# umount /service
```
删除LV
```
[root@t-luhx01-v-szzb ~]# lvremove /dev/dbvg/dblv
```
删除VG
```
[root@t-luhx01-v-szzb ~]# vgremove dbvg
```
删除PV
```
[root@t-luhx01-v-szzb ~]# pvremove /dev/sdb1
```

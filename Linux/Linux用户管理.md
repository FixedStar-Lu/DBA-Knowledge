[TOC]

# Linux用户管理

## 1. 用户

**添加用户**

```
[root@t-luhx01-v-szzb ~]#  useradd dba
```
当需要添加一个系统用户时，我们只需要通过useradd命令来创建即可。我们可以在创建用户时通过`-p`选项指定用户名，也可以在创建完成后通过`passwd`命令设置或修改密码
```
[root@t-luhx01-v-szzb ~]#  passwd dba
```
在一些特殊情况下，我们希望仅创建一个不允许登录但需要运行特定服务的账号，可以通过下列方式创建
```
[root@t-luhx01-v-szzb ~]#  useradd -s /bin/nologin -M mysql
```

**删除用户**
```
[root@t-luhx01-v-szzb ~]#  userdel -r dba
```
userdel只会删除用户，用户home目录依旧存在，想完全删除用户需要指定`-r`选项

**查看用户**
```
[root@t-luhx01-v-szzb ~]#  id dba
uid=1017(dba) gid=100(dba) groups=100(dba)
```
使用id命令，我们可以查看用户的UID以及用户组信息



## 2. 用户组

**创建用户组**
```
[root@t-luhx01-v-szzb ~]#  groupadd oinstall
```
**删除用户组**
```
[root@t-luhx01-v-szzb ~]#  groupdel oinstall
```
**修改用户组**
```
[root@t-luhx01-v-szzb ~]#  usermod -G oinstall dba
```
默认创建用户时，会同时创建一个同名的用户组，我们可以在创建用户时通过`-g`选项自定义一个已经存在的用户组，也可以通过`usermod -g`指定用户组。
```
[root@t-luhx01-v-szzb ~]#  usermod -g oinstall dba
[root@t-luhx01-v-szzb ~]#  id dba
uid=1017(dba) gid=1023(oinstall) groups=1023(oinstall)
[root@t-luhx01-v-szzb ~]#  usermod -g dba dba
[root@t-luhx01-v-szzb ~]#  usermod -G oinstall dba
[root@t-luhx01-v-szzb ~]#  id dba
uid=1017(dba) gid=1017(dba) groups=1017(dba),1023(oinstall)
```
> 这里需要注意`-g`和`-G`的区别，一个用户可以对应多个组，-G就是将用户添加到不同的组内，-g则会将用户当前的根组替换掉。

**查看所有组**

```
[root@t-luhx01-v-szzb ~]# cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
dba:x:1017:1017::/home/dba:/bin/bash
[root@t-luhx01-v-szzb ~]# cat /etc/group
root:x:0:
oinstall:x:1023:dba
dba:x:1017:
```
系统所有用户和组的信息默认都保存在`/etc/passwd`和`/etc/group`两个文件中，文件所有人可读。



## 3. sudo权限

新建的用户在权限方面会存在一定的限制，有时我们希望给他一些特殊的管理权限，例如useradd权限，我们可以通过sudo来实现，其配置文件为`/etc/sudoers`
```
# user=(username) host=(role) command
dba ALL=(root) NOPASSWD=/etc/sbin/useradd,/etc/sbin/userdel
mysql ALL=(ALL) NOPASSWD=ALL
```
添加对应权限后，用户只需要在命令之前加上sudo关键字即可
```
[dba@t-luhx01-v-szzb ~]$ sudo useradd test
[dba@t-luhx01-v-szzb ~]$ id test
uid=1018(test) gid=1018(test) groups=1018(test)
```
>NOPASSWD为免密选项，如果设置为ALL，则对应用户可以通过`sudo -i`直接免密切换到root用户，需注意安全问题


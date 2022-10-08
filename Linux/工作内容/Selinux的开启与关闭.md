#Linux #selinux
# Selinux的开启与关闭
## 开启selinux
修改配置文件(/etc/selinux/config)
```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
SELINUX由两种模式：
- enforcing：表示所有违反安全策略的行为都将被禁止
- permissive：表示所有违反安全策略的行为不被禁止，但是会在日志中作记录

修改conifg直接重启将会导致系统无法启动，需要现在根目录下面创建隐藏文件`.autorelabel`文件，这样重启后，selinux会自动重新标记所有系统文件
```
touch /.autorelabel
```

重启系统
```
shutdown -r now
```

## 验证Selinux状态
```
[root@test-luhengxing-dbatools ~]# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          disabled
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```

## 关闭selinux

临时关闭
```
setenforce 0
```

永久关闭则需要把配置文件中的SELINUX设置为disabled
```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
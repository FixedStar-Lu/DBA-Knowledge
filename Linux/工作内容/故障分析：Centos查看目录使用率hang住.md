**现象**

Centos7.2通过df -h查看目录使用率卡住，无法结束

**分析**

通过strace命令跟踪df命令，发现在stat(“/proc/sys/fs/binfmt_misc处卡住，查看unit状态

```
$ systemctl list-units -all | grep binfmt
proc-sys-fs-binfmt_misc.automount         loaded    active   running   Arbitrary Executable File Formats File System Automount Point
proc-sys-fs-binfmt_misc.mount             loaded    inactive dead      Arbitrary Executable File Formats File System
systemd-binfmt.service                    loaded    inactive dead      Set Up Additional Binary Formats
```
可见有两个服务的状态都已经是dead状态了，存在异常。

**解决方案**

bug链接：[Bug 1534701](https://bugzilla.redhat.com/show_bug.cgi?id=1534701)

>In systemd prior to 234 a race exists between .mount and .automount units such that automount requests from kernel may not be serviced by systemd resulting in kernel holding the mountpoint and any processes that try to use said mount will hang. A race like this may lead to denial of service, until mount points are unmounted.

大意就是systemd 234版本之前，mount和automount之前存在竞争，会导致服务中断，直到卸载安装点为止。我们可以通过systemctl restart proc-sys-fs-binfmt_misc.automount重启服务或升级到systemd-219-57版本来解决


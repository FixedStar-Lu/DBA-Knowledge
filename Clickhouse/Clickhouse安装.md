[TOC]

# ClickHouse安装

## 1. 环境配置

**查看系统是否支持SSE4.2**

```
[root@test-luhx ~]# grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
SSE 4.2 supported
```

**关闭防火墙**

```
[root@test-luhx ~]# systemctl stop firewalld.service
[root@test-luhx ~]# systemctl disable firewalld.service
```

**配置FQDN**

```
[root@test-luhx ~]# hostnamectl --static set-hostname test-luhx
[root@test-luhx ~]# cat >> /etc/hosts << EOF
192.168.56.10 test-luhx
EOF
```



## 2. 实例安装

**下载RPM介质**

[DownLoad Client](https://d28dx6y1hfq314.cloudfront.net/4574/4911/el/7/package_files/1014618.rpm?t=1628592155_9024290207359a0625871a87ccee1d7343db504a)

[DownLoad Common Static](https://d28dx6y1hfq314.cloudfront.net/4574/4911/el/7/package_files/1014619.rpm?t=1628592145_0945bf40ecc16f8937b17429747747a7d2ecfa8b)

[DownLoad Server](https://d28dx6y1hfq314.cloudfront.net/4574/4911/el/7/package_files/1014629.rpm?t=1628592136_7d7f0ce567bb02c19aecf8d19950c811d3575462)

[DownLoad Server Common](https://d28dx6y1hfq314.cloudfront.net/4574/4911/el/7/package_files/1014630.rpm?t=1628592122_06d118c1d528992d78a683ee4dbfe680bf527576)

**安装介质**

```
[root@test-luhx clickhouse]# rpm -ivh clickhouse-common-static-20.8.3.18-1.el7.x86_64.rpm 
准备中...                          ################################# [100%]
正在升级/安装...
   1:clickhouse-common-static-20.8.3.1################################# [100%]
[root@test-luhx clickhouse]# rpm -ivh clickhouse-server-common-20.8.3.18-1.el7.x86_64.rpm 
准备中...                          ################################# [100%]
正在升级/安装...
   1:clickhouse-server-common-20.8.3.1################################# [100%]
[root@test-luhx clickhouse]# rpm -ivh clickhouse-server-20.8.3.18-1.el7.x86_64.rpm 
准备中...                          ################################# [100%]
正在升级/安装...
   1:clickhouse-server-20.8.3.18-1.el7################################# [100%]
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse
[root@test-luhx clickhouse]# rpm -ivh clickhouse-client-20.8.3.18-1.el7.x86_64.rpm 
准备中...                          ################################# [100%]
正在升级/安装...
   1:clickhouse-client-20.8.3.18-1.el7################################# [100%]
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse
```

安装后的目录结构如下：

- /etc/clickhouse-server：配置文件目录
- /var/lib/clickhouse：默认的数据存储目录，需要在配置文件中修改为数据盘
- /var/log/clickhouse-server：默认的日志存储目录，需要在配置文件中修改为数据盘

- /etc/cron.d/clickhouse-server：定时任务配置，默认配置用于恢复因异常情况中断的clickhouse服务进程
- /usr/bin/clickhouse：主程序文件
- /usr/bin/clickhouse-client：客户端程序
- /usr/bin/clickhouse-server：服务启动程序
- /usr/bin/clickhouse-compressor：内置的压缩工具，可用于数据的压缩和解压

**创建数据目录**

```
[root@test-luhx ~]# mkdir /service/clickhouse/data -p
[root@test-luhx ~]# mkdir /service/clickhouse/log -p
[root@test-luhx ~]# chown -R clickhouse.clickhouse /service/clickhouse
```

**修改参数配置**

```
[root@test-luhx ~]# cat /etc/clickhouse-server/config.xml
<path>/service/clickhouse/</path>
<user_files_path>/service/clickhouse/user_files/</user_files_path>
<tmp_path>/service/clickhouse/tmp/</tmp_path>
<access_control_path>/service/clickhouse/access/</access_control_path>
<log>/service/clickhouse/log/clickhouse-server.log</log>
<errorlog>/service/clickhouse/log/clickhouse-server.err.log</errorlog>
<max_open_files>262144</max_open_files>
```

**启动服务**

```
[root@test-luhx ~]# service clickhouse-server start
Start clickhouse-server service: Path to data directory in /etc/clickhouse-server/config.xml: /service/clickhouse/
DONE
```

**登陆实例**

```
[root@test-luhx ~]# service clickhouse-server start
Start clickhouse-server service: Path to data directory in /etc/clickhouse-server/config.xml: /service/clickhouse/
DONE
[root@test-luhx ~]# clickhouse-client 
ClickHouse client version 20.8.3.18.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 20.8.3 revision 54438.

test-luhx :) 
```

> Clickhouse支持TCP和HTTP两种协议，TCP的端口为9000，主要用于集群通信和CLI客户端；HTTP端口为8123，可以通过REST服务的形式用于JAVA，python等客户端
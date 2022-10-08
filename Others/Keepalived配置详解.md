[TOC]

# Keepalived配置详解

keepalived是集群管理中保证集群高可用的一个服务软件，功能类似于heartbeat，防止单点故障。keepalived是以VRRP协议为实现基础的，VRRP即虚拟路由冗余协议。该协议将N台提供相同功能的路由器组成一个路由器组，这个组里面有个master和多个backup，master上绑定一个对外提供服务的VIP。当backup收不到vrrp包时，就会认为当前master宕机了，这时将根据定义的优先级从backup中选举一个新的master。



keepalived主要有三个模块：core、check和vrrp。core模块为keepalived的核心，负责进程的启动、维护以及全局配置文件的加载和解析；check负责健康检查，包括常见的各种检查方式。vrrp模块是实现VRRP协议的

## keepalived参数配置

**global_defs区域**

主要是配置故障发生时的通知对象以及机器标识

```
global_defs {
    notification_email {
        a@abc.com
        b@abc.com
        ...
    }
    notification_email_from alert@abc.com
    smtp_server smtp.abc.com
    smtp_connect_timeout 30
    enable_traps
    router_id host163
}
```

- notification_email：故障发生时发送邮件通知
- notification_email_from：通知邮件发出的账号
- smtp_server：邮箱服务器
- smtp_connect_timeout：连接smtp服务器的超时时间
- enable_traps：开启SNMP陷阱
- Router_id：标识本节点的字条串，通常可以设置为主机名

**static_ipaddress和static_routes区域**

Static_ipaddress和static_routes区域配置的是本节点的IP和路由信息。如果你的机器上已经配置了IP和路由，那么这两个区域可以不用配置

```
static_ipaddress {
    10.210.214.163/24 brd 10.210.214.255 dev eth0
    ...
}
static_routes {
    10.0.0.0/8 via 10.210.214.1 dev eth0
    ...
}
```

以上标识在启动关闭keepalived时在本机执行如下命令，因此强烈建议配置时忽略该区域

```
# /sbin/ip addr add 10.210.214.163/24 brd 10.210.214.255 dev eth0
# /sbin/ip route add 10.0.0.0/8 via 10.210.214.1 dev eth0
# /sbin/ip addr del 10.210.214.163/24 brd 10.210.214.255 dev eth0
# /sbin/ip route del 10.0.0.0/8 via 10.210.214.1 dev eth0
```

**vrrp_script区域**

该区域用于健康检查，当检查失败时会将vrrp_instance的priority减少相应的值

```
vrrp_script chk_http_port {
    script "</dev/tcp/127.0.0.1/80"
    interval 1
    weight -10
}
```

- script：执行的检查脚本
- interval：执行的时间间隔
- weight：检查失败时priority值的变化

**vrrp_instance和vrrp_sync_group区域**

vrrp_instance用来定义对外提供服务的VIP区域及其相关属性；vrrp_sync_group用来定义vrrp_instance组，使得组内成员动作一致

```
vrrp_sync_group VG_1 {
  group {
    http
    mysql
  }
  notify_master /path/to/to_master.sh
  notify_backup /path_to/to_backup.sh
  notify_fault "/path/fault.sh VG_1"
  notify /path/to/notify.sh
  smtp_alert
}

vrrp_instance http {
  state MASTER
  interface eth0
  use_vamc
  dont_track_primary
  track_interface {
    eth0
    eth1
  }
  mcast_src_ip <IPADDR>
  garp_master_delay 10
  lvs_sync_daemon_interface eth1
  virtual_router_id 51
  priority 100
  advert_int 1
  nopreempt
  authentication {
    auth_type PASS
    autp_pass 1234
  }
  virtual_ipaddress {
    10.240.204.169
  }
  virtual_routes {
    172.16.0.0/12 via 10.210.214.1
    192.168.1.0/24 via 192.168.1.1 dev eth1
    default via 202.102.152.1
  }
}
```

- notify_master/backup/fault：分别表示切换为主/备/出错时所执行的脚本；notify表示任何一种状态切换都会调用指定脚本
- smtp_alert：是否开启邮件通知
- State：可以选择master或backup，但是否为master是由prority的值决定的
- interface：节点网卡，用于发VRRP包
- use_vmac：是否使用VRRP的虚拟MAC地址
- dont_track_primary：忽略VRRP网卡错误
- track_interface：监控网卡
- mcast_src_ip：修改vrrp组播报的源地址，默认为master的ip
- garp_master_delay：当切为主状态多久更新ARP缓存，默认5秒
- lvs_sync_daemon_insterface：绑定lvs sync的网卡
- Virtual_router_id：取值在0-225之间，用于区分多个instance的VRRP组播。检查网络中存在的vrid：tcpdump -nn -i any net 224.0.0.0/8
- prority：设置优先级，取值在1-255，值最高的将成为master
- advert_int：发VRRP包的时间间隔，即健康检查的时间间隔
- authentication：认证区域，有PASS和HA
- virtual_ipaddress：VIP配置
- virtual_routes：虚拟路由，当IP漂过来之后需要添加的路由信息
- nopreempt：允许优先级较低的成为master

**virtual_server_group和virtual_server区域**

```
virtual_server IP Port {
    delay_loop <INT>
    lb_algo rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind NAT|DR|TUN
    persistence_timeout <INT>
    persistence_granularity <NETMASK>
    protocol TCP
    ha_suspend
    virtualhost <STRING>
    alpha
    omega
    quorum <INT>
    hysteresis <INT>
    quorum_up <STRING>|<QUOTED-STRING>
    quorum_down <STRING>|<QUOTED-STRING>
    sorry_server <IPADDR> <PORT>
    real_server <IPADDR> <PORT> {
        weight <INT>
        inhibit_on_failure
        notify_up <STRING>|<QUOTED-STRING>
        notify_down <STRING>|<QUOTED-STRING>
        # HTTP_GET|SSL_GET|TCP_CHECK|SMTP_CHECK|MISC_CHECK
        HTTP_GET|SSL_GET {
            url {
                path <STRING>
                # Digest computed with genhash
                digest <STRING>
                status_code <INT>
            }
            connect_port <PORT>
            connect_timeout <INT>
            nb_get_retry <INT>
            delay_before_retry <INT>
        }
    }
}
```

- delay_loop：延迟轮询时间，单位为秒
- lb_algo：后端调试算法
- lb_kind：调度类型NAT/DR/TUN
- virualhost：用来HTTP_GET和SSL_GET配置请求header
- Sorry_server：当所有real server宕掉时，sorry server顶替
- Real_server：提供服务的节点
- weight：权重
- notify_up/down：当real server宕机或启动时执行的脚本
- path：请求real server的路径
- digest/status_code：分别表示用genhash算出的结果和http状态码
- connect_port：健康检查，如果端口通则表示服务器正常
- connect_timeout,nb_get_delay,delay_before_retry分别表示超时时长、重试次数、下次尝试的时间延迟
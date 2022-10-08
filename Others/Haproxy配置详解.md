# Haproxy配置详解

**Haproxy简介**

Haproxy 是一款提供高可用性、负载均衡以及基于TCP（第四层）和HTTP（第七层）应用的代理软件，支持虚拟主机，它是免费、快速并且可靠的一种解决方案。可以用客户端IP地址或者任何其他地址来连接后端服务器. 这个特性仅在Linux 2.4/2.6内核打了cttproxy补丁后才可以使用



**Haproxy工作原理**

HAProxy由前端（frontend）和后端（backend），前端和后端都可以有多个。也可以只有一个listen块来同时实现前端和后端。这里主要讲一下frontend和backend工作模式。前端（frontend）区域可以根据HTTP请求的header信息来定义一些规则，然后将符合某规则的请求转发到相应后端（backend）进行处理

**Haproxy参数配置详解**

Haproxy的配置文件由两部分组成：全局设定和代理设定。共分为5段：global，defaults，frontend，backend，listen

- global：全局参数配置，属于进程级的配置
- defaults：默认参数配置，这些参数可以在frontend，backend，listen中使用。设置的默认参数值，会自动引用到frontend、backend、listen中，如果frontend、backend、listen部分也配置了与defaults一样的参数，defaults部分参数对应的值自动被覆盖
- Frontend：接收请求的前端虚拟节点，Frontend可以更加规则直接指定具体使用后端的backend
- backend：后端服务集群的配置，一个Backend对应一个或者多个后端实体服务器
- listen：Fronted和backend的组合体， 比如haproxy实例状态监控部分配置

global模块

```
global
    log 127.0.0.1 local0               /* 定义haproxy日志输出设置 */
    log 127.0.0.1 local1 notice        
    maxconn 4096                       /* 定义最大连接数 */
    user haproxy                       /* 运行haproxy的用户 */
    group haproxy                      /* 运行haproxy的组 */
    daemon                             /* 后台方式运行 */
    pidfile /usr/local/haproxy/run/haproxy.pid   /* haproxy进程pid文件 *
```

defaults模块

```
defaults
    log global                         /* 引入global定义的日志格式 */
    mode http                          /* 所处理的类别(http为7层，tcp为4层) */
    option tcplog                      /* 日志类别 [tcplog | httplog] */
    option dontlognull                 /* 不记录健康检查日志信息 */
    retries 3                          /* 尝试连接3次，如果3次连接失败则认为该服务不可用 */
    option redispatch                  /* 节点异常时重定向至正常节点 */
    maxconn 2000                       /* 最大连接数 */
    option  forwardfor except 127.0.0.0/8         /* 后端服务器获取真实的客户端IP */
    stats refresh 30                   /* 设置统计页面刷新时间间隔 */
    balance leastconn                                    /* 设置负载均衡方式 */
    timeout http-request 30s                    /* 客户端发送http请求超时时间 */
    timeout queue 5m                   /* 请求队列超时时间 */
    timeout connect 10s                /* ha与后端服务器连接超时时间 */ 
    timeout client 30m                 /* 客户端非活动连接超时时间 */
    timeout server 30m                 /* ha与后端服务器非活动连接超时时间 */
    timeout check 10s                  /* 心跳检查超时时间 */
```

frontend模块

```
frontend pxc-front                   /* 定义frontend名称 */
    bind *:3307                        /* 设置监听端口 */
    mode tcp                           /* 定义为tcp模式 */
    default_backend pxc-back           /* 设置后端服务器的backend */
```

backend模块

```
backend pxc-back                      /* 定义backend名称 */
    mode tcp                            /* 定义为tcp模式 */
    balance leastconn                   /* 这种均衡方式会把新连接发送到连接较少的节点 */                   
    #option httpchk GET /index.html     /* 心跳检测 */
    server node157 10.240.204.157:33006 check port 9200 inter 12000 rise 3 fall 3  /* 定义后端服务器列表 */
    server node165 10.240.204.165:33006 check port 9200 inter 12000 rise 3 fall 3  /* 定义后端服务器列表 */
    server node149 10.240.204.149:33006 check port 9200 inter 12000 rise 3 fall 3  /* 定义后端服务器列表 */
```

listen模块

```
backend pxc-back                      /* 定义backend名称 */
    mode tcp                            /* 定义为tcp模式 */
    balance leastconn                   /* 这种均衡方式会把新连接发送到连接较少的节点 */                   
    #option httpchk GET /index.html     /* 心跳检测 */
    server node157 10.240.204.157:33006 check port 9200 inter 12000 rise 3 fall 3  /* 定义后端服务器列表 */
    server node165 10.240.204.165:33006 check port 9200 inter 12000 rise 3 fall 3  /* 定义后端服务器列表 */
    server node149 10.240.204.149:33006 check port 9200 inter 12000 rise 3 fall 3  /* 定义后端服务器列表 */
```

均衡算法

- roundrobin：基于权重进行的轮询算法
- static-rr：静态方式的轮询算法，在运行时调整权重不生效
- source：基于请求IP的算法，对请求IP进行hash运算。这种方法可以使一个客户端IP始终请求同一个后端服务器
- leastconn：将请求发送到连接数最少的后端服务器，适用于数据库负载均衡
- uri：此算法会对部分或整个URI进行hash运算，再经过与服务器的总权重要除，最后转发到某台匹配的后端服务器上。
- uri_param：此算法会椐据URL路径中的参数时行转发，这样可以保证在后端真实服务器数量不变时，同一个用户的请求始终分发到同一台机器
- hdr：此算法根据httpd头时行转发，如果指定的httpd头名称不存在，则使用roundrobin算法进行策略转发。
- rdp-cookie(name)：示根据据cookie(name)来锁定并哈希每一次TCP请求
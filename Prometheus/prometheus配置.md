[TOC]

---

### Prometheus配置

**下载Prometheus**

[DownLoad](https://github.com/prometheus/prometheus/releases/download/v2.27.1/prometheus-2.27.1.linux-amd64.tar.gz)

**解压安装包**

```
[root@lhx media]# tar -xvf prometheus-2.27.0.linux-amd64.tar.gz
[root@lhx media]# mv prometheus-2.27.0.linux-amd64 /usr/local/prometheus
[root@lhx media]# echo "export PATH=$PATH:/usr/local/prometheus" >>/etc/profile
[root@lhx media]# source /etc/profile
```

**配置参数**
```
[root@lhx media]# cat /usr/local/prometheus/prometheus.yml 
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```

需要监听的服务      对象都需要添加到scrape_configs模块，例如node_exporter或者mysql_exporter。每次修改参数文件都需要重启才能生效，Prometheus也提供了多种服务发现的方式：
- azure_sd_configs
- consul_sd_configs
- dns_sd_configs
- ec2_sd_configs
- openstack_sd_configs
- file_sd_configs
- kubernetes_sd_configs
- marathon_sd_configs
- nerve_sd_configs
- serverset_sd_configs
- triton_sd_configs

具体可参考：[CONFIGURATION](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config)，这里我们可以采用file_sd_configs的方式，将配置都写入一个单独的文件，当文件发生更新时，Prometheus也会自动更新。
```
[root@lhx media]# mkdir /usr/local/prometheus/nodes && cd /usr/local/prometheus/nodes

[root@lhx media]# vi node.yml
- targets: ['192.168.56.20:9100']
  labels:
    name: linux-node1
    
[root@lhx media]# cat >> /usr/local/prometheus/prometheus.yml << EOF
  - job_name: 'linux'
    file_sd_configs:
    - files: ['/data/prometheus/linux.yml']
      refresh_interval: 5s
EOF
```

保存后可以`promtool check config promethus.yml`命令来检查配置文件

**创建用户**
```
[root@lhx media]# groupadd prometheus
[root@lhx media]# useradd -g prometheus -m -d /data/prometheus/ -s /sbin/nologin prometheus
```

**设置系统服务**
```
[root@lhx media]# cat /usr/lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/
After=network.target

[Service]
Type=forking
User=prometheus
ExecStart=/usr/local/prometheus/prometheus --config.file=/usr/local/prometheus/prometheus.yml --web.enable-lifecycle --storage.tsdb.path=/data/prometheus/ --storage.tsdb.retention=60d
Restart=on-failure

[Install]
WantedBy=multi-user.target

[root@lhx media]#  systemctl daemon-reload
[root@lhx media]#  systemctl enable prometheus.service
[root@lhx media]#  systemctl start prometheus.service
```
- --web.enable-lifecycle：2.0之后版本支持热加载(`curl -X POST http://host:9090/-/reload`)
- --storage.tsdb.path：TSDB数据保存位置
- --storage.tsdb.retention：数据保留周期

**访问prometheus**

prometheus Index：http://192.168.56.20:9090

### Exporter配置

exporter是Prometheus中非常重要的一个组件，主要负责数据采集任务，官方提供了node_exporter、mysql_exporter等，也有第三方市场提供了不少类型的exporter可供使用。

> 官方github：[Prometheus](https://github.com/prometheus)

**解压安装包**
```
[root@lhx media]# tar -xvf node_exporter-1.2.0.linux-amd64.tar.gz 
[root@lhx media]# cd node_exporter-1.2.0.linux-amd6
[root@lhx media]# cp node_exporter-1.1.2.linux-amd64 /usr/local/node_exporter
```

**配置系统服务**
```
[root@lhx media]# cat /usr/lib/systemd/system/node_exporter.service
[Unit]
Description=node_exporter
After=network.target

[Service]
Type=forking
User=prometheus
ExecStart=/usr/local/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target

[root@lhx media]#  systemctl daemon-reload
[root@lhx media]#  systemctl enable node_exporter.service
[root@lhx media]#  systemctl start node_exporter.service
```

**注册Prometheus**
```
[root@lhx media]# cat /service/prometheus/nodes/linux-host.yml
- targets: ['192.168.56.20:9100']
  labels:
    name: Prometheus-Host

[root@lhx media]# cat >> /usr/local/prometheus/prometheus.yml << EOF
  - job_name: 'linux-host'
    file_sd_configs:
    - files: ['/service/prometheus/nodes/linux-host.yml']
      refresh_interval: 5s
EOF
```

**重新加载**
```
[root@lhx media]# curl -X POST  http://localhost:9090/-/reload
```

**查看target**

![Targets]()


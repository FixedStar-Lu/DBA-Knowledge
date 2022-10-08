[TOC]

---

### 安装Grafana

Grafana是一款用Go语言开发的开源数据可视化工具，可以做数据监控和数据统计和展示，可以用于对接Prometheus的监控数据

**下载安装包**

[DownLoad](https://links.jianshu.com/go?to=https%3A%2F%2Fdl.grafana.com%2Foss%2Frelease%2Fgrafana-7.5.6-1.x86_64.rpm)

**安装字体包**
```
[root@lhx media]# yum install urw-fonts freetype* fontconfig -y
```

**安装grafana**
```
[root@lhx media]# rpm -ivh grafana-7.5.6-1.x86_64.rpm 
warning: grafana-7.5.6-1.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 24098cb6: NOKEY
Preparing...                ########################################### [100%]
   1:grafana                ########################################### [100%]
```

**配置参数**
```
[root@lhx media]# cat /etc/grafana/provisioning/dashboards/sample.yaml
# # config file version
apiVersion: 1

providers:
 - name: 'default'
   orgId: 1
   folder: ''
   folderUid: ''
   type: file
   options:
     path: /var/lib/grafana/dashboards
```

**开启插件功能**
```
[root@lhx media]# cat /etc/grafana/grafana.ini
plugins = /var/lib/grafana/plugins
```

**启动grafana**
```
[root@lhx media]# systemctl start grafana-server
```

**访问grafana**

Grafana Index：http://192.168.56.20:3000/login
user：admin
password：admin

**添加数据源**

将prometheus的数据添加到grafana中。主页选择Configuration->Add data source->prometheus，填写prometheus的配置信息

![add_datasource](https://gitee.com/dba_one/dba_note/raw/master/images/add_datasource.png)



**配置Dashboard**

选择Dashboards->Manage->New Dashboard->add an empty panel

> 通过[dashboards](https://grafana.com/grafana/dashboards)可以下载到很多Dashboard模板


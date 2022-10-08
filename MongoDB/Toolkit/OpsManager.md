[TOC]

# Ops Manager安装部署

## 1. 安装Ops Manager

**下载Ops Manager**

[DownLoad](https://www.mongodb.com/subscription/downloads/archived#archived_OnPrem)

**安装MongoDB**

安装Ops之前需要准备一个MongoDB用于存放Ops数据，安装教程可参考`环境部署`一章

**安装Ops**

```
$ rpm -ivh mongodb-mms-4.0.5.50245.20181031T0042Z-1.x86_64.rpm
```

**配置参数文件**

```
$ vi /opt/mongodb/mms/conf/conf-mms.properties
mms.centralUrl=http://10.0.139.162:8080
mongo.mongoUri=mongodb://10.0.139.162:27017
automation.versions.source=local
```

如果Ops数据库为副本集且开启了认证登录URI需要设置如下

```
mongo.replicaSet=repl01
mongo.mongoUri=mongodb://USER:PASSWORD@10.0.139.161:27017,10.0.139.162:27017,10.0.139.163:27017
```

> 链接用户需要有readWriteAnyDatabase、clusterAdmin和dbAdminAnyDatabase权限

**启动Ops**

```
$ service mongodb-mms start
Starting pre-flight checks
Successfully finished pre-flight checks

Migrate Ops Manager data
   Running migrations...[  OK  ]
Start Ops Manager server
   Instance 0 starting....[  OK  ]
Starting pre-flight checks
Successfully finished pre-flight checks
```



## 2. 配置实例

启动Ops之后，就可以通过web访问控制台

**设置邮件**

![register](https://gitee.com/dba_one/wiki_images/raw/master/images/img01.png)

**设置web服务**

![web server](https://gitee.com/dba_one/wiki_images/raw/master/images/img02.png)

**设置邮件**

![Email](https://gitee.com/dba_one/wiki_images/raw/master/images/img03.png)

**版本设置**

进入主页后，需要对Ops Manager上的MongoDB版本进行管理

![warring](https://gitee.com/dba_one/wiki_images/raw/master/images/img04.png)

拉到最下方，选择update version

![update version](https://gitee.com/dba_one/wiki_images/raw/master/images/img05.png)



由于内网机器没有连接网络，更新会失败，我们可以根据提示在本地联网电脑上访问复制到窗口中

![version manifest](https://gitee.com/dba_one/wiki_images/raw/master/images/img06.png)

将需要托管的MongoDB数据库版本安装包(TGZ)放到/opt/mongodb/mms/mongodb-releases/下，再回到version Manager页面，选中对应版本并点击左上角REVIEW & DEPLOY按钮确认

![DEPLOY](https://gitee.com/dba_one/wiki_images/raw/master/images/img07.png)



## 3. Agent安装 

### 3.1 Automation Agent安装

>  对需要加入到Ops Manager中的MongoDB节点都需要安装Automation Agent，并需要加入/etc/hosts解析。

点击Agents查看对应的版本的Agent安装说明

![Deployment](https://gitee.com/dba_one/wiki_images/raw/master/images/img08.png)

> **Automation Agent Installation Instructions**
> To save time, you can repeat each step of these instructions in parallel across servers with the same OS
>
> 1.Download the agent
>
> ```
> curl -OL http://10.0.139.162:8080/download/agent/automation/mongodb-mms-automation-agent-manager-5.4.13.5505-1.x86_64.rhel7.rpm
> ```
>
> and install the package.
>
> ```
> sudo rpm -U mongodb-mms-automation-agent-manager-5.4.13.5505-1.x86_64.rhel7.rpm
> ```
>
> 2.Create a new Agent API Key. After being generated, keys will only be shown once.Treat this API Key like a password
>
> 3.Next, open the config file
>
> ```
> sudo vi /etc/mongodb-mms/automation-agent.config
> ```
>
> and enter your API key, Project ID, and Ops Manager Base URL as shown below.
>
> ```
> mmsGroupId=5e86eedd4e57d5944be91d06
> mmsApiKey=<Insert Agent API Key Here>
> mmsBaseUrl=http://10.0.139.162:8080
> ```
>
> To manage your API keys, visit the Agent API Keys tab
>
> 4.Prepare the /data directory to store your MongoDB data. This directory must be owned by the mongod user
>
> ```
> sudo mkdir -p /data
> sudo chown mongod:mongod /data
> ```
>
> 5.Start the agent.
>
> ```
> sudo systemctl start mongodb-mms-automation-agent.service
> ```
>
> On SUSE, it may be necessary to run:
>
> ```
> sudo /sbin/service mongodb-mms-automation-agent start
> ```



### 3.2 Monitoring agent安装

根据官方建议，每一套监控环境之中只需要安装一个monitoring agent/backup agent即可；因此，我们选择ops manager的监控节点作为monitor agent的安装节点

![Monitoring Agent](https://gitee.com/dba_one/wiki_images/raw/master/images/img09.png)

点击左上角REVIEW & DEPLOY进行确认安装

![DEPLOY](http://dbapub.cn/images/img07.png)

![REVIEW](https://gitee.com/dba_one/wiki_images/raw/master/images/img10.png)

**创建监控用户**

```
> use admin
> db.createUser(
{
  user: "ops",
  pwd: "ops",
  roles: [ { role: "clusterAdmin", db: "admin" },{ role: "dbAdminAnyDatabase", db: "admin" },{ role: "readWriteAnyDatabase", db: "admin" },{ role: "userAdminAnyDatabase", db: "admin" }]
})
```

**添加已存在的项目**

![add project](https://gitee.com/dba_one/wiki_images/raw/master/images/img11.png)

填写节点相关信息，群集环境只需要填写一个节点的信息，会自动获取关联节点，分片环境可直接填入Mongos的信息

![node](https://gitee.com/dba_one/wiki_images/raw/master/images/img12.png)

等遍历完所有节点信息，即可点击Continue

![Continue](https://gitee.com/dba_one/wiki_images/raw/master/images/img13.png)

取消自动安装AUTOMATION

![canal automation](https://gitee.com/dba_one/wiki_images/raw/master/images/img14.png)

**查看监控信息**

![view monitor](https://gitee.com/dba_one/wiki_images/raw/master/images/img15.png)

点击对应节点能获取节点详细信息

![view node](https://gitee.com/dba_one/wiki_images/raw/master/images/img16.png)



### 3.3 Backup Agent安装

Ops Manager额外提供了MongoDB的热备份功能，备份对象是针对副本集或分片环境，单实例无法进行备份

![Ops Backup](https://gitee.com/dba_one/wiki_images/raw/master/images/ops.png)

安装Backup Agent

![Install Backup Agent](https://gitee.com/dba_one/wiki_images/raw/master/images/bak01.png)

点击配置BACKUP模块

![Setting Backup Module](https://gitee.com/dba_one/wiki_images/raw/master/images/backup01.png)

配置Head database目录，用以存放backup daemon实例

![Setting Backup Module](https://gitee.com/dba_one/wiki_images/raw/master/images/backup02.png)

选择备份类型

![Setting Backup Module](https://gitee.com/dba_one/wiki_images/raw/master/images/backup03.png)

> Ops Manager额外提供了MongoDB的热备份功能，支持三种备份方式：
>
> - File System：文件系统存储，每次备份都会生成一个目录单独存放备份快照，不支持增量备份
> - Database Storage：以MongoDB数据库作为备份存储，支持增量备份
> - AWS S3 Bucket：磁带库存储，备份快照以块格式存储在磁带库中，支持增量备份

回到首页，点击begin setup

![begin setup](https://gitee.com/dba_one/wiki_images/raw/master/images/backup04.png)

校验backup agent

![check agent](https://gitee.com/dba_one/wiki_images/raw/master/images/backup05.png)

选择要备份的对象

![select object](https://gitee.com/dba_one/wiki_images/raw/master/images/backup06.png)

第一次会自动进行备份

![Backup](https://gitee.com/dba_one/wiki_images/raw/master/images/backup07.png)

检查快照

![check snapshot](https://gitee.com/dba_one/wiki_images/raw/master/images/backup08.png)

检查日志

![check logs](https://gitee.com/dba_one/wiki_images/raw/master/images/backup09.png)

查看快照保留策略

![快照策略](https://gitee.com/dba_one/wiki_images/raw/master/images/backup10.png)

> 分片环境需要开启checkpoint才能进行恢复
#MongoDB #Percona 

## 简述
Percona for Mongodb是一个免费开源完全兼容的，可直接替代具有企业版功能的MongoDB版本，提供以下功能：
- MongoDB的MMAPv1存储引擎和默认的WiredTiger引擎
- 可选的Percona Memory Engine和MongoRocks存储引擎
- 使用OpenLDAP活AD进行外部SASL身份认证
- 审核日志记录以跟踪和查询用户或应用程序的数据库交互
- 默认WiredTiger和备用的MongoRocks存储引擎的热备份
- 分析率可降低分析器对性能的影响

**功能比较**


 \ | PSMDB | MongoDB | TokuMX
 -----|------|-------------|----------------
Storage Engines	| WiredTiger (default) / Percona Memory Engine | WiredTige(default) / In-Memory (Enterprise only) | Built-in storage engine based on the Fractal Tree index
Hot Backup | YES for WiredTiger | NO | YES
Audit Logging | YES | Enterprise only | YES
External SASL Authentication | YES | Enterprise only | NO
Profiling Rate Limit | YES | YES [1] | NO
Geospatial Indexes | YES | YES | YES
Text Search | YES | YES | NO
ACID and MVCC compliant | NO | NO | YES
Clustering Key Support | NO | NO | YES
Sharding with Clustering Keys | NO | NO | YES
Point-in-time Recovery | NO | Enterprise only | YES

## 实例安装

**包装清单**

- Percona-Server-MongoDB-34：安装mongo shell，导入/导出工具，其它客户端实用程序，服务器软件，默认配置init.d脚本
- Percona-Server-MongoDB-34-server：包含mongod服务器，默认配置文件和init.d脚本
- Percona-Server-MongoDB-34-shell：包含mongoshell
- Percona-Server-MongoDB-34-mongos：包含分片mongos群集查询路由器
- Percona-Server-MongoDB-34-tools：包含来自Percona的高性能MongoDB fork的Mongo工具
- Percona-Server-MongoDB-34-debuginfo：包含服务器的调试

**安装server和shell**

```
$ rpm -ivh Percona-Server-MongoDB-34-shell-3.4.18-2.16.el6.x86_64.rpm
$ rpm -ivh Percona-Server-MongoDB-34-server-3.4.18-2.16.el6.x86_64.rpm
```

**编辑参数文件**

```
storage:
    dbPath: /mongodb/data
    journal:
        enabled: true
    engine: wiredTiger

systemLog:
    destination: file
    logAppend: true
    path: /mongodb/log/mongod.log

processManagement:
    fork: true
    pidFilePath: /mongodb/data/mongod.pid

net:
    port: 27017
    bindIp: all
```

**创建管理员**

```
test> use admin
switched to db admin
admin> db.createUser({user: 'root', pwd: 'Abcd123#', roles: ['root'] });
Successfully added user: { "user" : "root", "roles" : [ "root" ] }
```

**启用认证登陆**

```
security:
    authorization: enabled
```

**重启实例**

```
$ service mongod restart
```

## Hot Backup

Percona MongoDB 3.2开始默认支持WiredTiger引擎在线热备份，需要管理员权限在admin数据库下执行createBackup，并指定备份目录

**备份恢复原理**

备份原理：
- 首先启动一个后台检测的进程，实时检测MongoDB Oplog的变化，将新产生的日志写入到日志文件WiredTiger.backup中
- 复制MongoDB dbpath目录到指定的备份目录中
	
恢复原理：
- 将WiredTiger.backup日志进行回放，将操作日志应用到WiredTiger引擎里，最终得到一致性快照恢复
- 把备份目录里的数据文件直接拷贝到dbpath下，然后启动MongoDB

**PHP备份脚本**

安装依赖
```
yum install -y php-pear php-devel php gcc openssl openssl-devel cyrus-sasl cyrus-sasl-devel
```

**安装php-mongo驱动**

[Download](https://pecl.php.net/get/mongo-1.6.16.tgz)
```
$ tar -xvf mongo-1.6.16.tar
$ cd mongo-1.6.16
$ phpize
$ ./configure
$ make && make install -j 4
```

<p>在php.ini中加入extension=mongo.so</p>

**创建PHP脚本**

```
<?php
  
ini_set('date.timezone','Asia/Shanghai');
error_reporting(7);
 
$user = "root";
$pwd = 'Abcd123#'; 
$host = '127.0.0.1';
$port = '27017';
$authdb = 'admin';
$BAKDIR = "/data/bak/";
$BAKDIR .= date('Y_m_d_H_i_s');
 
 
$m = new MongoBak($user,$pwd,$host,$port,$authdb,$BAKDIR);
 
Class MongoBak{
 
    function __construct(){ 
         $a = func_get_args();
            $f = 'backup';
            call_user_func_array(array($this,$f),$a); 
    }
 
    function backup($p1,$p2,$p3,$p4,$p5,$p6){
           $this->mkdirs("{$p6}");
        $mongo = new MongoClient("mongodb://{$p1}:{$p2}@{$p3}:{$p4}/{$p5}");
       # $mongo->setReadPreference(MongoClient::RP_SECONDARY);
           $db = $mongo->admin;
        echo "\n".date('Y-m-d H:i:s')."    backup is running......\n";
           $r = $db->command(
              array(
                   'createBackup' => 1,
                    'backupDir' => $p6),
                  array('timeout' => -1)    
        );
 
    //print_r($r);
 
        if($r['ok'] == 1){
               echo date('Y-m-d H:i:s')."    backup is success.\n";
        } else{
            echo date('Y-m-d_H:i:s')."    backup is failure.\n";
        }   
    }
     
    function mkdirs($dir, $mode = 0777){
        if (!is_dir($dir)){
            mkdir($dir, $mode, true);
            chmod($dir, $mode);
            echo "BackupDir : ${dir} is created.\n";
        }
    }
 
}
 
?>
```
*注意：该脚本在PHP7下无效*

**执行备份**

```
$ php pmongo_backup.sh
```

> 恢复时只需将备份文件覆盖到数据目录下启动即可

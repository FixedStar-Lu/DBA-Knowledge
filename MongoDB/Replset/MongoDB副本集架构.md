[TOC]

# MongoDB副本集

为了解决MongoDB单点故障问题，推出了副本集复制功能，其中主节点用于处理客户端读写请求，从节点用于复制主节点的数据，当主节点故障时，副本集自动切换提升一个成员作为新的主节点继续提供服务。

![replica](https://docs.mongodb.com/manual/_images/replica-set-read-write-operations-primary.bakedsvg.svg)



## 1. 成员角色

primary节点是副本集中唯一能够执行写操作的成员，一个副本集只能存在一个primary节点，其它secondary节点通过应用primary上的oplog完成数据同步。

secondary节点实时同步primary节点数据，并作为主节点的候选人，当主节点发生故障，会提升一个secondary作为新的主节点。除此之外，secondary还可以设置如下属性：

- 优先级：通过选项priority来设置优先级权重。拥有最高优先级的成员会优先选举作为主节点，只要其满足大多数的要求。如果节点优先级为0，则无法升级为primary，但能参加选举投票

  ```
  > var config = rs.config()
  > config.members[2].priority=20
  > rs.reconfig(config)
  ```

- 隐藏成员：通过配置选项hidden:true来隐藏指定节点，前提是节点优先级为0。设置后客户端不会向隐藏成员发送请求，隐藏成员也不会作为复制源，因此通常可以作为备份节点。

  ```
  > var config = rs.config()
  > config.members[2].priority=0
  > config.members[2].hidden=0
  > rs.reconfig(config)
  ```

- 延迟节点：为防止数据被破坏，可以使用配置选项slaveDelay设置一个延迟节点，延迟节点的数据会比主节点延迟指定的时间，单位为秒。设置延迟节点时，需要先将优先级设置为0并隐藏该节点，避免接收客户端请求。

  ```
  > var config = rs.config()
  > config.members[2].priority=0
  > config.members[2].hidden=0
  > config.members[2].slaveDelay=3600
  > rs.reconfig(config)
  ```

如果成员被定义为仲裁者，则不保存任何数据，也不对客户端提供服务，它仅作为投票者，具有选举功能，来帮助副本集达到大多数这个选举条件。我们可以通过rs.addArb(node)来添加仲裁节点或者在定义配置文件时对仲裁节点添加arbiterOnly:true来声明该节点是仲裁节点。



## 2. 数据同步

### 2.1 初始化

**初始化同步**

副本集成员启动后，就会检查自身状态，确定是否可以从某个节点进行同步。如果不行，则尝试进行完整的数据复制。这个过程就是初始化同步，它主要包含以下步骤：

1. 选择一个复制源，在local.me中为创建一个标识符，删除已经存在的数据库
2. 将同步源的数据复制到本地
3. 记录克隆过程中所有操作到oplog中
4. 将上述的oplog同步到从节点，确保有足够的空间暂存
5. 数据复制完成，创建索引
6. 应用创建索引过程中产生的oplog
7. 完成初始化同步，从STARTUP2切换到secondary状态

如果在初始化过程中遇到了非瞬时的网络错误，同步需要重新启动。MongoDB4.4开始，如果遇到临时的网络错误，集合删除等情况，会尝试恢复同步进程。默认情况下，从节点会尝试在24小时内恢复初始化同步，也可以通过参数`initialSyncTransientErrorRetryPeriodSeconds`控制尝试恢复初始化同步的时间，时间范围内无法恢复的话，会重新选择一个复制源进行初始化同步。

初始化同步过程中，第二步和第五步都比较耗时，可能会导致节点远远落后同步源，从而导致复制数据被覆盖。另外初始化同步会强制将当前成员的所有数据加载到内存中，导致频繁的访问的数据不能常驻内存，导致请求变慢。因此建议在副本集空闲时间执行并确保oplog足够大。

在MongoDB4.4中，我们可以通过参数`initialSyncSourceReadPreference`手动指定初始化复制源，可以设置为primary(从主节点同步)，也可以设置为primaryPreferred(尝试从primary同步，如果primary失败则从其它成员进行同步)。

初始化同步时，会对所有副本集内成员遍历两次，第一次遍历成员需要满足以下条件：

- 同步源必须是primary或secondary状态
- 同步源必须是online并可达的
- 参数`initialSyncSourceReadPreference`设置为secondary或者secondaryPreferred，同步源必须为secondary
- 同步源必须是可见的
- 同步源与primary延迟在30秒以内
- 如果成员为build index，同步源也必须为build index
- 如果成员有投票权，同步源也必须有投票权
- 如果成员没有设置延迟节点，同步源也不允许设置延迟
- 如果成员是延迟节点，同步源必须设置较短的延迟
- 同步源必须比当前最佳的同步源更快

如果第一次没有符合的成员，将放松要求进行第二次遍历，第二次成员需要满足以下条件：

- 同步源必须是primary或secondary状态
- 同步源必须是online并可达的
- 参数`initialSyncSourceReadPreference`设置为secondary或者secondaryPreferred，同步源必须为secondary
- 如果成员为build index，同步源也必须为build index
- 同步源必须比当前最佳的同步源更快

如果成员通过两次遍历依旧无法选择同步源，将会抛出一个错误并等待1秒后重新遍历，10次都不成功将退出。



### 2.2 增量复制

初始化同步完成之后，从同步源复制oplog，并异步应用这些操作。成员会根据ping时间和复制状态的变化，自动选择复制源。

从4.4开始，复制源会将连续的oplog条目发送到其同步的secondary，减轻了高负载和网络延迟情况下的复制延迟，它还具有以下特点：

- 减少从secondary读取的过时
- 减少primary failover时w:1的写操作丢失的风险
- 使用w:majority和w:>1来减少写操作的延迟

在MongoDB4.2开始，管理员可以通过流控限制主节点写入速度，目的是将大多数提交的延迟保持在`flowControlTargetLagSeconds`之下。流控默认是启用的，当延迟增长到接近`flowControlTargetLagSeconds`时，主节点写入必须先获取tickets，然后利用锁来应用写入，通过限制每秒的tickets进行流控。

MongoDB通过多个线程批量应用写入操作来提高系统并发性，其根据文档ID进行分组，使用不同的线程来应用操作。MongoDB始终以原始写入顺序对文档应用写入操作。



## 3. 高可用选举

副本集通过选举来确定主节点成员，副本集中以下事件会触发选举：

- 副本集添加新成员
- 初始化副本集
- 使用rs.stepDown()或rs.reconfig维护副本集
- secondary节点丢失主节点链接超过了超时时间(默认为10秒)

在选举成功之前，副本集不能处理写入操作，在secondary上执行的查询请求不受影响。默认情况下，选择新的primary时间不应超过12秒，可以通过参数settings.electionTimeoutMillis配置选举所需的时间，网络延迟等因素会延长副本集选举的时间。3.6开始，MongoDB驱动程序能够检测到primary链接丢失，并自动重试写入操作，这需要我们在连接串中添加`retryWrites = true`显示启用

影响副本集选举的因素和条件包括：

- 选举协议：MongoDB4.0之前使用pv0(protocol version 0)，MongoDB4.0开始使用pv1(protocol version 1)，其可以配置catchUpTimeoutMillis在更快的故障转移和保留w:1写入之间确定优先级
- 心跳：复制集成员每两秒钟互相发送心跳(ping)。如果心跳在10秒内没有返回，则其他成员将错误成员标记为不可访问。
- 成员优先级：副本集会尽可能将优先级最高的节点选举为primary，当它满足大多数原则，数据也是最新的等条件即刻就能选举成为primary，即使不满足也会待其满足成为primary的条件后重新选举成为primary
- 镜像读取：从mongodb4.4开始，MongoDB提供镜像读取，将预热的可选举secondary与最近访问的数据一起存储。通过镜像读取，primary可以镜像其接收操作的子集，并将其发送到可选择的secondary的子集。预热secondary的缓存可以帮助在选举后更快地恢复性能。
- 数据中心丢失：对于分布式副本集，数据中心丢失可能会影响一个或多个数据中心其余成员选举primary的能力，可能无法满足大多数的原则
- 网络分割：网络分割可能会将primary分配到一个少数成员的区中，primary会退化为secondary，大多数成员区将重新选举出一个primary

为了获取到副本集中其它成员的状态，成员每隔两秒会向其它节点发送一个心跳请求，主要用于检查成员状态，判断主节点是否满足大多数要求，不满足则重新选举。成员状态除了primary和secondary，也包含一些其它状态：

- STARTUP：成员刚启动时处于这个状态，加载完配置后切换到STARTUP2状态
- STARTUP2：MongoDB会创建几个线程，用于处理复制和选举，然后切换到RECOVERY状态
- RECOVERY：该状态下暂时无法处理请求，在处理非常耗时的操作时，成员也可能进入该状态。当成员与其它成员脱节时，也会进入该状态，这时可能需要重新同步
- ARBITER：仲裁者始终处于该状态
- DOWN：如果成员不可达时，就会处于DOWN状态
- UNKNOWN：如果成员无法到达其其它任何成员，其它成员就无法知道它处于什么状态，会将其报告为UNKNOWN状态
- REMOVED：成员被移除副本集时，它处于REMOVED状态
- ROLLBACK：如果成员正在进行回滚，它就处于ROLLBACK状态，回滚完成切换到RECOVERY状态，然后成为secondary
- FATAL：如果一个成员发生了不可挽回的错误，也不再尝试恢复正常的话，就处于FATAL状态



## 4. 数据回滚

如果主节点执行完写操作后就挂了，从节点可能没来得及复制该操作，那么新选举出来的主节点就会遗漏这次写操作。当挂掉的主节点恢复后，就会向其它复制源进行数据复制，当其无法从其它节点获取到最后的写操作就会进行回滚，撤销这次写操作，从共同点继续复制。

对于每个回滚数据的集合，回滚文件位于/rollback/目录中，文件名为removed..bson

```
mongorestore --db temp --collection test /mongodb/data/rollback/20f74796-d5ea-42f5-8c95-f79b39bad190/removed.2020-06-27T15-30-00.0.bson
```

> 4.0开始，参数`createRollbackDataFiles`可以控制回滚期间是否创建回滚文件

如果想要获取对应的集合名，我们可以在日志文件中过滤rollback file，也可以使用下列语句进行遍历

```
var mydatabases=db.adminCommand("listDatabases").databases;
var foundcollection=false;

for (var i = 0; i < mydatabases.length; i++) {
   let mdb = db.getSiblingDB(mydatabases[i].name);
   collections = mdb.getCollectionInfos( { "info.uuid": UUID("20f74796-d5ea-42f5-8c95-f79b39bad190") } );

   for (var j = 0; j < collections.length; j++) {   // Array of 1 element
      foundcollection=true;
      print(mydatabases[i].name + '.' + collections[j].name);
      break;
   }

   if (foundcollection) { break; }
}
```

如果回滚操作是集合删除或文档删除，则不会将集合删除和文档删除的回滚写入回滚数据目录。如果想要读取回滚文件的内容，我们可以使用bsondump来解析。

回滚注意事项：

- 4.2开始，当成员进入回滚状态，MongoDB终止所有正在执行的用户操作
- 4.0会等待所有后台创建索引完成后再开始回滚，4.2会等待所有索引创建完成后再回滚
- 禁用”majority”读关注可以防止修改索引的callmod命令回滚
- 在之前的版本中，MongoDB不会回滚超过300M的数据，4.0开始不进行限制
- 4.0开始，回滚限制时间为24小时，可以修改TimeLimitSecs参数进行配置

为避免数据回滚的情况，我们希望不管发生什么都将写操作都确认写入操作复制到了副本集的大多数。我们可以通过getLastError命令配合w选项强制要求getLastError等待，一直到给定数量的成员都执行完最后的写入操作，仅阻塞当前会话。w选项的值可设置为majority来要求大多数成员都执行完成。

当执行该命令时，如果副本集只有一个主节点和仲裁节点，主节点无法将操作复制到副本集任何成员。getLastError无法确定要等待多久，会一直等待下去。因此，我们还应该设置wtimeout选项指定超时时间，如果超时还没有返回则返回错误。

```
db.runCommand({"getLastError : 1, "w" : "majoriry", "wtimeout" : 1000})
```



## 5. 副本集设计

**确定成员数量**

在设计副本集时，需要考虑到一个概念就是”大多数”。在MongoDB的选举规则中，选择的主节点要求得到大多数节点的支持才能成为主节点，这里的大多数被定义为大于群集成员数量一半以上，因此通常节点数量设计为奇数

> 副本集最大只能有50个成员，其中只有7个节点有投票权，超过7个需要将多余成员的votes设置为0，并不是副本集成员越多越好，副本集越大心跳请求的网络流量和选举花费的时间就越大。虽然没这些成员不可参加主节点选举，但可以在选举中投否决票。

**最多只能一个仲裁节点**

有时，基于成本和系统重要性考虑，我们可能不需要太多的数据副本，会引入仲裁节点来满足”大多数”原则。但需要注意的事，如果节点已经是奇数，就不需要添加仲裁者，添加仲裁者也不会提升副本集的稳定性，因此往往建议最多添加一个仲裁节点即可。

仲裁节点的缺点也十分明显，在一个包含主-从-仲裁三节点的架构中，其中一个数据节点down了无法恢复，此时副本集将面临后续恢复初始化同步的压力以及主节点的读写压力，带来了较大的不稳定性。如果可能，尽量避免使用仲裁者

**隐藏节点和延迟节点**

在设计中，我们也可以引入隐藏节点或延迟节点的特性来实现一些特定需求，例如备份或者报表

**读负载均衡**

在一个读请求非常高的场景下，可以通过将读分发给secondary来提供读取的吞吐量，降低primary性能压力。

**多数据中心**

如果副本集成员都集中在一个数据中心，当数据中心出现断电等异常情况，副本集将出现不可用状态。为了提高副本集的容错性，我们应该采用多数据中心的方案进行部署。基于大多数原则，推荐下列两种配置方式：

- 将大多数成员放在同一个数据中心，适用于双中心配置，但如果大多数所在的机房出现访问，副本集将无法选举出新的主节点
- 双中心部署相同数量的节点，在第三方位置添加一个决定性的副本集成员，进一步提升了可靠性。



## 6. 副本集维护

**修改成员状态**

当我们进行副本集维护时，会手动修改副本集成员状态。

```
rs.stepDown()
```

这个命令会让主节点退化为从节点，并维持60秒，也可以手动设置一个时间值。如果这段时间没有选举出新的主节点，这个节点就会重新参加选举

如果不希望维护期间，其它成员选举为主节点，可以在每个从节点上执行freeze命令，以强制它们处于secondary

```
rs.freeze(10000)
```

维护完成后，如果想提前释放其它成员，可以再次执行freeze(0)

除此之外，我们也可以认为让成员进入维护模式，成员会变成RECOVERING状态，客户端将不再发送请求到这个成员上，也不能作为复制源。

```
db.adminCommand({"replSetMaintenanceMode" : "true"})
```

**配置复制链路**

在从节点执行rs.status()命令时，输出信息中会有一个”syncingTo”的字段，其用于表示当前成员以哪个节点作为复制源进行复制。如果在所有从节点上执行replSetGetStatus命令就能弄清楚副本集整个的复制链路情况。

MongoDB根据ping时间选择复制源，一个成员向另一个成员发送心跳请求，就可以知道ping所耗费的时间。MongoDB维护着不同节点成员间心跳请求的平均花费时间。选择同步源时，会选择一个离自己比较近而且数据也比自己新的成员。

自动复制链路也存在一些缺点，复制链路越长，将写操作复制到所有节点所花费的时间就越长。极端情况可能会形成一种串行的复制链路，每个从节点都要比前面的从节点要落后，这种情况可以通过rs.syncFrom命令修改成员的复制源。

当然，我们也可以禁用复制链路，要求所有从节点都从主节点进行复制，只需要将allowChaining设置为false

```
var config = rs.config()
config.settings = config.settings || {}
config.settings.allowChaining = false
rs.reconfig(config)
```

**跟踪延迟**

延迟是指从节点相对于主节点的落后程度，是主节点最后一次操作的时间戳与从节点最后一次操作时间戳的差值。在主节点执行db.printReplicationInfo()或在从节点执行db.printSlaveReplicationInfo()能够快速获得同步信息

```
shard1:PRIMARY> db.printReplicationInfo()
configured oplog size:   2048MB
log length start to end: 8659767secs (2405.49hrs)
oplog first event time:  Thu Mar 19 2020 11:39:52 GMT+0800 (CST)
oplog last event time:   Sat Jun 27 2020 17:09:19 GMT+0800 (CST)
now:                     Sat Jun 27 2020 17:09:28 GMT+0800 (CST)
```

输出信息中包含了oplog的大小，以及oplog包含的操作时间范围，如果log length start to end较小可能就需要对oplog进行扩容了。

```
shard3:SECONDARY> db.printSlaveReplicationInfo()
source: 10.0.139.161:22000
        syncedTo: Sat Jun 27 2020 17:12:05 GMT+0800 (CST)
        0 secs (0 hrs) behind the primary
```

从上面的输出可以看出当前成员的复制源以及相对于主节点的复制延迟。
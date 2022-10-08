MongoDB3.2开始默认使用WiredTiger存储引擎，

### 数据结构

**write flow**

**read flow**

### Compression

### Transaction isolation

**read-uncommited**

**read-commited**

**snapshot**

### Journal

**checkpoint**
checkpoint的目的在于：
- 实现数据库在某个时刻意外发生故障，再次启动时，缩短数据库的恢复时间

触发checkpoint的条件默认有两个：
- 默认每60秒做一次checkpoint
- journal日志达到2G
- 任何打开的数据文件被修改，关闭时将自动执行一次checkpoint

### Block Manager

### Cache

MongoDB默认情况下采用tcmalloc作为内存管理器，而非glibc。WiredTiger引擎通过参数cacheSizeGB来限制引擎内存缓存的最大上限，其大小为下列两种情况下的最大值：
- 50% of (RAM - 1 GB)
- 256MB

除了MongoDB cache外，还会在主机内存划分一块最大500M的内存区域用于创建索引。其它剩余内存作为文件系统缓存，MongoDB能够将压缩的磁盘文件缓存到内存中，减少磁盘I/O。

**Page淘汰机制**

MongoDB通过LRU队列算法进行page clean，Wiredtiger引擎目前有4个参数来配置eviction策略

参数 | 默认值 | 描述
-- | -- | --
eviction_target | 80 | 当cache used使用超过eviction_target后，evict线程开始淘汰CLEAN PAGE
eviction_trigger | 95 | 当cache used使用超过eviction_trigger，用户线程开始淘汰CLEAN PAGE，用户请求出现缓慢
eviction_dirty_target | 5 | 当cache dirty超过eviction_dirty_target，evict线程开始淘汰DIRTY PAGE
eviction_dirty_trigger | 20 | 当cache dirty超过eviction_dirty_trigger，用户线程开始淘汰DIRTY PAGE，用户请求出现缓慢

> 存在另一种情况就是当页随着插入更新等操作后大小超过memory_page_max参数的限制时，也会强制触发page eviction动作。先将大的page拆分为多个小page，再通过reconcile将这些小的pages保存到磁盘上，reconcile完成之后page就可以淘汰出去了。

默认只有一个线程负责page clean，如果想要加快page clean的速度，我们可以试着提高eviction的线程数
```
db.adminCommand({setParameter : 1,"wiredtigerEngineRuntimeConfig" : "eviction=(threads_min=4,threads_max=8)"})
```
淘汰一个页时，会先锁住相应的page并检查该页上是否存在其它请求，如果有则不会对该页进行eviction

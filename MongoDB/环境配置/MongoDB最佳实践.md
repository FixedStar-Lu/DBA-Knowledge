### 安全
1. 开启认证鉴权
2. 权限管理
3. 白名单设置(authenticationRestrictions)
4. 备份策略

### 操作系统配置
1. 采用SSD及RAID10
2. 采用XFS文件系统
3. 避免超大缓存
默认是60秒做一次checkpoint。做checkpoint需要对内存内所有脏数据遍历以便整理然后把这些数据写入硬盘。如果缓存超大（如大于128G），那么这个checkpoint时间就需要较长时间
4. 关闭Transparent Huge Pages
Transparent Huge Pages (THP) 是Linux的一种内存管理优化手段，通过使用更大的内存页来减少Translation Lookaside Buffer(TLB)的额外开销
5. 定时Log Rotation，避免日志过大
6. 分配足够大的oplog
7. 增加文件描述符和线程数
8. 禁用NUMA
9. NTP时间同步
10. 关闭文件系统的atime
11. 设置readahead

### 监控指标
1. CPU和内存
2. 磁盘及IO
3. 复制延迟
4. 连接数
5. oplog窗口
6. 慢查询
7. 脏页比例
8. 等待情况

### 索引

### 模式设计

### 连接驱动


参考链接：[MongoDB 最佳实践 - MongoDB中文社区](https://mongoing.com/archives/3895)

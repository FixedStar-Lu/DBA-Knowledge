# DMS同步任务配置

## 复制实例

## 连接端点
### Kafka Target
1. 通过offset Explorer创建Topic
2. 创建kafka类型的终端节点，添加如下配置

参数 | 值 | 说明
-- | -- | --
Topic | 具体的Topic名称 | 目标kafka Topic
MessageFormat | json-unformatted | 数据格式
PartitionIncludeSchemaTable | true | Topic数据中key默认为主键，当同步多张表到同一个topic中，开启该参数key则为schema.table_name.primary

## 同步任务
### Kafka Target

**全量加载(性能优化)：**
`MaxFullLoadSubTasks` — 使用此选项指示要并行加载的表的最大数目。AWS DMS使用专用的子任务将每个表加载到其对应的 Kafka 目标表。默认值为 8；最大值为 49。

`ParallelLoadThreads` — 使用此选项指定以下内容的线程数：AWS DMS使用将各个表加载到其 Kafka 目标表中。Apache Kafka 目标的最大值为 32。您可以请求提高此最大值限制。

`ParallelLoadBufferSize` — 使用此选项指定在缓冲区（并行加载线程将数据加载到 Kafka 目标时使用）中存储的最大记录数。默认值是 50。最大值为 1,000。将此设置与 ParallelLoadThreads 一起使用；仅在有多个线程时ParallelLoadBufferSize 才有效。

`ParallelLoadQueuesPerThread` — 使用此选项指定每个并发线程访问的队列数，以便从队列中取出数据记录并为目标生成批处理负载。默认值为 1。但是，对于各种负载大小的 Kafka 目标，有效范围为每个线程 5-512 个队列。

**CDC加载(性能优化)：**
`ParallelApplyThreads` — 指定并行线程的数量：AWS DMS在 CDC 加载期间使用来将数据记录推送到 Kafka 目标终端节点。默认值为零 (0)，最大值为 32。

`ParallelApplyBufferSize` — 指定在 CDC 加载期间要在每个缓冲区队列中存储的最大记录数，以便将并发线程推送到 Kafka 目标终端节点。默认值为 100，最大值为 1,000。当 ParallelApplyThreads 指定多个线程时，请使用此选项。

`ParallelApplyQueuesPerThread`— 指定每个线程访问的队列数，以便从队列中取出数据记录并在 CDC 期间为 Kafka 终端节点生成批处理负载。
查看当前集群配置

```
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port
FROM system.clusters

Query id: 5724f05a-d5b8-46a3-8b9c-ec7d0d6b6e59

┌─cluster───────────────────────┬─shard_num─┬─replica_num─┬─host_name─────┬─port─┐
│ quickstart_clickhouse_cluster │         1 │           1 │ 172.30.36.212 │ 9000 │
│ quickstart_clickhouse_cluster │         1 │           2 │ 172.30.37.73  │ 9000 │
│ quickstart_clickhouse_cluster │         2 │           1 │ 172.30.36.161 │ 9000 │
│ quickstart_clickhouse_cluster │         2 │           2 │ 172.30.37.214 │ 9000 │
└───────────────────────────────┴───────────┴─────────────┴───────────────┴──────┘
```

创建数据库

```
create database test_log on cluster quickstart_clickhouse_cluster
```

创建本地表

```
CREATE TABLE test.test_20210812_1 on cluster quickstart_clickhouse_cluster
(
	`id` UInt64,
	`name` String,
	`age` UInt64,
	`address` String,
    `birtyday` DateTime
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}','{replica}')
PARTITION BY toYYYYMM(birtyday)
ORDER BY id
```

创建分布式表

```
CREATE TABLE test.test_20210812_1_D on cluster quickstart_clickhouse_cluster AS test.test_20210812_1 ENGINE = Distributed(
 'quickstart_clickhouse_cluster',
 'test',
 'test_20210812_1',
 rand())
```

插入一条测试数据

```
insert into test_20210812_1 values(1,'lu',18,'shenzhen',now())
```

查看每个节点是否能访问

```
SELECT * FROM test_20210812_1_D

Query id: 2c26f40c-a16d-453e-a4d5-92e495e51733

┌─id─┬─name─┬─age─┬─address──┬────────────birtyday─┐
│  1 │ lu   │  18 │ shenzhen │ 2021-08-12 14:54:00 │
└────┴──────┴─────┴──────────┴─────────────────────┘
```


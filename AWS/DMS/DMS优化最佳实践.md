
# DMS优化最佳实践



## 1. 并行加载多个表

AWS DMS默认一次加载8张表，当复制实例配置较高时，可以考虑扩大该值来提升性能。复制实例压力较大时也可以考虑减小该值

修改方式：修改任务->高级设置->Tuning Settings->Maximum number of tables to load in parallel

## 2. 并行加载单个表

对于带有分区或子分区的Oracle、MSSQL、MySQL、Sybase、DB2源时，添加并行可以改善FULL LOAD的效率，加快大表或分区表的迁移。

要使用并行加载，需要在表映射配置中添加parallel-load选项，其中type类型可选下列5个值：
- partitions-auto：表或视图的所有分区都是并行加载的
- subpartitions-auto：仅针对Oracle，表或视图的所有子分区都是并行加载的
- partitions-list：表或视图的所有指定分区都是并行加载的
- ranges：表或视图的所有指定段都是并行加载的
- none：表或视图是在单线程任务中加载的，默认值
```
{
    "rules": [{
            "rule-type": "selection",
            "rule-id": "1",
            "rule-name": "1",
            "object-locator": {
                "schema-name": "%",
                "table-name": "%"
            },
            "rule-action": "include"
        },
        {
            "rule-type": "table-settings",
            "rule-id": "2",
            "rule-name": "2",
            "object-locator": {
                "schema-name": "HR",
                "table-name": "SALES"
            },
            "parallel-load": {
                "type": "ranges",
                "columns": [
                    "SALES_NO",
                    "REGION"
                ],
                "boundaries": [
                    [
                        "1000",
                        "NORTH"
                    ],
                    [
                        "3000",
                        "WEST"
                    ]
                ]
            }
        }
    ]
}
```
上面的并行配置含义如下：
1. SALES_NO 小于或等于 1000 且 REGION 小于“NORTH”的行。也就是EAST区域最高为 1,000 的销售数量
2. 除上面1以外SALES_NO 小于或等于 3000 且 REGION 小于“WEST”的行。也就是NORTH 和 SOUTH 区域超过 1,000 且不超过 3,000 的销售数量
3. 除上面1和2以外的所有行

更多内容参考：[表和集合设置规则和操作](https://docs.aws.amazon.com/zh_cn/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TableMapping.SelectionTransformation.Tablesettings.html)

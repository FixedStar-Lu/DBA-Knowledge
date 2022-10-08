[TOC]

# MySQL8参数优化

## 1. resultset_metadata

| System Variable      | `resultset_metadata` |
| :------------------- | -------------------- |
| Scope                | Session              |
| Dynamic              | Yes                  |
| SET_VAR Hint Applies | No                   |
| Type                 | Enumeration          |
| Default Value        | FULL                 |
| Valid Values         | FULL<br>NONE         |

resulset_metadata是在MySQL8.0.3中引入的参数，用于控制是否返回结果集的元数据，例如：列的数量，列名，列的类型等信息。FULL表示返回，NONE则表示不返回，在不需要元数据信息时，可以在客户端配置中NONE来提升性能。

[resultset_metadata性能测试]:https://mp.weixin.qq.com/s/Kpa3KWjRz4ndJVlaYlACEw




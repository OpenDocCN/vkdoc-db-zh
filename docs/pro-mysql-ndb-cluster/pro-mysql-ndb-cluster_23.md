# 9. NDB 集群中的维护任务

| 名称 | 描述 |
| --- | --- |
| `allow` | 是否允许查询。通常与 `ndb_index_stat_enable` 值相同，但如果索引统计信息尚未初始化，则可能为 0。 |
| `enable` | 根据 `ndb_index_stat_enable` 的关闭或开启状态，值为 0 或 1。 |
| `busy` | 统计线程当前是否繁忙。 |
| `loop` | 统计线程当前的等待时间，单位为毫秒。 |
| `list` | 各种列表（shares）的统计信息。每个列表中的条目代表了在受相应批次大小限制的循环中，索引统计线程需要完成的工作。 |
| `analyze` | `ANALYZE TABLE` 的统计信息，包括排队等待计算索引统计信息的表数量。 |
| `stats` | 特殊计数器。 |
| `total` | 总计数器。例如，`analyze` 中的 `all` 值对应 `ANALYZE TABLE` 语句的数量，而 `query` 中的 `all` 值对应索引查询统计信息的查找次数。查询值都可能因常规查询需要索引或检查索引统计信息而增加。 |
| `cache` | 索引统计信息缓存的各种统计信息。`query` 值是当前缓存中使用的字节数（与 `Ndb_index_stat_cache_query` 相同），`usedpct` 是可用缓存的百分比，`highpct` 是自上次重置统计信息以来的最高使用率。这些值对于确定缓存大小是否合适很有用。 |

清单 9-9 显示了在多次执行 `ANALYZE TABLE` 语句和使用索引的查询后，`Ndb_index_stat_status` 以及 `Ndb_index_stat_cache_query` 和 `Ndb_index_stat_cache_clean` 中的统计信息示例。在生成输出时，一个更新四张表索引统计信息的 `ANALYZE TABLE` 操作正在进行中。这反映在 `Ndb_index_stat_status` 的 `analyze` 字段值中。

```
mysql> SHOW GLOBAL STATUS LIKE 'Ndb\_index\_stat\_%'\G
*************************** 1. row ***************************
Variable_name: Ndb_index_stat_status
Value: allow:1,enable:1,busy:1,loop:100,list:(new:0,update:3,read:0,idle:5,check:0,delete:0,error:0,total:8),analyze:(queue:3,wait:1),stats:(nostats:0,wait:0),total:(analyze:(all:43,error:0),query:(all:22,nostats:16,error:0),event:(act:0,skip:0,miss:0),cache:(refresh:45,clean:8,pinned:0,drop:1,evict:0)),cache:(query:145667,clean:247816,drop:0,evict:0,usedpct:1.17,highpct:1.17)
*************************** 2. row ***************************
Variable_name: Ndb_index_stat_cache_query
Value: 145667
*************************** 3. row ***************************
Variable_name: Ndb_index_stat_cache_clean
Value: 247816
3 rows in set (0.00 sec)
Listing 9-9.
索引统计信息索引变量中的统计信息示例
```

### 总结

重要的是要记住，设置数据库系统从来不是一劳永逸的任务，而是一个持续进行的项目。本章讨论了与 `NDBCluster` 表相关的若干日常维护任务。讨论的主题包括：

*   在线与离线模式变更
*   重新分区表
*   碎片整理
*   维护索引统计信息及其重要性

许多模式变更可以在线进行，对系统的影响相对较小。这使得集群在大多数情况下能够保持在线。然而，对于某些变更，需要离线复制模式变更；如果需要对表进行彻底的碎片整理，同样适用这种情况。

索引统计信息对于优化器确定最优查询计划非常重要。本章讨论了 `NDBCluster` 表的索引统计信息如何工作以及如何保持其最新。

下一章将介绍集群中节点的重启，包括几个示例。

脚注 1

请记住，SQL 数据库中的列是有序的，即使在关系理论中并非如此。


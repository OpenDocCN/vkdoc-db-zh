# IORM 相关指标概览

我们关注的 IORM 监控指标具有以下 `objectType`：`IORM_DATABASE`、`IORM_CATEGORY`、`IORM_PLUGGABLE_DATABASE` 和 `IORM_CONSUMER_GROUP`。这些指标通过 `METRICCURRENT` 对象中的 `name` 属性进行进一步组织。指标名称是表示 I/O 消费者组类型、存储设备类型和描述性名称的缩写组合。`name` 属性的元素格式如下：
```
{consumer_type}_{device type}_{metric}
```
其中 `consumer_type` 代表 IORM 资源组，为以下之一：
* `DB` = 跨数据库 IORM 计划
* `CT` = 类别 IORM 计划
* `CG` = 数据库内 IORM 计划
* `PDB` = 可插拔数据库 IORM 计划

而 `device_type` 是服务 I/O 请求的存储类型，为以下之一：
* `FC` = 闪存缓存
* `FD` = 基于闪存的网格磁盘
* `’’` = 如果既非前者也非后者，则指标代表对物理磁盘的 I/O

属性的最后一部分 `{metric}` 是指标的描述性名称。指标名称可能进一步由 `SM` 或 `LG` 限定，表示其代表小型 I/O 请求或大型 I/O 请求。例如：
* `CG_FC_IO_RQ_LG`：为 DBRM 消费者组从闪存缓存（FC）服务的大型 I/O 请求总数。
* `CG_FD_IO_RQ_LG`：为 DBRM 消费者组从基于闪存的网格磁盘（FD）服务的大型 I/O 请求总数。
* `CG_IO_RQ_LG`：为 DBRM 消费者组从物理磁盘（网格磁盘）服务的大型 I/O 请求总数。

下表列出了跨数据库、类别、数据库内和可插拔数据库资源计划所共有的指标。此列表并非详尽无遗；还有更多指标。

| 名称 | 描述 |
| --- | --- |
| `{CG,CT,DB,PDB}_IO_BY_SEC` | 为此 I/O 消费者调度的每秒兆字节数。 |
| `{CG,CT,DB,PDB}_IO_LOAD` | 此 I/O 消费者的平均 I/O 负载。I/O 负载指定磁盘队列长度。它类似于 iostat 的 `avgqu-sz`，但该值根据磁盘类型加权；对于硬盘，大型 I/O 的权重是小型 I/O 的三倍。对于闪存盘，大型和小型 I/O 权重相同。 |
| `{CG,CT,DB,PDB}_IO_RQ_{SM,LG}` | 来自此 I/O 消费者的小型或大型 I/O 请求的累计数量。 |
| `{CG,CT,DB,PDB}_IO_RQ_{SM,LG}_SEC` | 此 I/O 消费者每秒发出的小型或大型 I/O 请求数量。 |
| `{CG,CT,DB,PDB}_IO_WT_{SM,LG}` | 来自此 I/O 消费者的 I/O 请求等待 IORM 调度所花费的累计时间（毫秒）。 |
| `{CG,CT,DB,PDB}_IO_WT_{SM,LG}_RQ` | 由上面的 `{CG,CT,DB}_IO_WT_{SM,LG}` 派生而来。它存储了在过去一分钟内 I/O 请求等待 IORM 调度所花费的平均等待时间（毫秒）。大量的等待表明此 I/O 消费者的 I/O 工作负载超出了其分配量。对于较低优先级的消费者，这可能是期望的效果。对于较高优先级的消费者，这可能表明应分配更多 I/O 资源以满足组织的目标。 |
| `{CG,CT,DB,PDB}_IO_UTIL_{SM,LG}` | 此 I/O 消费者消耗的总 I/O 资源的百分比。 |
| `{CG,CT,DB,PDB}_IO_TM_{SM,LG}` | 此 I/O 消费者读取小型和大型块的累计延迟。 |
| `{CG,CT,DB,PDB}_IO_TM_{SM,LG}_RQ` | 此 I/O 消费者读取块的平均延迟。 |

以上所有累计指标在 `cellsrv` 重启、IORM 计划启用或该 I/O 消费者组的 IORM 计划发生变化时，都会被重置为 0。例如，如果 IORM 类别计划被更改，以下累计指标将被重置：
```
CT_IO_RQ_SM
CT_IO_RQ_LG
CT_IO_WT_SM
CT_IO_WT_LG
CT_IO_TM_SM
CT_IO_TM_LG
```
这些 IORM 指标进一步使用 `metricObjectName` 属性进行分类。跨数据库资源计划指标存储在详细记录中，其中 `metricObjectName` 设置为相应的数据库名称。类似地，IORM 类别计划的指标通过与类别名称匹配的 `metricObjectName` 来标识。IORM 消费者组由数据库名称和 DBRM 消费者组名称的串联来标识。最后，与 PDB 相关的 IORM 信息在 `metricObjectName` 字段中使用 `<database name>.<PDB name>` 语法。


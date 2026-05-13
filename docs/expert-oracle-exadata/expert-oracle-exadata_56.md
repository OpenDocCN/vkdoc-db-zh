# cellsrvstat 实用程序

长久以来，Oracle 一直将 `cellsrvstat` 实用程序作为其 cell 软件发行版的一部分提供。这是一个用于深入排查故障和研究 cell 软件工作原理的有用工具。该工具带有内置帮助，但除此之外，Oracle 并未真正为其提供文档。你可以在一些出版物和博客中找到若干参考资料。为了让你对该工具的功能有所了解，请参见以下内容：

```
用法：
cellsrvstat [-stat_group=<group name>,<group name>,]
[-offload_group_name=<offload_group_name>,]
[-database_name=<database_name>,]
[-stat=<stat name>,<stat name>,] [-interval=<interval>]
[-count=<count>] [-table] [-short] [-list]
stat                     一个代表统计项的、用逗号分隔的短字符串列表。
                         默认为全部。（除非指定了 -stat）。
                         -list 选项会显示所有统计项。
                         示例：-stat=io_nbiorr_hdd,io_nbiowr_hdd
stat_group               一个代表统计组的、用逗号分隔的短字符串列表。
                         默认：除 database 外的全部（除非指定了 -stat_group）。
                         -list 选项会显示所有统计组。
                         有效组包括：io, mem, exec, net,
                         smartio, flashcache, offload, database.
                         示例：-stat_group=io,mem
offload_group_name       一个代表卸载组名称的、用逗号分隔的短字符串列表。
                         默认：cellsrvstat -stat_group=offload
                         （所有卸载组，除非指定了 -offload_group_name）。
                         示例：-offload_group_name=SYS_121111_130502
database_name            一个代表数据库组名称的、用逗号分隔的短字符串列表。
                         默认：cellsrvstat -stat_group=database
                         （所有数据库，除非指定了 -database_name）。
                         示例：-database_name=testdb,proddb
interval                 获取并打印统计信息的间隔时间（以秒为单位）。默认为 1 秒。
count                    统计信息应被打印的次数。默认为一次。
list                     列出所有指标缩写及其描述。所有其他选项将被忽略。
table                    以表格格式输出。如果指定的所有指标都不是基于整数的指标，
                         此选项将被忽略。
short                    使用缩写的指标名称，而非描述性名称。
error_out                用于打印错误消息的输出文件，主要用于调试。
```

在研究智能扫描的机制时，将范围缩小到 `io`、`smartio` 或 `offload` 证明是有效的。秉承 UNIX 的优良传统，对该工具的一次调用会打印自开始收集以来的所有统计数据。如果你对当前的统计信息感兴趣，应指定 `interval` 和 `count` 参数。五秒的间隔时间配合至少为二的计数被证明是有效的。与 `vmstat` 和 `iostat` 一样，你可以放心地忽略第一批输出，而专注于第二批，因为后者代表了 cell 上的当前统计信息。以下是一个针对一个 80GB 表进行单次串行模式智能扫描期间 `io` 统计组的输出示例。统计名称后的数字是自上次快照以来的增量；其后的较大数字是自 cell 开始记录以来的事件累计总数：

```
== 输入/输出相关统计 ==
硬盘块 IO 读请求次数                                     1860       47269919
硬盘块 IO 写请求次数                                       18        1481441
硬盘块 IO 读取量 (KB)                                1881618    46815620582
硬盘块 IO 写入量 (KB)                                  170      210753174
闪存盘块 IO 读请求次数                               135071        6106538
闪存盘块 IO 写请求次数                                    7        2761123
闪存盘块 IO 读取量 (KB)                            8641696      372843144
闪存盘块 IO 写入量 (KB)                              188       52813784
磁盘 IO 错误次数                                          0              4
任务期间达到延迟阈值的警告次数                             0              2
检查程序引发的延迟阈值警告次数                             0              0
智能 IO 相关的延迟阈值警告次数                             0              0
重做日志写入相关的延迟阈值警告次数                         0              0
当前待发出的读块 IO 量 (KB)                              0              0
累计待发出的读块 IO 量 (KB)                            202       42461566
当前待发出的写块 IO 量 (KB)                             0              0
累计待发出的写块 IO 量 (KB)                           249      132637393
当前正在 IO 中的读块量 (KB)                              0              0
累计已发出的读块 IO 量 (KB)                            202       42461566
当前正在 IO 中的写块量 (KB)                              0              0
累计已发出的写块 IO 量 (KB)                           249      132637393
当前在网络发送中的读块 IO 量 (KB)                        0              0
累计在网络发送中的读块 IO 量 (KB)                      202       42461566
当前在网络发送中的写块 IO 量 (KB)                       0              0
累计在网络发送中的写块 IO 量 (KB)                     249      132637393
当前正在填充到闪存中的块 IO 量 (KB)                      0              0
累计已填充到闪存中的块 IO 量 (KB)                         0          401680
```

如果你跟随本章内容并在你的环境中尝试了这些示例，`cellsrvstat` 的输出看起来会非常熟悉。你从该命令行工具获得的很多信息，也可以从 `V$CELL` 视图中获取，尤其是从 `V$CELL_STATE` 视图。

## 总结

本章的重点是理解 Exadata 性能以及 Oracle 为性能分析师和研究者提供的各种相关指标。重要的是要记住，Exadata 智能扫描有可能加速数据检索，但智能扫描仅在使用直接路径读取和全段扫描时才会发生。同时，要记住仅凭孤立地查看执行计划，无法确定是否实际发生了智能扫描。

你应该始终检查额外的指标，例如在你的会话中是否看到 `cell smart table/index scan` 等待事件，以及在运行 SQL 时 `IO_CELL_OFFLOAD_ELIGIBLE_BYTES`（在 `V$SQL` 中）或 `cell physical I/O bytes eligible for predicate offload` 统计信息（在 `V$SESSTAT` 中）是否增加。正如第 10 章所解释的，对等待事件进行追踪是你可用来确认是否使用了智能扫描的另一种方法。希望所解释的许多其他指标对于理解和排查高级性能问题会有所帮助，例如当智能扫描启动但因诸多特殊条件（如链接行、一致性读回滚，或仅仅是 cell 服务器资源耗尽）而被节流时。在第 12 章中，我们将看到如何运用这些知识来监控和排查 Exadata 性能问题，并且我们也将更深入地研究来自 `cellsrv` 和操作系统的单元级性能指标。



# 理解时间戳格式

ExaWatcher 输出样本以一行时间戳为前缀，该行以“zzz”开头，如上所示。这向您显示了操作系统命令执行的确切时间，精确到秒。请注意，这个“zzz”行只显示监控命令何时执行。有些命令每小时只执行一次，但它们会在那一小时内持续收集并转储数据样本。在这种情况下，每个数据样本都会打印特定于监控命令的时间戳。以下示例展示了 `cellsrvstat` 命令如何使用 `===Current Time===` 来指示其下方性能数据的确切时间。您可以在自定义脚本中使用此前缀。示例如下：

```
# bzcat 2015_01_17_09_02_55_CellSrvStatExaWatcher_enkx3cel01.enkitec.com.dat.bz2 \
> | head -30
zzz <01/17/2015 09:02:55> Count:720
===Current Time===                                              Sat Jan 17 09:02:55 2015
== Input/Output related stats ==
Number of hard disk block IO read requests                         0      167987444
Number of hard disk block IO write requests                        0      150496829
Hard disk block IO reads (KB)                                      0   111186344274
Hard disk block IO writes (KB)                                     0    25224016152
Number of flash disk block IO read requests                        0       67230332
Number of flash disk block IO write requests                       0       15528794
Flash disk block IO reads (KB)                                     0      870806808
Flash disk block IO writes (KB)                                    0      347337424
Number of disk IO errors                                           0              0
Number of reads from flash cache                                   0       57890992
Number of writes to flash cache                                    0       13119701
Flash cache reads (KB)                                             0      870014160
Flash cache writes (KB)                                            0      347236056
Number of flash cache IO errors                                    0              0
Size of eviction from flash cache (KB)                             0              0
Number of outstanding large flash IOs                              0              0
Number of latency threshold warnings during job                    0          13400
Number of latency threshold warnings by checker                    0              0
Number of latency threshold warnings for smart IO                  0              0
Number of latency threshold warnings for redo log writes           0           6568
Current read block IO to be issued (KB)                            0              0
Total read block IO to be issued (KB)                              0     9066623802
Current write block IO to be issued (KB)                           0              0
Total write block IO to be issued (KB)                             0    25054696317
Current read blocks in IO (KB)                                     0              0
Total read block IO issued (KB)                                    0     9066623802
.... a lot of output removed ....
```

该输出仅是 `cellsrvstat` 每小时转储的五秒样本中的第一个。第一列显示单元指标名称，第二列（全为零）显示当前指标间隔（快照）内的指标值。因为命令刚刚执行（并获取了它的第一个样本），所以每个指标都显示为零，因为增量计算仅在获取第二个快照后才开始。如果您在 ExaWatcher 转储文件中向下浏览，您将看到从第二个快照开始，许多指标出现非零值。最后一列显示了每个指标的累积值（自 `cellsrv` 进程启动以来）。您可能应该忽略这个累积值，因为它包含了自 `cellsrv` 启动（可能是数月前）以来的信息。查看单个累积值，它累积了如此长时间的信息，对于当前正在发生什么或上周五上午 8 点发生了什么，不会告诉您太多。较短时间范围内的指标增量才是正确的方法，这就是您在第二列中看到的：截至采样指标时的当前间隔值。`cellsrvstat` 工具在 第 11 章 中有更详细的描述。


#### 使用 `metric_iorm.pl` 分析 IO 指标

另一个对性能分析师非常有用的工具是 Oracle 提供的名为 `metric_iorm.pl` 的脚本。它可以在 My Oracle Support note 1337265.1 中找到，是一个 Perl 脚本，需要部署到存储节点（cell）上以收集信息。这在某种程度上违反了“不允许向存储节点部署任何软件”的规则，但由于它是一个 Oracle 提供的 Perl 脚本，通常是允许的。

一旦部署完成，可以直接运行该脚本而无需参数，它会显示大量信息。在底层，它使用 `cellcli` 来收集指标，但结合了巧妙的计算以提供有用的输出。它不仅限于展示非 CDB 或 11.2 实例的 I/O 资源管理器（I/O Resource Manager）信息。它还能捕获 12c 可插拔数据库（Pluggable Databases）的 I/O 资源管理器信息。为了保持输出的简洁性，以下示例使用了一个 12c 非容器数据库（non-CDB）。在深入细节之前，让我们先查看一个 X2-2 存储节点的输出示例，该节点执行了一系列针对同一数据段的智能扫描（Smart Scans）（这是一个在现实中不太真实的用例，但很好地展示了 I/O 压力）：

```
[root@enkcel04 ∼]# ./metric_iorm.pl

Database: _OTHER_DATABASE_

Utilization:      Small=0%    Large=0%

Flash Cache:      IOPS=98.7

Disk Throughput: MBPS=0

Small I/O’s:      IOPS=0.4    Avg qtime=0.0ms

Large I/O’s:      IOPS=0.0    Avg qtime=0.0ms

Consumer Group: _ORACLE_BACKGROUND_GROUP_

Utilization:      Small=0%    Large=0%

Flash Cache:      IOPS=98.7

Disk Throughput: MBPS=0

Small I/O’s:      IOPS=0.3    Avg qtime=0.0ms

Large I/O’s:      IOPS=0.0    Avg qtime=0.0ms

...

Database: DBM01

Utilization:      Small=0%    Large=86%

Flash Cache:      IOPS=42595

Disk Throughput: MBPS=962

Small I/O’s:      IOPS=1.4    Avg qtime=0.0ms

Large I/O’s:      IOPS=919    Avg qtime=1611ms

Consumer Group: HIGHPRIO_GROUP

Utilization:      Small=0%    Large=82%

Flash Cache:      IOPS=40501

Disk Throughput: MBPS=915

Small I/O’s:      IOPS=0.0    Avg qtime=0.0ms

Large I/O’s:      IOPS=874    Avg qtime=913ms

Consumer Group: _ORACLE_BACKGROUND_GROUP_

Utilization:      Small=0%    Large=0%

Flash Cache:      IOPS=4.6

Disk Throughput: MBPS=0

Small I/O’s:      IOPS=1.4    Avg qtime=0.0ms

Large I/O’s:      IOPS=0.0    Avg qtime=0.0ms

Consumer Group: LOWPRIO_GROUP

Utilization:      Small=0%    Large=3%

Flash Cache:      IOPS=2090

Disk Throughput: MBPS=47

Small I/O’s:      IOPS=0.0    Avg qtime=0.0ms

Large I/O’s:      IOPS=45.5    Avg qtime=15027ms

Consumer Group: _ORACLE_LOWPRIBG_GROUP_

Utilization:      Small=0%    Large=0%

Flash Cache:      IOPS=0.2

Disk Throughput: MBPS=0

Small I/O’s:      IOPS=0.0    Avg qtime=0.0ms

Large I/O’s:      IOPS=0.0    Avg qtime=0.0ms

...

CELL METRICS SUMMARY

Cell Total Utilization:      Small=0%    Large=86%

Cell Total Flash Cache:      IOPS=42730.9

Cell Total Disk Throughput: MBPS=965

Cell Total Small I/O’s:      IOPS=39.1

Cell Total Large I/O’s:      IOPS=919

Cell Avg small read latency:   76.17 ms

Cell Avg small write latency: 32.64 ms

Cell Avg large read latency:   52.82 ms

Cell Avg large write latency: 0.00 ms
```

从上面的输出可以看出，该脚本将信息按数据库分段显示，并提供了整个存储节点的摘要。如果你的监控范围是整个网格（Grid），你需要通过 `dcli` 执行该脚本，并可能将输出整合到一个文件中以便进一步处理。否则，半机架和全机架配置会产生过多的输出。在此特定示例中，数据库 `DBM01` 是 I/O 活动的唯一目标。你可以看到利用率主要集中在大型 I/O 请求和闪存设备上。实际上，由于 11.2.3.3.0 及更高版本引入的透明缓存，记录到的磁盘吞吐量非常少。这些数据应该是不言自明的。有趣的是，该脚本还按消费者组报告了细分信息。你在上面输出中看到的就是 I/O 资源管理器在起作用。作为示例目的，激活了一个数据库内计划，优先处理 `HIGHPRIO` 组的所有活动，而不是 `LOWPRIO` 组。

## 总结

Oracle 数据库和存储节点提供了成千上万的指标。我们在本章中只触及了其中一小部分。另外一些指标已在第 11 章中讨论过。这就引出了显而易见的问题：所有这些指标都重要吗？它们的“良好”值应该是多少？我们应该针对哪些指标采取行动？等等。对所有这些问题的总体回答是，不，你不需要学习、记忆和“调优”所有这些指标（这是不可能的）。你应该始终测量重要的东西——通常这是响应时间。理想情况下，你应该从测量最终用户响应时间开始，然后在需要时向下钻取，有时深入到数据库，有时深入到应用服务器或网络指标。正如你所知，问题并不总是由数据库引起。然而，由于现代多层应用的复杂性，这种端到端的诊断可能不可用，并且实施或改造也不可行。

在这种情况下，你会退一步，监控数据库查询和事务的响应时间。你只是从应用栈中稍低一点的层次开始采用自顶向下方法，同时记住问题可能实际上仍然发生在应用层或更高层。在监控用户报告和数据仓库查询的响应时间时，如果你拥有相应的许可，可能应该从 SQL Monitoring 页面（或 ASH 数据）开始，从中识别出有问题的用户查询，然后深入钻取它。使用自顶向下方法并跟随 SQL 计划中最大的时间消耗者，比采用相反的自底向上方法要容易得多——后者可能意味着查看一些数据库范围内的 Top SQL 报告，并希望弄清楚你的特定用户工作负载出了什么问题。

当然，在某些情况下，你会监控系统范围内的聚合匿名性能数据（不绑定到任何特定用户或会话），例如在容量规划和利用率监控中。然而，“我的数据库运行良好吗？”这类问题绝不应该通过查看系统利用率报告来回答。这些工具都无法告诉你你的数据库系统是否运行良好；只有你的用户能告诉你。如果用户不满意，你可以从一个不满意的用户开始，并从那里深入钻取响应时间。Oracle 提供了你所需的所有指标！



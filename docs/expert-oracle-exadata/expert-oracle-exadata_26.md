# Exadata 闪存缓存监控与统计

## 闪存缓存相关统计

以下是查询输出的性能统计示例：

```
`264 physical read bytes                                         3,050,496,000.00`
`264 physical read requests optimized                           23,364.00`
`264 physical read total bytes optimized                        3,050,487,808.00`
`264 physical write requests optimized                              .00`
`264 physical write total bytes optimized                           .00`
`12 rows selected.`
```

通常，高达 97% 的 I/O 请求应由闪存提供服务，但这里看到的读取命中率很低。其中一部分是由递归 SQL 引起的。另一方面，可以看到在 23,365 次 I/O 请求中，有 23,364 次得到了优化。显然，这些优化是由存储索引节省造成的。

到目前为止，在本节中你看到的都是单元闪存缓存读取命中率，没有与写入相关的统计。这些在前台进程中找不到，因为对数据文件的写入是由数据库写入器进程批量执行的。以下是同一实例上数据库写入器统计信息的示例：

```
SQL> select sid, name, value from v$sesstat natural join v$statname
  2    where sid in (select sid from v$session where program like ’%DBW%’)
  3    and name in (
  4      ’cell flash cache read hits’,
  5      ’cell overwrites in flash cache’,
  6      ’cell partial writes in flash cache’,
  7      ’cell physical IO bytes saved by storage index’,
  8      ’cell writes to flash cache’,
  9      ’physical read IO requests’,
 10      ’physical read requests optimized’,
 11      ’physical read total bytes optimized’,
 12      ’physical write requests optimized’,
 13      ’physical write total bytes optimized’)
 14  order by name;

SID NAME                                                      VALUE
---------- ------------------------------------------------ ----------
        65 cell flash cache read hits                                 0
         1 cell flash cache read hits                                 1
      1473 cell flash cache read hits                               556
         1 cell overwrites in flash cache                    344868411
        65 cell overwrites in flash cache                    344836755
      1473 cell overwrites in flash cache                    344936768
         1 cell partial writes in flash cache                       665
        65 cell partial writes in flash cache                       518
      1473 cell partial writes in flash cache                       643
        65 cell physical IO bytes saved by storage index              0
      1473 cell physical IO bytes saved by storage index              0
         1 cell physical IO bytes saved by storage index              0
        65 cell writes to flash cache                        353280230
         1 cell writes to flash cache                        353317405
      1473 cell writes to flash cache                        353382073
      1473 physical read IO requests                                 11
        65 physical read IO requests                                  0
         1 physical read IO requests                                  1
         1 physical read requests optimized                           1
        65 physical read requests optimized                           0
      1473 physical read requests optimized                         556
         1 physical read total bytes optimized                     8192
      1473 physical read total bytes optimized                  8994816
        65 physical read total bytes optimized                        0
         1 physical write requests optimized                176644066
        65 physical write requests optimized                176625494
      1473 physical write requests optimized                176676770
      1473 physical write total bytes optimized            1.5990E+12
        65 physical write total bytes optimized            1.5981E+12
         1 physical write total bytes optimized            1.5986E+12
30 rows selected.
```

在此处可以看到回写闪存缓存正在工作。

### V$CELL% 动态性能视图族

本节只能是对 `V$CELL%` 视图族的一个介绍。它们在第 11 章也有涉及，你可以了解到这些视图在 Oracle 11.1.0.2 和 `cellOS` 12.1.2.1.x 及更高版本中得到了极大增强。从分析闪存缓存的角度来看，最有趣的视图列在表 5-2 中。

**表 5-2. 用于监控闪存缓存的 V$CELL 视图列表**

| 视图名称 | 内容 |
| --- | --- |
| `V$CELL_DB` | 列出全局层面对所有单元的 I/O 请求。将查询限制到特定源数据库，可以了解其在所有 I/O 请求中的份额。 |
| `V$CELL_DISK` | 列出可以为单元磁盘测量的所有与 I/O 相关的统计信息。应将查询限制到单个单元，或者更好的是，限制到名称以 FD（闪存磁盘）开头的单个单元磁盘。如果您使用的是全闪存存储服务器，则不应依赖此视图中的信息来衡量闪存缓存性能。由于没有硬盘，所有网格磁盘都创建在闪存设备上，您无法轻易在视图中区分闪存缓存 I/O 和网格磁盘相关 I/O。对于所有其他 Exadata 用户，很可能所有闪存卡都用于闪存缓存（减去用于闪存日志的 512MB），在这种情况下，该视图可以显示一些关于 I/O 请求数量、大小和延迟的有趣数据。 |
| `V$CELL_GLOBAL` | 此表中最有趣的视图之一，因为它提供的输出与前面展示的 `cellcli` 命令 “list metriccurrent” 和 `cellsrvstat` 非常相似。最好通过一个示例来解释，紧随本表之后。 |

`V$CELL_DISK` 和 `V$CELL_GLOBAL` 的信息在 `V$CELL_DISK_HISTORY` 和 `V$CELL_GLOBAL_HISTORY` 中分别有一些历史化记录。以下是对 `V$CELL_GLOBAL` 的示例查询，用于显示与闪存缓存相关的指标。

```
SQL> select metric_name, metric_value, metric_type
  2  from V$CELL_GLOBAL
  3  where lower(metric_name) like ’%flash cache%’
  4  and cell_name = ’192.168.12.10’;

METRIC_NAME                                          METRIC_VALUE METRIC_TYPE
-------------------------------------------------- -------------- -----------------
Flash cache read bytes                              358836707328 bytes
Flash cache write bytes (first writes, overwrites,  500606863872 bytes
partial writes)
Flash cache bytes used                              360777785344 bytes
Flash cache bytes used (keep objects)                  4966318080 bytes
Flash cache keep victim bytes                                  0 bytes
Flash cache write bytes - first writes              107450368000 bytes
Flash cache write bytes - overwrites                392564908032 bytes
Flash cache write bytes - population writes due to   50094088192 bytes
read misses
[and many more]
```

你可以在 `V$CELL_METRIC_DESC` 中找到指标定义。在撰写本文时，并非所有通过 `cellcli` 在单元上暴露的指标都在数据库层被同等暴露，但与所有 Exadata 特性一样，这在未来可能会发生变化。



## AWR 报告

Oracle Exadata 文档在《数据库机器系统概述》文档中包含一个名为"Oracle Exadata 数据库一体机的新特性"的附录。你应该不时查看一下——其内容非常有价值，尤其是因为 Exadata 功能繁多，有时很难记住某个特定功能是在哪个版本引入的。从版本 `12.1.2.1.0` 开始，Oracle 在 AWR 报告中引入了与 Exadata 性能相关的信息。AWR 报告可能更适合展示和呈现性能相关数据。除了性能数据，报告还会显示更多关于系统配置的详细信息以及一份健康报告。该报告的一个优点是，大量信息适用于全局层面，因此你可以查看关于整个系统的信息，而不必局限于单个数据库。

## 总结

`Exadata 智能闪存缓存` 为降低与 Oracle 数据库相关的 `I/O` 成本提供了又一种方法。Exadata 平台提供的大多数优化都需要使用 `智能扫描`（全表扫描或快速全索引扫描）。`ESFC` 不依赖于 `智能扫描`，事实上，它对于加速单块随机读取最为有用。得益于 Exadata `11.2.3.3.0` 引入的全扫描数据透明缓存，多块读取现在也能受益于 `闪存缓存`。单块读取操作通常与 `OLTP` 工作负载相关，因此，`ESFC` 是 Exadata 用于 `OLTP` 或混合负载的关键组件。在本书的前一版中，我们认为 `ESFC` 不提供写入缓存这一事实，严重限制了它在写入瓶颈系统中的有效性。Exadata `11.2.3.3.1` 解决了这一问题，现在你可以在 `写回` 模式下操作 `闪存缓存`。即使是全部使用闪存的 `X5-2` 存储阵列也是如此，其部分容量被专门用作（`写回` 模式）`闪存缓存`。`写回闪存缓存` 对写入密集型工作负载提供了显著的性能提升。Exadata 系统中的闪存卡提供的写入 `IOPS` 数值优于所有磁盘组合在一起的性能。在一个根据数据表支持 6000 磁盘 `IOPS` 的 `X2-2` 四分之一机架上，我们毫不费力地将 `IOPS` 提升到了 22000。这个数字大大超过了系统纯基于磁盘的 `IOPS` 能力。

大容量缓存以及 Oracle 存储软件使用的智能缓存算法，使得 `ESFC` 能够提供与基于固态硬盘的磁盘系统相似的读取性能。将大量随机读取活动从硬盘上卸载，也间接有利于处理尚未缓存的 `智能扫描` 等操作。一般来说，`X4-2` 和 `X5-2` 硬件中可用的充裕缓存容量使得 Exadata 管理员的工作轻松了许多，而自动数据缓存（这是本书作者近期最喜爱的 Exadata 功能之一）也同样如此。

虽然 `ESFC` 最初被认为是纯粹针对减少小读取延迟的优化，但现在它对大型 `数据仓库型` 查询也相当有效。事实上，Oracle 引用的高吞吐量数据取决于在可能的情况下同时扫描磁盘和 `闪存缓存`。在这种情况下，`闪存缓存` 实际上承担了大部分负担。调整存储子句来修改缓存策略应该不再需要。由于 `闪存缓存` 已成为 Exadata 平台的关键组件，必须对其进行某种形式的控制。Oracle 已经认识到这一点，并允许使用 `I/O 资源管理器` 来控制 `闪存缓存`。


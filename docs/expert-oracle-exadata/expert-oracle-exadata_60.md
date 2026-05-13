# Exadata I/O 性能分析与 iostat 使用

## 服务时间正常但 I/O 性能仍差的情况

如果服务时间正常，但 Oracle (`cellsrv`) 仍然看到较差的 I/O 性能，这可能意味着在单元 Linux 服务器内部仍然存在排队现象。`iostat` 命令同样可以显示这些信息。

针对存储控制器 LUN 的“飞行中”未完成 I/O 数量是有限的。存储控制器需要跟踪每个未完成的 I/O（例如，它必须记住当数据块从磁盘读取并到达控制器时，应在主机 RAM 的哪个位置写入），而这些 I/O 插槽是有限的。Linux 内核不允许发送超出存储控制器处理能力的 I/O；否则，将发生 SCSI 重置。这些被限流的 I/O 将不得不在操作系统磁盘设备 I/O 队列中等待——并且它们是无用地等待在那里；它们甚至尚未被发送到存储控制器。只有当之前某个未完成的 I/O 操作完成后，队列中的第一个项目才会被发送到磁盘（假设某些请求的 I/O 延迟截止期限尚未达到；`cellsrv` 使用 Linux 的 “deadline” I/O 调度器）。

## Flash 设备的优势

好消息是，从 X5-2 开始，PCIe Flash 设备采用了 NVMe（非易失性内存主机控制器接口），这有望大幅增加 I/O 队列数量以及每个队列的队列深度。但这仅适用于 Flash 设备；硬盘仍然受限于上述限制。随着时间的推移，Flash 设备在承担 I/O 工作负载方面承担了更多责任。在 X5-2 高性能单元的情况下，这一点被推向极致。但即使在混合硬盘/Flash 内存配置中，您也可以启用 **写回闪存缓存** 来帮助解决硬盘成为写入瓶颈的罕见情况。**智能闪存日志** 可帮助您处理提交操作。而且从 Exadata 11.2.3.3 开始，无论对于小型还是大型 I/O 请求，读取操作都会自动缓存在 **智能闪存缓存** 中。总之，您的大部分 I/O 请求理想情况下将由 Flash 而非硬盘满足。小型随机 I/O 请求从 Flash 内存中获益良多。

## 硬盘 I/O 队列分析

回到对硬盘的讨论。如果您观察到 I/O 等待（排队时间），`iostat` 中的 `await`（平均 I/O 完成时间）列将显著高于 `svctm`（估算的 I/O 服务时间）列。请注意，“await”这个名称有些误导性，因为它不仅显示等待（排队）时间——它显示的是总等待时间加上服务时间。同样，`avgqu-sz` 列显示的并非仅仅是平均 I/O 等待队列长度，而是 I/O 请求队列中未完成 I/O 的平均总数，无论它们是否已经发送到存储硬件（正在被服务）或仍在 I/O 队列中等待（尚未被服务）。

Linux `iostat` 显示磁盘分区和单元软件 RAID 设备（用于单元操作系统分区镜像）的统计信息。我们可以过滤掉这些行，只列出物理磁盘统计信息；LVM 和分区信息被排除。下一个示例适用于除 X5-2 高性能单元外的所有型号；这些单元没有任何旋转硬盘。

```
$ iostat -xm 5 | egrep -v "sd.[0-9]|^md"
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
   56.38    0.03   18.65   15.33    0.00    9.61
Device:
rrqm/s  wrqm/s      r/s   w/s   rMB/s  wMB/s  avgrq-sz  avgqu-sz   await  svctm  %util
sdd    0.00    0.00   156.20  0.20   78.10   0.00   1022.69     20.78  116.79   5.77  90.20
sde    0.00    0.00   165.40  0.00   82.70   0.00   1024.00     27.96  163.97   5.69  94.08
sdh    0.00    0.00   163.20  3.20   81.34   0.01   1001.18     24.67  148.05   5.82  96.92
sdi    0.00    0.00   154.40  0.20   77.20   0.00   1022.68     12.81   82.68   5.75  88.82
sdj    0.00    0.00   163.20  0.00   81.31   0.00   1020.39     20.27  124.22   5.58  91.12
sdk    0.00    0.00   157.60  0.40   78.51   0.01   1017.76     30.54  168.64   5.91  93.42
sdc    0.00    0.00   166.80  3.60   80.96   0.03    973.48     30.23  169.08   5.70  97.20
sdf    0.00    0.00   161.80  0.00   80.51   0.00   1019.08     30.66  177.49   6.08  98.44
sdl    0.00    0.00   147.40  0.00   73.70   0.00   1024.00     17.52  119.09   6.01  88.64
sda    0.00   15.40   166.00  8.40   83.00   0.09    975.71     37.20  209.58   5.73 100.00
sdg    0.00    0.00   172.40  0.00   86.20   0.00   1024.00     32.10  171.85   5.75  99.18
sdn   83.00    0.00  2850.00  0.60  183.90   0.00    132.13     18.18    6.37   0.33  93.12
sdr   76.00    0.00  2788.20  0.40  179.60   0.01    131.91     16.43    5.89   0.33  92.12
sdv  108.60    0.00  2797.60  0.40  182.40   0.00    133.51     19.04    6.79   0.33  91.82
sdz  130.00    0.00  3119.80  0.80  203.63   0.00    133.64     35.88   11.47   0.30  95.12
sds   72.80    0.00  2969.60  0.60  190.57   0.00    131.40     21.37    7.19   0.32  94.46
sdo   88.80    0.00  2847.80  1.20  184.19   0.01    132.41     18.22    6.40   0.33  93.22
sdw   95.20    0.00  2711.40  0.40  176.20   0.00    133.07     16.79    6.18   0.33  90.14
sdaa  70.00    0.00  2556.40  0.60  164.43   0.00    131.71     14.96    5.85   0.34  87.42
sdq   56.00    0.00  2565.80  0.40  164.14   0.00    130.99     12.27    4.78   0.35  89.34
sdm   79.20    0.00  2863.00  1.20  184.33   0.01    131.81     18.99    6.62   0.33  93.64
sdp   91.20    0.00  2863.40  4.40  185.18   0.04    132.28     18.82    6.56   0.33  93.54
sdu  100.60    0.00  2774.20  0.40  180.51   0.00    133.24     18.84    6.78   0.33  91.88
sdy  105.60    0.00  2830.00  0.80  184.05   0.01    133.16     21.14    7.46   0.32  91.26
sdt  160.40    0.00  3054.40  0.80  201.74   0.01    135.24     35.70   11.64   0.32  96.48
sdx  111.60    0.00  2752.00  0.40  179.93   0.00    133.88     17.63    6.39   0.33  91.12
sdab  85.60    0.00  2828.80  0.40  182.62   0.00    132.20     20.28    7.15   0.32  90.88
sdb    0.00   15.40   162.80  7.80   81.40   0.09    978.27     23.15  132.24   5.63  96.04
sdac   0.00    0.00     0.00  0.00    0.00   0.00      0.00      0.00    0.00   0.00   0.00
```

## 解释输出结果

等一下！看到这么多磁盘，不应该感到奇怪吗？每个单元不是只有 12 块硬盘吗？之所以看到这么多磁盘，是因为除了 12 块硬盘外，单元还将其 Flash 卡呈现为独立的 SCSI “磁盘”。上面的输出来自一个 X3-2 单元，显示了除硬盘外，每张 Flash 卡对应 4 个 SCSI 设备。X5-2 的输出非常相似，不同之处在于设备通过 NVMe 访问，因此具有不同的名称。正如您在第 5 章中读到的那样，它们的 FMOD（闪存模块）数量也更少。

最后，还有一个用于系统恢复的 USB 磁盘，它不为单元软件所知。您可以在单元中使用 `lsscsi` 命令或 `cellcli` 来查看操作系统检测到的所有磁盘，但有一个例外：由于 X5-2 单元中的 Flash 设备不是 SCSI 设备，因此您无法在那里看到它们。在上面的例子中，为了演示目的，我们用 50 个用户发出的多个 **智能扫描** 对这台特定的 Exadata X3-2 单元施加了很大压力。这些数字仅用于说明；当前一代的 Exadata 硬件比这里使用的型号更强大。查看以下输出并与之前的 `iostats` 列表相关联，您会看到真正繁忙并提供最佳吞吐量的磁盘是 Flash 磁盘——返回给数据库会话的大部分数据都缓存在 **智能闪存缓存** 中。

```
CellCLI> list lun attributes deviceName,diskType,status
/dev/sda        HardDisk        normal
/dev/sdb        HardDisk        normal
/dev/sdc        HardDisk        normal
/dev/sdd        HardDisk        normal
/dev/sde        HardDisk        normal
/dev/sdf        HardDisk        normal
/dev/sdg        HardDisk        normal
/dev/sdh        HardDisk        normal
```



`/dev/sdi        硬盘            正常`
`/dev/sdj        硬盘            正常`
`/dev/sdk        硬盘            正常`
`/dev/sdl        硬盘            正常`
`/dev/sdq        闪存卡          正常`
`/dev/sdr        闪存卡          正常`
`/dev/sds        闪存卡          正常`
`/dev/sdt        闪存卡          正常`
`/dev/sdy        闪存卡          正常`
`/dev/sdz        闪存卡          正常`
`/dev/sdaa       闪存卡          正常`
`/dev/sdab       闪存卡          正常`
`/dev/sdm        闪存卡          正常`
`/dev/sdn        闪存卡          正常`
`/dev/sdo        闪存卡          正常`
`/dev/sdp        闪存卡          正常`
`/dev/sdu        闪存卡          正常`
`/dev/sdv        闪存卡          正常`
`/dev/sdw        闪存卡          正常`
`/dev/sdx        闪存卡          正常`

此处的 LSI 磁盘指的是 LSI SCSI RAID 控制器呈现给主机的硬盘（设备`sda`到`sdl`）；其他磁盘则是闪存卡。直到 Exadata X5-2，每块物理闪存卡都被细分为四个不同的“域”，每个域作为单独的设备报告给操作系统。此外，这种所谓的`FMOD`的寻址方式与其他 SCSI 设备一样，其设备名以`sd`开头。如果你比较`iostat`中的平均 I/O 服务时间（`svctm`），你会发现属于闪存卡的设备的服务时间比硬盘设备低一个数量级。因此，如果你只想监控硬盘设备，可以通过在`iostat`命令后附加要过滤的设备名来过滤其输出。Exadata 12.1.2.1 提供的`iostat`工具甚至能识别正则表达式。以下输出是使用`iostat –x –m 5 1 /dev/sd[a-l]`命令生成的：

```
Device:
rrqm/s  wrqm/s     r/s     w/s   rMB/s   wMB/s  avgrq-sz  avgqu-sz   await  svctm  %util
sdd   0.00    0.00    9.54    0.39    3.49    0.05    729.81      0.05    5.29   3.18   3.16
sde   0.00    0.00    9.56    0.44    3.49    0.05    725.00      0.06    5.51   3.29   3.29
sdh   0.00    0.00    9.54    0.42    3.49    0.05    727.57      0.05    5.43   3.17   3.16
sdi   0.00    0.00    9.56    0.41    3.49    0.05    727.11      0.05    5.37   3.25   3.24
sdj   0.00    0.00    9.62    0.43    3.52    0.05    729.15      0.05    5.22   3.13   3.15
sdk   0.00    0.00    9.56    0.92    3.49    0.06    693.37      0.05    5.10   3.07   3.22
sdc   0.00    0.00   12.83    3.21    3.49    0.07    454.96      0.06    4.05   2.52   4.04
sdf   0.00    0.00    6.97    0.38    3.35    0.05    948.27      0.05    7.11   4.17   3.06
sdl   0.00    0.00    9.63    0.37    3.49    0.05    725.13      0.05    5.41   3.24   3.24
sda   0.01   17.61   10.35    8.22    3.49    0.15    400.95      0.07    3.85   1.80   3.34
sdg   0.00    0.00    8.20    0.43    3.42    0.05    822.55      0.05    6.25   3.70   3.20
sdb   0.01   17.61   10.34    8.10    3.49    0.15    404.44      0.08    4.26   1.83   3.38
```

使用 Exadata 的一个优点是，尽管每一代新硬件都会增加新功能，但其核心概念保持不变。在上面的例子中，设备是作为块设备还是闪存卡来访问根本不重要——变化的只是设备名。你可以通过限制要报告的设备来调整刚才展示的`iostat`输出。

#### 使用 ExaWatcher 进行高级 Cell 监控

虽然数据库级别的等待概要文件和 SQL 监控应作为性能监控和故障排除的起点，但有时这些方法还不够，你需要更深入地探究。本节介绍一个额外的数据源。Exadata 计算节点和存储单元预装了一个名为`ExaWatcher`的工具，它从 Exadata 软件版本 11.2.3.3 开始取代了`OSWatcher`。它位于`/opt/oracle.ExaWatcher`目录下。这个工具只是一组 shell 脚本，它们定期运行标准的 OS 工具（如`vmstat`、`iostat`和`netstat`）来收集数据。`ExaWatcher`的优势在于它在操作系统级别运行，而不是在数据库内部，因此不受数据库挂起、性能问题或数据库的`V$`视图未显示真实情况或细节不足的影响。此外，高频的`ExaWatcher`收集器每隔几秒采样一次数据，使其能够检测到短暂的“故障”或活动突发。

`ExaWatcher`在机器启动时自动启动。你可以通过简单地搜索进程名中包含“`ExaWatcher`”的任何进程来检查`ExaWatcher`是否正在运行：

```
# pgrep -lf "ExaWatcher"
4372 /bin/bash ./ExaWatcher.sh --fromconf
4500 sh -c /usr/bin/mpstat -P ALL  5  720 2>/dev/null >> /opt/oracle.ExaWatcher/archive/Mpstat.ExaWatcher/2015_01_19_03_02_48_MpstatExaWatcher_enkx3cel01.enkitec.com.dat
4734 sh -c /usr/bin/iostat -t -x  5  720 2>/dev/null >> /opt/oracle.ExaWatcher/archive/Iostat.ExaWatcher/2015_01_19_03_03_13_IostatExaWatcher_enkx3cel01.enkitec.com.dat
5116 /usr/bin/perl /opt/oracle.ExaWatcher/ExecutorExaWatcher.pl /opt/oracle.ExaWatcher/ExaWatcher.execonf
5571 sh -c /opt/oracle.ExaWatcher/ExaWatcherCleanup.sh 1417968473 1733501253 3600 /opt/oracle.ExaWatcher/archive/ 614400 2>>/dev/null
5572 /bin/bash /opt/oracle.ExaWatcher/ExaWatcherCleanup.sh 1417968473 1733501253 3600 /opt/oracle.ExaWatcher/archive/ 614400
5578 /usr/bin/perl /opt/oracle.ExaWatcher/ExecutorExaWatcher.pl /opt/oracle.ExaWatcher/ExaWatcher.execonf
6171 sh -c /opt/oracle/cell/cellsrv/bin/cellsrvstat -interval=5 -count=720 2>/dev/null >> /opt/oracle.ExaWatcher/archive/CellSrvStat.ExaWatcher/2015_01_19_03_04_23_CellSrvStatExaWatcher_enkx3cel01.enkitec.com.dat
8202 sh -c /usr/bin/vmstat  5  2 >> /opt/oracle.ExaWatcher/archive/Vmstat.ExaWatcher/2015_01_19_03_04_58_VmstatExaWatcher_enkx3cel01.enkitec.com.dat
16614 sh -c /usr/bin/top -b -d 5 -n 720 2>/dev/null >> /opt/oracle.ExaWatcher/archive/Top.ExaWatcher/2015_01_19_02_09_22_TopExaWatcher_enkx3cel01.enkitec.com.dat
```

如果你想查看`ExaWatcher`进程守护程序的层次结构，在 Linux 上可以使用`ps -H`命令或`pstree`命令，如下所示：

```
[root@enkcel04 oracle.ExaWatcher]# pstree -Ahlup $(pgrep ExaWatcher.sh)
ExaWatcher.sh(19804)---perl(20519)-+-perl(20995)---sh(20996)---ExadataDiagColl(20999)
                                    |-sh(4960)---mpstat(4961)
                                    |-sh(4993)---iostat(4994)
                                    |-sh(5690)---cellsrvstat(5691)
                                    |-sh(9208)-+-grep(9211)
                                    |          |-sed(9210)
                                    |          `-top(9209)
                                    |-sh(9214)---FlexIntervalMod(9215)---sleep(20129)
                                    |-sh(11658)---FlexIntervalMod(11659)---sleep(20310)
                                    |-sh(14104)---FlexIntervalMod(14105)---sleep(20306)
                                    |-sh(20312)---vmstat(20313)
                                    |-sh(20987)---ExaWatcherClean(20988)---sleep(4294)
                                    |-{perl}(20581)
                                    |-{perl}(20584)
                                    |-{perl}(20587)
                                    |-{perl}(20604)
                                    |-{perl}(20630)
                                    |-{perl}(20647)
                                    |-{perl}(20651)
                                    |-{perl}(20660)
                                    |-{perl}(20672)
                                    |-{perl}(20700)
                                    |-{perl}(20723)
                                    |-{perl}(20743)
                                    |-{perl}(20749)
                                    |-{perl}(20755)
                                    |-{perl}(20758)
                                    |-{perl}(20789)
                                    |-{perl}(20986)
                                    `-{perl}(20994)
[root@enkcel04 oracle.ExaWatcher]#
```



# 单元磁盘故障

`ASM` 通过其冗余故障组技术处理单元磁盘的临时或永久丢失。因此，只要磁盘组定义为普通冗余，丢失一个单元磁盘就不会对数据库造成任何中断。如果使用高冗余，磁盘组可以承受同一故障组内两个单元磁盘同时丢失。回想一下，在 `Exadata` 上，每个存储单元构成一个独立的故障组。这意味着，使用普通冗余时，您可以丢失整个存储单元（12 个单元磁盘）而不会影响您的数据库。使用高冗余时，您可以同时丢失两个存储单元，而您的数据库将继续为客户端提供服务而不中断。这相当令人印象深刻。然而，冗余并不便宜。例如，考虑一个具有 30 `TB` 原始空间（配置为外部冗余）的磁盘组。使用普通冗余时，这 30 `TB` 变为 15 `TB` 的可用空间。使用高冗余时，它变为 10 `TB` 的可用存储空间。还要记住，除非不可用，否则数据库通常会读取数据的主副本。在 `Oracle 12c` 上，磁盘故障将启用均衡读取功能，该功能将读取负载最轻的磁盘，无论其包含的是主副本还是镜像副本。普通和高冗余不提供性能优势。它们严格用于容错。关键在于选择一个在弹性和预算之间取得平衡的冗余级别。

## 模拟磁盘故障

在本节中，我们将测试当一个单元磁盘发生故障时会发生什么。用于这些测试的系统是一个四分之一机架的 Exadata V2。我们创建了一个名为 `SCRATCH_DG` 的磁盘组，定义如下：

```
SYS:+ASM2> CREATE DISKGROUP SCRATCH_DG NORMAL REDUNDANCY
FAILGROUP CELL01 DISK ’o/192.168.12.3/SCRATCH_DG_CD_05_cell01’
FAILGROUP CELL02 DISK ’o/192.168.12.4/SCRATCH_DG_CD_05_cell02’
FAILGROUP CELL03 DISK ’o/192.168.12.5/SCRATCH_DG_CD_05_cell03’
attribute ’compatible.rdbms’=’12.1.0.2.0’,
’compatible.asm’  =’12.1.0.2.0’,
’au_size’=’4M’,
’cell.smart_scan_capable’=’true’;
```

请注意，该磁盘组是使用三个网格磁盘创建的。遵循 Exadata 最佳实践，我们从每个存储单元中使用了一个网格磁盘。有趣的是，即使我们没有在三个故障组中各指定一个磁盘，ASM 也会自动完成此操作。然后，我们使用这个磁盘组创建了一个名为 `SCRATCH` 的小型单实例数据库。该磁盘组配置为普通冗余（每个数据块有两个镜像副本），这意味着我们的数据库应该能够承受丢失一个网格磁盘而不丢失对数据的访问或导致崩溃。由于每个网格磁盘都位于单独的存储单元上，我们甚至可以承受整个存储单元丢失而不丢失数据。我们将在本章后面讨论存储单元故障时会发生什么。

稍后，我们将看看从存储单元中移除网格磁盘（模拟磁盘故障）时会发生什么。但在这样做之前，我们需要做几件事：

-   验证没有再平衡或其他卷管理操作正在运行
-   确保 `SCRATCH_DG` 磁盘组的所有网格磁盘都处于联机状态
-   验证使磁盘脱机不会影响数据库操作
-   检查磁盘修复计时器，以确保磁盘在我们能将其重新联机之前不会被自动丢弃

有几种方法可以验证卷管理活动没有进行。首先，让我们使用 `asmcmd` 检查磁盘组的当前状态。`ls –l` 命令显示磁盘组、冗余类型以及当前是否正在进行再平衡操作。顺便说一下，您也可以使用 `lsdg` 命令获取此信息，该命令还包括其他有用信息，如空间利用率、联机/脱机状态等。以下列表中的 `Rebal` 列表明当前没有正在执行的再平衡操作。

```
> asmcmd -p
ASMCMD [+] > ls -l
State    Type    Rebal  Name
MOUNTED  NORMAL  N      DATA_DG/
MOUNTED  NORMAL  N      RECO_DG/
MOUNTED  NORMAL  N      SCRATCH_DG/
MOUNTED  NORMAL  N      STAGE_DG/
MOUNTED  NORMAL  N      SYSTEM_DG/
```

请注意，并非所有卷管理操作都会在 `asmcmd` 命令中显示。如果某个网格磁盘已脱机一段时间，为了使其保持最新状态，可能有大量积压数据需要复制到其上。根据数据量大小，重新同步一个磁盘可能需要几分钟才能完成。尽管此操作与维护所有磁盘间的平衡直接相关，但从技术上讲，它并不是一个“再平衡”操作。因此，它不会出现在列表中。例如，即使前面列表中的 `ls –l` 命令显示再平衡操作的状态为 `N`，通过运行下一个查询，您可以清楚地看到有一个磁盘当前正在被联机：

```
SYS:+ASM2> select dg.name "Diskgroup", disk.name, disk.failgroup, disk.mode_status
from v$asm_disk disk,
v$asm_diskgroup dg
where dg.group_number = disk.group_number
and disk.mode_status <> ’ONLINE’;
Diskgroup         NAME                       FAILGROUP  MODE_ST
----------------- ------------------------------ -------  -------
SCRATCH_DG        SCRATCH_CD_05_CELL01        CELL01     SYNCING
```


检查磁盘的在线/离线状态，只需在 SQL*Plus 中运行以下查询即可轻松完成。在下面的列表中，您可以看到 `SCRATCH_CD_05_CELL01` 磁盘因其 `MOUNT_STATUS` 为 `MISSING` 且 `HEADER_STATUS` 为 `UNKNOWN` 而处于离线状态：

```
SYS:+ASM2> select d.name, d.MOUNT_STATUS, d.HEADER_STATUS, d.STATE
from v$asm_disk d
where d.name like 'SCRATCH%'
order by 1;
NAME                                         MOUNT_S HEADER_STATU STATE
-------------------------------------------------- ------- ------------ ----------
SCRATCH_CD_05_CELL01                         MISSING UNKNOWN      NORMAL
SCRATCH_CD_05_CELL02                         CACHED  MEMBER       NORMAL
SCRATCH_CD_05_CELL03                         CACHED  MEMBER       NORMAL
```

不过，检查 `SCRATCH_DG` 磁盘组中所有磁盘状态的一个更好方法可能是查看 `V$ASM_DISK_STAT` 中的 `mode_status`。下面的列表显示 `SCRATCH_DG` 磁盘组中的所有网格磁盘都处于在线状态：

```
SYS:+ASM2> select name, mode_status from v$asm_disk_stat where name like 'SCRATCH%';
NAME                                         MODE_ST
-------------------------------------------------- -------
SCRATCH_CD_05_CELL03                         ONLINE
SCRATCH_CD_05_CELL01                         ONLINE
SCRATCH_CD_05_CELL02                         ONLINE
```

接下来我们要看的是磁盘修复计时器。回想一下，磁盘组属性 `disk_repair_time` 决定了当发生读/写错误时，ASM 在将磁盘从磁盘组中永久移除并将数据重新平衡到幸存的网格磁盘之前会等待多长时间。在将磁盘置于离线状态之前，我们应该检查这个计时器，以确保在 ASM 自动删除磁盘之前，我们有足够的时间将磁盘重新恢复在线。可以使用 SQL*Plus 运行以下查询来显示此属性。（顺便说一下，无论您连接到 ASM 实例还是数据库实例，`V$ASM` 视图都是可见的。）

```
SYS:+ASM2> select dg.name "DiskGroup",
attr.name,
attr.value
from v$asm_diskgroup dg,
v$asm_attribute attr
where dg.group_number = attr.group_number
and attr.name like '%repair_time';
DiskGroup         NAME                      VALUE
----------------- ------------------------- ----------
DATA_DG           disk_repair_time          3.6h
DATA_DG           failgroup_repair_time     24.0h
DBFS_DG           disk_repair_time          3.6h
DBFS_DG           failgroup_repair_time     24.0h
RECO_DG           disk_repair_time          3.6h
RECO_DG           failgroup_repair_time     24.0h
SCRATCH_DG        disk_repair_time          8.5h
SCRATCH_DG        failgroup_repair_time     24h
STAGE_DG          disk_repair_time          72h
STAGE_DG          failgroup_repair_time     24h
```

磁盘修复计时器的默认值是 3.6 小时。由于此查询是在运行 Oracle 12c 的集群上执行的，因此还有一个 `failgroup_repair_time` 属性。这是指在发生整个故障组丢失的情况下，在删除磁盘之前将花费的时间。这在硬件故障波及整个存储单元时非常有用。这些属性在存储单元重启或磁盘暂时离线时生效，但在极少数情况下，当发生实际的硬件故障时，它们也可能自发触发。有时，只需将磁盘从机箱中拔出再重新插入，就能清除意外的瞬时错误。任何通常应写入故障磁盘的数据都会排队等待，直到磁盘恢复在线或磁盘修复时间过期。如果 ASM 删除了磁盘，可以手动将其添加回磁盘组，但这需要进行一次完整的重新平衡操作，这可能是一个漫长的过程。以下命令用于将 `SCRATCH_DG` 磁盘组的磁盘修复计时器设置为 8.5 小时：



`SYS:+ASM2> alter diskgroup SCRATCH_DG set attribute 'disk_repair_time'='8.5h';`

现在，我们来验证将一个单元磁盘离线是否会影响磁盘组的可用性。我们可以通过检查网格磁盘的 `asmdeactivationoutcome` 和 `asmmodestatus` 属性来做到这一点。例如，下面的列表显示了当正常冗余磁盘组中的一个网格磁盘被离线时，`LIST GRIDDISK` 命令的输出。在此示例中，我们有一个 `SCRATCH_DG` 磁盘组，它由来自三个故障组（`enkcel01`、`enkcel02` 和 `enkcel03`）的各一个网格磁盘组成。首先，我们检查所有磁盘都处于活动状态时网格磁盘的状态：

```
[enkdb02:root] /root
> dcli -g cell_group -l root " cellcli -e list griddisk \
attributes name, asmdeactivationoutcome, asmmodestatus " | grep SCRATCH
enkcel01: SCRATCH_DG_CD_05_cell01   Yes     ONLINE
enkcel02: SCRATCH_DG_CD_05_cell02   Yes     ONLINE
enkcel03: SCRATCH_DG_CD_05_cell03   Yes     ONLINE
```

现在，我们将在存储单元上停用其中一个网格磁盘，并再次运行该命令：

```
CellCLI> alter griddisk SCRATCH_DG_CD_05_cell01 inactive
GridDisk SCRATCH_DG_CD_05_cell01 successfully altered

[enkdb02:root] /root
> dcli -g cell_group -l root " cellcli -e list griddisk \
attributes name, asmdeactivationoutcome, asmmodestatus " | grep SCRATCH
enkcel01: SCRATCH_DG_CD_05_cell01   Yes     OFFLINE
enkcel02: SCRATCH_DG_CD_05_cell02   "Cannot de-activate due to other offline disks in the diskgroup"        ONLINE
enkcel03: SCRATCH_DG_CD_05_cell03   "Cannot de-activate due to other offline disks in the diskgroup"        ONLINE
```

如你所见，已离线的网格磁盘的 `asmmodestatus` 属性现在被设为 `OFFLINE`，而磁盘组中另外两个磁盘的 `asmdeactivationoutcome` 属性则警告我们，这些网格磁盘无法被离线。这样做会导致 ASM 卸载 `SCRATCH_DG` 磁盘组。

注意：我们使用 `dcli` 命令在存储网格的每个单元上运行 `CellCLI` 命令 `LIST GRIDDISK ATTRIBUTES`。基本上，`dcli` 允许我们在多个节点上并发地运行一个命令。`cell_group` 参数是一个文件，其中包含所有存储单元的列表。

如果 `LIST GRIDDISK` 命令的输出表明这样做是安全的，我们可以测试将 `SCRATCH_DG` 磁盘组的一个网格磁盘离线时会发生什么。对于此测试，我们将从存储单元机箱中物理移除磁盘驱动器。测试配置将如下所示：

*   为此测试，我们将创建一个包含一个数据文件的新表空间。该数据文件设置为 `autoextend`，以便随着数据加载而增长到磁盘组中。
*   接下来，我们将通过创建一个大表在该表空间中生成大量数据；从 `DBA_SEGMENTS` 中生成几十亿行数据应该就够了。
*   在将数据加载到大表的过程中，我们将从单元机箱中物理移除该磁盘。
*   数据加载完成后，我们将重新安装磁盘，并观察 Exadata 自动磁盘恢复的过程。

首要任务是确定磁盘驱动器在存储单元中的位置。为此，我们将使用网格磁盘名称来查找其所在的单元磁盘。然后，我们将使用单元磁盘名称来查找该磁盘驱动器在存储单元内的插槽地址。一旦获得插槽地址，我们将在前面板上点亮服务 LED，以便知道要移除哪块磁盘。

在存储单元 3 上，我们可以使用 `LIST GRIDDISK` 命令来查找我们寻找的单元磁盘名称：

```
CellCLI> list griddisk attributes name, celldisk where name like 'SCRATCH.*' detail
name:                   SCRATCH_DG_CD_05_cell03
cellDisk:               CD_05_cell03
```

现在我们有了单元磁盘名称，可以使用 `LIST LUN` 命令来查找我们要移除的物理磁盘的插槽地址。在下面的列表中，我们看到了寻找的插槽地址：16:5。




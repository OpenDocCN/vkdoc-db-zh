# 启用回写闪存缓存

回写闪存缓存通常在典型代表性工作负载中，读写比例倾向于写入时启用。传统上，人们可能假设工作负载中读取数量多于写入数量。启用 WBFC 的另一个原因是当您发现“`空闲缓冲区等待`”占您等待事件的很大比例时。在您的 AWR 报告或跟踪中发现此事件，表明（一个或多个）数据库写入进程无法及时将脏块写入磁盘，以释放新读入缓冲区缓存的缓冲区。

或许可以安全地说，大多数用户以默认的直写模式操作智能闪存缓存。从直写模式切换到回写模式涉及本节说明的步骤（除非您使用的是全闪存单元——即 X5-2 高性能存储服务器——其中回写模式默认已启用）。您可以以滚动或非滚动方式执行切换。

## 先决条件

作为先决条件，您必须确保集群中的所有网格磁盘都处于联机并可用状态：

```
[root@enkx3db01 ∼]# dcli -g cell_group -l root cellcli -e list griddisk attributes \
> asmdeactivationoutcome, asmmodestatus, name, status
```

## 验证当前状态

以下列表显示了以滚动方式启用 WBFC 所执行的步骤。在验证集群磁盘的第一个命令返回无问题后，您应继续检查现有闪存缓存和闪存日志的状态，如下所示。状态不应显示任何问题：

```
[root@enkx3db01 ∼]# dcli -g ./cell_group -l root \
> cellcli -e list flashcache attributes name,size,status
enkx3cel01: enkx3cel01_FLASHCACHE      1488.75G        normal
enkx3cel02: enkx3cel02_FLASHCACHE      1488.75G        normal
enkx3cel03: enkx3cel03_FLASHCACHE      1488.75G        normal

[root@enkx3db01 ∼]# dcli -g ./cell_group -l root \
> cellcli -e list flashlog attributes name,size,status
enkx3cel01: enkx3cel01_FLASHLOG        512M    normal
enkx3cel02: enkx3cel02_FLASHLOG        512M    normal
enkx3cel03: enkx3cel03_FLASHLOG        512M    normal
```

在以下示例中，智能闪存缓存正从直写模式更改为回写模式。以下是闪存缓存处于直写模式的证明：

```
[root@enkx3db01 ∼]# dcli -g cell_group -l root \
> "cellcli -e list cell detail" | grep "flashCacheMode"
enkx3cel01: flashCacheMode:            WriteThrough
enkx3cel02: flashCacheMode:            WriteThrough
enkx3cel03: flashCacheMode:            WriteThrough
```

## 删除闪存缓存

接下来，您需要以 `root` 身份连接到第一个单元并删除闪存缓存：

```
[root@enkx3cel01 ∼]# cellcli -e drop flashcache
Flash cache enkx3cel01_FLASHCACHE successfully dropped
```

作为此操作的一部分，由于 ASM 正常或高冗余配置，ASM 应保持运行，能够容忍单元的丢失。下一个列表的输出应显示所有磁盘均为 `YES`。为使本章篇幅合理，输出被截断：

```
[root@enkx3cel01 ∼]# cellcli -e list griddisk attributes name,asmmodestatus,asmdeactivationoutcome
DATA_CD_00_enkx3cel01           ONLINE  Yes
...
DATA_CD_11_enkx3cel01           ONLINE  Yes
DBFS_DG_CD_02_enkx3cel01        ONLINE  Yes
...
DBFS_DG_CD_11_enkx3cel01        ONLINE  Yes
RECO_CD_00_enkx3cel01           ONLINE  Yes
...
RECO_CD_11_enkx3cel01           ONLINE  Yes
```

## 停用网格磁盘

验证后，您将单元上的网格磁盘更改为非活动状态。同样，输出经过删节以增强可读性，每个网格磁盘将显示一行：

```
[root@enkx3cel01 ∼]# cellcli -e alter griddisk all inactive
GridDisk DATA_CD_00_enkx3cel01 successfully altered
...
GridDisk DATA_CD_11_enkx3cel01 successfully altered
GridDisk DBFS_DG_CD_02_enkx3cel01 successfully altered
...
GridDisk DBFS_DG_CD_11_enkx3cel01 successfully altered
GridDisk RECO_CD_00_enkx3cel01 successfully altered
...
GridDisk RECO_CD_11_enkx3cel01 successfully altered
```

## 重新配置缓存模式

现在单元上剩余的步骤不多了。总之，您需要关闭 `cellsrv` 进程，删除闪存缓存，将 `flashCacheMode` 属性设置为“`WriteBack`”，创建闪存缓存，并使所有服务重新联机。但是，首先让我们使用另一个会话，以 `root` 身份连接到 RDBMS 集群中的第一个节点，验证闪存缓存是否确实已被删除：

```
[root@enkx3db01 ∼]# dcli -g ./cell_group -l root \
> cellcli -e list flashcache attributes name,size,status
enkx3cel02: enkx3cel02_FLASHCACHE      1488.75G        normal
enkx3cel02: enkx3cel03_FLASHCACHE      1488.75G        normal
```

确实，第一个单元没有报告智能闪存缓存的存在。如您所见，闪存日志不受影响：

```
[root@enkx3db01 ∼]# dcli -g ./cell_group -l root \
> cellcli -e list flashlog attributes name,size,status
enkx3cel01: enkx3cel01_FLASHLOG        512M    normal
enkx3cel02: enkx3cel02_FLASHLOG        512M    normal
enkx3cel03: enkx3cel03_FLASHLOG        512M    normal
```

在此阶段，您关闭单元软件，更改属性以启用回写缓存，然后再次启动服务：

```
[root@enkx3cel01 ∼]# cellcli -e alter cell shutdown services cellsrv
Stopping CELLSRV services...
The SHUTDOWN of CELLSRV services was successful.

[root@enkx3cel01 ∼]# cellcli -e alter cell flashCacheMode=WriteBack
Cell enkx3cel01 successfully altered

[root@enkx3cel01 ∼]# cellcli -e alter cell startup services cellsrv
Starting CELLSRV services...
The STARTUP of CELLSRV services was successful.
```

## 重新激活网格磁盘

在成功启动 `cellsrv` 守护进程后，您将网格磁盘重新带入 ASM：

```
[root@enkx3cel01 ∼]# cellcli -e alter griddisk all active
GridDisk DATA_CD_00_enkx3cel01 successfully altered
...
GridDisk DATA_CD_11_enkx3cel01 successfully altered
GridDisk DBFS_DG_CD_02_enkx3cel01 successfully altered
...
GridDisk DBFS_DG_CD_11_enkx3cel01 successfully altered
GridDisk RECO_CD_00_enkx3cel01 successfully altered
...
GridDisk RECO_CD_11_enkx3cel01 successfully altered
```

这不是一个瞬时操作——您必须耐心等待一小会儿，直到每个网格磁盘的状态都变为 `ONLINE`。在每个磁盘都联机之前，请不要继续。以下是一些仍在恢复为联机状态的网格磁盘的示例输出：

```
[root@enkx3cel01 ∼]# cellcli -e list griddisk attributes name, asmmodestatus
DATA_CD_00_enkx3cel01           SYNCING
DATA_CD_01_enkx3cel01           SYNCING
DATA_CD_02_enkx3cel01           SYNCING
DATA_CD_03_enkx3cel01           SYNCING
DATA_CD_04_enkx3cel01           SYNCING
DATA_CD_05_enkx3cel01           SYNCING
DATA_CD_06_enkx3cel01           SYNCING
DATA_CD_07_enkx3cel01           SYNCING
DATA_CD_08_enkx3cel01           SYNCING
DATA_CD_09_enkx3cel01           SYNCING
DATA_CD_10_enkx3cel01           SYNCING
DATA_CD_11_enkx3cel01           SYNCING
DBFS_DG_CD_02_enkx3cel01        ONLINE
...
RECO_CD_11_enkx3cel01           ONLINE
```

## 重建闪存缓存

当所有磁盘都恢复为 `ONLINE` 后，您可以在单元上重新创建闪存缓存：

```
[root@enkx3cel01 ∼]# cellcli -e create flashcache all
Flash cache enkx3cel01_FLASHCACHE successfully created
```

## 最终验证

辛勤工作的成果是单元使用了回写闪存缓存：

```
[root@enkx3db01 ∼]# dcli -g cell_group -l root \
> "cellcli -e list cell detail" | grep "flashCacheMode"
enkx3cel01: flashCacheMode:            WriteBack
enkx3cel02: flashCacheMode:            WriteThrough
enkx3cel03: flashCacheMode:            WriteThrough
```


很遗憾，工作尚未完成——您需要在下一个 Exadata 存储单元节点上重复上述步骤。然而，在关闭其他存储单元之前，您必须确保这不会影响数据库的可用性。使用本节引言中见过的命令列出网格磁盘，并确保属性 `asmmodestatus` 和 `asmdeactivationoutcome` 允许您将网格磁盘状态更改为“非活动”，以便为删除该单元的闪存缓存做准备。此过程在未来的版本中可能会有变化，因此请务必查阅 `My Oracle Support` 获取最新文档。

注意

当然，也可以从 `write-back` 模式切换回 `write-through` 模式。`My Oracle Support` 的说明文档 `1500257.1` 解释了如何执行这些步骤。

### 闪存缓存压缩

Oracle Exadata 软件 `11.2.3.3.0` 是首个引入闪存缓存压缩功能的版本。在 Exadata `X3` 和 `X4` 存储单元中发现的 `F40` 和 `F80` 卡内置了一个压缩引擎，允许在用户数据写入闪存缓存时进行压缩。由于压缩技术内置于卡的硬件中，与软件解决方案相比，其相关开销应该更小。

与任何压缩技术一样，节省的空间取决于您要压缩的数据。最差的压缩比很可能出现在 `HCC 压缩单元 (CUs)` 上。由于这些单元已包含压缩信息，几乎没有什么可再压缩的了。同样，如果 `OLTP`（现称为“高级”）压缩块已处于大部分块已去重的状态，它们也不是压缩的最佳候选者。另一方面，未压缩的块是良好的压缩候选对象。

在撰写本文时，`My Oracle Support` 说明文档 `1664257.1` “EXADATA Flash Cache Compression - FAQ” 指出，只有 `F40` 和 `F80` 卡可以使用压缩功能，并且作为先决条件，必须授权许可 `Advanced Compression Option`。`X5-2` 中的 `F160` 卡不支持本节所述的闪存压缩功能。您应查阅此说明文档以确保您的补丁级别符合最低要求。启用闪存缓存压缩的过程与刚才描述的启用 `Write-back` 闪存缓存类似。为了避免重复，我们将指引您查阅 `My Oracle Support` 的说明文档以获取具体步骤。

当您启用闪存缓存压缩后，您的闪存缓存报告大小将远大于物理设备大小。考虑一下在 `X4-2` 存储服务器上启用压缩后的闪存缓存：

```
[root@enkx4cel01 ∼]# cellcli -e list flashcache detail
name:                       enkx4cel01_FLASHCACHE
cellDisk:                   FD_04_enkx4cel01,FD_06_enkx4cel01,FD_11_enkx4cel01,
                            FD_02_enkx4cel01,FD_13_enkx4cel01,FD_12_enkx4cel01,
                            FD_00_enkx4cel01,FD_14_enkx4cel01,FD_03_enkx4cel01,
                            FD_09_enkx4cel01,FD_10_enkx4cel01,FD_15_enkx4cel01,
                            FD_08_enkx4cel01,FD_07_enkx4cel01,FD_01_enkx4cel01,
                            FD_05_enkx4cel01
creationTime:               2015-01-19T21:33:37-06:00
degradedCelldisks:
effectiveCacheSize:         5.8193359375T
id:                         3d415a32-f404-4a27-b9f2-f6a0ace2cee2
size:                       5.8193359375T
status:                     normal
```

如果您观察非常仔细，会发现闪存缓存的大小是 `5.8TB`。这是逻辑缓存大小，因为 `X4-2` 存储服务器有四张 `800 GB` 大小的闪存卡，总共 `3.2 TB`。要获得这些数字，您必须已启用闪存缓存压缩：

```
CellCLI> list cell attributes FlashCacheCompress
TRUE
```

正常情况下，闪存上的每个单元磁盘是 `186GB`，但在启用压缩后会报告更大的空间。这个空间是虚拟的，Oracle 在内部管理闪存缓存中的空间：

```
CellCLI> list celldisk attributes name,diskType,freeSpace,size,status where name like 'FD.*'
FD_00_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_01_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_02_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_03_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_04_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_05_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_06_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_07_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_08_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_09_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_10_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_11_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_12_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_13_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_14_enkx4cel01         FlashDisk         0          372.515625G          normal
FD_15_enkx4cel01         FlashDisk         0          372.515625G          normal
```

闪存缓存压缩是一个很好的功能，可以在 Exadata 系列的特定型号上逻辑地扩展闪存缓存。

### 控制 ESFC 使用

一般而言，对象会根据存储软件的自动缓存策略被缓存在 ESFC 中。不过，你可以通过使用 `CELL_FLASH_CACHE` 存储子句属性来覆盖单个数据库对象的自动策略，尽管你可能应该避免这样做。正如你目前已经读过几遍的那样，自 Exadata 11.2.3.3.1 及更高版本起，自动缓存效果非常好。如果你坚持在少数合理的情况下将对象固定在闪存缓存中，你可以将该属性设置为以下三个有效值：

*   `NONE`: 从不缓存此对象。
*   `DEFAULT`: 自动缓存机制生效。这是默认值。
*   `KEEP`: 此对象应被赋予优先状态。请注意，此指定也会更改智能扫描的默认行为，允许它们从缓存和磁盘同时读取。

你可以在创建对象时指定存储子句。存储子句的某些选项也可以使用 `ALTER` 命令进行修改。以下是使用 `ALTER` 命令更改 `CELL_FLASH_CACHE` 存储子句的示例：

```sql
SQL> alter table martin.bigt storage (cell_flash_cache keep);
```

你还可以通过查看 `dba_tables` 或 `dba_indexes` 及其分区相关视图的 `cell_flash_cache` 列，来查看哪些对象被指定为更积极的缓存：

```sql
SQL> @esfc_keep_tables

SQL> select owner, table_name, status, last_analyzed,
  2  num_rows, blocks, degree, cell_flash_cache
  3  from dba_tables
  4  where cell_flash_cache like nvl('&cell_flash_cache','KEEP');
Enter value for cell_flash_cache:
old   4: where cell_flash_cache like nvl('&cell_flash_cache','KEEP')
new   4: where cell_flash_cache like nvl('','KEEP')

OWNER                TABLE_NAME      STATUS   LAST_ANAL   NUM_ROWS     BLOCKS DEGREE   CELL_FL
-------------------- --------------- -------- --------- ---------- ---------- -------- -------
MARTIN               BIGTAB_QL       VALID    28-JAN-15  256000000     890768 1        KEEP
```

你之前已经读到，自 `cellsrv` 11.2.3.3.1 及更高版本起，将对象固定在闪存缓存中已不再是真正必需。升级到更新版本的存储软件是一个好机会，可以测试是否可能允许 Oracle 根据其算法自主缓存对象。固定对象也可能适得其反，尤其是在早期的 Exadata 硬件代（如 V2 和 X2）中，因为可用的闪存容量有限。在我们访问的现场，仍然能看到 X2 和 V2 系统。

在第 7 章中，你可以阅读关于资源管理的内容。I/O 资源管理器允许 Exadata 管理员限制甚至禁止使用智能闪存缓存。

### 监控

你可以在多个地方监控 Exadata 智能闪存缓存的使用情况。粗略地说，你可以选择查询某些自动工作负载存储库视图或其他最近引入的动态性能视图。另一种选择是向单元软件查询更多信息。本节只能作为数据库可访问信息的一个引子，更多内容你可以在第 11 章（关于会话统计）和第 12 章（基于 Enterprise Manager 12c 的图形监控解决方案）中阅读。

自本书第一版印刷以来，存储单元和数据库级别可用的度量指标已大大增强。归根结底，图形工具——AWR 报告、Enterprise Manager 12c 和 Enterprise Manager 12c Express——不能凭空捏造数字——它们拥有智能的接口来显示系统提供的度量指标。为了让读者更容易理解，将首先讨论通过 `cellcli` 可用的存储服务器相关度量指标，然后再将重点转向数据库层。

#### 在存储层

每个 Exadata 存储服务器都会记录自己的度量指标，这些指标最终可以传递到 RDBMS 层。如果你希望基于命令行深入进行诊断，可以通过连接到每个单元来实现，或者使用计算节点上的 `dcli` 工具从每个存储服务器收集信息。性能工程师可用的第一个选项是使用 `cellcli` 实用程序中的 `metriccurrent` 类别。关于闪存相关的性能指标，可以查询多种不同的对象类型。连接到 Exadata 12.1.2.1 存储单元后，可以识别出以下度量类别：

```bash
[root@enkx4cel01 ~]# cellcli -e list metriccurrent attributes objecttype | sort | uniq | nl
1                  CELL
2                  CELLDISK
3                  CELL_FILESYSTEM
4                  FLASHCACHE
5                  FLASHLOG
6                  GRIDDISK
7                  HOST_INTERCONNECT
8                  IBPORT
9                  IORM_CATEGORY
10                 IORM_CONSUMER_GROUP
11                 IORM_DATABASE
12                 IORM_PLUGGABLE_DATABASE
13                 SMARTIO
```

仅出于本次讨论的目的，粗体标出的 `objectType` 值值得关注。由于这些度量指标随着每个版本不断变化，你应该查看 `list metricdefinition` 命令的输出，看看是否有任何新的值得关注的指标。仍然连接到 12.1.2.1 单元服务器，你可以找到以下闪存缓存相关的度量指标：

```sql
CellCLI> LIST METRICDEFINITION attributes name, description WHERE objectType = 'FLASHCACHE'
```

表 5-1 显示了该命令的输出，仅限于那些具有实际关联值的统计信息。在 `cellsrv` 12.1.2.1 中有 111 个与闪存缓存相关的度量指标，其中约 54 个在此处显示，在我们的环境中这些指标的值 > 0。表 5-1 提供了你在存储层可以报告的每个统计信息的简要描述。

表 5-1.

闪存缓存度量定义选摘

| 指标 | 描述 |
| --- | --- |
| `FC_BYKEEP_USED` | 闪存缓存上用于保留对象的兆字节数 |
| `FC_BY_ALLOCATED` | 闪存缓存中已分配的兆字节数 |
| `FC_BY_DIRTY` | 闪存缓存中未刷新的兆字节数 |
| `FC_BY_STALE_DIRTY` | 闪存缓存中因缓存磁盘不可访问而无法刷新的未刷新兆字节数 |
| `FC_BY_USED` | 闪存缓存上已使用的兆字节数 |
| `FC_IO_BYKEEP_R` | 从闪存缓存为保留对象读取的兆字节数 |
| `FC_IO_BY_ALLOCATED_OLTP` | 闪存缓存中为 OLTP 数据分配的兆字节数 |
| `FC_IO_BY_DISK_WRITE` | 从闪存缓存写入硬盘的兆字节数 |
| `FC_IO_BY_R` | 从闪存缓存读取的兆字节数 |
| `FC_IO_BY_R_ACTIVE_SECONDARY` | 从闪存缓存满足的活动辅助读取的兆字节数 |
| `FC_IO_BY_R_ACTIVE_SECONDARY_MISS` | 未从闪存缓存满足的活动辅助读取的兆字节数 |
| `FC_IO_BY_R_DW` | 从闪存缓存读取的 DW 数据兆字节数 |
| `FC_IO_BY_R_MISS` | 因请求数据不全在闪存缓存中而从磁盘读取的兆字节数 |
| `FC_IO_BY_R_MISS_DW` | 从磁盘读取的 DW 数据兆字节数 |
| `FC_IO_BY_R_SEC` | 每秒从闪存缓存读取的兆字节数 |
| `FC_IO_BY_R_SKIP` | 绕过闪存缓存的 IO 请求从磁盘读取的兆字节数 |
| `FC_IO_BY_R_SKIP_NCMIRROR` | 因 IO 位于非主、非活动辅助镜像而绕过闪存缓存的 IO 请求从磁盘读取的兆字节数 |
| `FC_IO_BY_R_SKIP_SEC` | 每秒绕过闪存缓存的 IO 请求从磁盘读取的兆字节数 |
| `FC_IO_BY_W` | 写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_FIRST` | 首次写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_FIRST_SEC` | 每秒首次写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_OVERWRITE` | 覆盖写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_OVERWRITE_SEC` | 每秒覆盖写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_POPULATE` | 因读取未命中而填充写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_SEC` | 每秒写入闪存缓存的兆字节数 |
| `FC_IO_BY_W_SKIP` | 绕过闪存缓存的 IO 请求写入磁盘的兆字节数 |
| `FC_IO_BY_W_SKIP_LG` | 因 IO 大小较大而绕过闪存缓存的 IO 请求写入磁盘的兆字节数 |
| `FC_IO_BY_W_SKIP_LG_SEC` | 每秒因 IO 大小较大而绕过闪存缓存的 IO 请求写入磁盘的兆字节数 |
| `FC_IO_BY_W_SKIP_SEC` | 每秒绕过闪存缓存的 IO 请求写入磁盘的兆字节数 |
| `FC_IO_RQKEEP_R` | 从闪存缓存为保留对象读取的请求数 |
| `FC_IO_RQ_DISK_WRITE` | 从闪存缓存写入硬盘的请求数 |
| `FC_IO_RQ_R` | 从闪存缓存读取的请求数 |
| `FC_IO_RQ_REPLACEMENT_ATTEMPTED` | 尝试在闪存缓存中寻找空间的请求数 |
| `FC_IO_RQ_REPLACEMENT_FAILED` | 在闪存缓存中寻找空间失败的请求数 |
| `FC_IO_RQ_R_ACTIVE_SECONDARY` | 从闪存缓存满足的活动辅助读取请求数 |
| `FC_IO_RQ_R_ACTIVE_SECONDARY_MISS` | 未从闪存缓存满足的活动辅助读取请求数 |
| `FC_IO_RQ_R_DW` | 从闪存缓存读取数据的 DW IO 数 |
| `FC_IO_RQ_R_MISS` | 未在闪存缓存中找到所有数据的读取请求数 |
| `FC_IO_RQ_R_MISS_DW` | 从磁盘读取数据的 DW IO 数 |
| `FC_IO_RQ_R_SEC` | 每秒从闪存缓存读取的请求数 |
| `FC_IO_RQ_R_SKIP` | 绕过闪存缓存从磁盘读取的请求数 |
| `FC_IO_RQ_R_SKIP_NCMIRROR` | 因 IO 位于非主、非活动辅助镜像而绕过闪存缓存从磁盘读取的请求数 |
| `FC_IO_RQ_R_SKIP_SEC` | 每秒绕过闪存缓存从磁盘读取的请求数 |
| `FC_IO_RQ_W` | 导致数据填充到闪存缓存的请求数 |
| `FC_IO_RQ_W_FIRST` | 首次写入闪存缓存的请求数 |
| `FC_IO_RQ_W_FIRST_SEC` | 每秒首次写入闪存缓存的请求数 |
| `FC_IO_RQ_W_OVERWRITE` | 覆盖写入闪存缓存的请求数 |
| `FC_IO_RQ_W_OVERWRITE_SEC` | 每秒覆盖写入闪存缓存的请求数 |
| `FC_IO_RQ_W_POPULATE` | 因读取未命中而填充写入闪存缓存的请求数 |
| `FC_IO_RQ_W_SEC` | 每秒导致数据填充到闪存缓存的请求数 |
| `FC_IO_RQ_W_SKIP` | 绕过闪存缓存写入磁盘的请求数 |
| `FC_IO_RQ_W_SKIP_LG` | 因 IO 大小较大而绕过闪存缓存写入磁盘的请求数 |
| `FC_IO_RQ_W_SKIP_LG_SEC` | 每秒因 IO 大小较大而绕过闪存缓存写入磁盘的请求数 |
| `FC_IO_RQ_W_SKIP_SEC` | 每秒绕过闪存缓存写入磁盘的请求数 |

根据表 5-1 中的度量指标，该值可以是 `cellsrv` 启动以来的累积值或瞬时值。`LIST METRICCURRENT` 命令显示单个存储单元的度量指标当前值。以下是一个 `cellcli` 命令示例，显示当前报告值非 0 的所有与闪存缓存相关的度量指标：

```
CellCLI> list metriccurrent attributes name,metricType,metricValue –
> where objectType = 'FLASHCACHE' and metricValue not like '0.*'
FC_BYKEEP_USED                  Instantaneous        4,395 MB
FC_BY_ALLOCATED                 Instantaneous        313,509 MB
FC_BY_DIRTY                     Instantaneous        28,509 MB
FC_BY_STALE_DIRTY               Instantaneous        1,052 MB
FC_BY_USED                      Instantaneous        342,890 MB
FC_IO_BY_ALLOCATED_OLTP         Instantaneous        327,733 MB
FC_IO_BY_DISK_WRITE             Cumulative          39,456 MB
FC_IO_BY_R                      Cumulative          233,829 MB
FC_IO_BY_R_ACTIVE_SECONDARY     Cumulative          17.445 MB
FC_IO_BY_R_ACTIVE_SECONDARY_MISS Cumulative         5.000 MB
FC_IO_BY_R_DW                   Instantaneous        82,065 MB
FC_IO_BY_R_MISS                 Cumulative          19,303 MB
FC_IO_BY_R_MISS_DW              Instantaneous        59,824 MB
FC_IO_BY_R_SKIP                 Cumulative          36,943 MB
FC_IO_BY_R_SKIP_NCMIRROR        Cumulative          14,714 MB
FC_IO_BY_W                      Cumulative          344,600 MB
FC_IO_BY_W_FIRST                Cumulative          86,593 MB
FC_IO_BY_W_OVERWRITE            Cumulative          216,326 MB
FC_IO_BY_W_POPULATE             Cumulative          41,680 MB
FC_IO_BY_W_SKIP                 Cumulative          532,343 MB
FC_IO_BY_W_SKIP_LG              Cumulative          401,333 MB
FC_IO_RQKEEP_R                  Cumulative          11 IO requests
FC_IO_RQ_DISK_WRITE             Cumulative          202,358 IO requests
FC_IO_RQ_R                      Cumulative          20,967,497 IO requests
FC_IO_RQ_REPLACEMENT_ATTEMPTED  Cumulative          1,717,959 IO requests
FC_IO_RQ_REPLACEMENT_FAILED     Cumulative          427,146 IO requests
FC_IO_RQ_R_ACTIVE_SECONDARY     Cumulative          2,233 IO requests
FC_IO_RQ_R_ACTIVE_SECONDARY_MISS Cumulative         80 IO requests
FC_IO_RQ_R_DW                   Cumulative          328,018 IO requests
FC_IO_RQ_R_MISS                 Cumulative          307,935 IO requests
FC_IO_RQ_R_MISS_DW              Cumulative          83,356 IO requests
FC_IO_RQ_R_SEC                  Rate                15.8 IO/sec
FC_IO_RQ_R_SKIP                 Cumulative          1,513,946 IO requests
FC_IO_RQ_R_SKIP_NCMIRROR        Cumulative          1,350,743 IO requests
FC_IO_RQ_R_SKIP_SEC             Rate                4.6 IO/sec
FC_IO_RQ_W                      Cumulative          26,211,256 IO requests
FC_IO_RQ_W_FIRST                Cumulative          5,961,364 IO requests
FC_IO_RQ_W_OVERWRITE            Cumulative          19,887,434 IO requests
FC_IO_RQ_W_OVERWRITE_SEC        Rate                5.0 IO/sec
FC_IO_RQ_W_POPULATE             Cumulative          362,458 IO requests
FC_IO_RQ_W_SEC                  Rate                5.0 IO/sec
FC_IO_RQ_W_SKIP                 Cumulative          14,121,057 IO requests
FC_IO_RQ_W_SKIP_LG              Cumulative          864,020 IO requests
FC_IO_RQ_W_SKIP_SEC             Rate                8.8 IO/sec
```


除了性能指标，您还可以查看缓存中包含哪些对象。`LIST FLASHCACHECONTENT`命令可用于实现此效果。该命令会为每个缓存对象显示一条条目，包括其占用的空间量以及各种其他统计数据。以下是查看特定存储节点上闪存缓存内容的示例。该命令的输出将列出排名前 20 的缓存对象：

```
CellCLI> list flashcachecontent where dbUniqueName like 'MBACH.*' -
>  attributes objectNumber, cachedKeepSize, cachedSize, cachedWriteSize, hitCount, missCount -
>  order by hitcount desc limit 20
103456              0          2845298688           2729680896          6372231          23137
103457              0          320430080            318562304           2031937          2293
94884               0          32874496             12853248            664676           6569
103458              0          103858176            101097472           662069           3051
4294967294          0          1261568              1032192             346445           2
4294967295          0          11259322368          5793267712          25488            440
102907              0          404414464            154648576           21243            551
93393               0          65232896             64184320            20019            53
103309              0          383328256            137814016           19930            342
102715              0          362323968            141139968           19585            273
93394               0          73457664             71581696            19148            48
93412               0          55427072             53739520            19122            55
103365              0          388464640            146743296           18938            335
103367              0          390332416            151314432           18938            347
103319              0          385581056            149807104           18908            408
102869              0          373628928            142901248           18596            383
103373              0          383008768            141934592           18515            288
103387              0          375513088            139116544           18194            427
103323              0          354279424            129171456           18117            313
103313              0          397303808            154607616           18018            318
CellCLI>
```

遗憾的是，属性列表中仍未包含对象名称。这意味着您必须返回数据库以确定哪个对象对应哪个（例如，通过查询`dba_objects`）。请注意，`cellcli`中的`ObjectNumber`属性等同于数据库视图（如`dba_objects`）中的`data_object_id`。以下是一个如何将存储节点的输出与数据库匹配的示例：

```
SQL> select owner, object_name, object_type
2 from dba_objects where data_object_id = 103456;
OWNER      OBJECT_NAME                     OBJECT_TYPE
---------- ------------------------------ ------------------------------
SOE        STRESSTESTTABLE                 TABLE
```

鉴于本章准备过程中进行的所有基准测试，Swingbench 压力测试表及其两个索引（数据对象 ID 103457 和 103458）成为命中率最高且（写入）缓存最多的段，这并不令人意外。

一些较新版本的 Exadata 还有一个有用的补充工具：`cellsrvstat`。虽然在第 11 章中会详细介绍，但在此提及也很有用。该命令行工具允许性能分析师将输出限制为所谓的统计信息组。您有许多统计信息组可用，但如果想将调查范围限定在闪存缓存，`flashcache`组无疑是最有用的。以下是`cellsrvstat`输出的示例：

```
[root@enkcel04 ∼]# cellsrvstat -stat_group flashcache
===Current Time===                                       Wed May  6 15:23:37 2015
== FlashCache related stats ==
Number of read hits                                               0       21065931
Read on flashcache hit(KB)                                        0      241004568
Number of keep read hits                                          0             11
Read on flashcache keep hit(KB)                                   0             88
Number of read misses                                             0          307947
Total IO size for read miss(KB)                                   0       19767368
Number of keep read misses                                        0              0
Total IO size for keep read miss(KB)                              0              0
Number of no cache reads                                          0        1547921
Total size for nocache read(KB)                                   0       38403468
Number of keep no cache reads                                     0              0
Number of partial reads                                           0          26596
Total size for partial reads(KB)                                  0        6897768
Number of optimized partial reads                                 0          26528
Number of keep partial reads                                      0              0
Number of cache writes                                            0       25900731
Total size for cache writes(KB)                                   0      311166944
Number of partial cache writes                                    0          15155
Number of redirty                                                 0       19923003
Number of keep cache writes                                       0              0
Total size for keep cache writes(KB)                              0              0
Number of partial keep cache writes                               0              0
Number of keep redirty                                            0              0
[and many more]
```

该工具的输出是累计的。左侧是指标及其名称。这里的零值列表示当前值，而大数值表示该统计信息的累计值。秉承优秀的 UNIX 传统，您也可以使用`cellsrvstat`来测量持续活动。为此，您需要指定`interval`（间隔）和`count`（计数）参数。如果您指定的间隔为 15 秒，重复计数为 2 次，那么您应该关注在初始性能指标显示之后产生的输出。与`iostat`及相关工具类似，第一次输出代表自启动以来的累计数据，而第二次输出则真正测量了指定间隔内的当前活动。

#### 在数据库层

直到 Oracle Database 12c，数据库都没有真正提供太多关于闪存缓存如何使用的可见信息。更糟糕的是，在 11.2.0.3 版本中，没有指标能够区分对（回写）闪存缓存的写入和读取——一切都混在一起。Oracle 12.1.0.2.x 提供了最多的信息，11.2.0.4 紧随其后，尽管本节中您将读到的大部分内容仅适用于 12.1.0.2。

您可以查询数据库的以下位置以获取有关闪存使用的信息：

*   会话统计信息，见于`v$mystat`、`v$sesstat`及相关视图
*   `V$CELL%`系列动态性能视图
*   AWR 报告

这些将在以下各节中讨论。



#### 与闪存缓存相关的性能计数器

性能计数器在第 11 章有详细介绍，但在此简要列出很重要，以便您对 Exadata 平台上闪存存储性能监控有一个基本了解。如果您需要关于这些计数器或相关计数器的更多信息，请翻阅第 11 章。

在引入写回闪存缓存之前，真正可用于闪存缓存使用或效率的统计信息只有几个：`cell flash cache read hits`以及两个与物理 I/O 相关的统计信息，即`physical read requests optimized`和密切相关的`physical read total bytes optimized`。后两者可能有点误导，因为优化的 I/O 请求可能因为（a）I/O 请求受益于存储索引，或（b）从闪存缓存提供服务而被优化。存储索引节省和闪存优势都归入同一个物理 I/O 统计中。如果您的查询只能利用闪存缓存，则该统计信息是相关的。这里有一个例子。会话 264 中的用户执行了以下语句：

```
SQL> select /* fctest001 */ count(*) from SOE.ORDER_ITEMS where order_id < 0;
COUNT(*)
----------
0
Elapsed: 00:00:16.45
```

性能分析师从实例范围的`v$sesstat`性能视图中筛选了以下统计信息：

```
SQL> select sid, name, value from v$sesstat natural join v$statname
  2  where sid = 264
  3  and name in (
  4    'cell flash cache read hits',
  5    'cell overwrites in flash cache',
  6    'cell partial writes in flash cache',
  7    'cell physical IO bytes saved by storage index',
  8    'cell writes to flash cache',
  9    'physical read IO requests',
 10    'physical read requests optimized',
 11    'physical read total bytes optimized',
 12    'physical write requests optimized',
 13    'physical write total bytes optimized')
 14  order by name;
SID NAME                                                                  VALUE
---------- ---------------------------------------------------------------- ----------
264 cell flash cache read hits                                              5895
264 cell overwrites in flash cache                                             0
264 cell partial writes in flash cache                                         0
264 cell physical IO bytes saved by storage index                              0
264 cell writes to flash cache                                                 0
264 physical read IO requests                                               6042
264 physical read requests optimized                                        5895
264 physical read total bytes optimized                                 810237952
264 physical write requests optimized                                          0
264 physical write total bytes optimized                                       0
10 rows selected.
```

请忽略写统计信息是统计名称列表一部分的事实——该脚本是通用的，可用于查询前台和后台进程。从输出可以看出，查询的执行没有受益于存储索引，这就是为什么`cell physical IO bytes saved by storage index`统计信息未填充的原因。然而，从其他 I/O 统计信息中，您可以推导出，在 6042 个物理读 I/O 请求中，有 5895 个（约 97%）的 I/O 请求是从闪存缓存中提供服务的。如果您回顾一下查看语句，您可以看到它查询来自 Swingbench 套件一部分的`SOE`模式的数据。如果您之前使用过 Swingbench，您会知道 Swingbench 模拟基于大量索引的联机事务处理风格工作负载。实际上，使用`fsx`（查找 SQL 执行）和`dplan`脚本，您可以看到查询并未被卸载：

```
SQL> @fsx4.sql
Enter value for sql_text: %fctest001%
Enter value for sql_id:
```



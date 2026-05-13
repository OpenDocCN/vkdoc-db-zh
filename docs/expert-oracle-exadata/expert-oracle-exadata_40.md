# Exadata 存储监控与 `sundiag.sh` 脚本

## 磁盘状态监控

文件中的信息包括事件日志以及控制器和磁盘驱动器的状态摘要。例如，下面的列表显示了连接到我们的一个数据库服务器的物理磁盘驱动器状态摘要（来自 `megacli64-status.out` 文件）：

```
Checking RAID status on enkx3db01.enkitec.com
Controller a0:  LSI MegaRAID SAS 9261-8i
No of Physical disks online : 4
Degraded : 0
Failed Disks : 0
```

很难说 Exadata 是否使用 `MegaCli64` 命令来监控磁盘驱动器的预测性故障，或者开发人员是否通过 API 利用了 SMART 指标，但这些信息都可以在命令行获取。关于 `MegaCli64` 的信息不多，但如果你有兴趣深入了解并仔细查看 Exadata 收集的用于确定磁盘子系统健康状况的一些指标，`sundiag.sh` 脚本是一个很好的起点。

## `sundiag.sh` 脚本与日志文件

如果你在存储单元上运行 `sundiag.sh` 脚本，它会收集有关单元配置、警报和数据库服务器上不存在的特殊日志文件的附加数据。以下列表描述了由 `sundiag.sh` 收集的这些附加日志文件。

*   `cell-detail`：`cell-detail` 文件包含有关存储单元的详细站点特定信息。这是 `CellCLI` 命令 `LIST CELL DETAIL` 的输出。
*   `celldisk-detail`：此文件包含单元磁盘的详细报告。该报告使用 `CellCLI` 命令 `LIST CELLDISK DETAIL` 创建。除其他信息外，它还显示单元磁盘的状态、逻辑单元号 (LUN) 和物理设备分区。
*   `lun-detail`：此报告使用 `CellCLI` 命令 `LIST LUN DETAIL` 生成。它包含有关配置单元磁盘的底层 LUN 的详细信息。此报告中包括 LUN 的名称、设备类型和物理设备名称（例如 `/dev/sdw`）。
*   `physicaldisk-detail`：`physicaldisk-detail` 文件包含存储单元用于数据库类型存储和 Flash Cache 的所有物理磁盘和 FMOD 的详细报告。它使用 `CellCLI` 命令 `LIST PHYSICALDISK DETAIL` 生成，包括有关这些设备的重要信息，例如设备类型（硬盘或闪盘）、制造商和型号、插槽地址以及设备状态。
*   `physicaldisk-fail`：此文件包含所有状态不是“正常”的物理磁盘（包括闪盘）的列表。这包括状态为“不存在”的磁盘，即已更换但尚未从配置中移除的故障磁盘。更换物理磁盘后，其旧配置会在系统中保留七天，之后自动清除。
*   `griddisk-detail`：此文件包含存储单元上配置的所有网格磁盘的详细报告。它使用 `CellCLI` 命令 `LIST GRIDDISK DETAIL` 创建，除其他信息外，还包括你在存储单元上配置的所有网格磁盘的网格磁盘名称、单元磁盘名称、大小和状态。
*   `griddisk-status`：此文件包含存储单元上配置的每个网格磁盘的名称和状态。它使用 `CellCLI` 命令 `LIST GRIDDISK ATTRIBUTES NAME, STATUS, ASMMODESTATUS, ASMDEACTIVATIONOUTCOME` 创建，并包含从存储服务器和 ASM 两个角度看网格磁盘状态的详细信息。
*   `flashcache-detail`：此报告包含构成单元闪存缓存的所有 FMOD 的列表。它是 `CellCLI` 命令 `LIST FLASHCACHE DETAIL` 的输出，包括闪存缓存的大小和状态。此报告中还包含所有以降级模式运行的闪存单元磁盘的列表。
*   `flashlog-detail`：此报告包含构成单元闪存日志区域的所有 FMOD 的列表。它是 `CellCLI` 命令 `LIST FLASHLOG DETAIL` 的输出，包括闪存日志区域的大小和状态。此报告中还包含所有以降级模式运行的闪存单元磁盘的列表。
*   `alerthistory`：`alerthistory` 文件包含存储单元上发生的所有警报的详细报告。它使用 `CellCLI` 命令 `LIST ALERTHISTORY` 创建。
*   `alert.log`：`alert.log` 文件由 `cellsrv` 进程写入。与数据库或 ASM 警报日志文件类似，存储单元的 `alert.log` 包含有关存储单元及其磁盘驱动器状态的重要运行时信息。此文件对于诊断单元存储问题非常有用。在运行 12c 版本的 Exadata 存储单元上，有多个警报日志，每个卸载服务器一个。
*   `ms-odl.trc`：`ms-odl.trc` 包含来自单元管理服务器进程的详细运行时跟踪级别信息。
*   `ms-odl.log`：此文件由单元的管理服务器进程写入。它不包含在 `sundiag.sh` 脚本创建的集合中，但我们发现它对于诊断存储单元中出现的问题非常有用。它还包含日常的操作消息。存储单元通过轮转方式维护其日志文件，类似于操作系统轮转系统日志 (`/var/log/messages`) 的方式。`ms-odl.log` 文件记录这些任务以及更关键的任务（如磁盘故障）。


#### 单元警报

作为监控功能的一部分，Exadata 在存储单元中跟踪超过 70 种警报指标类型。还可以使用 Grid Control 的监控和告警功能定义额外的警报。警报严重性分为四个类别：信息（Information）、警告（Warning）、关键（Critical）和清除（Clear）。这些类别用于管理警报通知。例如，您可以选择仅接收关键警报的电子邮件通知。清除严重性用于在组件恢复正常状态时通知您。可以使用 `LIST ALERTHISTORY DETAIL` 命令生成系统所产生警报的详细报告。以下是存储单元生成的一个警报示例：

```
name:                   209_1
alertMessage:           "All Logical drives are in WriteThrough caching mode.
                        Either battery is in a learn cycle or it needs to be
                        replaced. Please contact Oracle Support"
alertSequenceID:        209
alertShortName:         Hardware
alertType:              Stateful
beginTime:              2011-01-17T04:42:10-06:00
endTime:                2011-01-17T05:50:29-06:00
examinedBy:
metricObjectName:       LUN_CACHE_WT_ALL
notificationState:      1
sequenceBeginTime:      2011-01-17T04:42:10-06:00
severity:               critical
alertAction:            "Battery is either in a learn cycle or it needs
                        replacement. Please contact Oracle Support"
```

当电池随后恢复到正常状态时，会生成一个后续警报，其严重性为清除，表明该组件已恢复正常运行状态：

```
name:                   209_2
alertMessage:           "Battery is back to a good state"
...
severity:               clear
alertAction:            "Battery is back to a good state. No Action Required"
```

在检查警报时，您应养成设置警报的 `examinedBy` 属性的习惯，以便跟踪哪些警报正在调查中。如果设置了 `examinedBy` 属性，就可以在 `LIST ALERTHISTORY` 命令中将其用作过滤条件，以报告所有当前未被处理的警报。通过添加严重性过滤条件，您可以进一步将输出缩减为仅关键警报。例如：

```
LIST ALERTHISTORY WHERE severity = 'critical' AND examinedBy = ' ' DETAIL
```

要设置警报的 `examinedBy` 属性，请使用 `ALTER ALERTHISTORY` 命令并指定您希望更改的警报名称。例如，我们可以如下设置电池警报的 `examinedBy` 属性：

```
CellCLI> alter alerthistory 209_1 examinedBy="acolvin"
Alert 209_1 successfully altered

CellCLI> list alerthistory attributes name, alertMessage, examinedby where name=209_1 detail
         name:                   209_1
         alertMessage:           "All Logical drives are in WriteThrough caching mode.
                                 Either battery is in a learn cycle or it needs to be
                                 replaced. Please contact Oracle Support"
         examinedBy:             acolvin
```

关于管理、报告和自定义 Exadata 警报，还有很多内容需要讨论。要详细涵盖这个主题，需要整整一章的篇幅。在本节中，我们只触及了基础。幸运的是，一旦您配置好了用于告警通知的电子邮件，管理这些警报几乎不需要做太多工作。在许多环境中，仅使用电子邮件通知就足以捕获和报告关键警报。

## 备份 Exadata

当我们收到第一台 Exadata 系统时，我们的一个主要问题是：“我们如何备份所有内容，以便在出现严重问题时将其恢复到工作状态？” 2010 年 5 月 Exadata 到货时，最新的 Cell 软件版本是 11.2.1.2.1。当时，备份数据库服务器的唯一方法是使用第三方备份软件或标准的 Linux 命令，如 `tar`。Oracle 不断为 Exadata 开发新功能，不到一年后，搭载原生 Linux 逻辑卷管理器（LVM）的 Exadata X-2 数据库服务器发布。这是一个重大进步，因为 LVM 具有内置的快照功能，为备份操作系统提供了一种简便的方法。存储单元使用内置的方法进行备份和恢复。在本节中，我们将看看 Oracle 推荐用于备份 Exadata 数据库存储服务器和存储单元的各种方法。我们还将简要介绍 Recovery Manager（RMAN）以及 Exadata 提供的可改进数据库备份和恢复性能的一些功能。之后，我们将研究如何从一些更常见的系统故障类型中恢复。您可能会感到惊讶，但本章的重点不是数据库恢复。对于数据库备份和恢复，几乎没有特定于 Exadata 的考虑因素。大多数特定于产品的备份和恢复方法涉及备份和恢复包含操作系统和 Exadata 软件的系统卷。因此，我们将花相当多的时间讨论恢复，从单元磁盘的丢失到数据库服务器或存储单元上系统卷的丢失。

#### 备份数据库服务器

Exadata 计算节点默认配置利用 Linux 逻辑卷管理（LVM）。逻辑卷管理器为物理磁盘分区提供了一个抽象层，类似于 ASM 为其底层物理存储设备所做的那样。LVM 具有类似于 ASM 磁盘组的卷组。这些卷组由一个或多个物理磁盘（或磁盘分区）组成，就像 ASM 磁盘组由一个或多个物理磁盘（或磁盘分区）组成一样。LVM 卷组被划分为可以创建文件系统的逻辑卷。类似地，数据库利用 ASM 磁盘组创建用于存储表、索引和其他数据库对象的表空间。将物理存储从文件系统中抽象出来，允许系统管理员根据需要扩展和收缩逻辑卷（以及文件系统）。使用 LVM 管理 Exadata 数据库存储服务器的存储还有许多其他优势，但我们的重点将是 Linux LVM 提供的新备份和恢复功能，即 LVM 快照。除了方便和易用性之外，LVM 快照消除了使用 `tar` 命令或第三方备份产品进行简单备份时面临的许多典型挑战。例如，根据备份集中的数据量，文件系统备份可能需要相当长的时间才能完成。这些备份不是时间点一致的，这意味着如果您必须从备份中恢复文件系统，文件中的数据将代表从备份过程开始到结束的不同时间点。在备份周期内继续运行的应用程序可能持有文件锁，导致这些文件被跳过（未备份）。同样，打开的应用程序不可避免地会在备份周期内更改数据。即使您能够备份这些打开的文件，除非在进行备份前关闭应用程序，否则您无法知道它们是否处于可用状态。LVM 快照是瞬时完成的，因为实际上没有复制数据。您可以将快照视为指向构成文件系统内容的物理数据块的指针索引。当文件被更改或删除时，该文件的原始块会被写入快照卷。因此，即使备份需要几个小时才能完成，它仍然与创建快照的时刻保持一致。现在，让我们看看如何使用 LVM 快照来创建数据库服务器的一致文件系统备份。


## 使用 LVM 快照进行系统备份

使用 LVM 快照创建文件系统备份是一个相当简单的过程。首先，你需要为备份的最终副本创建一个目标位置。这可以是 SAN 或 NAS 存储，或者仅仅是来自另一台服务器共享的 NFS 文件系统。如果你在卷组中有足够的可用空间来存储备份文件，你可以创建一个临时逻辑卷，用于在将备份发送到磁带之前进行暂存。这可以通过 `lvcreate` 命令来完成。在创建新的逻辑卷之前，请使用 `vgdisplay` 命令确保你的卷组中有足够的可用空间：

```
[root@enkx4db01 ~]# vgdisplay
--- Volume group ---
VG Name               VGExaDb
...
VG Size               1.63 TB
PE Size               4.00 MB
Total PE              428308
Alloc PE / Size       47104 / 184.00 GB
Free  PE / Size       381204 / 1.45 TB
...
```

`vgdisplay` 命令显示了我们卷组的大小、当前正在使用的物理扩展（PE）以及卷组中可用的剩余空间量。`Free PE/Size` 属性表明我们的卷组中还有 1.45TB 的剩余空间。

### 创建备份目标位置

首先，我们将从另一台系统挂载一个 NFS 共享作为备份的目标位置。我们将其称为 `/mnt/nfs`：

```
[root@enkx4db01 ~]# mount -t nfs -o rw,intr,soft,proto=tcp,nolock <ip>/share /mnt/nfs
```

### 创建并挂载快照

接下来，我们将使用 `lvcreate` 和 `e2label` 命令为 `/` 和 `/u01` 创建并标记 LVM 快照。请注意我们使用 `–L1G` 和 `-L5G` 选项来创建这些快照。`–L` 参数决定了快照卷的大小。在快照创建后，当数据块被修改或删除时，该块的原始副本将被写入快照。为快照分配足够的大小以存储所有已更改块的原始副本非常重要。快照不会被长期使用，因此通常 5GB-10GB 的空间就足够了。如果快照空间用尽，它将被停用。

```
[root@enkx4db01 ~]# lvcreate –L1G -s -n root_snap /dev/VGExaDb/LVDbSys1
Logical volume "root_snap" created

[root@enkx4db01 ~]# e2label /dev/VGExaDb/root_snap DBSYS_SNAP

[root@enkx4db01 ~]# lvcreate –L5G -s -n u01_snap /dev/VGExaDb/LVDbOra1
Logical volume "u01_snap" created

[root@enkx4db01 ~]# e2label /dev/VGExaDb/u01_snap DBORA_SNAP
```

接下来，挂载快照卷。我们使用文件系统标签（`DBSYS_SNAP` 和 `DBORA_SNAP`）来确保挂载的是正确的卷。挂载后，它们就可以被复制到 NFS 挂载点。`df` 命令显示了我们新的文件系统以及我们想要包含在系统备份中的逻辑卷：`VGExaDb-LVDbSys1`（根文件系统的逻辑卷）和 `VGExaDb-LVDbOra1`（`/u01` 文件系统的逻辑卷）。请注意，`/boot` 文件系统不使用 LVM 进行存储。该文件系统必须使用 `tar` 命令进行备份。这不是问题，因为 `/boot` 文件系统相当小且静态，因此我们不担心这些文件在备份周期中被修改、锁定或打开。

```
[root@enkx4db01 ~]# mkdir –p /mnt/snaps/u01

[root@enkx4db01 ~]# mount –L DBSYS_SNAP /mnt/snaps

[root@enkx4db01 ~]# mount –L DBORA_SNAP /mnt/snaps/u01

[root@enkx4db01 mnt]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VGExaDb-LVDbSys1
                      30G   27G  1.6G  95% /
/dev/sda1             496M   40M  431M   9% /boot
/dev/mapper/VGExaDb-LVDbOra1
                      99G   53G   41G  57% /u01
tmpfs                 252G     0  252G   0% /dev/shm
192.168.10.9:/nfs/backup
                      5.4T  182G  5.2T   4% /mnt/nfs
/dev/mapper/VGExaDb-root_snap
                      30G   27G  1.4G  96% /mnt/snaps
/dev/mapper/VGExaDb-u01_snap
                      99G   53G   41G  57% /mnt/snaps/u01
```

### 验证快照一致性

现在我们有了确保 `/` 和 `/u01` 文件系统一致性的快照，我们准备好进行备份了。为了证明这些快照是一致的，我们将 `/etc/hosts` 文件复制到 `/root` 目录中的一个测试文件。如果快照按预期工作，这个文件将不会被包含在我们的备份中，因为它是在快照创建之后创建的。命令如下：

```
[root@enkx4db01 ~]# cp /etc/hosts /root/test_file.txt
```

因为快照已经挂载，我们可以像浏览任何其他文件系统一样浏览它们。快照文件系统看起来和感觉起来都像原始文件系统，只有一个例外。如果我们查看挂载的快照中是否有我们创建的测试文件（或快照拍摄后发生的任何其他更改），我们看不到它。它不在那里，因为该文件是在快照创建之后创建的：

```
[root@enkx4db01 ~]# ls -l /root/testfile
-rw-r--r-- 1 root root 1724 Nov 24 14:23 /root/test_file.txt   <- 我们创建的测试文件

[root@enkx4db01 ~]# ls -l /mnt/snaps/root/test_file.txt
ls: /mnt/snaps/root/testfile: No such file or directory    <- 快照中没有测试文件
```

### 执行备份

一旦快照挂载，就可以使用任何标准的 Linux 备份软件进行备份。在本测试中，我们将使用 `tar` 命令将 `/` 和 `/u01` 文件系统的 tar 包备份创建到 NFS 共享。因为我们备份的是快照，所以不必担心备份期间有文件被打开、锁定或更改。请注意，我们还在此备份中包含了 `/boot` 目录。

```
[root@enkx4db01 ~]# cd /mnt/snaps

[root@enkx4db01 snap]# tar -pjcvf /mnt/nfs/backup.tar.bz2 * /boot --exclude \
nfs/backup.tar.bz2 --exclude /mnt/nfs >         \
/tmp/backup_tar.stdout 2> /tmp/backup_tar.stderr
```

### 清理

备份完成后，你应该检查错误文件 `/tmp/backup_tar.stderr`，查看备份期间记录的任何问题。如果你对备份满意，你可以卸载并删除快照。每次运行备份时，你都将创建一组新的快照。备份副本完成后，你可以选择卸载并删除你创建的临时逻辑卷：

```
[root@enkx4db01 snap]# cd /

[root@enkx4db01 /]# umount /mnt/snaps/u01

[root@enkx4db01 /]# rm -Rf /mnt/snaps/u01

[root@enkx4db01 /]# umount /mnt/snaps

[root@enkx4db01 /]# rm -Rf /mnt/snaps

[root@enkx4db01 /]# lvremove /dev/VGExaDb/root_snap
Do you really want to remove active logical volume root_snap? [y/n]: y
Logical volume "root_snap" successfully removed

[root@enkx4db01 /]# lvremove /dev/VGExaDb/u01_snap
Do you really want to remove active logical volume u01_snap? [y/n]: y
Logical volume "u01_snap" successfully removed
```

### 总结

早期的 Exadata V2 模型没有采用 LVM 来管理文件系统存储。没有 LVM 快照，要获得干净的系统备份就需要关闭服务器上的应用程序（包括数据库），或者购买第三方备份软件。即便如此，也无法创建所有文件在时间点上一致的备份。LVM 快照填补了 Exadata 备份和恢复架构中的一个重要空白，并为备份数据库服务器提供了一种简单、可管理的策略。在本章后面，我们将讨论当文件系统丢失或损坏时，如何使用这些备份来恢复数据库服务器。



#### 备份存储单元

存储单元中的前两个磁盘包含 Linux 操作系统。这些 Linux 分区通常被称为系统卷。**不建议**使用行业标准的 Linux 备份软件来备份系统卷。那么，如何备份系统卷呢？答案是：你无需手动备份。Exadata 会通过一个名为 `CELLBOOT` USB 闪存驱动器的内置 USB 设备自动为你完成此操作。如果你比较谨慎，也可以使用外部 USB 闪存驱动器创建自己的单元恢复镜像。除了 `CELLBOOT` USB 闪存驱动器外，Exadata 还会在一组独立的磁盘分区上，保留系统卷在上次补丁安装之前的完整副本。这些备份分区用于回滚补丁。现在，让我们来看看这些备份方法是如何工作的。

#### `CELLBOOT` USB 闪存驱动器

你可以像对待任何插入笔记本电脑的外部 USB 驱动器一样看待内置的 `CELLBOOT` USB 闪存驱动器。可以使用 `parted` 命令查看该设备，如下所示：

```
[root@enkx4cel01 ∼]# parted /dev/sdac print
Model: ORACLE UNIGEN-UFD (scsi)
Disk /dev/sdac: 4010MB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Number  Start   End     Size    Type     File system  Flags
 1      11.3kB  4008MB  4008MB  primary  ext3
```

为了有趣起见，我们挂载了这个内置 USB 闪存驱动器，想看看 Oracle 在这个备份中包含了什么。下面的列表显示了该设备的内容：

```
[root@enkx4cel01 ∼]# mount /dev/sdm1 /mnt/usb
[root@enkx4cel01 ∼]# ls -al /mnt/usb
total 95816
drwxr-xr-x  7 root root      4096 Oct  3 21:26 .
drwxr-xr-x  9 root root      4096 Nov 23 15:49 ..
-r-xr-x---  1 root root      2048 Aug 17  2011 boot.cat
-r-xr-x---  1 root root        16 Oct  9  2013 boot.msg
drwxr-----  2 root root      4096 Oct  3 20:14 cellbits
drwxrwxr-x  2 root root      4096 Oct  3 20:15 grub
-rw-r-----  1 root root        16 Oct  3 20:14 I_am_CELLBOOT_usb
-rw-r-----  1 root root       805 Oct  3 19:53 image.id
-rw-r-----  1 root root       441 Oct  3 19:55 imgboot.lst
-rw-rw-r--  1 root root   8280755 Jul 14 04:12 initrd-2.6.32-300.19.1.el5uek.img
-rw-r-----  1 root root   7381429 Oct  3 20:14 initrd-2.6.39-400.128.17.el5uek.img
-rw-r-----  1 root root   70198394 Oct  3 19:55 initrd.img
-r-xr-x---  1 root root     10648 Aug 17  2011 isolinux.bin
-r-xr-x---  1 root root       155 Apr 14  2014 isolinux.cfg
-rw-r-----  1 root root        25 Oct  3 20:14 kernel.ver
drwxr-----  4 root root      4096 Nov  7 16:28 lastGoodConfig
drwxr-xr-x  3 root root      4096 Oct  3 21:38 log
drwx------  2 root root     16384 Oct  3 20:11 lost+found
-r-xr-x---  1 root root      94600 Aug 17  2011 memtest
-r-xr-x---  1 root root      7326 Aug 17  2011 splash.lss
-r-xr-x---  1 root root      1770 Oct  9  2013 trans.tbl
-rwxr-x---  1 root root   4121488 Jul 14 04:12 vmlinuz
-rwxr-xr-x  1 root root   3688864 Jul 14 04:12 vmlinuz-2.6.32-300.19.1.el5uek
-rwxr-----  1 root root   4121488 Oct  3 20:08 vmlinuz-2.6.39-400.128.17.el5uek
```

在这个备份中，我们看到了 Linux 启动镜像以及启动 Linux 和恢复操作系统所需的所有文件。请注意，你还会看到一个名为 `lastGoodConfig` 的目录。此目录是我们存储单元上 `/opt/oracle.cellos/iso/lastGoodConfig` 目录的备份。还有一个名为 `cellbits` 的目录，其中包含单元服务器软件。我们不仅在内置 USB 驱动器上拥有将存储单元恢复到可启动状态所需的完整副本，还拥有所有重要单元配置文件和单元服务器二进制文件的在线备份。

#### 外部 USB 驱动器

除了内置的 `CELLBOOT` USB 闪存驱动器外，Exadata 还提供了一种方法，可以使用你在当地电子商店购买的普通 1-8GB USB 闪存驱动器创建自己的外部可启动恢复镜像。Exadata 会在它找到的第一个外部 USB 驱动器上创建救援镜像，因此在创建此恢复镜像之前，必须从系统中移除所有其他外部 USB 驱动器，否则脚本将发出警告并退出。

回想一下，Exadata 存储单元维护着操作系统和单元软件的两个版本：活动和非活动版本。它们作为两组独立的磁盘分区进行管理，分别对应 `/` 和 `/opt/oracle` 文件系统，可以使用 `imageinfo` 命令确认，如下所示：

```
[root@enkx4cel01 ∼]# imageinfo | grep device
Active system partition on device: /dev/md6
Active software partition on device: /dev/md8
Inactive system partition on device: /dev/md5
Inactive software partition on device: /dev/md7
```

`imageinfo` 命令显示了存储单元上当前（活动）和先前（非活动）的系统卷。使用 `df` 命令，我们可以看到我们确实正在使用 `imageinfo` 命令输出中标识的活动分区（`/dev/md6` 和 `/dev/md8`）：

```
[root@enkx4cel01 ∼]# df | egrep 'Filesystem|md6|md8'
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/md6              10317752   6632836   3160804  68% /
/dev/md8               2063440    654956   1303668  34% /opt/oracle
```

默认情况下，`make_cellboot_usb` 命令将基于你的活动配置（你当前正在运行的配置）创建救援镜像。使用 `–inactive` 选项，你可以基于先前的配置创建救援镜像。非活动分区是上次安装补丁时处于活动状态的系统卷。

`make_cellboot_usb` 命令用于创建可启动的救援镜像。要创建外部救援镜像，你只需将 USB 闪存驱动器插入存储单元前面板上的一个 USB 端口，然后运行 `make_cellboot_usb` 命令即可。

**注意**

救援镜像将在系统找到的第一个外部 USB 驱动器上创建。在创建外部救援镜像之前，请从系统中移除所有其他外部 USB 驱动器。

例如，以下列表展示了创建外部 USB 救援镜像的过程。`make_cellboot_usb` 脚本的输出相当长，超过 100 行，因此我们不会在此处显示全部内容。以下列表中省略的部分输出包括用于在 USB 驱动器上创建分区的 `fdisk` 命令输出、文件系统格式化过程以及为创建可启动救援盘而复制的众多文件。

```
[root@enkx4cel01 oracle.SupportTools]# ./make_cellboot_usb
[WARNING] More than one USB devices suitable for use as Oracle Exadata Cell start up boot device.
Candidate for the Oracle Exadata Cell start up boot device     : /dev/sdad
Partition on candidate device                                   : /dev/sdad1
The current product version                                     : 12.1.1.1.1.140712
Label of the current Oracle Exadata Cell start up boot device   :
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: cell usb        : /dev/sdad
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: mnt sys         : /
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: preserve        : preserve
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: mnt usb         : /mnt/usb.make.cellboot
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: lock            : /tmp/usb.make.cellboot.lock
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: serial console  :
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: kernel mode     : kernel
2014-11-25 10:12:27 -0600  [DEBUG] set_cell_boot_usb: mnt iso save    :
2014-11-25 10:12:27 -0600   Create CELLBOOT USB on device /dev/sdad
...
```



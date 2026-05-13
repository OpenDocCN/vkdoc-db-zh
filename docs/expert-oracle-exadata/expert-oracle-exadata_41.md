# `make_cellboot_usb` 脚本日志

```
2014-11-25 10:15:11 -0600  Copying ./isolinux.cfg to /mnt/usb.make.cellboot/. ...
2014-11-25 10:15:44 -0600  Copying ./trans.tbl to /mnt/usb.make.cellboot/. ...
2014-11-25 10:15:48 -0600  Copying ./isolinux.bin to /mnt/usb.make.cellboot/. ...
2014-11-25 10:15:48 -0600  Copying ./boot.cat to /mnt/usb.make.cellboot/. ...
2014-11-25 10:15:48 -0600  Copying ./initrd.img to /mnt/usb.make.cellboot/. ...
2014-11-25 10:16:26 -0600  Copying ./memtest to /mnt/usb.make.cellboot/. ...
2014-11-25 10:16:29 -0600  Copying ./boot.msg to /mnt/usb.make.cellboot/. ...
2014-11-25 10:16:30 -0600  Copying ./vmlinuz-2.6.39-400.128.17.el5uek to /mnt/usb.make.cellboot/. ...
2014-11-25 10:16:31 -0600  Copying ./cellbits/ofed.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:16:38 -0600  Copying ./cellbits/commonos.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:17:51 -0600  Copying ./cellbits/sunutils.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:18:11 -0600  Copying ./cellbits/cellfw.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:19:30 -0600  Copying ./cellbits/doclib.zip to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:20:34 -0600  Copying ./cellbits/debugos.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:26:07 -0600  Copying ./cellbits/exaos.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:27:19 -0600  Copying ./cellbits/cellboot.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:27:26 -0600  Copying ./cellbits/cell.bin to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:30:52 -0600  Copying ./cellbits/kernel.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:31:37 -0600  Copying ./cellbits/cellrpms.tbz to /mnt/usb.make.cellboot/./cellbits ...
2014-11-25 10:33:59 -0600  Copying ./initrd-2.6.39-400.128.17.el5uek.img to /mnt/usb.make.cellboot/. ...
2014-11-25 10:34:20 -0600  Copying ./splash.lss to /mnt/usb.make.cellboot/. ...
2014-11-25 10:34:26 -0600  Copying ./image.id to /mnt/usb.make.cellboot/. ...
2014-11-25 10:34:32 -0600  Copying ./imgboot.lst to /mnt/usb.make.cellboot/. ...
2014-11-25 10:34:37 -0600  Copying ./vmlinuz to /mnt/usb.make.cellboot/. ...
2014-11-25 10:34:44 -0600  Copying lastGoodConfig/* to /mnt/usb.make.cellboot/lastGoodConfig ...
/opt/oracle.cellos
...
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: mnt sys        : /
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: grub template  : USB_grub.in
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: boot dir       : /mnt/usb.make.cellboot
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: kernel param   : 2.6.39-400.128.17.el5uek
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: marker         : I_am_CELLBOOT_usb
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: mode           :
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: sys dev        :
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: Image id file: //opt/oracle.cellos/image.id
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: System device where image id exists: /dev/md5
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: Kernel version: 2.6.39-400.128.17.el5uek
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: System device with image_id (/dev/md5) and kernel version (2.6.39-400.128.17.el5uek) are in sync
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: Full kernel version: 2.6.39-400.128.17.el5uek
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: System device for the next boot: /dev/md5
2014-11-25 10:37:01 -0600  [DEBUG] set_grub_conf_n_initrd: initrd for the next boot: /mnt/usb.make.cellboot/initrd-2.6.39-400.128.17.el5uek.img
2014-11-25 10:37:01 -0600  [INFO] set_grub_conf_n_initrd: Set /dev/md5 in /mnt/usb.make.cellboot/I_am_CELLBOOT_usb
2014-11-25 10:37:01 -0600  [INFO] Set kernel 2.6.39-400.128.17.el5uek and system device /dev/md5 in generated /mnt/usb.make.cellboot/grub/grub.conf from //opt/oracle.cellos/tmpl/USB_grub.in
2014-11-25 10:37:01 -0600  [INFO] Set /dev/md5 in /mnt/usb.make.cellboot/initrd-2.6.39-400.128.17.el5uek.img
33007 blocks
2014-11-25 10:37:12 -0600  [WARNING] restore_preserved_cell_boot_usb: Unable to restore logs and configs. Archive undefined
```

```
GNU GRUB  version 0.97  (640K lower / 3072K upper memory)
[ Minimal BASH-like line editing is supported.  For the first word, TAB
lists possible command completions.  Anywhere else TAB lists the possible
completions of a device/filename.]
grub> root (hd0,0)
Filesystem type is ext2fs, partition type 0x83
grub> setup (hd0)
Checking if "/boot/grub/stage1" exists... no
Checking if "/grub/stage1" exists... yes
Checking if "/grub/stage2" exists... yes
Checking if "/grub/e2fs_stage1_5" exists... yes
Running "embed /grub/e2fs_stage1_5 (hd0)"...  16 sectors are embedded.
succeeded
Running "install /grub/stage1 (hd0) (hd0)1+16 p (hd0,0)/grub/stage2 /grub/grub.conf"... succeeded
Done.
```

在这里，你可以看到 `make_cellboot_usb` 脚本复制了恢复存储单元所需的所有存储单元软件 (`cellbits`) 和配置文件 (`lastGoodConfig`)。最后，你会看到 Grub 引导加载程序被安装到了 USB 驱动器上，这样你就可以从它启动系统了。脚本完成后，你可以从系统上移除外接 USB 磁盘。这个救援盘以后可以在需要时用来将你的存储单元恢复到工作状态。

#### 备份数据库

Exadata 在容量和性能上代表了一次飞跃。就在几年前，大型数据库还以千兆字节来描述。如今，以太字节为单位的数据库已很常见。就在不久之前，一个包含数千万行的表被认为是非常巨大的。如今，我们通常会看到包含数百亿行的表。这一趋势清楚地表明，我们很快将看到以艾字节为单位的数据库。正如你可能想象的那样，这给备份和恢复带来了一些独特的挑战。备份 Exadata 数据库的工具并未发生根本性变化，而在合理时间内完成备份的需求却变得越来越难以实现。我们将在这里讨论的一些策略并非全新；然而，我们将探讨如何利用平台的速度，使备份性能能够跟上数据库数据量不断增长的步伐。

#### 基于磁盘的备份

Oracle 10g 为我们引入了一项名为 `闪回恢复区` 的新特性，它扩展了 `恢复管理器` 管理备份的结构化方法。最近，该特性已更名为 `快速恢复区` (`FRA`)。`FRA` 是一个存储区域，与其他数据库存储非常相似。它可以在原始设备、块设备、文件系统以及 `ASM` 上创建。由于 `FRA` 利用了基于磁盘的存储，它为数据库恢复提供了一种非常快速的存储介质。在使用 Exadata 的高性能存储架构时尤其如此。无需从磁带检索备份可以节省数小时甚至数天的时间来恢复数据库。而且，由于 `FRA` 是数据库的扩展，Oracle 会自动为你管理该空间。当 `FRA` 中的文件备份到磁带时，它们不会立即被删除。相反，只要有足够的可用空间，它们就会保持在线状态。当需要更多空间时，数据库会（以 FIFO 方式）删除足够多的这些文件以提供所需的空间。



该请求因被视为高风险而被拒绝。


#### 等待事件

有两个与 Exadata 特定相关的等待事件，它们由 Exadata 平台上的数据库备份和恢复操作触发：`cell smart incremental backup` 和 `cell smart restore from backup`。这些等待事件在第 10 章中有更详细的介绍。

- `cell smart incremental backup`：当 Exadata 将增量备份处理卸载到存储单元时，会发生此等待事件。`V$SESSION_WAIT` 视图的 P1 列包含单元哈希号。此哈希值可用于比较每个存储单元的相对备份性能，并确定是否存在性能问题。
- `cell smart restore from backup`：当 Exadata 在恢复操作期间将初始化文件的任务卸载到存储单元时，会发生此等待事件。`V$SESSION_WAIT` 的 P1 列包含单元哈希号。此哈希值可用于比较每个存储单元的相对恢复性能，并确定是否存在性能问题。

## 恢复 Exadata

本节更合适的标题可能是“当系统出错时”。毕竟，通常正是在那个时候，我们才意识到自己在系统恢复方面的实践经验是多么匮乏。随着企业界不断榨取我们工作周中每一滴高效产出，数据库管理员和系统管理员即使不是全部，也大部分清醒时间（有时甚至是睡眠时间）都花在了“维持系统运转”上。因此，实际上，练习系统恢复往往被视为那个众所周知的“受气包”——很少被想起，也极少被关注。即使我们发现自己处于令人羡慕的有时间练习系统恢复的境地，也很难有备用设备可供练习。所以，如果你正在阅读本节而系统并未真正出故障，那值得称赞。在本节中，我们将讨论如何使用本章前面“备份 Exadata”部分介绍的备份方法来进行 Exadata 系统恢复。

### 恢复数据库服务器

备份和恢复数据库服务器可以使用第三方备份软件完成，也可以使用熟悉的命令（如 `tar` 和 `zip`）通过自定义脚本完成。Linux 逻辑卷管理器 (`LVM`) 提供了通过快照备份数据库服务器的能力，以创建基于时间点的、基于 `tar` 的备份集。恢复 Exadata 数据库服务器的过程是一个非常结构化的过程，且特定于 Exadata。在本节中，我们将逐步介绍此过程，并假设备份是使用本章前面讨论的备份过程创建的。因此，如果你尚未阅读本章的该部分内容，可能需要在继续之前查看一下。

**注意**

在执行本节列出的任何恢复步骤之前，最好先向 Oracle 支持部门提交服务请求。此处描述的许多工具都需要密码或协助，而这通常只能由 Oracle 的支持组织提供。这些步骤应仅作为最后手段来执行。

#### 使用基于 LVM 快照的备份映像进行恢复

使用本章前面讨论的 `LVM` 快照备份过程来恢复数据库服务器是一个相当直接的过程。我们在此过程中使用的备份映像 `backup.tar.bz2`，是本章前面创建的，包含了 `/`、`/boot` 和 `/u01` 文件系统。你需要做的第一件事是将备份映像放置在故障数据库服务器可以挂载的 `NFS` 文件系统上。然后，从所有 Exadata 服务器上包含的特殊诊断 `ISO` 启动映像启动服务器。当系统从诊断 `ISO` 启动时，你将逐步收到恢复过程的提示。让我们看看从本章前面获取的基于 `LVM` 快照的备份中恢复故障数据库服务器的基本步骤：

1.  将 `LVM` 快照备份映像放置在故障服务器可通过 `IP` 地址访问的 `NFS` 共享文件系统上。我们将要操作的文件名为 `backup.tar.bz2`。
2.  通过 `ILOM` 远程控制台，将 `/opt/oracle.SupportTools/diagnostics.iso` 启动映像（从一台正常运行的服务器获取）附加到故障服务器。
3.  重新启动故障服务器，并选择 `CD-ROM` 作为启动设备。当系统从诊断 `ISO` 启动时，它将进入特殊的服务器恢复过程。
4.  从此时起，恢复过程将包含分步指导。例如，以下过程从备份映像 `backup.tar.bz2` 恢复数据库服务器。提示的答案以 _ 粗斜体 _ 显示：

```
Choose from following by typing letter in ’()’:
(e)nter interactive diagnostics shell. Must use credentials from Oracle support to login (reboot or power cycle to exit the shell),
(r)estore system from NFS backup archive,
Select: r
Are you sure (y/n) [n]: y
The backup file could be created either from LVM or non-LVM based compute node. Versions below 11.2.1.3.1 and 11.2.2.1.0 or higher do not support LVM based partitioning. Use LVM based scheme(y/n): y
Enter path to the backup file on the NFS server in format:
<ip_address_of_the_NFS_share>:/<path>/:<archive_file>
For example, 10.10.10.10:/export/:operating_system.tar.bz2
NFS line: 10.160.242.200:/export/:backup.tar.bz2
IP Address of this host: 10.160.242.170
Netmask of this host: 255.255.255.0
Default gateway: 10.160.242.1
```
5.  当所有上述信息输入后，Exadata 将继续通过网络挂载备份映像并恢复系统。恢复完成后，系统将提示你登录。使用 Oracle 文档中提供的密码以 `root` 身份登录。
6.  从 `ILOM` 分离诊断 `ISO`。
7.  使用 `reboot` 命令重新启动系统。此时，故障服务器应已完全恢复。

系统启动完成后，你可以使用 `imagehistory` 命令验证恢复情况。以下列表显示该映像作为 `restore from nfs backup` 创建并且已成功完成：

```
[enkdb01:oracle:EXDB1] /home/oracle
> su -
Password:
[enkdb01:root] /root
> imagehistory
Version                              : 11.2.1.2.3
Image activation date                : 2010-05-15 05:58:56 -0700
Imaging mode                         : fresh
Imaging status                       : success
...
Version                              : 11.2.2.2.0.101206.2
Image activation date                : 2010-12-17 11:51:53 -0600
Imaging mode                         : patch
Imaging status                       : success
Version                              : 11.2.2.2.0.101206.2
Image activation date                : 2010-01-23 15:23:05 -0600
Imaging mode                         : restore from nfs backup
Imaging status                       : success
```

一般来说，在定制 Exadata 数据库服务器时最好不要过于标新立异。Oracle 允许你为数据库服务器创建新的 `LVM` 分区并添加文件系统，但如果你这样做，你的恢复将需要一些额外的步骤。这些步骤并不特别困难，但如果你选择自定义 `LVM` 分区，请准备好将更改记录在某处，并熟悉 Oracle 文档中针对自定义系统的恢复过程。此外，Oracle 提供的脚本不会感知到文件系统布局的自定义更改。这可能会在运行这些脚本时导致意外结果——特别是 Exadata 文档中提供的 `LVM` 备份脚本。



#### 重建数据库服务器

如果必须更换或从头重建数据库服务器，且没有备份镜像可供恢复，则可以使用 Oracle 支持部门提供的安装镜像来创建一个镜像。这是一个漫长且高度复杂的过程，但我们会在此介绍主要步骤，以便您对其有个大致了解。

在重建服务器之前，必须先将其从 RAC 集群中移除。这是从任何 11gR2 或 12cR1 RAC 集群中删除节点的标准流程。首先，必须关闭并禁用故障服务器上的监听程序。然后，从 Oracle 清单中删除数据库二进制文件的 `ORACLE_HOME`。接着，停止 VIP 并将其从集群配置中删除，最后从集群中删除该节点。最后，从 Oracle 清单中删除网格基础设施的 `ORACLE_HOME`。

Oracle 软件交付云（原 e-Delivery）托管着一个 `computeImageMaker` 文件，该文件用于从一台幸存的数据库服务器创建安装镜像。这个镜像制作文件特定于您的 Exadata 系统的版本和平台，命名如下：

```
`computeImageMaker_` `{exadata_release}` `_LINUX.X64_` `{release_date}`. `{platform}` `.tar`
```

使用外部 USB 闪存驱动器在故障服务器上启动恢复镜像。USB 驱动器不需要很大，一个 2-4GB 的 U 盘即可。下一步是在机架中的另一台 Exadata 数据库服务器上，解压您从 Oracle 支持下载的镜像制作文件。存储单元的类似恢复过程会使用系统找到的第一个 USB 驱动器，因此在继续操作之前，您应该从系统中移除所有其他外部 USB 设备。要为恢复故障数据库服务器创建可启动的系统镜像，您需要运行 `makeImageMedia.sh` 脚本。当 `makeImageMedia.sh` 脚本完成后，您就可以在故障服务器上安装镜像了。将 USB 驱动器从完好的服务器上拔下，插入故障服务器。登录到故障服务器的 ILOM 并重启它。服务器启动时，会自动在外部 USB 驱动器上找到可启动的恢复镜像，并开始重建过程。此后，过程是自动化的。首先，它会检查服务器上的固件和 BIOS 版本，并根据需要将其更新到与其他数据库服务器匹配的版本。如果您重建的服务器原本就是 Exadata 系统的一部分，则不必期望此步骤会执行任何操作，但如果损坏的服务器已被新设备替换，则此步骤是必要的。一旦硬件组件更新完毕，将安装新的镜像。重建过程完成后，您可以拔下外部 USB 驱动器，并对服务器进行电源循环以启动新的系统镜像。

重建过程完成且数据库服务器重新上线后，它将被设置为出厂默认值。无论如何，您都应该将重建后的服务器视为一台全新的服务器。服务器将进入首次启动 (`firstboot`) 过程，期间会询问您完成安装所需的任何相关网络信息，包括主机名、IP 地址、DNS 和 NTP 服务器。操作系统配置完成后，您需要重新安装网格基础设施和数据库软件，并将该节点重新添加到集群中。这是一个记录完善的过程，许多 RAC DBA 称之为“添加节点”程序。如果您不熟悉此过程，请放心——它远没有您想象的那么令人生畏或耗时。一旦操作系统准备好进行安装，Oracle 安装程序会为您完成大部分繁重的工作。《Exadata 用户指南》出色地引导您完成此过程的每一步。

#### 恢复存储单元

存储单元恢复是一个非常广泛的主题。它可能像更换一个性能低下或故障的数据磁盘一样简单，也可能像应对主板芯片故障等导致的全系统故障一样复杂。在本节中，我们将讨论各种类型的单元恢复，包括物理磁盘的拆卸和更换，以及故障的闪存缓存模块。我们还将讨论如果整个存储单元死亡且必须更换时该怎么办。

#### 系统卷故障

回想一下，存储单元中的前两个磁盘包含 Linux 操作系统，通常被称为“系统卷”。Exadata 通过 Linux 操作系统使用软件镜像来保护这些卷。尽管如此，某些情况可能需要您从备份中恢复这些磁盘。执行单元恢复的一些原因如下：

*   系统卷（磁盘 1 和 2）同时故障。
*   启动分区损坏到无法修复的程度。
*   文件系统损坏。
*   补丁安装或升级失败。

如果您遇到以上任何一种情况，从备份恢复系统卷可能是必要的，或者至少是更便捷的方法。如前所述，Exadata 使用一个名为 `CELLBOOT` USB 闪存驱动器的 4GB 内部 USB 闪存驱动器，自动维护最后一个良好启动配置的备份。使用这个内部 USB 闪存磁盘恢复系统卷通常被称为存储单元救援程序。执行单元救援程序的步骤基本上涉及从内部 USB 驱动器启动，并按照提示选择您想要执行的救援类型。顺便提一下，由于 Exadata 配备了集成的远程管理模块 (`ILOM`)，您可以跨网络远程执行所有单元恢复操作，无需站在机架前从内部 USB 闪存磁盘执行完整的单元恢复。

注意

为了对 USB 恢复介质进行完整性检查，每个 Exadata 存储单元都被配置为使用 USB 设备作为其主要启动介质。单元将利用安装在 USB 恢复介质上的引导加载程序，该程序然后指回系统卷。如果 USB 恢复介质不可用，单元将回退到从系统卷启动，并生成一个关于 USB 恢复介质不存在或已损坏的警报。

本节并非存储单元恢复的分步指南，因此我们不会深入探讨从 `CELLBOOT` USB 闪存磁盘恢复的所有细节。应参考 Oracle 文档获取详细信息，但我们会探讨在开始此类恢复之前需要考虑的事项。

*   单元磁盘和网格磁盘：救援程序仅恢复 Linux 系统卷。单元磁盘及其内容不会被救援程序恢复。如果这些分区损坏，必须先删除再重新创建。一旦网格磁盘上线，就可以将它们添加回 ASM 磁盘组，随后的重平衡操作将恢复数据。
*   `ASM` 冗余：从 USB 备份恢复存储单元可能会导致系统卷上的所有数据丢失。这包括这些磁盘驱动器上网格磁盘中的数据库数据。如果您的 `ASM` 磁盘组使用普通冗余，我们强烈建议在执行 USB 磁盘单元恢复之前进行数据库备份。使用 `ASM` 高冗余时，您总共有三份数据副本，因此无需进行数据库备份即可安全地执行单元恢复。即便如此，如果可能，我们仍然建议进行备份。恢复过程不会破坏数据卷（单元/网格磁盘），除非您在救援程序提示时明确选择这样做。
*   软件和补丁：救援程序将使单元恢复到执行备份时的状态，包括补丁。恢复中还包括网络设置以及 `root`、`celladmin` 和 `cellmonitor` 帐户的 `SSH` 密钥。




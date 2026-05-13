# 升级 Exadata 计算节点

本练习演示如何将 Exadata 计算节点升级到版本 12.1.1.1.1。

从 MOS 笔记 #888828.1 中找到并下载最新版本的 `dbnodeupdate.sh` 实用程序和所需的补丁 ISO。在此示例中，`dbnodeupdate.sh` 可作为补丁号 16486998 获取，而 ISO 文件位于补丁号 18889969 中。在此示例中，两个文件都下载到 `/u01/stage/patches` 目录。解压包含 `dbnodeupdate.sh` 的补丁文件。

```
# cd /u01/stage/patches
# unzip p16486998_121111_Linux-x86-64.zip
```

运行 `dbnodeupdate.sh` 脚本，指定补丁文件位置。

```
# ./dbnodeupdate.sh –u –l /u01/stage/patches/p18889969_121111_Linux-x86-64.zip -s
```

计算节点重启且 `imageinfo` 显示成功状态后，使用 `dbnodeupdate.sh` 完成升级。

```
# imageinfo
Kernel version: 2.6.39-400.128.17.el5uek #1 SMP Tue May 27 13:20:24 PDT 2014 x86_64
Image version: 12.1.1.1.1.140712
Image activated: 2014-12-08 04:42:43 -0500
Image status: success
System partition on device: /dev/mapper/VGExaDb-LVDbSys1

# cd /u01/stage/patches
# ./dbnodeupdate.sh –c
```

这些步骤可以逐节点执行以进行滚动补丁，也可以在每个节点上并行执行以进行完全停机更新。




#### 使用 `dbnodeupdate.sh` 回滚补丁

回滚计算节点的过程与升级过程非常相似。调用 `dbnodeupdate.sh` 脚本并使用 `-r` 标志即可回滚到计算节点操作系统的先前版本。此操作将更改非活动逻辑卷的文件系统标签（即运行更新时创建的备份），并重新配置 GRUB 引导加载程序以使用此卷。主机将重新启动，当它恢复运行时，将处于补丁发布之前的状态。因此，在决定回滚之前，务必避免在主机上发生过多变更。任何密码或配置设置都将与补丁发布时的系统相匹配。

### 升级 InfiniBand 交换机

虽然更新发布很少，但 Exadata 环境中的 InfiniBand 交换机必须定期更新。这些更新一直如此罕见，以至于几乎每次发布都包含了不同的安装方法。Oracle 似乎已标准化将交换机固件与 Exadata 存储服务器补丁版本捆绑在一起，并使用 `patchmgr` 实用程序来应用更新。这是合理的，因为 InfiniBand 交换机补丁通常与存储服务器版本同时发布。通过使用 `patchmgr`，语法将保持熟悉，方法也将趋于稳定。

与存储服务器补丁一样，升级 InfiniBand 交换机是一个两步过程。首先进行先决条件检查，然后是交换机的实际升级。虽然步骤相同，但语法略有不同。存储服务器补丁使用 `-patch_check_prereq` 标志，而 InfiniBand 补丁使用 `-ibswitch_precheck` 标志。存储服务器上的补丁使用 `-patch` 标志应用，而 `-upgrade` 用于将更新应用到 InfiniBand 交换机。`patchmgr` 脚本仍然使用一个包含 InfiniBand 交换机名称的文件，就像修补存储服务器时一样。InfiniBand 交换机更新总是以滚动方式应用，一次一个交换机。使用 `patchmgr` 应用这些补丁时，会在每个交换机修补前后执行验证测试。这些补丁可以在 Clusterware 堆栈联机时应用，无需系统范围内的停机。

#### 升级 Exadata InfiniBand 交换机

本练习演示将 Exadata InfiniBand 交换机升级到包含在 Exadata 存储服务器版本 `12.1.1.1.1` 中的 `2.1.3-4` 版本。

在用于应用 Exadata 存储服务器补丁的同一节点上，收集 InfiniBand 交换机的名称。请注意，在包含其他 Oracle 工程系统的环境中，并非所有 InfiniBand 交换机都需要升级。（`Exalogic` 和 `Big Data Appliance` 包含使用不同固件的“网关”交换机。）寻找标记为 "SUN DCS 36P" 的交换机，这些是需要修补的交换机。

```
[root@enkx3db01 ~]# ibswitches
Switch : 0x002128ac ports 36 "SUN DCS 36P QDR enkx3sw-ib3 x.x.x.x" enhanced port 0 lid 1 lmc 0
Switch : 0x002128ab ports 36 "SUN DCS 36P QDR enkx3sw-ib2 x.x.x.x" enhanced port 0 lid 2 lmc 0
```

创建一个名为 `ibswitches.lst` 的文本文件，其中包含要修补的 InfiniBand 交换机的名称。

```
# cd /u01/stage/patches/patch_12.1.1.1.1.140712
# cat ibswitches.lst
enkx3sw-ib2
enkx3sw-ib3
```

运行带有 `-ibswitch_precheck` 标志的 `patchmgr` 以确保一切已准备就绪。`patchmgr` 脚本将要求输入所有交换机的 root 密码，并验证连接性以及其他测试（可用空间、`verify-topology` 输出等）。如果脚本成功返回，即可准备应用补丁。

```
# cd /u01/stage/patches/patch_12.1.1.1.1.140712
# ./patchmgr –ibswitches ibswitches.lst –upgrade –ibswitch_precheck
```

使用 `patchmgr` 脚本应用补丁。每个交换机将按顺序修补，就像 `patchmgr` 执行滚动 Exadata 存储服务器补丁一样。在所有交换机修补完成或发生错误之前，该脚本不会返回到提示符。

```
# cd /u01/stage/patches/patch_12.1.1.1.1.140712
# ./patchmgr –ibswitches ibswitches.lst –upgrade
```

与 Exadata 存储服务器补丁不同，InfiniBand 交换机上不需要任何清理操作。补丁完成后，即可进入流程的下一个环节。

### 将补丁应用于备用系统

回顾本章描述的所有补丁，很容易会想，“我永远无法获批停工来应用所有这些补丁！”Oracle 为了减少与补丁相关的实际停机时间，提供了一项功能：自数据库版本 `11.2.0.2` 以来，每个 Exadata 补丁都支持“备用优先”打补丁。使用此方法，唯一的数据库停机时间就是执行 Data Guard 切换到备用系统所需的时间。只需提前将所有补丁应用到备用系统，将数据库切换到新修补的系统，然后修补原始的主系统。此时，您可以选择切换回原始主系统，或者继续以角色反转的状态运行。在下一个补丁周期，只需重复此过程。使用此方法可以展示审计员喜欢的两点——成功的灾难恢复测试和最新的数据库。在“备用优先”模式下应用补丁的步骤非常简单：

1.  通过滚动或全停机方法将季度补丁应用于备用系统。补丁可包括 Exadata 存储服务器、计算节点、QDPE 和 InfiniBand 交换机升级。也可以执行 Grid Infrastructure 补丁集升级，因为它们不影响 Oracle Data Guard。不要在备用数据库上运行修补后脚本（`catbundle.sql` 或 `datapatch`），因为这些脚本需要在所有 Oracle 主目录修补完成后运行。
2.  使用 Active Data Guard、快照备用数据库或其他方式对备用数据库执行测试。
3.  对主集群上的所有数据库执行 Data Guard 角色切换到辅助集群。
4.  通过滚动或全停机方法将季度补丁应用于主系统。这些补丁应与步骤 1 中应用于备用系统的补丁相同。
5.  按照补丁自述文件中的说明，在主数据库实例上执行修补后步骤（`catbundle.sql` 或 `datapatch`）。
6.  如果需要，将数据库切换回原始主集群。

唯一不能使用“备用优先”方法应用的补丁类型是数据库补丁集升级。这些升级涉及更改版本号（从 `11.2.0.3` 到 `11.2.0.4`，或从任何版本升级到 Oracle 数据库 `12c`）。这些升级无法通过“备用优先”方法完成，因为它们需要升级数据库目录。这不是 Exadata 的限制，而是 Oracle 数据库本身的限制。从高层次来看，在 Data Guard 配置中升级数据库需要在具有新版本的主目录中启动备用数据库，并从主库运行升级脚本。这适用于主要版本内的补丁集升级（`11.2.0.3` 到 `11.2.0.4` 或 `12.1.0.1` 到 `12.1.0.2`），或主要版本之间的升级（例如，`11.2.0.3` 到 `12.1.0.2`）。数据库内的组件被升级，更改通过 Data Guard 传播。有关此过程的更多详细信息，请参阅 附录 B 中提到的 MOS 说明。它们实际上非常详尽，并系统地逐步讲解了整个过程。





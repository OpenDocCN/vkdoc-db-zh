# 存储重新配置

![image](img/frontdot.jpg)

默认情况下，Exadata 系统会在 `+DATA_MYEXA1` 和 `+RECO_MYEXA1` 磁盘组之间预先分配大部分物理存储，并且采用 80/20 的拆分比例。在购买的预安装阶段，Oracle 会向 DBA、网络管理员和系统管理员提供一份工作表。每个小组为配置过程提供信息，其中 DBA 和网络管理员提供大部分数据。工作表完成后，会进行讨论以澄清任何模糊的设置，并确定存储将如何划分。可能并不明显需要调整默认设置，因此系统按标准的 80/20 分配进行配置。还会提供第三个磁盘组 `DBFS_DG`，其空间约为 32GB，且此磁盘组的大小不会改变。

在配置完成后，可以更改这些分配，但这是一个破坏性的过程，需要仔细考虑。我们曾在一台将用于生产的 Exadata 系统上执行过此操作，但那是在任何生产数据库创建之前很久进行的。该操作也是在 Oracle 高级客户支持的协助下完成的，尽管我们会提供执行此重新配置的步骤，但我们强烈建议您让 Oracle 高级客户支持参与此过程。我们将讨论实际步骤，以便您理解从头到尾的整个过程。本章的目标是提供有关存储重新配置过程的实用知识。如果您对执行此类操作感到不太放心，最好的方法可能是安排 Oracle 高级客户支持在需要时为您执行此操作。即使您不亲自执行，了解将运行哪些任务以及最终结果会是什么样也会很有帮助。考虑到这一点，我们将继续下去。

## 我希望我们能有更多...

在配置完成后，但在创建任何数据库（除了安装时提供的默认数据库 DBM）之前，您可能会发现，例如，您希望或需要比最初分配更多的恢复存储。或者，您可能不使用恢复区域（不太可能），并希望使数据磁盘组稍大一些，作为另一个例子。有一种方法可以在创建任何重要内容之前完成此操作。我们再次声明，重新配置存储分配是一个破坏性的过程，需要在执行前很久就进行规划。我们将演示一个在 `+DATA_MYEXA1` 和 `+RECO_MYEXA1` 磁盘组之间更均匀分配存储的过程，再次提醒，**适当的规划绝对必要**，并且应让 Oracle 高级客户支持参与。

## 重新分配资源

Exadata 系统中的每个磁盘在逻辑上划分为有时称为分区的区域，这些区域包含在配置时确定的固定存储量。查看标准 Exadata 安装上单个磁盘的配置方式，以下输出显示了各种分区的分配情况：

```
CellCLI> list griddisk attributes name, size, offset where name like '.*CD_11.*'
         DATA_MYEXA1_CD_11_myexa1cel05   2208G           32M
         DBFS_DG_CD_11_myexa1cel05       33.796875G      2760.15625G
         RECO_MYEXA1_CD_11_myexa1cel05   552.109375G     2208.046875G

CellCLI>
```

![image](img/sq.jpg) **注意** 对于 UNIX/Linux 系统，slice 和 partition 这两个术语在某些资料中互换使用。两者都用于指示物理磁盘的逻辑划分。在本文中，我们将使用术语 **partition**，因为它是最准确的术语。

请注意，对于给定的磁盘，有三个 ASM 可以看到并使用的分区。还要注意，对于给定的分区，其大小加上偏移量给出了下一个分区的起始“点”。不幸的是，CellCLI 的输出无法按照您希望的方式排序。在此示例中，默认排序方式是按名称。

从提供的输出看，第一个分区被分配为 `+DATA_MYEXA1` 存储分区，从存储的前 32MB 之后开始。那 32MB 保留给操作系统进行磁盘管理。下一个可供 ASM 使用的分区被分配给 `+RECO_MYEXA1` 磁盘组，第三个分区被分配给 `+DBFS_DG` 磁盘组。磁盘组的大小取决于您选择的存储类型以及您订购的 Exadata 机架大小。本章图示的系统使用高容量磁盘，每块磁盘提供 3TB 存储。高速磁盘每块提供 600GB 存储。配置过程根据 `/opt/oracle.SupportTools/onecommand/onecommand.param` 文件中名为 `SizeArr` 的参数，为数据和恢复磁盘组配置分区。两个系统都采用 `+DATA_MYEXA1` 和 `+RECO_MYEXA1` 之间的默认存储划分进行配置。这导致这两个磁盘组之间采用 80/20 的分配。如前所述，提供的输出是针对标准 Exadata 配置的。

要改变 ASM 看到存储的方式，第一步将是删除原始配置，以便可以在物理磁盘上创建新的分区。下一个基本步骤是创建新分区，然后配置 ASM 磁盘组。由于第一步是删除所有已配置的分区，因此当 Exadata 系统上已创建开发、测试或生产数据库时，**不建议**执行此过程。我们再次声明，此过程是**破坏性的**，在删除原始逻辑磁盘配置的初始步骤完成后，**将没有任何数据库保留**。GRID 基础架构也会被删除；它将作为定义新逻辑存储分区的 26 个步骤之一被重新创建。



### 准备

不仅逻辑存储区域被重新定义，“Oracle”操作系统的用户主目录也会被重新创建，因此在执行此过程之前，必须保留对该位置所做的任何更改。这需要在您的 Exadata 系统中所有可用的数据库服务器上执行，即使您让 Oracle 高级客户支持执行实际的存储工作也是如此。因此，执行的第一步是使用 `tar`（最初是 *Tape ARchiver* 的首字母缩写，因为在旧时代，唯一可用的介质是磁带）或 `cpio`（*CoPy In/Out* 的首字母缩写，一个相对于 `tar` “较新”的实用程序，设计为不仅可以在磁带上操作，还可以在磁盘驱动器等块设备上操作）来保留“Oracle”用户主目录。这两个实用程序在操作系统级别提供。如果您选择 `tar`，以下命令将保留“Oracle”操作系统用户的主目录：

```
$ tar cvf /tmp/oraclehome.tar ./*
```

如果您不熟悉 `tar` 命令，提供的选项执行以下操作：

```
c -- 创建归档文件
v -- 使用详细模式。这将显示所有处理的目录和文件及其关联路径
f -- 用于归档的文件名
```

使用 `./*` 语法指定要归档的文件，可以更轻松地将文件恢复到您选择的任何基本目录。“Oracle”操作系统用户主目录不太可能更改名称。了解此语法会很有帮助，如果您想将完整的目录树复制到另一个位置，这是我们通常调用 `tar` 来复制文件和目录的方式。

使用 `tar` 不是强制性的；您也可以使用 `cpio`。如果您更习惯使用 `cpio` 实用程序，以下命令将归档“Oracle”操作系统用户主目录：

```
$ find . -depth -print | cpio -ov > /tmp/oraclehome.cpio
```

提供的选项执行以下操作：

```
o -- 复制输出到列出的归档文件
v -- 详细输出，列出处理的文件和路径
```

我们建议在开始任何存储重新配置之前保留“Oracle”用户主目录，因为它将被重新配置过程覆盖。在所有可用的数据库服务器上保留现有的“Oracle”操作系统用户主目录，因为所有“Oracle”操作系统用户主目录都将被重建。此外，“Oracle”操作系统用户密码将被重置为其原始值，该值在交付、安装和初始配置之前填写的 Exadata 工作表中列出。

用于 Exadata 配置的文件仅允许您指定数据磁盘组分区的大小。恢复磁盘组分区的大小源自为调整数据磁盘组大小而提供的值。DBFS_DG 的大小不受更改此值的影响。在删除原始存储配置之前，无法修改此文件。

![image](img/sq.jpg) **注意** 列出的存储重新配置命令必须以“root”用户身份运行。由于此用户在操作系统级别拥有无限权力，因此在执行这些命令时必须格外谨慎，这一点再怎么强调也不为过。如果您不习惯使用“root”帐户，最好与 Oracle 高级客户支持安排，将存储重新配置到您所需的级别。

您应该对存储重新配置在 +DATA_MYEXA1 磁盘组和 +RECO_MYEXA1 磁盘组大小方面希望提供的内容有很好的了解，并且您应该已经使用 `tar` 或 `cpio` 保留了“Oracle”操作系统用户主目录。我们现在继续进行实际的存储重新配置过程。

此时的其他任务包括了解安装了哪些 OEM 代理，并记录 SCAN 监听器是否使用默认端口。如有必要，重新安装 OEM 代理和为非默认端口重新配置 SCAN 监听器，将不得不在恢复步骤中作为存储重新配置完成和“Oracle”操作系统用户主目录恢复后的一部分来完成。本章将不涵盖重新安装 OEM 代理和重新配置 SCAN 监听器的步骤，但了解可能需要执行这些操作是好的。

### 执行

您应该充分准备好开始实际的重新配置过程。第一步是删除原始存储配置，包括存储单元的配置。让我们开始。

## 步骤 1

以“root”用户身份连接到数据库服务器 1。您应该在 shell 提示符中看到 `#`，表示登录/su 过程成功。您现在必须将目录更改为 `/opt/oracle.SupportTools/onecommand`。这是完成此操作所需的脚本和配置文件所在的位置。了解您可以执行哪些命令来进行此重新配置是一个好主意，该信息可以通过使用列出的参数执行以下脚本来找到：

```
./deploy11203.sh -a -l
```

将显示以下列表：

```
Step  0 = CleanworkTmp
Step  1 = DropUsersGroups
Step  2 = CreateUsers
Step  3 = ValidateGridDiskSizes
Step  4 = CreateOCRVoteGD
Step  5 = TestGetErrNode
Step  6 = DeinstallGI
Step  7 = DropCellDisk
Step  8 = DropUsersGroups
Step  9 = CheckCssd
Step 10 = testfn
Step 11 = FixAsmAudit
Step 12 = DropExpCellDisk
Step 13 = testHash
Step 14 = DeconfigGI
Step 15 = GetVIPInterface
Step 16 = DeleteDB
Step 17 = ApplyBP3
Step 18 = CopyCrspatchpm
Step 19 = SetupMultipath
Step 20 = RemovePartitions
Step 21 = SetupPartitions
Step 22 = ResetMultipath
Step 23 = DoT4Copy
Step 24 = DoAlltest
Step 25 = ApplyBPtoGridHome
Step 26 = Apply112CRSBPToRDBMS
Step 27 = RelinkRDSGI
Step 28 = RunConfigAssistV2
Step 29 = NewCreateCellCommands
Step 30 = GetPrivIpArray
Step 31 = SetASMDefaults
Step 32 = GetVIPInterface
Step 33 = Mychknode
Step 34 = UpdateOPatch
Step 35 = GetGroupHash
Step 36 = HugePages
Step 37 = ResetPermissions
Step 38 = CreateGroups
Step 39 = FixCcompiler
Step 40 = CollectLogFiles
Step 41 = OcrVoteRelocate
Step 42 = OcrVoteRelocate
Step 43 = FixOSWonCells
Step 44 = Dbcanew
Step 45 = testgetcl
Step 46 = setupASR
Step 47 = CrossCheckCellConf
Step 48 = SetupCoreControl
Step 49 = dropExtraCellDisks
Step 50 = CreateCellCommands
```

对于此重新配置步骤，有三个感兴趣的命令：

```
Step  6 = DeinstallGI
Step  7 = DropCellDisk
Step  8 = DropUsersGroups
```

这些是将执行以“清除”现有存储配置并为下一步做准备的命令。每个命令都从 `deploy11203.sh` 脚本单独运行。您必须按列出的顺序运行这些步骤，以确保完全删除现有配置。实际的命令如下：

```
./deploy11203.sh -a -s 6
./deploy11203.sh -a -s 7
./deploy11203.sh -a -s 8
```

查看 `V$ASM_DISKGROUP` 的输出，您可以看到原始的存储分配。以下输出说明了这一点：

```
SQL> select name, free_mb, usable_file_mb
  2  from v$asm_diskgroup
  3  where name like '%MYEXA%';

NAME                           FREE_MB USABLE_FILE_MB
------------------------------  --------- --------------
DATA_MYEXA1                   81050256       26959176
RECO_MYEXA1                   20324596        6770138

SQL>
```

您将看到，当此过程完成时，所示查询的输出将报告不同的值，这些值是新存储设置的结果。您应该在过程开始前和过程结束后运行此查询，以记录原始存储和新配置的存储。

考虑到这些注意事项，我们继续进行重新配置存储所需的步骤。



### 概述

在八分之一机架或四分之一机架配置上运行这些步骤大约需要一个小时，输出将显示在屏幕上，描述待处理和已完成的任务。我们建议您运行每个命令并观察输出。这些流程不应产生任何错误；然而，我们认为通过观察进度来确保安全是最佳做法。

这些步骤的作用是什么？第 6 步移除了 GRID 基础设施，这基本上是 ASM（以及 RAC，但我们在此操作中关心的是它与 ASM 的链接）的核心。一旦该基础设施消失，我们就可以自由运行第 7 步来删除已创建的网格磁盘，然后删除 ASM 用于识别和访问存储的分区定义。最后是删除访问这些分区和网格磁盘的用户和组。这里会删除“oracle”用户账户及其操作系统主目录。请注意，只需 3 个步骤即可删除现有的存储配置，但需要 26 个步骤才能重新创建它。

#### Act 2

一旦这些操作完成，就该修改存储阵列参数以提供所需的存储划分。要编辑的配置文件是`/opt/oracle.SupportTools/onecommand/onecommand.params`。我们使用`vi`来编辑此类文件，但您可能希望改用 emacs。使用您习惯的任何编辑器。要更改的参数值名为`SizeArr`。对于标准默认安装，此行应如下所示：

```
SizeArr=2208G
```

了解使用什么值来获得所需的存储划分是很有用的。表 11-1 显示了几个`SizeArr`值以及创建的大约存储划分。

##### 表 11-1. 某些 SizeArr 值及生成的存储分配

| SizeArr 设置 | 数据磁盘组百分比 | 恢复磁盘组百分比 |
| --- | --- | --- |
| 2208 | 80 | 20 |
| 1932 | 70 | 30 |
| 1656 | 60 | 40 |
| 1487 | 55 | 45 |
| 1380 | 50 | 50 |

请记住，这些百分比是基于默认 80/20 存储分配的近似值。我们建议您使用标准 shell 脚本注释字符`#`来注释该行，以便保留原始设置。您现在可以复制注释的行，或者在其正下方打开一个新行并提供以下文本：

```
SizeArr=<您选择的值>G
```
例如，如果您希望对可用存储配置 55/45 的划分，基于表 11-1 中的信息，新行将如下所示：

```
SizeArr=1487G
```
编辑完成后，应该有两行列出了`SizeArr`，如下所示：

```
###### SizeArr=2208G
SizeArr=1487G
```
这是您需要对此参数文件进行的唯一更改，因此请保存您的工作并退出编辑器。您现在已准备好建立新的存储配置。

#### Act 3

与之前的操作一样，将使用`deploy11203.sh`脚本，尽管步骤数量更多。以下是此过程此部分可用的完整步骤列表：

```
Step  0 = ValidateEnv
Step  1 = CreateWorkDir
Step  2 = UnzipFiles
Step  3 = setupSSHroot
Step  4 = UpdateEtcHosts
Step  5 = CreateCellipinitora
Step  6 = ValidateIB
Step  7 = ValidateCell
Step  8 = PingRdsCheck
Step  9 = RunCalibrate
Step 10 = CreateUsers
Step 11 = SetupSSHusers
Step 12 = CreateGridDisks
Step 13 = GridSwInstall
Step 14 = PatchGridHome
Step 15 = RelinkRDSGI
Step 16 = GridRootScripts
Step 17 = DbSwInstall
Step 18 = PatchDBHomes
Step 19 = CreateASMDiskgroups
Step 20 = DbcaDB
Step 21 = DoUnlock
Step 22 = RelinkRDSDb
Step 23 = LockUpGI
Step 24 = ApplySecurityFixes
Step 25 = setupASR
Step 26 = SetupCellEmailAlerts
Step 27 = ResecureMachine
```
与第一个列表不同，这里只显示了 28 个步骤。由于这是一个现有的 Exadata 安装，验证环境的第 0 步和保护机器的第 27 步将不会运行。剩下的 26 步将用于重新配置存储和存储单元。以下命令将用于完成此过程的存储重新配置部分：

```
./deploy11203.sh -r 1-26
```
`-r`选项按顺序执行提供的步骤，只要没有步骤产生错误。如果某个步骤确实产生错误，它将停在该步骤；应采取任何纠正措施，然后使用修改后的步骤列表重新启动`deploy11203.sh`脚本。例如，如果执行因错误停在第 21 步，并且错误原因已纠正，以下命令将从生成错误的点重新启动重新配置：

```
./deploy11203.sh -r 21-26
```
如果在执行中途出现问题，则无需从头开始整个过程。

对于八分之一机架或四分之一机架 Exadata 配置，最后执行的`deploy11203.sh`脚本将运行一个半到两个小时。对于更大的 Exadata 配置，这将运行更长时间，因为磁盘数量更多。脚本将为每个执行的步骤提供反馈。

在“干净”的系统上——即仍处于原始配置且没有任何新数据库的系统——我们在重新配置存储时没有遇到任何错误或困难。这是一个耗时的过程，但它应该可以毫无问题地执行。

![image](img/sq.jpg) **注意** 我们在新配置的 Exadata 系统上执行过此操作，并且运行没有困难。如果您计划在已创建其他数据库、存在其他`ORACLE_HOME`位置的系统上执行此操作，请注意此处提供的步骤可能会遇到与这些更改相关的问题。包含的数据库（通常名为`DBM`）将在重新配置过程中从脚本重新创建。请注意，恢复在初始配置完成后创建的数据库可能会因为重新分配的存储而导致错误，特别是如果`+DATA_MYEXA1`磁盘组已变小。在这种情况下，最好联系 Oracle 高级客户支持。

##### 步骤详解

让我们看看每个步骤在基础层面上完成了什么。步骤 1 和 2 是基本的“内务处理”操作，创建临时工作目录并将软件档案解压缩到该临时工作区。步骤 3 开始了此操作的重要方面，它为“root”账户设置用户等效性（意味着无密码的`ssh`访问）到所有服务器和交换机。这是必需的，以便后续步骤无需询问密码即可继续。步骤 4 更新了`/etc/hosts`文件，其中包含 Exadata 系统的预配置存储单元 IP 地址和服务器名称。这使得通过使用服务器名称而不是 IP 地址来连接系统中的其他服务器更加容易。

下一步，步骤 5，在可用的存储单元上创建`cellinit.ora`文件。`cellinit.ora`包含的内容示例（实际 IP 地址和端口号已遮蔽）如下。

```
#CELL Initialization Parameters
version=0.0
HTTP_PORT=####
bbuChargeThreshold=800
SSL_PORT=#####
RMI_PORT=#####
ipaddress1=###.###.###.###/##
bbuTempThreshold=60
DEPLOYED=TRUE
JMS_PORT=###
BMC_SNMP_PORT=###
```
此步骤还在可用的数据库服务器上创建`cellip.ora`文件。示例内容如下。

```
cell="XXX.XXX.XXX.3"
cell="XXX.XXX.XXX.4"
cell="XXX.XXX.XXX.5"
cell="XXX.XXX.XXX.6"
cell="XXX.XXX.XXX.7"
```
此文件告知 ASM 集群中可用的存储单元。



步骤 6、7 和 8 执行必要的 InfiniBand 连通性、存储单元配置以及数据库服务器与存储单元之间的 InfiniBand 进程间通信（`IPC`）验证。步骤 8 通过检查`oracle`可执行文件的链接方式来执行此验证。`InfiniBand`上的`IPC`是通过使用`ipc_rds`选项链接`oracle`内核来设置的，如果缺少此选项，`IPC`将通过非`InfiniBand`网络层进行。不幸的是，大多数重新链接内核的数据库补丁并不包含此参数，我猜测是因为它们主要面向非 Exadata 系统。因此，为`oracle`软件打补丁可能导致此连接性丢失，进而因使用速度慢得多的网络连接进行`IPC`而引发性能问题。通过使用`ipc_rds`选项重新链接`oracle`内核可以轻松解决此问题。确保这一点的一种方法是编辑打补丁集中的脚本，该脚本用于重新链接`oracle`内核。另一种选择是在将数据库发布给用户之前，简单地使用`ipc_rds`选项再执行一次重新链接。从数据库服务器命令行复制而来的具体语句如下：

```
$ make -f $ORACLE_HOME/rdbms/lib/ins_rdbms.mk ipc_rds
```

这也是`exachk`脚本执行的一项检查，该报告的输出会列出那些未链接以提供`InfiniBand`上`RDS`连接性的`oracle`内核。

步骤 9 校准存储，这是一个非常重要的步骤。每个可用的存储单元都会运行此操作。首先，关闭`CELLSRV`，然后运行`calibrate`过程。将生成类似于以下内容的输出：

```
CellCLI> calibrate;
Calibration will take a few minutes...
Aggregate random read throughput across all hard disk luns: 137 MBPS
Aggregate random read throughput across all flash disk luns: 2811.35 MBPS
Aggregate random read IOs per second (IOPS) across all hard disk luns: 1152
Aggregate random read IOs per second (IOPS) across all flash disk luns: 143248
Controller read throughput: 5477.08 MBPS
Calibrating hard disks (read only) ...
Calibrating flash disks (read only, note that writes will be significantly slower) ...
...
```

一旦所有存储单元的`calibrate`步骤完成，就可以创建单元磁盘和网格磁盘，即步骤 12。在此之前，步骤 10 和 11 会创建`oracle`操作系统用户，并建立关联的免密`ssh`访问。这些步骤完成后，在步骤 12 中创建网格磁盘分区，以便`ASM`可以识别它们。这是您的新设置生效的地方，并建立了`+DATA_MYEXA1`和`+RECO_MYEXA1`之间所需的比例。

接下来是设置`GRID`主目录，通过安装`GRID`软件（步骤 13）、修补`GRID`主目录（步骤 14）、重新链接`GRID` `oracle`可执行文件以提供`InfiniBand`上的`IPC`（步骤 15）以及运行`GRID`的`root.sh`脚本（步骤 16）。除了步骤 15，这些操作与您在任何其他`GRID`安装中执行的操作相同。安装和配置`GRID`软件后，安装并修补数据库软件（步骤 17 和 18）。同样，这些步骤是您在安装和修补`Oracle`数据库软件时，在任何系统上都会执行的步骤。

由于网格磁盘现已创建，让`ASM`识别它们并在步骤 19 中创建`+DATA_MYEXA1`、`+RECO_MYEXA1`和`+DBFS_DG`磁盘组就是一项简单的任务。这为在步骤 20 中创建`DBM`集群数据库铺平了道路，这是`裸机`Exadata 安装附带的默认数据库。可以在配置工作表中更改此数据库名称，但我们尚未见过任何安装中`DBM`不是可用数据库的情况。创建`DBM`数据库后，会解锁选定的账户（步骤 21），并执行`DBM` `oracle`可执行文件的最终重新链接，以确保`InfiniBand`上的`IPC`功能正常（步骤 22）。步骤 23 中`GRID`基础设施被“锁定”以防止任何配置更改，然后，在步骤 24 中，任何安全修复/补丁都会应用于数据库主目录。

步骤 25 可能会也可能不会执行，这取决于您的企业是否决定允许`Oracle`通过自动服务请求（`ASR`）服务器直接访问 Exadata。这需要配置一个单独的服务器与 Exadata 通信以监控系统，并将发现的任何问题自动报告回`Oracle`，自动生成服务请求来解决问题。此类问题可能是实际或即将发生的硬件故障（磁盘驱动器、Sun PCIe 卡、内存）。服务请求会生成零件订单并派遣服务技术员安装故障或即将故障的零件。如果您没有为此配置服务器，该步骤将运行然后成功终止，不执行任何操作。如果您确实需要此服务并已安装所需的额外服务器，此步骤会使用提供的 My `Oracle` Support 凭据配置服务，并进行测试以确保凭据有效。

步骤 26 通过电子邮件设置任何所需的单元警报。它需要一个有效的电子邮件地址，该地址会被使用和验证。将单元警报通过电子邮件发送给您很方便；您无需连接到各个单元来列出警报历史记录，因此节省时间。这也是一个可选步骤；您不必配置将单元警报通过电子邮件发送给您，但建议这样做。

至此，您的 Exadata 系统已恢复到安装时的状态，但您修改的存储百分比除外。现在是恢复在此过程开始之前拥有的`oracle`操作系统用户主目录的时候了。

### 恢复

既然存储已经重新配置，现在该解决重新创建的`oracle`操作系统用户主目录和重置的`oracle`操作系统密码了。请记住，在清除旧存储配置的最后一步中删除了`oracle`操作系统用户。假设您仍然以`root`身份连接，那么在每个数据库服务器的 shell 提示符下，您将键入以下内容：

```
# passwd oracle
```

这将提示您输入新的`oracle`操作系统用户密码，需要输入两次进行确认。由于已为`root`账户配置了免密`ssh`访问，您可以从节点 1 上的“基地”使用`ssh`连接到所有其他数据库服务器。对于不熟悉`ssh`的用户，以下命令将连接到数据库服务器 2：

```
# ssh myexa1db02
```

更改目标服务器名称以连接到其余的数据库服务器。

现在该恢复之前的`oracle`操作系统用户主目录文件和配置了。使用`su`以`oracle`身份连接。这也会将您置于用户的主目录中。如果您使用`tar`来归档目录及其内容，以下命令将恢复归档的项目：

```
$ tar xvf /tmp/oraclehome.tar . | tee oraclehome_restore.log
```




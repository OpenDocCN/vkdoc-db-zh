# 运行备份

在运行 RMAN 备份之前，请务必阅读第 17 章，了解如何为生产环境配置 RMAN 设置。对于生产数据库，应从 Oracle Cloud Control 或类似于第 17 章末尾所示的 shell 脚本来运行 RMAN。在 shell 脚本中，为特定数据库配置 RMAN 的每个使用方面是很有用的。如果你使用其默认设置开箱即用地运行 RMAN，你将能够备份你的数据库。然而，对于大多数生产数据库应用来说，这些设置是不够的。

## 备份整个数据库

如果你不确定 RMAN 会将数据库文件备份到哪里，你需要阅读第 17 章，因为它描述了如何配置 RMAN 以在你选择的位置创建备份文件。要配置 RMAN 写入磁盘上的特定位置（请注意，`CONFIGURE` 命令必须在运行 `BACKUP` 命令之前执行）：

```
RMAN> configure channel 1 device type disk format '/u01/O18C/rman/rman1_%U.bk';
```

配置好备份位置后，使用类似于下面显示的命令来备份整个数据库：

```
RMAN> backup incremental level=0 database plus archivelog;
```

此命令确保 RMAN 将备份数据库中的所有数据文件、备份前生成的所有可用归档日志，以及备份期间生成的所有归档日志。此命令还确保你拥有恢复和修复数据库所需的所有数据文件和归档日志。

如果你启用了控制文件的自动备份功能，请接着运行此命令：

```
RMAN> configure controlfile autobackup on;
```

RMAN 作为备份一部分执行的最后一项任务是生成一个包含控制文件备份的备份集。此控制文件将包含有关已发生的备份以及备份期间生成的任何归档日志的所有信息。

**提示**
始终启用控制文件的自动备份功能。

RMAN 的 `BACKUP` 命令有许多细微差别。对于生产数据库，通常建议使用 `BACKUP INCREMENTAL LEVEL=0 DATABASE PLUS ARCHIVELOG` 命令备份数据库。这通常是足够的。然而，你会遇到许多情况，需要运行使用特定 RMAN 功能的备份，或者你可能需要排查一个要求你了解调用 RMAN 备份的其他方法的问题。这些方面将在接下来的几个部分中讨论。

## 完整备份 vs. 增量级别=0

术语 *RMAN full backup*（RMAN 完整备份）有时会引起混淆。更贴切地描述这个任务的方式是 *RMAN 备份一个或多个数据文件中所有已修改的块*。术语 *full*（完整）并不意味着所有块都被备份或所有数据文件都被备份。它仅仅意味着备份了重建数据文件（在发生故障时）所需的所有块。你可以对单个数据文件进行完整备份，而该备份片的内容可能比数据文件本身小得多。

## 备份集 vs. 镜像副本

RMAN 的默认备份模式指示其仅备份数据文件中已使用的块；这些被称为备份集。RMAN 还可以对数据文件进行字节对字节的复制；这些被称为镜像副本。创建备份集是 RMAN 创建的默认备份类型。下一个命令创建数据库的备份集备份：

```
RMAN> backup database;
```

如果你愿意，可以在创建备份时显式放置 `AS BACKUPSET` 命令：

```
RMAN> backup as backupset database;
```

你可以通过使用 `AS COPY` 命令指示 RMAN 创建镜像副本。此命令为数据库中的每个数据文件创建镜像副本：

```
RMAN> backup as copy database;
```



由于映像副本是数据文件的完全相同副本，DBA 可以通过操作系统命令直接访问它们。例如，假设发生了介质故障，而你不想使用 `RMAN` 来还原映像副本。你可以使用操作系统命令将数据文件的映像副本复制到数据库可以使用的位置。相反，备份集由二进制文件组成，只有 `RMAN` 工具能够写入或读取这些文件。

使用 `RMAN` 时，更推荐使用备份集。备份集往往比数据文件更小，并且可以应用真正的二进制压缩。此外，使用 `RMAN` 作为创建只有 `RMAN` 才能还原的备份文件的机制，并非不便。将 `RMAN` 与备份集结合使用，高效、非常可靠，并且在恢复时极其有用。

## 备份表空间

`RMAN` 能够在数据库级别（如前一节所示）、表空间级别，或者更细粒度地在数据文件级别进行备份。当你备份表空间时，`RMAN` 会备份与你所指定的表空间相关联的任何数据文件。例如，以下命令将备份与 `SYSTEM` 和 `SYSAUX` 表空间相关联的所有数据文件：

```
RMAN> backup tablespace system, sysaux;
```

使用表空间级别备份的一种场景是，如果最近添加了一个新的表空间，而你只想备份与这个新添加表空间相关联的数据文件。请注意，在处理还原和备份问题时，通常更高效的方法是针对一个表空间，尤其是一个数据块进行操作（因为备份一个表空间通常比备份整个数据库快得多）。

## 备份数据文件

你可能偶尔需要备份单个数据文件。例如，在排查备份问题时，尝试成功备份一个数据文件通常很有帮助。你可以通过文件名或文件号来指定数据文件，如下所示：

```
RMAN> backup datafile '/u01/dbfile/o18c/system01.dbf';
```

在这个例子中，指定了文件号：

```
RMAN> backup datafile 1,4;
```

以下是使用各种功能备份数据文件的其他示例：

```
RMAN> backup as copy datafile 4;
RMAN> backup incremental level 1 datafile 4;
```

提示：使用 `RMAN` 的 `REPORT SCHEMA` 命令来列出表空间、数据文件名和数据文件编号信息。

## 备份控制文件

备份控制文件最可靠的方法是配置自动备份功能：

```
RMAN> configure controlfile autobackup on;
```

此命令确保在发出 `BACKUP` 或 `COPY` 命令时自动备份控制文件。启用控制文件的自动备份功能后，你就不必再担心需要显式发出单独的命令来备份控制文件。在此模式下，控制文件总是在数据文件备份片创建完成后，在其自己的备份集和备份片中创建。

如果你需要手动备份控制文件，可以这样做：

```
RMAN> backup current controlfile;
```

备份的位置可以是默认的操作系统位置、FRA（快速恢复区），或者是手动配置的位置：

```
RMAN> configure controlfile autobackup format for device type disk to '/u01/O18C/rman/rman_ctl_%F.bk';
```

## 备份 spfile

如果你启用了控制文件的自动备份功能，那么每当发出 `BACKUP` 或 `COPY` 命令时，`spfile` 也会（随控制文件一起）被自动备份。如果你需要手动备份 `spfile`，请使用以下命令：

```
RMAN> backup spfile;
```

包含 `spfile` 备份的文件位置取决于你为控制文件自动备份所配置的设置（请参阅上一节中的示例）。默认情况下，如果你没有使用 FRA，并且也没有通过通道显式配置位置，那么对于 Linux/Unix 服务器，备份将写入 `ORACLE_HOME/dbs` 目录。



## 注意

RMAN 只有在实例使用 `spfile` 启动的情况下，才能备份该 `spfile`。

## 备份归档日志

归档日志可以单独于数据库备份运行，也可以与备份一起运行。由于快速恢复区的空间以及生成的归档日志数量，归档日志备份可能每天运行多次。即使它是单独运行的，也应该使用以下命令与数据库备份一起运行：

```
RMAN> backup incremental level=0 database plus archivelog;
```

然而，您偶尔会遇到需要进行特殊的、一次性的归档日志备份的情况。您可以使用以下命令来备份归档日志文件：

```
RMAN> backup archivelog all;
```

如果您有一个即将满载的挂载点，并且您确定想要备份归档日志（以便它们存在于备份文件中），但随后又想立即将这些刚备份的文件从磁盘上删除，您可以使用以下语法来备份归档日志，然后让 RMAN 将它们从存储介质中删除：

```
RMAN> backup archivelog all delete input;
```

接下来列出的是您可以备份归档日志文件的其他一些方法：

```
RMAN> backup archivelog sequence 300;
RMAN> backup archivelog sequence between 300 and 400 thread 1;
RMAN> backup archivelog from time "sysdate-7" until time "sysdate-1";
```

如果某个归档日志已通过操作系统删除命令从磁盘上手动移除，那么在尝试备份这个不存在的归档日志文件时，RMAN 将抛出以下错误：

```
RMAN-06059: expected archived log not found, loss of archived log compromises recoverability
```

在这种情况下，首先运行 `CROSSCHECK` 命令，让 RMAN 知道哪些文件在磁盘上是物理可用的：

```
RMAN> crosscheck archivelog all;
```

## 备份快速恢复区

如果您使用了快速恢复区，RMAN 的一个便利功能是您可以通过一条命令备份该位置的所有文件。如果您使用了介质管理器并启用了磁带备份通道，您可以将快速恢复区中的所有内容备份到磁带，如下所示：

```
RMAN> backup device type sbt_tape recovery area;
```

您也可以将快速恢复区备份到磁盘上的某个位置。使用 `TO DESTINATION` 命令来实现这一点：

```
RMAN> backup recovery area to destination '/u01/O18C/fra_back';
```

RMAN 会在 `TO DESTINATION` 命令指定的目录下，根据需要自动创建目录。

**注意**
`TO_DESTINATION>` 目录下的子目录格式为 `db_uniuqe_name/backupset/YYYY_MM_DD`。

RMAN 将备份完全备份、增量备份、控制文件自动备份以及归档日志文件。请记住，闪回日志、在线重做日志文件和当前控制文件不会被备份。

## 从备份中排除表空间

假设您有一个包含非关键数据的表空间，并且您永远不想备份它。可以配置 RMAN 从备份中排除此类表空间。要确定 RMAN 当前是否被配置为排除任何表空间，请运行此命令：

```
RMAN> show exclude;
RMAN configuration parameters for database with db_unique_name O18C are:
RMAN configuration has no stored or default parameters
```

使用 `EXCLUDE` 命令指示 RMAN 哪些表空间不需备份：

```
RMAN> configure exclude for tablespace users;
```

现在，对于任何数据库级别的备份，RMAN 都将排除与 `USERS` 表空间关联的数据文件。您可以使用以下命令指示 RMAN 备份所有数据文件以及任何被排除的表空间：

```
RMAN> backup database noexclude;
```

您可以通过以下命令清除排除设置：

```
RMAN> configure exclude for tablespace users clear;
```

## 备份尚未备份的数据文件

假设您刚刚向数据库添加了几个数据文件，并且希望确保有它们的备份。您可以发出以下命令来指示 Oracle 备份尚未备份的数据文件：

```
RMAN> backup database not backed up;
```

您还可以为这些尚未备份的文件指定一个时间范围。假设您发现备份在过去几天没有运行，而您想备份在过去 24 小时内未备份的所有内容。以下命令将备份在过去一天内未备份的所有数据文件：

```
RMAN> backup database not backed up since time='sysdate-1';
```

如果由于任何原因（数据中心断电、备份期间备份目录变满，等等）导致您的备份中止，前面的命令也很有用。在您解决了导致备份作业失败的问题后，可以发出前一个命令，RMAN 将仅备份在指定时间段内未备份的数据文件。

## 跳过只读表空间

由于只读表空间中的数据不会更改，您可能只想备份只读表空间一次，然后在后续备份中跳过它们。使用 `SKIP READONLY` 命令来实现这一点：

```
RMAN> backup database skip readonly;
```

请记住，当您跳过只读表空间时，您将需要保留一个包含这些表空间的可用备份。只要您只发出 RMAN 命令 `DELETE OBSOLETE`，包含只读表空间的 RMAN 备份集就会被保留而不会被删除，即使该备份集也包含其他可读写表空间。

## 跳过离线或不可访问的文件

假设您有一个数据文件丢失或损坏，并且您没有它的备份，因此无法还原和恢复它。在这种情况下，您无法启动数据库：

```
SQL> startup;
ORA-01157: cannot identify/lock data file 6 - see DBWR trace file
ORA-01110: data file 6: '/u01/dbfile/o18c/reg_data01.dbf'
```

在这种场景下，您必须将该数据文件脱机才能启动数据库：

```
SQL> alter database datafile '/u01/dbfile/o18c/reg_data01.dbf' offline for drop;
```

现在，您可以打开数据库了：

```
SQL> alter database open;
```

假设您随后尝试运行 RMAN 备份：

```
RMAN> backup database;
```

当 RMAN 遇到一个无法备份的数据文件时，会抛出以下错误：

```
RMAN-03002: failure of backup command at ...
RMAN-06056: could not access datafile 6
```

在这种情况下，您必须指示 RMAN 从备份中排除该离线数据文件。`SKIP OFFLINE` 命令指示 RMAN 忽略状态为离线的数据文件：

```
RMAN> backup database skip offline;
```

如果文件已完全丢失，请使用 `SKIP INACCESSIBLE` 指示 RMAN 忽略磁盘上不可用的文件。如果数据文件是使用操作系统命令删除的，可能会发生这种情况。以下是从 RMAN 备份中排除不可访问数据文件的示例：

```
RMAN> backup database skip inaccessible;
```

您可以通过一条命令跳过只读、离线和不可访问的数据文件：

```
RMAN> backup database skip readonly skip offline skip inaccessible;
```

在处理离线和不可访问的文件时，您应该找出文件离线或不可访问的原因，并尝试解决任何问题。

## 并行备份大文件

通常，RMAN 只会使用一个通道来备份单个数据文件。当您启用并行性时，它允许 RMAN 生成多个进程来备份多个文件。然而，即使启用了并行性，RMAN 也不会使用并行通道同时备份一个数据文件。

您可以指示 RMAN 使用多个通道并行备份一个数据文件。这称为多段备份。此功能可以加快非常大的数据文件的备份速度。使用 `SECTION SIZE` 参数进行多段备份。以下示例配置两个并行通道来备份一个文件：



## 将 RMAN 备份信息添加到存储库

假设您不得不重新创建了控制文件。重新创建控制文件的过程会清除所有关于 RMAN 备份的信息。然而，您希望让新创建的控制文件能够识别磁盘上已有的 RMAN 备份。在这种情况下，可以使用 `CATALOG` 命令将 RMAN 元数据填充到控制文件中。例如，如果所有 RMAN 备份文件都保存在 `/u01/O18C/rman` 目录中，您可以使控制文件识别该目录中的这些备份文件，操作如下：

```
RMAN> configure device type disk parallelism 2;
RMAN> configure channel 1 device type disk format '/u01/O18C/rman/r1%U.bk';
RMAN> configure channel 2 device type disk format '/u02/O18C/rman/r2%U.bk';
RMAN> backup section size 2500M datafile 10;
```

当这段代码运行时，RMAN 将分配两个通道来并行备份指定的数据文件。

**注意**
如果您指定的节区大小大于数据文件的大小，RMAN 将不会并行备份该文件。

```
RMAN> catalog start with '/u01/O18C/rman';
```

这会指示 RMAN 在指定目录中查找任何备份片、映像副本、控制文件副本和归档日志，如果找到，则使用相应的元数据填充控制文件。对于此示例，在给定目录中找到了两个备份片文件：

```
searching for all files that match the pattern /u01/O18C/rman
List of Files Unknown to the Database
=====================================
File Name: /u01/O18C/rman/r1otlns90o_1_1.bk
File Name: /u01/O18C/rman/r1xyklnrveg_1_1.bk
Do you really want to catalog the above files (enter YES or NO)?
```

如果您输入 `YES`，则有关备份文件的元数据将被添加到控制文件中。通过这种方式，`CATALOG` 命令允许您让 RMAN 存储库（控制文件和恢复目录）知晓 RMAN 可用于备份和恢复操作的文件。

您还可以指示 RMAN 编目快速恢复区（FRA）中任何控制文件当前未知的文件，如下所示：

```
RMAN> catalog recovery area;
```

此外，您可以编目特定的文件。此示例指示 RMAN 为一个特定的备份片文件添加元数据到控制文件：

```
RMAN> catalog backuppiece '/u01/O18C/rman/r159nv562v_1_1.bk';
```

## 对可插拔数据库进行备份

从 Oracle Database 12c 开始，您可以在根容器数据库内创建可插拔数据库（更多详情请参见第 22 章）。如果您使用此选项，关于备份有几个需要注意的功能点：

*   连接到根容器时，您可以备份数据库内的所有数据文件或仅根数据库数据文件；特定的可插拔数据库；或特定的表空间或数据文件，或这些的组合。
*   连接到可插拔数据库时，您只能备份与该可插拔数据库相关联的数据文件。

上述要点将在以下两个小节中详细说明。

### 连接到根容器时

假设您以 `SYS` 身份连接到根容器，并希望备份所有数据文件（包括与可插拔数据库关联的任何数据文件）。首先，确认您确实是以 `SYS` 身份连接到根容器：

```
RMAN> SELECT SYS_CONTEXT('USERENV', 'CON_ID')   AS con_id,
SYS_CONTEXT('USERENV', 'CON_NAME')       AS cur_container,
SYS_CONTEXT('USERENV', 'CURRENT_SCHEMA') AS cur_user
FROM DUAL;
```

以下是一些示例输出：

```
CON_ID               CUR_CONTAINER        CUR_USER
-------------------- -------------------- --------------------
1                    CDB$ROOT             SYS
```

现在，要备份根容器及任何关联的可插拔数据库中的所有数据文件，请按如下方式操作：

```
RMAN> backup database;
```

如果您只想备份与根容器关联的数据文件，请指定 `ROOT`：

```
RMAN> backup database root;
```



您也可以备份特定的可插拔数据库：

```
RMAN> backup pluggable database salespdb;
```

此外，您可以备份可插拔数据库内的特定表空间：

```
RMAN> backup tablespace SALESPDB:SALES;
```

同时，您可以通过指定文件名来备份根容器或关联可插拔数据库中的任何数据文件：

```
RMAN> backup datafile '/ora01/app/oracle/oradata/CDB/salespdb/sales01.dbf';
```

## 连接到可插拔数据库时

首先，启动 RMAN，并连接到您要备份的可插拔数据库。您必须以具有 `SYSDBA` 或 `SYSBACKUP` 权限的用户身份连接。此外，必须有一个针对 PDB 服务的监听器正在运行。此示例连接到 `SALESPDB` 可插拔数据库：

```
$ rman target sys/foo@salespdb
```

一旦连接到可插拔数据库，您只能备份特定于该数据库的数据文件。因此，在此示例中，以下命令仅备份与 `SALESPDB` 可插拔数据库关联的数据文件：

```
RMAN> backup database;
```

此示例备份与可插拔数据库 `SYSTEM` 表空间关联的数据文件：

```
RMAN> backup tablespace system;
```

我需要再次强调，当您直接连接到可插拔数据库时，您只能备份与该数据库关联的数据文件。您无法备份与根容器或容器内任何其他可插拔数据库关联的数据文件。图 18-1 说明了这一概念。作为 `SYSDBA` 连接到 `SALESPDB` 可插拔数据库只能备份和查看与该数据库相关的数据文件。在数据文件以及数据文件的 RMAN 备份方面，`SYSDBA` 连接无法“看到”其可插拔框之外的内容。相比之下，连接到根容器的 `SYSDBA` 连接可以备份所有数据文件（根、种子和所有可插拔数据库），并可以访问从可插拔数据库连接发起的 RMAN 备份。

![../images/214899_3_En_18_Chapter/214899_3_En_18_Fig1_HTML.jpg](img/214899_3_En_18_Fig1_HTML.jpg)

图 18-1 连接到根容器的 `SYSDBA` 连接与连接到可插拔数据库的 `SYSDBA` 连接的权限范围

## 创建增量备份

RMAN 有三个独立且不同的增量备份功能：

*   增量级别备份
*   增量更新备份
*   块变化跟踪

使用增量级别备份，RMAN 仅备份自上次备份以来已修改的块。增量备份可应用于整个数据库、表空间或数据文件。增量级别备份是 RMAN 中最常用的增量功能。

增量更新备份是与增量级别备份独立的功能。这些备份首先对数据文件进行映像副本，然后使用增量备份来更新这些映像副本。这为您提供了一种高效的方法来实施和维护映像副本，作为备份策略的一部分。您只需进行一次映像副本备份，然后使用增量备份来使映像副本保持最新事务的状态。

块变化跟踪是另一个旨在加速增量备份性能的功能。其理念是使用一个操作系统文件来记录自上次备份以来哪些块发生了变化。在执行增量备份时，RMAN 可以使用块变化跟踪文件快速识别需要备份的块。此功能可以显著提升增量备份的性能。

### 执行增量级别备份

RMAN 通过级别来实现增量备份。只有两个有文档记载的增量备份级别：级别 0 和级别 1。在 Oracle 10g 版本之前，提供了五个级别，0 到 4。这些级别（0-4）仍然可用，但未在 Oracle 文档中指定。您必须首先进行级别 0 增量备份以建立基线，之后才能进行级别 1 增量备份。

注意
完全备份备份的块与级别 0 备份相同。但是，您不能将完全备份与增量备份一起使用。此外，您必须从级别 0 备份开始增量备份策略。如果您尝试进行级别 1 备份，但不存在级别 0 备份，RMAN 将自动进行级别 0 备份。

以下是一个进行增量级别 0 备份的示例：

```
RMAN> backup incremental level=0 database;
```

假设在接下来的几次备份中，您只想备份自上次增量备份以来发生变化的块。这行代码进行级别 1 备份：

```
RMAN> backup incremental level=1 database;
```

增量备份有两种不同的类型：差异增量备份和累积增量备份。您使用哪种类型取决于您的需求。差异备份（默认）较小，但恢复需要更多时间。累积备份比差异备份大，但需要的恢复时间较少。

差异增量级别 1 备份指示 RMAN 备份自上次级别 1 或级别 0 备份以来发生变化的块，而累积增量级别 1 备份指示 RMAN 备份自上次级别 0 备份以来发生变化的块。累积增量备份实际上会忽略任何级别 1 增量备份。

注意
RMAN 增量级别 0 备份用于还原数据文件，而 RMAN 增量级别 1 备份用于恢复数据文件。

使用增量备份时，默认是差异模式。如果您需要累积备份，则必须指定关键字 `CUMULATIVE`。以下是一个进行累积级别 1 备份的示例：

```
RMAN> backup incremental level=1 cumulative database;
```

以下是在比数据库更细粒度的级别上进行增量备份的一些示例：

```
RMAN> backup incremental level=0 tablespace sysaux;
RMAN> backup incremental level=1 tablespace sysaux plus archivelog;
RMAN> backup incremental from scn 4343352 datafile 3;
```

### 进行增量更新备份

增量更新备份的基本思路是创建数据文件的映像副本，然后使用增量备份来更新这些映像副本。这样，您就能拥有基本保持最新的数据库映像副本。这可以是一种将映像副本备份与增量备份相结合的高效方式。

要了解这种备份技术的工作原理，您需要检查执行增量更新备份的命令。需要以下 RMAN 代码行来启用此功能：

```
run{recover copy of database with tag 'incupdate';
backup incremental level 1 for recover of copy with tag 'incupdate' database;}
```

在第一行中指定了一个标签（此示例使用 `incupdate`）。您可以使用任何您想要的标签名称；该标签名称让 RMAN 在每次运行命令时关联所使用的备份文件。首次运行脚本时，此代码将按如下方式执行：

*   `RECOVER COPY` 生成一条消息，称其无需执行任何操作。
*   如果不存在映像副本，`BACKUP INCREMENTAL` 会创建数据库数据文件的映像副本。

当 `RECOVER COPY` 和 `BACKUP INCREMENTAL` 命令第一次运行时，您应在输出中看到类似这样的消息：

```
no copy of datafile 1 found to recover
...
no parent backup or copy of datafile 1 found
...
```

第二次运行增量更新备份时，它执行以下操作：

*   `RECOVER COPY` 再次生成一条消息，称其无需执行任何操作。
*   `BACKUP INCREMENTAL` 进行一次增量级别 1 备份，并分配指定的标签名称；此备份随后将被 `RECOVER COPY` 命令使用。

第三次运行增量更新备份时，它执行以下操作：

*   既然已经创建了增量备份，`RECOVER COPY` 会将增量备份应用到映像副本。




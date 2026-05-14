# Oracle 闪回技术详解

## 闪回表到删除前

在输出中，当对象被删除时，其对应的表也会有一个被重命名的主键。要撤销表的删除操作，可以执行：

```
SQL> flashback table inv to before drop;
```

前面的命令将表恢复为其原始名称。但是，此语句不会将索引恢复为其原始名称：

```
SQL> select index_name from user_indexes where table_name='INV';
INDEX_NAME
------------------------------
BIN$0zIqhEFjcprgQ4TQTwq2uA==$0
```

在这种情况下，你必须重命名索引：

```
SQL> alter index "BIN$0zIqhEFjcprgQ4TQTwq2uA==$0" rename to inv_pk;
```

你还需要以相同方式重命名任何触发器对象。如果在表删除前存在引用约束，则必须手动重新创建它们。

如果出于某种原因，你需要将表闪回到与原始名称不同的名称，可以按如下方式操作：

```
SQL> flashback table inv to before drop rename to inv_bef;
```

## 闪回表到之前的时间点

如果表被错误地删除或修改，你可以选择将表闪回到之前的时间点。闪回表功能使用回滚表空间中的信息来恢复表。过去的时间点取决于你的回滚表空间保留周期，该周期指定了回滚信息被保留的最短时间。

如果所需的闪回信息不在回滚表空间中，你将收到如下错误：

```
ORA-01555: snapshot too old
```

换句话说，要能够闪回到过去的某个时间点，回滚表空间中所需的信息不能已被覆盖。

### 闪回表到 SCN

假设你正在测试一个应用程序功能，并希望快速将表恢复到特定的系统变更号。作为应用程序测试的一部分，你在测试开始前记录下 SCN：

```
SQL> select current_scn from v$database;
CURRENT_SCN
-----------
    4760089
```

你执行了一些测试，然后希望将表闪回到之前记录的 SCN。首先，确保为表启用了行移动：

```
SQL> alter table inv enable row movement;
SQL> flashback table inv to scn 4760089;
```

现在，表应该反映了在 `FLASHBACK` 语句中指定的历史 SCN 值时已提交的事务。

### 闪回表到时间戳

你也可以将表闪回到过去的时间点。例如，要将表闪回到 15 分钟前，首先启用行移动，然后使用 `FLASHBACK TABLE`：

```
SQL> alter table inv enable row movement;
SQL> flashback table inv to timestamp(sysdate-1/96);
```

你提供的时间戳必须可以计算为 Oracle 时间戳的有效格式。你也可以显式指定时间，如下所示：

```
SQL> flashback table inv to timestamp
    to_timestamp('14-jan-18 12:07:33','dd-mon-yy hh24:mi:ss');
```

### 闪回表到还原点

还原点是与数据库中的时间戳或 SCN 关联的名称。你可以创建一个包含数据库当前 SCN 的还原点，如下所示：

```
SQL> create restore point point_a;
```

之后，如果你决定将表闪回到该还原点，首先启用行移动：

```
SQL> alter table inv enable row movement;
SQL> flashback table inv to restore point point_a;
```

现在，表应该包含与指定还原点关联的 SCN 时的事务。

## 闪回数据库

闪回数据库功能可以将整个数据库闪回到过去的时间点。闪回数据库使用存储在闪回日志中的信息；它不依赖于还原数据库文件（与冷备份、热备份和 RMAN 不同）。

> **提示**
> 闪回数据库不能替代你的数据库备份。如果你遇到数据文件的介质故障，不能使用闪回数据库闪回到故障之前。如果数据文件损坏，你必须使用物理备份（热备份、冷备份或 RMAN）进行还原和恢复。

在需要一致地将数据库重置到过去某个时间点的情况下，闪回数据库功能可能非常有用。例如，你可能希望定期将测试或培训数据库重置为已知基线。或者，你可能正在升级应用程序，在对应用程序数据库对象进行大规模更改之前，标记起始点。升级后，如果情况不理想，你希望能够快速将数据库重置到升级发生前的时间点。

闪回数据库有几个先决条件：

*   数据库必须处于归档日志模式。
*   你必须使用闪回恢复区。
*   必须启用闪回数据库功能。

有关启用归档日志模式和/或启用 FRA 的详细信息，请参见第 5 章。你可以使用以下 SQL*Plus 语句验证这些功能的状态：

```
SQL> archive log list;
SQL> show parameter db_recovery_file_dest;
```

要启用闪回数据库功能，需将数据库切换到闪回模式，如下所示：

```
SQL> alter database flashback on;
```

你可以按如下方式验证闪回状态：

```
SQL> select flashback_on from v$database;
```

启用闪回数据库后，你可以使用此查询查看 FRA 中的闪回日志：

```
SQL> select name, log#, thread#, sequence#, bytes
    from v$flashback_database_logfile;
```

你可以闪回的时间范围由 `DB_FLASHBACK_RETENTION_TARGET` 参数决定。该参数以分钟为单位，指定了你的数据库可以闪回的最长时间上限。

你可以通过运行以下 SQL 来查看可以闪回到的最旧 SCN 和时间：

```
SQL> select
        oldest_flashback_scn,
        to_char(oldest_flashback_time,'dd-mon-yy hh24:mi:ss')
    from v$flashback_database_log;
```

如果出于任何原因需要禁用闪回数据库，可以将其关闭，如下所示：

```
SQL> alter database flashback off;
```

你可以使用 RMAN 或 SQL*Plus 来闪回数据库。你可以指定过去的某个时间点，使用以下方法之一：

*   SCN
*   时间戳
*   还原点
*   上次 `RESETLOGS` 操作（仅适用于 RMAN）

此示例创建一个还原点：

```
SQL> create restore point flash_1;
```

接下来，应用程序执行一些测试，之后数据库被闪回到该还原点，以便开始新一轮测试：

```
SQL> shutdown immediate;
SQL> startup mount;
SQL> flashback database to restore point flash_1;
SQL> alter database open resetlogs;
```

此时，你的数据库应该与关联该还原点的 SCN 时的状态在事务上保持一致。

## 恢复到不同的服务器

当你考虑设计备份策略时，作为过程的一部分，你还必须考虑将如何还原和恢复。你的备份只有在上次测试还原和恢复时才是可靠的。如果没有良好的还原和恢复策略，备份策略可能会变得毫无价值。你最不希望发生的情况是：遇到介质故障，去还原数据库，然后发现缺少关键部分、没有足够的空间来还原、某些东西已损坏等等。

测试 RMAN 备份的最佳方法之一是将其还原和恢复到不同的数据库服务器。这将全面锻炼你备份、还原和恢复的 DBA 技能。如果你能在不同的服务器上成功还原和恢复 RMAN 备份，那么在真正的灾难发生时，这将给你带来信心。你可以将本书前面的所有材料视为备份和恢复工作原理的基础构建块。




## 注意

RMAN 确实有一个 ``DUPLICATE DATABASE`` 命令，它能很好地将数据库从一台服务器复制到另一台。如果你需要经常执行此类任务，我建议你使用 RMAN 的复制数据库功能。然而，你可能仍然需要手动将数据库备份从一台服务器复制到另一台，特别是在安全策略要求生产服务器不能直接连接到开发环境的情况下。你可以使用 RMAN，基于从目标服务器复制到辅助服务器的备份来复制数据库。有关无目标复制的详细信息，请参阅 MOS 注释 874352.1。

## 高级步骤

在此示例中，源服务器和目标服务器的挂载点不同。接下来列出了所需的高级步骤，以便执行 RMAN 备份并在独立的服务器上重新创建数据库：

1.  在源数据库上创建 RMAN 备份。
2.  将 RMAN 备份复制到目标服务器。接下来的所有步骤都在目标数据库服务器上执行。
3.  确保已安装 Oracle。
4.  设置所需的 OS 变量。
5.  为要还原的数据库创建一个 `init.ora` 文件。
6.  为数据文件、控制文件和转储/跟踪文件创建所有必需的目录。
7.  以 nomount 模式启动数据库。
8.  从 RMAN 备份中还原控制文件。
9.  以 mount 模式启动数据库。
10. 使控制文件知晓 RMAN 备份的位置。
11. 重命名并还原数据文件以反映新的目录位置。
12. 恢复数据库。
13. 设置联机重做日志的新位置。
14. 打开数据库。
15. 添加临时文件。
16. 重命名数据库（可选）。

前面的每个步骤都将在接下来的几个小节中详细说明。步骤 1 和 2 在源数据库服务器上执行。所有剩余步骤在目标服务器上执行。在本示例中，源数据库名为 `o18c`，目标数据库将命名为 `DEVDB`。

此外，源服务器和目标服务器的挂载点名称不同。在源数据库上，数据文件和控制文件位于此处：
```
/u01/dbfile/o18c
```
在目标数据库上，数据文件和控制文件将被重命名并还原到此目录：
```
/ora01/dbfile/DEVDB
```
目标数据库的联机重做日志将放置在此目录中：
```
/ora01/oraredo/DEVDB
```
目标数据库的归档重做日志文件位置将设置如下：
```
/ora01/arc/DEVDB
```
请记住，这些是我测试环境中使用的目录。你需要调整这些目录名称以反映你数据库服务器上的目录结构。

### 步骤 1. 在源数据库上创建 RMAN 备份
备份数据库时，请确保已开启控制文件自动备份功能。同时，将归档重做日志作为备份的一部分包含进来，如下所示：
```
RMAN> backup database plus archivelog;
```
你可以通过 `LIST BACKUP` 命令来验证备份片的名称和位置。例如，源数据库的备份片如下所示：
```
rman1_bonvb2js_1_1.bk
rman1_bqnvb2k5_1_1.bk
rman1_bsnvb2p3_1_1.bk
rman_ctl_c-3423216220-20130113-06.bk
```
在上面的输出中，文件名中包含 `c-3423216220` 字符串的那个是包含控制文件的备份片。你需要检查 `LIST BACKUP` 命令的输出，以确定哪个备份片包含控制文件。在步骤 8 中，你将需要引用该备份片。

### 步骤 2. 将 RMAN 备份复制到目标服务器
对于此步骤，使用 `rsync` 或 `scp` 等实用程序将备份片从一台服务器复制到另一台。此示例使用 `scp` 命令复制备份片：
```
$ scp rman*  oracle@DEVBOX:/ora01/rman/DEVDB
```
在此示例中，必须在复制备份文件之前在目标服务器上创建 `/ora01/rman/DEVDB` 目录。根据你的环境，此步骤可能需要复制 RMAN 备份两次：一次从生产服务器到安全服务器，另一次从安全服务器到测试服务器。

**注意**
如果 RMAN 备份在磁带上而不是在磁盘上，那么目标服务器上必须安装/配置相同的介质管理器软件。此外，该服务器必须能直接访问磁带上的 RMAN 备份。

### 步骤 3. 确保已安装 Oracle
确保在目标服务器上安装了与源数据库上相同版本的 Oracle 二进制文件。

### 步骤 4. 设置所需的 OS 变量
你需要设置 OS 变量，例如 `ORACLE_SID`、`ORACLE_HOME` 和 `PATH`。通常，`ORACLE_SID` 变量最初设置为与原始数据库上的名称匹配。数据库名称将作为此配方最后一步的一部分（可选）进行更改。以下是目标服务器上 `ORACLE_SID` 和 `ORACLE_HOME` 的设置：
```
$ echo $ORACLE_SID
o18c
$ echo $ORACLE_HOME
/ora01/app/oracle/product/18.1.0.1/db_1
```
此时，还要考虑将 Oracle SID 添加到 `oratab` 文件中。如果你计划在复制后使用此数据库，那么你应该有一个自动设置所需 OS 变量的方法。有关结合 `oratab` 文件设置 OS 变量的详细信息，请参见第 2 章。

### 步骤 5. 为要还原的数据库创建 init.ora 文件
将 `init.ora` 文件从原始服务器复制到目标服务器，并对其进行修改，使其在目录路径方面与目标服务器匹配。确保你更改了参数，例如 `CONTROL_FILES`，以反映目标服务器上的新路径目录（在本示例中为 `/ora01/dbfile/DEVDB`）。

最初，`init.ora` 文件名为 `ORACLE_HOME/dbs/inito18c.ora`，数据库名称为 `o18c`。两者都将在后面的步骤中重命名。以下是 `init.ora` 文件的内容：
```
control_files='/ora01/dbfile/DEVDB/control01.ctl',
'/ora01/dbfile/DEVDB/control02.ctl'
db_block_size=8192
db_name='o18c'
log_archive_dest_1='location=/ora01/arc/DEVDB'
job_queue_processes=10
memory_max_target=300000000
memory_target=300000000
open_cursors=100
os_authent_prefix="
processes=100
remote_login_passwordfile='EXCLUSIVE'
resource_limit=true
shared_pool_size=80M
sql92_security=TRUE
undo_management='AUTO'
undo_tablespace='UNDOTBS1'
workarea_size_policy='AUTO'
```

### 步骤 6. 为数据文件、控制文件和转储/跟踪文件创建所有必需的目录
对于本示例，需要创建目录 `/ora01/dbfile/DEVDB` 和 `/ora01/oraredo/DEVDB`：
```
$ mkdir -p /ora01/dbfile/DEVDB
$ mkdir -p /ora01/oraredo/DEVDB
$ mkdir -p /ora01/arc/DEVDB
```

### 步骤 7. 以 nomount 模式启动数据库
你现在应该能够以 nomount 模式启动数据库：
```
$ rman target /
RMAN> startup nomount ;
```

### 步骤 8. 从 RMAN 备份中还原控制文件
接下来，从先前复制的备份中还原控制文件；例如，
```
RMAN> restore controlfile from
'/ora01/rman/DEVDB/rman_ctl_c-3423216220-20130113-06.bk';
```
控制文件将被还原到 `CONTROL_FILES` 初始化参数指定的所有位置。以下是一些示例输出：
```
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
output file name=/ora01/dbfile/DEVDB/control01.ctl
output file name=/ora01/dbfile/DEVDB/control02.ctl
```

### 步骤 9. 以 mount 模式启动数据库
你现在应该能够以 mount 模式启动你的数据库：
```
RMAN> alter database mount ;
```
此时，你的控制文件已存在并已打开，但数据文件或联机重做日志都还不存在。

### 步骤 10. 使控制文件知晓 RMAN 备份的位置



## 第 11 步\. 重命名并恢复数据文件以反映新的目录位置

首先，使用 `CROSSCHECK` 命令让控制文件知道，没有备份或归档重做日志位于原始服务器上的相同位置：

```
RMAN> crosscheck backup; # 交叉校验备份
RMAN> crosscheck copy;   # 交叉校验映像副本和归档日志
```

然后，使用 `CATALOG` 命令，让控制文件知晓复制到目标服务器的备份片的名称和位置。

> **注意**
> 请勿将 `CATALOG` 命令与恢复目录模式混淆。`CATALOG` 命令将 RMAN 元数据添加到控制文件，而恢复目录模式是一个用户（通常在单独的数据库中创建），用于存储 RMAN 元数据。

在此示例中，位于 `/ora01/rman/DEVDB` 目录中的所有 RMAN 文件都将被编录到控制文件中：

```
RMAN> catalog start with '/ora01/rman/DEVDB';
```

以下是一些示例输出：

```
数据库未知的文件列表
=====================================
文件名: /ora01/rman/DEVDB/rman1_bqnvb2k5_1_1.bk
文件名: /ora01/rman/DEVDB/rman1_bonvb2js_1_1.bk
文件名: /ora01/rman/DEVDB/rman_ctl_c-3423216220-20130113-06.bk
文件名: /ora01/rman/DEVDB/rman1_bsnvb2p3_1_1.bk
您确实要编录以上文件吗（输入 YES 或 NO）？
```

现在，输入 `YES`（如果一切看起来正常）。之后，您应该能够使用 RMAN 的 `LIST BACKUP` 命令查看新编录的备份片：

```
RMAN> list backup;
```

如果您的目标服务器目录结构与原始服务器目录完全相同，您可以直接发出 `RESTORE` 命令：

```
RMAN> restore database;
```

但是，当将数据文件恢复到与原始目录不同的位置时，您必须使用 `SET NEWNAME` 命令。创建一个包含适当 `SET NEWNAME` 和 `RESTORE` 命令的 RMAN `run{}` 块文件。以下是一个用于生成 `run{}` 块起始点的 SQL 脚本示例：

```
SQL> set head off feed off verify off echo off pages 0 trimspool on
SQL> set lines 132 pagesize 0
SQL> spo newname.sql
--
SQL> select 'run{' from dual;
--
SQL> select
'set newname for datafile ' || file# || ' to ' || "" || name || "" || ';'
from v$datafile;
--
SQL> select
'restore database;' || chr(10) ||
'switch datafile all;' || chr(10) ||
'}'
from dual;
--
SQL> spo off;
```

运行该脚本后，生成的 `newname.sql` 脚本内容如下：

```
run{
set newname for datafile 1 to '/u01/dbfile/o18c/system01.dbf';
set newname for datafile 2 to '/u01/dbfile/o18c/sysaux01.dbf';
set newname for datafile 3 to '/u01/dbfile/o18c/undotbs01.dbf';
set newname for datafile 4 to '/u01/dbfile/o18c/users01.dbf';
restore database;
switch datafile all;
}
```

然后，修改 `newname.sql` 脚本的内容，以反映目标数据库服务器上的目录。在此示例中，最终编辑后的 `newname.sql` 脚本如下所示：

```
run{
set newname for datafile 1 to '/ora01/dbfile/DEVDB/system01.dbf';
set newname for datafile 2 to '/ora01/dbfile/DEVDB/sysaux01.dbf';
set newname for datafile 3 to '/ora01/dbfile/DEVDB/undotbs01.dbf';
set newname for datafile 4 to '/ora01/dbfile/DEVDB/users01.dbf';
restore database;
switch datafile all;
}
```

现在，连接到 RMAN，并运行前面的脚本，将数据文件恢复到新位置：

```
$ rman target /
RMAN> @newname.sql
```

以下是输出片段：

```
数据文件 1 已切换为数据文件副本
输入的数据文件副本 RECID=5 STAMP=790357985 文件名=/ora01/dbfile/DEVDB/system01.dbf
```

所有数据文件都已恢复到新的数据库服务器。您可以使用 RMAN 的 `REPORT SCHEMA` 命令来验证文件是否已恢复并处于正确的位置：

```
RMAN> report schema;
```

以下是一些示例输出：

```
RMAN-06139: 警告: 用于 REPORT SCHEMA 的控制文件不是当前的
数据库模式报告 (数据库唯一名称: O12C)
永久数据文件列表
===========================
文件 大小(MB) 表空间           RB 段 数据文件名
---- -------- -------------------- ------- ------------------------
1    500      SYSTEM               ***     /ora01/dbfile/DEVDB/system01.dbf
2    500      SYSAUX               ***     /ora01/dbfile/DEVDB/sysaux01.dbf
3    800      UNDOTBS1             ***     /ora01/dbfile/DEVDB/undotbs01.dbf
4    50       USERS                ***     /ora01/dbfile/DEVDB/users01.dbf
临时文件列表
=======================
文件 大小(MB) 表空间           最大大小(MB) 临时文件名
---- -------- -------------------- ----------- --------------------
1    500      TEMP                 500         /u01/dbfile/o12c/temp01.dbf
```

从前面的输出可以看到，数据库名称和临时表空间数据文件仍未反映目标数据库 (`DEVDB`)。这些将在后续步骤中进行修改。

## 第 12 步\. 恢复数据库

接下来，您需要应用备份期间生成的任何归档重做文件。由于备份时使用了 `ARCHIVELOG ALL` 子句，这些文件应包含在备份中。通过 `RECOVER DATABASE` 命令启动重做应用：

```
RMAN> recover database ;
```

RMAN 将恢复并应用备份片中包含的所有归档重做日志，当遇到一个不存在的归档重做日志时，可能会抛出错误；例如，

```
RMAN-06054: 媒体恢复正在请求未知的归档日志，用于...
```

该错误信息是正常的。恢复过程将恢复并应用备份中包含的归档重做日志，这应该足以打开数据库。恢复过程不知道应在何处停止应用归档重做日志，因此将继续尝试，直到找不到下一个日志为止。话虽如此，现在正是验证您的数据文件是否联机且不处于模糊状态的好时机：

```
SQL> select file#, status, fuzzy, error, checkpoint_change#,
to_char(checkpoint_time,'dd-mon-rrrr hh24:mi:ss') as checkpoint_time
from v$datafile_header;
```

## 第 13 步\. 为联机重做日志设置新位置

如果您的源服务器和目标服务器的目录结构完全相同，那么您无需为联机重做日志设置新位置（因此可以跳过此步骤）。

但是，如果目录结构不同，那么您将需要更新控制文件以反映联机重做日志的新目录。我有时会使用一个生成 SQL 的 SQL 脚本来辅助完成此步骤：

```
SQL> set head off feed off verify off echo off pages 0 trimspool on
SQL> set lines 132 pagesize 0
SQL> spo renlog.sql
SQL> select
'alter database rename file ' || chr(10)
|| "" || member || "" || ' to ' || chr(10) || "" || member || "" ||';'
from v$logfile;
SQL> spo off;
```

对于此示例，生成的 `renlog.sql` 文件片段如下：

```
SQL> alter database rename file
'/u01/oraredo/o18c/redo01a.rdo' to
'/u01/oraredo/o18c/redo01a.rdo';
...
SQL> alter database rename file
'/u02/oraredo/o18c/redo03b.rdo' to
'/u02/oraredo/o18c/redo03b.rdo';
```

`renlog.sql` 的内容需要被修改以反映目标服务器上的目录结构。编辑后的 `renlog.sql` 如下所示：

```
SQL> alter database rename file
'/u01/oraredo/o18c/redo01a.rdo' to
'/ora01/oraredo/DEVDB/redo01a.rdo';
...
SQL> alter database rename file
'/u02/oraredo/o18c/redo03b.rdo' to
'/ora01/oraredo/DEVDB/redo03b.rdo';
```

通过运行 `renlog.sql` 脚本来更新控制文件：

```
SQL> @renlog.sql
```

您可以从 `V$LOGFILE` 中选择数据以验证联机重做日志名称是否正确：

```
SQL> select member from v$logfile;
```

以下是此示例的输出：




## 14. 打开数据库

必须使用 `OPEN RESETLOGS` 命令打开数据库（因为此时没有重做日志，必须重新创建）：

```
SQL> alter database open resetlogs ;
```

如果成功，您应该会看到此消息：

```
Statement processed
```

## 注意
请记住，新恢复副本中的所有密码都与源数据库中的相同。您会希望在复制的数据库中更改密码，特别是如果它是从生产环境复制的。

## 15. 添加临时文件

当您启动数据库时，Oracle 会自动尝试将任何缺失的临时文件添加到数据库中。如果目标服务器上的目录结构与源服务器不同，Oracle 将无法执行此操作。在此场景中，您必须手动添加任何缺失的临时文件。为此，首先将临时表空间的临时文件脱机。来自源数据库的文件定义如下脱机：

```
SQL> alter database tempfile '/u01/dbfile/o18c/temp01.dbf' offline;
SQL> alter database tempfile '/u01/dbfile/o18c/temp01.dbf' drop;
```

接下来，向 `TEMP` 表空间添加一个临时表空间文件，该文件与目标数据库服务器的目录结构相匹配：

```
SQL> alter tablespace temp add tempfile '/ora01/dbfile/DEVDB/temp01.dbf'
size 100m;
```

您可以运行 `REPORT SCHEMA` 命令来验证所有文件是否位于正确的位置。

## 16. 重命名数据库

此步骤是可选的。如果您需要重命名数据库以反映开发或测试数据库的名称，请创建一个包含 `CREATE CONTROLFILE` 语句的跟踪文件，并使用它来重命名您的数据库。

## 提示
如果您不重命名数据库，请注意连接和重新同步到原始/源数据库使用的同一个恢复目录的操作。这会导致恢复目录混淆哪个是真正的源数据库，并可能危及您恢复和还原真正源数据库的能力。

重命名数据库的步骤如下：

1.  生成包含重新创建控制文件 SQL 命令的跟踪文件：

    ```
    SQL> alter database backup controlfile to trace as '/tmp/cf.sql' resetlogs;
    ```

2.  关闭数据库：

    ```
    SQL> shutdown immediate;
    ```

3.  修改 `/tmp/cf.sql` 跟踪文件；务必在输出的顶部指定 `SET DATABASE "<NEW DATABASE NAME>"`：

    ```
    CREATE CONTROLFILE REUSE SET DATABASE "DEVDB" RESETLOGS ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 4
    MAXDATAFILES 1024
    MAXINSTANCES 1
    MAXLOGHISTORY 876
    LOGFILE
    GROUP 1 (
    '/ora01/oraredo/DEVDB/redo01a.rdo',
    '/ora01/oraredo/DEVDB/redo01b.rdo'
    ) SIZE 50M BLOCKSIZE 512,
    GROUP 2 (
    '/ora01/oraredo/DEVDB/redo02a.rdo',
    '/ora01/oraredo/DEVDB/redo02b.rdo'
    ) SIZE 50M BLOCKSIZE 512,
    GROUP 3 (
    '/ora01/oraredo/DEVDB/redo03a.rdo',
    '/ora01/oraredo/DEVDB/redo03b.rdo'
    ) SIZE 50M BLOCKSIZE 512
    DATAFILE
    '/ora01/dbfile/DEVDB/system01.dbf',
    '/ora01/dbfile/DEVDB/sysaux01.dbf',
    '/ora01/dbfile/DEVDB/undotbs01.dbf',
    '/ora01/dbfile/DEVDB/users01.dbf'
    CHARACTER SET AL32UTF8;
    ```

    如果您未在脚本的顶部指定 `SET DATABASE`，那么当您运行该脚本时（如本示例稍后所示），您将收到如下错误：

    ```
    ORA-01161: database name ... in file header does not match...
    ```

4.  创建一个与新数据库名称匹配的 `init.ora` 文件：

    ```
    $ cd $ORACLE_HOME/dbs
    $ cp init.ora init.ora
    $ cp inito18c.ora initDEVDB.ora
    ```

5.  修改新的 `init.ora` 文件中的 `DB_NAME` 变量（在此示例中，它被设置为 `DEVDB`）：

    ```
    db_name='DEVDB'
    ```

6.  将 `ORACLE_SID` 操作系统变量设置为新的 `SID` 名称（在此示例中，设置为 `DEVDB`）：

    ```
    $ echo $ORACLE_SID
    DEVDB
    ```

7.  在 nomount 模式下启动实例：

    ```
    SQL> startup nomount;
    ```

8.  运行跟踪文件（来自步骤 2）以重新创建控制文件：

    ```
    SQL> @/tmp/cf.sql
    ```

## 注意
在此示例中，控制文件已存在于 `CONTROL_FILES` 初始化参数指定的位置；因此，在 `CREATE CONTROL FILE` 语句中使用了 `REUSE` 参数。

9.  使用 `OPEN RESETLOGS` 打开数据库：

    ```
    SQL> alter database open resetlogs;
    ```

    如果成功，您应该拥有一个作为原始数据库副本的数据库。所有数据文件、控制文件、归档重做日志和在线重做日志都在新位置，并且数据库具有一个新名称。

10. 最后一步，确保您的临时表空间存在：

    ```
    ALTER TABLESPACE TEMP ADD TEMPFILE '/ora01/dbfile/DEVDB/temp01.dbf'
    SIZE 104857600  REUSE AUTOEXTEND OFF;
    ```

## 提示
您也可以使用 `NID` 实用程序来更改数据库名称和数据库标识符（DBID）。有关更多详细信息，请参阅 MOS 注释 863800.1。

## 总结

RMAN 是 Recovery Manager 的缩写。值得注意的是，Oracle 并未将此工具命名为 Backup Manager。Oracle 团队认识到，尽管备份很重要，但备份和恢复工具的真正价值在于其恢复和还原数据库的能力。能够管理恢复过程是关键技能。当数据库损坏需要恢复时，每个人都指望 DBA 来执行平稳快速的数据库恢复。Oracle DBA 应该使用 RMAN 来保护、确保和维护公司数据资产的可用性。

还原和恢复过程类似于骨骼断裂时的愈合过程。从备份还原数据文件并将其放回原始目录，可以比作将骨骼复位到其原始位置。恢复数据文件类似于断裂骨骼的愈合——将骨骼恢复到断裂前的状态。但是，数据库恢复不会像骨骼愈合那样耗时。当您恢复数据文件时，您应用事务（从归档重做和在线重做中获得）以将还原的数据文件转换回介质故障发生前的状态。

RMAN 可用于任何类型的还原和恢复场景。根据具体情况，可以使用 RMAN 还原整个数据库、特定数据文件、控制文件、服务器参数文件、归档重做日志，或者仅还原特定的数据块。您可以指示 RMAN 执行完全恢复或不完全恢复。设置场景并练习每一个场景非常重要。这不仅可以测试备份，还可以验证可以使用备份成功进行恢复。定期安排的练习，或者甚至使用备份文件恢复到可安排的 DEV 环境，都将提供这些重要的测试。一旦您可以成功地将数据库恢复到与原始服务器不同的服务器，您将获得极大的信心并完全理解备份和恢复的内部原理。

## 20. 自动化作业

在几乎任何类型的数据库环境中——从开发、测试到生产——DBA 都严重依赖 SQL 语句、代码块和脚本来执行任务。DBA 自动化的典型作业包括：

*   数据库和监听器的关闭和启动
*   备份
*   验证备份的完整性
*   检查错误
*   删除旧的跟踪或日志文件
*   检查异常进程
*   检查异常情况




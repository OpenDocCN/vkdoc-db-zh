# RMAN `UNTIL` 命令与不完全恢复概述

`RMAN` 的 `RESTORE DATABASE UNTIL` 命令指导 `RMAN` 从过去的一个时间点检索数据文件，该时间点基于以下方法之一确定：

*   时间 (Time)
*   SCN
*   日志序列号 (Log sequence number)
*   还原点 (Restore point)

`RMAN` 的 `RESTORE DATABASE UNTIL` 命令将从最近的备份集或映像副本中检索所有数据文件。`RMAN` 会根据 `UNTIL` 子句自动确定哪个备份集包含所需的数据文件。如果省略 `RESTORE` 命令的 `UNTIL` 子句，`RMAN` 将从最新可用的备份集或映像副本中检索数据文件。在某些情况下，这可能正是您期望的行为。建议使用 `UNTIL` 子句以确保 `RMAN` 从正确的备份集还原。

当您发出 `RESTORE DATABASE UNTIL` 命令时，`RMAN` 将确定如何从以下类型的备份中提取数据文件：

*   完全数据库备份 (Full database backup)
*   增量级别 0 备份 (Incremental level-0 backup)
*   由 `BACKUP AS COPY` 命令生成的映像副本备份 (Image copy backup)

您无法对数据库在线数据文件的子集执行不完全恢复。在执行不完全数据库恢复时，所有在线数据文件的检查点 SCN 必须同步，然后才能使用 `ALTER DATABASE OPEN RESETLOGS` 命令打开数据库。您可以通过以下 SQL 查询查看数据文件头 SCN 和每个数据文件的状态：

```
SQL> select file#, status, fuzzy,
error, checkpoint_change#,
to_char(checkpoint_time,'dd-mon-rrrr hh24:mi:ss') as checkpoint_time
from v$datafile_header;
```

**注意**：`V$DATAFILE_HEADER` 视图的 `FUZZY` 列包含那些有一个或多个块的 SCN 值大于或等于数据文件头中检查点 SCN 的数据文件。如果还原的数据文件的 `FUZZY` 值为 `YES`，则需要进行介质恢复。

此规则（不在数据库文件的子集上执行不完全恢复）的唯一例外是**表空间时间点恢复 (TSPITR)**，它使用 `RECOVER TABLESPACE UNTIL` 命令。TSPITR 用于罕见情况；它仅还原和恢复您指定的表空间。

不完全数据库恢复的**恢复部分**总是使用 `RECOVER DATABASE UNTIL` 命令启动。`RMAN` 将自动将您的数据库恢复到使用 `UNTIL` 子句指定的点。与 `RESTORE` 命令类似，您可以恢复到特定时间、SCN、日志序列号或还原点。当 `RMAN` 达到指定点时，它将自动终止恢复过程。

**注意**：无论您在 `UNTIL` 子句中指定什么，`RMAN` 都会将其转换为相应的 `UNTIL SCN` 子句并分配适当的 SCN。这是为了避免任何时序问题，特别是由夏令时引起的问题。

在恢复期间，`RMAN` 将自动确定如何应用重做。首先，`RMAN` 将应用任何可用的增量备份。接下来，将应用磁盘上的任何归档日志文件。如果归档日志文件不在磁盘上，则 `RMAN` 将尝试从备份集中检索它们。如果希望作为不完全数据库恢复的一部分应用重做，则必须满足以下条件：

*   您的数据库处于归档日志模式 (archivelog mode)。
*   您拥有所有数据文件的良好备份。
*   您拥有恢复到指定点所需的所有重做日志。

使用 `RMAN` 执行不完全数据库恢复时，必须将数据库置于装载 (mount) 模式。`RMAN` 需要数据库处于装载模式才能读写控制文件。此外，在不完全数据库恢复期间，任何 `SYSTEM` 表空间的数据文件总是会被恢复。Oracle 不允许在还原 `SYSTEM` 表空间数据文件时打开数据库。

**注意**：执行不完全数据库恢复后，您必须使用 `ALTER DATABASE OPEN RESETLOGS` 命令打开数据库。在执行 `ALTER DATABASE OPEN RESETLOGS` 后的任何时间，请确保进行新的备份，以便在此时间点之后可用，因为如果尝试还原到 resetlogs 之后的时间点，其他备份可能会失效。

根据具体场景，您可以使用 `RMAN` 执行各种不完全恢复方法。下一节讨论如何确定要执行的不完全恢复类型。

## 确定不完全恢复的类型

基于时间的还原和恢复通常在您知道要恢复数据库的大致日期和时间时使用。例如，您可能大致知道希望停止恢复过程的时间，但不知道具体的 SCN。

基于日志序列和基于取消的恢复在您有丢失或损坏的日志文件时很有效。在这种情况下，您只能恢复到最后一个好的归档日志文件。

基于 SCN 的恢复在您能够精确定位希望停止恢复过程的 SCN 时很有效。您可以从 `V$LOG` 和 `V$LOG_HISTORY` 等视图中检索 SCN 信息。您也可以使用 LogMiner 等工具来检索特定 SQL 语句的 SCN。

还原点恢复仅在您已建立还原点的情况下有效。在这些情况下，您还原并恢复到与指定还原点关联的 SCN。

TSPITR 用于需要仅还原和恢复少数几个表空间的情况。您可以使用 `RMAN` 自动化与此类不完全恢复相关的许多任务。

## 执行基于时间的恢复

要将数据库还原并恢复到过去的某个时间点，您可以使用 `RESTORE` 和 `RECOVER` 命令的 `UNTIL TIME` 子句，也可以使用 `run{}` 块内的 `SET UNTIL TIME` 子句。拥有语法正确的 `run{}` 代码块非常有用，您可以替换其中的时间值来执行还原，而无需搜索语法。使用本书中的这些示例并运行测试和练习还原，将为您提供准备好的代码块。`RMAN` 将还原并恢复数据库到但不包括指定的时间。换句话说，`RMAN` 将还原在指定时间之前提交的任何事务。`RMAN` 在达到您指定的时间时自动停止恢复过程。

`RMAN` 期望的默认日期格式是 `YYYY-MM-DD:HH24:MI:SS`。但是，建议使用 `TO_DATE` 函数并指定格式掩码。这消除了不同国家日期格式的歧义，也无需设置操作系统 `NLS_DATE_FORMAT` 变量。以下示例在发出 `restore` 和 `recover` 命令时指定了时间：

```
$ rman target /
RMAN> startup mount;
RMAN> restore database until time
"to_date('15-jan-2018 12:20:00', 'dd-mon-rrrr hh24:mi:ss')";
RMAN> recover database until time
"to_date('15-jan-2018 12:20:00', 'dd-mon-rrrr hh24:mi:ss')";
RMAN> alter database open resetlogs;
```

如果一切顺利，您应该看到类似这样的输出：

```
Statement processed
```

## 执行基于日志序列的恢复

通常，这种类型的不完全数据库恢复是因为您有一个丢失或损坏的归档日志文件而启动的。如果是这种情况，您只能恢复到最后一个好的归档日志文件，因为您不能跳过丢失的归档日志。

您如何确定要还原到（但不包括）哪个归档日志文件会有所不同。例如，如果您物理上丢失了一个归档日志文件，并且 `RMAN` 无法在备份集中找到它，那么在尝试应用丢失的文件时，您将收到类似这样的消息：

```
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 19...
```



## 基于日志序列的恢复

根据之前的错误消息，您将恢复到（但不包括）日志序列号 19。

```
$ rman target /
RMAN> startup mount;
RMAN> restore database until sequence 19;
RMAN> recover database until sequence 19;
RMAN> alter database open resetlogs;
```

如果成功，您应该会看到类似以下的输出：

```
Statement processed
```

注意
基于日志序列的恢复类似于用户管理的基于取消的恢复。有关用户管理的基于取消的恢复的详细信息，请参见第 16 章。


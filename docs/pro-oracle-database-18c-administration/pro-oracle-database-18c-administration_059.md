# 使用 SQL

有多个数据字典视图可用于查询备份信息。表 18-1 描述了与 RMAN 相关的数据字典视图。无论您是否使用恢复目录，这些视图都可用（这些视图中的信息源自控制文件）。

## 表 18-1
RMAN 备份数据字典视图描述

| 视图名称 | 提供的信息 |
| --- | --- |
| `V$RMAN_BACKUP_JOB_DETAILS` | RMAN 备份作业 |
| `V$BACKUP` | 处于备份模式（用于热备份）的联机数据文件的备份状态 |
| `V$BACKUP_ARCHIVELOG_DETAILS` | 已备份的归档日志 |
| `V$BACKUP_CONTROLFILE_DETAILS` | 已备份的控制文件 |
| `V$BACKUP_COPY_DETAILS` | 控制文件和数据文件副本 |
| `V$BACKUP_DATAFILE` | 已备份的控制文件和数据文件 |
| `V$BACKUP_DATAFILE_DETAILS` | 在备份集、映像副本和代理副本中备份的数据文件 |
| `V$BACKUP_FILES` | 已备份的数据文件、控制文件、`spfile` 和归档日志 |
| `V$BACKUP_PIECE` | 备份片文件 |
| `V$BACKUP_PIECE_DETAILS` | 备份片详细信息 |
| `V$BACKUP_SET` | 备份集 |
| `V$BACKUP_SET_DETAILS` | 备份集详细信息 |

有时，刚接触 RMAN 的 DBA 很难理解备份、备份集、备份片和数据文件的概念以及它们之间的关系。在讨论 RMAN 备份组件时，我发现以下查询很有用。此查询将显示备份集、备份集中的备份片以及备份片内备份的数据文件：

```
SQL> SET LINES 132 PAGESIZE 100
SQL> BREAK ON REPORT ON bs_key ON completion_time ON bp_name ON file_name
SQL> COL bs_key    FORM 99999 HEAD "BS Key"
SQL> COL bp_name   FORM a40   HEAD "BP Name"
SQL> COL file_name FORM a40   HEAD "Datafile"
SQL> --
SQL> SELECT
  s.recid                  bs_key
  ,TRUNC(s.completion_time) completion_time
  ,p.handle                 bp_name
  ,f.name                   file_name
FROM v$backup_set      s
    ,v$backup_piece    p
    ,v$backup_datafile d
    ,v$datafile        f
WHERE p.set_stamp = s.set_stamp
AND   p.set_count = s.set_count
AND   d.set_stamp = s.set_stamp
AND   d.set_count = s.set_count
AND   d.file#     = f.file#
ORDER BY
  s.recid
  ,p.handle
  ,f.name;
```

此处的输出已缩短以适应页面：

```
BS Key COMPLETIO BP Name                           Datafile
------ --------- --------------------------------  -------------------------------
159    11-JAN-18 /u01/O18C/rman/r16qnv59jj_1_1.bk /u01/dbfile/o18c/inv_data2.dbf
                                              /u01/dbfile/o18c/lob_data01.dbf
                                              /u01/dbfile/o18c/p14_tbsp.dbf
                                              /u01/dbfile/o18c/p15_tbsp.dbf
                                              /u01/dbfile/o18c/p16_tbsp.dbf
```

有时，报告 RMAN 备份的性能很有用。以下查询报告了每个会话的 RMAN 备份所花费的时间。

```
SQL> COL hours              FORM 9999.99
SQL> COL time_taken_display FORM a20
SQL> SET LINESIZE 132
SQL> --
SQL> SELECT
  session_recid
  ,compression_ratio
  ,time_taken_display
  ,(end_time - start_time) * 24 as hours
  ,TO_CHAR(end_time,'dd-mon-yy hh24:mi') as end_time
FROM v$rman_backup_job_details
ORDER BY end_time;
```

这是一些示例输出：

```
SESSION_RECID COMPRESSION_RATIO TIME_TAKEN_DISPLAY      HOURS END_TIME
------------- ----------------- -------------------- -------- ---------------
15            1                 00:05:08                  .09 11-jan-18 13:41
27            3.79407176        00:00:09                  .00 11-jan-18 13:52
33            1.19992137        00:05:01                  .08 11-jan-18 14:07
```

`V$RMAN_BACKUP_JOB_DETAILS` 的内容是根据与 RMAN 的会话连接进行汇总的。因此，如果您连接到 RMAN（建立会话），然后在备份作业完成后退出 RMAN，报告输出会更准确。如果您在运行多个备份作业时保持与 RMAN 的连接，则查询输出会报告该会话连接期间的所有备份活动。

您应该有一个自动化的方法来检测 RMAN 备份是否正在运行以及数据文件是否正在被备份。自动化此类任务的一个可靠方法是将 SQL 嵌入到 shell 脚本中，然后通过 `cron` 等调度工具定期运行该脚本。

我通常对 RMAN 备份运行两种基本类型的检查：
*   RMAN 备份最近运行过吗？
*   是否有任何数据文件最近未被备份？

以下 shell 脚本检查这些条件。您需要修改脚本，并为其提供一个能够查询脚本中引用的数据字典对象的用户的用户名和密码，同时更改消息发送到的电子邮件地址。运行脚本时，您需要传入两个变量：Oracle SID 以及您希望检查的上次备份运行时间或数据文件备份时间的阈值天数。

```
#!/bin/bash
#
if [ $# -ne 2 ]; then
  echo "Usage: $0 SID threshold"
  exit 1
fi
# source oracle OS variables
. /var/opt/oracle/oraset $1
crit_var=$(sqlplus -s / as sysdba <<EOF
SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF
SELECT count(*)
FROM v\$rman_backup_job_details
WHERE start_time > sysdate - $2
  AND status = 'COMPLETED';
EOF)
#
if [ $crit_var -ne 0 ]; then
  echo "rman backups not running on $1" | mailx -s "rman problem" dkuhn@gmail.com
else
  echo "rman backups ran ok"
fi
#--------------------------------------------
crit_var2=$(sqlplus -s / as sysdba <<EOF
SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF
SELECT count(*)
FROM v\$datafile
WHERE file# NOT IN (
  SELECT file#
  FROM v\$backup_datafile
  WHERE completion_time > sysdate - $2
);
EOF)
#
if [ $crit_var2 -ne 0 ]; then
  echo "datafile not backed up on $1" | mailx -s "backup problem" dkuhn@gmail.com
else
  echo "datafiles are backed up..."
fi
#
exit 0
```

例如，要检查备份在过去 2 天内是否成功运行，请运行脚本（名为 `rman_chk.bsh`）：

```
$ rman_chk.bsh INVPRD 2
```

前面的脚本是基础但有效的。您可以根据您的 RMAN 环境需求对其进行增强。

## 总结

RMAN 为备份提供了许多灵活且功能丰富的选项。默认情况下，RMAN 仅备份数据库中已修改的块。增量功能允许您仅备份自上次备份以来已修改的块。这些增量功能在减少大型数据库环境中的备份大小方面特别有用，因为从一次备份到下一次备份，数据库中只有一小部分数据发生变化。

您可以指示 RMAN 通过映像副本备份每个数据文件中的每个块。映像副本是数据文件的逐块相同副本。映像副本的优点是能够直接从备份中恢复备份文件（无需使用 RMAN）。您可以使用增量更新备份功能来实现映像副本备份和增量备份的高效混合。

存在一些配置，有助于将映像副本的恢复策略和要求与增量备份、备份集和归档日志相匹配。CDB 和 PDB 数据库都可以在 CDB 中备份，但当连接到 PDB 作为目标时，只能备份该 PDB。数据库数据块更改可以与表空间、数据文件和数据库一起备份。

RMAN 包含用于报告备份许多方面的内置命令。`LIST` 命令报告备份活动。`REPORT` 命令对于根据保留策略确定需要备份哪些文件非常有用。

成功配置 RMAN 并创建备份后，您就能够在发生介质故障时恢复和恢复您的数据库。备份的好坏取决于恢复数据库的能力。恢复和还原主题将在下一章中详细介绍。




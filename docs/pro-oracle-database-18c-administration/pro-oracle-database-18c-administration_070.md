# 将表恢复到之前的某个时间点

从 Oracle Database 12c 开始，您可以通过 `RECOVER TABLE` 命令从 RMAN 备份中恢复单个表。这使您能够将表恢复并恢复到过去某个时间点。

表级恢复功能使用临时辅助实例和 Data Pump 实用程序。辅助实例和 Data Pump 在恢复表时都会创建临时文件。在启动表级恢复之前，首先创建两个目录：一个用于存放辅助实例使用的文件，另一个用于存储 Data Pump 转储文件：

```
$ mkdir /tmp/oracle
$ mkdir /tmp/recover
```

前面的两个目录在 `RECOVER TABLE` 命令中通过 `AUXILIARY DESTINATION` 和 `DATAPUMP DESTINATION` 子句引用。在以下代码片段中，`MV_MAINT` 拥有的 `INV` 表被恢复到之前某个 SCN 的状态：

```
RMAN> recover table mv_maint.inv
until scn 4689805
auxiliary destination '/tmp/oracle'
datapump destination '/tmp/recover';
```

只要存在包含该表在指定 SCN 时状态的 RMAN 备份，就会执行表级恢复和恢复。

注意
您也可以将表恢复到某个 SCN、时间点或日志序列号。

当 RMAN 执行表级恢复时，它会自动创建一个临时辅助数据库，使用 Data Pump 导出该表，然后将该表作为它在指定恢复点时的状态导入回目标数据库。恢复完成后，辅助数据库会被删除，Data Pump 转储文件也会被移除。

提示
虽然 `RECOVER TABLE` 命令是一个不错的增强功能，但我建议，如果您意外删除了一个表，首先探索使用回收站或“闪回表到删除前”功能来恢复该表。或者，如果表被错误地删除了数据，则使用“闪回表”功能将表闪回到过去的某个时间点。甚至可以使用 CTAS（创建表为查询）从 `FLASHBACK QUERY` 恢复。如果以上选项都不可行，则考虑使用 RMAN 的“恢复表”功能。


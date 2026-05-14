# 恢复到某个恢复点

恢复点有两种类型：普通恢复点和保证恢复点。保证恢复点与普通恢复点之间的主要区别在于，保证恢复点最终不会从控制文件中老化清除；保证恢复点将一直保留，直到您删除它。保证恢复点确实需要一个闪回恢复区（FRA）。但是，对于使用保证恢复点进行不完全恢复，您无需启用闪回数据库。

您可以使用 SQL*Plus 创建普通恢复点，如下所示：

```
SQL> create restore point MY_RP;
```

此命令创建一个名为 `MY_RP` 的恢复点，该恢复点与命令发出时数据库的 SCN 相关联。您可以查看数据库的当前 SCN，如下所示：

```
SQL> select current_scn from v$database;
```

您可以在 `V$RESTORE_POINT` 视图中查看恢复点信息，如下所示：

```
SQL> select name, scn from v$restore_point;
```

恢复点类似于特定 SCN 的别名。恢复点允许您恢复和恢复到某个 SCN，而无需指定一个数字。RMAN 将恢复和恢复到但不包括与恢复点关联的 SCN。

以下示例恢复并恢复到 `MY_RP` 恢复点：

```
$ rman target /
RMAN> startup mount;
RMAN> restore database until restore point MY_RP;
RMAN> recover database until restore point MY_RP;
RMAN> alter database open resetlogs;
```


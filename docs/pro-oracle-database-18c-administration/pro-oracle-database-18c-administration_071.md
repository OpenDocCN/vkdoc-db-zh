# 闪回表

为了简化意外删除表的恢复，Oracle 引入了“闪回表”功能。Oracle 提供两种不同类型的闪回表操作：

*   `FLASHBACK TABLE TO BEFORE DROP` 快速取消删除之前删除的表。此功能使用一个名为回收站的逻辑容器。
*   `FLASHBACK TABLE` 将表闪回到最近的某个时间点，以撤消不期望的 DML 语句的影响。您可以闪回到某个 SCN、时间戳或恢复点。

Oracle 引入 `FLASHBACK TABLE TO BEFORE DROP` 是为了让您能够快速恢复已删除的表。当您删除表时，如果没有指定 `PURGE` 子句，Oracle 不会删除该表——而是重命名该表。您删除的任何表（Oracle 重命名的）都会被放入回收站。回收站为您提供了一种高效查看和管理已删除对象的方法。

注意
要使用“闪回表”功能，您不需要实现 FRA，也不需要启用闪回数据库。

`FLASHBACK TABLE TO BEFORE DROP` 操作仅在您的数据库启用了回收站功能（默认情况下是启用的）时才有效。您可以检查回收站的状态，如下所示：

```
SQL> show parameter recyclebin
NAME                                 TYPE        VALUE
------------------------------------ ----------- --------------------------
recyclebin                           string      on
```

## FLASHBACK TABLE TO BEFORE DROP

以下是一个示例。假设 `INV` 表被意外删除：

```
SQL> drop table inv;
```

通过查看回收站的内容来验证表已被重命名：

```
SQL> show recyclebin;
ORIGINAL NAME   RECYCLEBIN NAME                OBJECT TYPE  DROP TIME
--------------- ------------------------------ ------------ --------------
INV             BIN$0zIqhEFlcprgQ4TQTwq2uA==$0 TABLE        2018-01-11:12:16:49
```

`SHOW RECYCLEBIN` 语句仅显示已删除的表。要获取重命名对象的更完整视图，请查询 `RECYCLEBIN` 视图：

```
select object_name, original_name, type
from recyclebin;
```

输出如下：

```
OBJECT_NAME                         ORIGINAL_NAM TYPE
----------------------------------- ------------ -------------------------
BIN$0zIqhEFjcprgQ4TQTwq2uA==$0      INV_PK       INDEX
BIN$0zIqhEFkcprgQ4TQTwq2uA==$0      INV_TRIG     TRIGGER
BIN$0zIqhEFlcprgQ4TQTwq2uA==$0      INV          TABLE
```




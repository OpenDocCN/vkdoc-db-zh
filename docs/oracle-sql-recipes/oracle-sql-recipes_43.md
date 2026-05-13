# 第 5 部分

## 管理

#### 第 17 章：数据库管理

*数据库管理* 指的是数据库管理员（DBA）为确保数据库管理系统的可用性、可扩展性、易用性和可维护性而执行的各种任务。关于这个主题已有整本的著作。本章并不会涵盖 DBA 可能用到的所有 SQL 方面。相反，它的目标是向您展示如何使用 SQL 来执行常见的数据库管理员任务。

本章涉及的许多任务也可以使用像 Enterprise Manager 这样的图形化工具来执行。图形化工具非常适合快速执行基本的数据库管理任务，特别是当您对直接运行 SQL 命令不太熟悉时。对于大多数场景来说，它们是够用的。

然而，您最终会遇到需要实现某个 SQL 功能，而该功能恰恰无法通过图形化工具工作的情况。例如，您可能需要使用图形化工具中尚未提供的新特性或特殊命令语法。或者您可能想编写一个稍复杂的、必须以批处理模式运行的脚本。在这种情况下，您就需要具备执行该操作所需的 SQL 脚本编写技能。

### 五命令法则

我们中的一位曾效力于一家公司，当时刚招聘了一位初级 DBA。当轮到这位初级 DBA 值班时，这位年轻人担心自己是否具备在生产环境中执行数据库管理任务的知识。一位资深 DBA 将这位初级 DBA 叫到一边，解释说实际上只需要知道五条 SQL 命令就可以成为一名 DBA：

* `STARTUP`
* `SHUTDOWN`
* `RECOVER`
* `ALTER DATABASE`
* `ALTER SYSTEM`

这位资深 DBA 说 DBA 只需要少数几条 SQL 命令，是有点开玩笑的。当然，DBA 还需要知道如何备份和恢复数据库——以及其他任务。

尽管如此，五命令法则还是有些道理的。关键在于要打好 Oracle 数据库架构的坚实基础，以及掌握使用 SQL 与数据库交互的方法。当出现严重问题时，SQL 是 DBA 用来排查和修复问题的工具。您必须熟悉 DBA 使用的基本命令，以及如何为超出基本 SQL 命令范围的场景查找相应命令。

##### 17-1. 创建数据库

#### 问题

您刚刚安装好了 Oracle 二进制文件。您想要创建一个数据库。

#### 解决方案

在运行 `CREATE DATABASE` 命令之前，您必须连接到 SQL*Plus 并以 `NOMOUNT` 模式启动数据库：

```
$ sqlplus / as sysdba
SQL> startup nomount;
```

下面是一个典型的 Oracle `CREATE DATABASE` 语句：

```
create database invrep controlfile reuse
maxlogfiles 16
maxlogmembers 4
maxdatafiles 1024
maxinstances 1
maxloghistory 680
character set "UTF8"
logfile group 1
('/ora01/oradata/INVREP/redo01a.log',
'/ora01/oradata/INVREP/redo01b.log') size 200m reuse,
group 2
('/ora01/oradata/INVREP/redo02a.log',
'/ora01/oradata/INVREP/redo02b.log' ) size 200m reuse,
group 3
('/ora01/oradata/INVREP/redo03a.log',
'/ora01/oradata/INVREP/redo03b.log' ) size 200m reuse
datafile
'/ora01/oradata/INVREP/system01.dbf'
size 500m
reuse
undo tablespace undotbs1 datafile
```

考虑到所需的开销，例如让 LOB 列成为主键，这些限制中的大多数似乎是合理的。然而，在其他操作中，您可以像对待任何其他数据库列一样对待它们，例如在同一张表或另一张表中将一个 LOB 列更新为另一个 LOB 列，或者使用 `EMPTY_CLOB()` 将 LOB 列设置为 `NULL` 或空值。

LOB 在事务中也可以回滚，尽管 LOB 的旧版本存储在 LOB 段中，而不是撤销表空间中。




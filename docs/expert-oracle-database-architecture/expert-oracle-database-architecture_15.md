# 快速恢复区 (FRA)

Oracle 中的**快速恢复区 (FRA)** 是一个由数据库管理的位置，用于存放与数据库备份和恢复相关的多种文件。在此区域（这里的“区域”指磁盘上为此目的划出的一部分，例如一个目录），您可能会找到以下内容：

-   RMAN 备份集（全备份和/或增量备份）
-   RMAN 映像副本（数据文件和控制文件的字节级副本）
-   在线重做日志
-   归档重做日志
-   多路复用的控制文件
-   闪回日志

Oracle 使用这个新区域来管理这些文件，因此服务器会知道哪些文件在磁盘上，哪些不在磁盘上（可能在其他磁带中）。利用这些信息，数据库可以执行诸如磁盘到磁盘还原受损数据文件，或将数据库“闪回”（一种“倒退”操作）以撤销本不该执行的操作。例如，您可以使用 `FLASHBACK DATABASE` 命令将数据库恢复到五分钟前的状态（而无需对数据库进行完整还原和基于时间点的恢复）。这样您就可以“撤销删除”那个意外删除的用户账户。

要设置快速恢复区，您需要设置两个参数：`db_recovery_file_dest_size` 和 `db_recovery_file_dest`。您可以通过以下命令查看这些参数的当前设置：

```sql
SQL> show parameter db_recovery_file_dest
NAME                                 TYPE        VALUE
------------------------------------ ----------- --------------------------
db_recovery_file_dest                string      /opt/oracle/fra
db_recovery_file_dest_size           big integer 100G
```

快速恢复区更多是一个逻辑概念。它是本章讨论的文件类型的一个存放区。其使用是可选的——您不一定需要使用它，但如果您想使用某些高级功能，例如 `FLASHBACK DATABASE`，则必须使用此区域来存储相关信息。

## 数据泵文件

**数据泵 (Data Pump)** 是 Oracle 中至少两种工具使用的文件格式。外部表可以以数据泵格式加载和卸载数据，而导入/导出工具 `IMPDP` 和 `EXPDP` 也使用此文件格式。它们是跨平台（可移植）的二进制文件，包含元数据（不是存储在 `CREATE/ALTER` 语句中，而是以 XML 格式存储）以及可能的数据。它们使用 XML 作为元数据表示结构，这一点作为工具的最终用户，与您我其实息息相关。`IMPDP` 和 `EXPDP` 拥有一些复杂的过滤和转换能力。这部分归因于 XML 的使用，也因为 `CREATE TABLE` 语句并非存储为 `CREATE TABLE` 原样，而是存储为一种标记化文档。这使得轻松实现类似“请将所有对表空间 FOO 的引用替换为表空间 BAR”这样的请求成为可能。`IMPDP` 只需应用一个简单的 XML 转换即可完成此操作。当 `FOO` 指代一个 `TABLESPACE` 时，它会被 `<TABLESPACE>FOO</TABLESPACE>` 标签（或某些其他类似表示形式）包围。

在第 15 章，我们将更详细地探讨这些工具。不过在此之前，让我们先看看如何利用这种数据泵格式快速从数据库 A 提取数据并移动到数据库 B。这里我们将使用一种“反向外部表”。

外部表使我们能够读取平面文件——即普通的旧文本文件——就像它们是数据库表一样。我们可以使用 SQL 的全部功能来处理它们。它们是只读的，旨在将 Oracle 外部的数据导入。外部表也可以反向操作：可用于以数据泵格式将数据导出数据库，以方便将数据移动到其他机器或平台。要开始此练习，我们需要一个 `DIRECTORY` 对象，告知 Oracle 卸载的目标位置：

```sql
SQL> create or replace directory tmp as '/tmp';
Directory created.
SQL> create table all_objects_unload
organization external
( type oracle_datapump
default directory TMP
location( 'allobjects.dat' )
)
as
select * from all_objects
/
Table created.
```

实际上，操作就是这么简单：我们在 `/tmp` 下有了一个名为 `allobjects.dat` 的文件，其中包含了查询 `select * from all_objects` 的内容。我们可以窥视一下这些信息：

```sql
SQL> !strings /tmp/allobjects.dat | head
x86_64/Linux 2.4.xx
AL32UTF8
19.00.00.00.00
001:001:000001:000001
i<?xml version="1.0" encoding="UTF-8"?
...
```

这只是文件的头部信息。现在，使用二进制 SCP，您可以将该文件移动到任何其他已安装 Oracle 的平台上，通过发出 `CREATE DIRECTORY` 命令（告知数据库文件所在位置）和 `CREATE TABLE` 命令，如下所示：

```sql
SQL> create table t
( OWNER            VARCHAR2(30),
OBJECT_NAME      VARCHAR2(30),
SUBOBJECT_NAME   VARCHAR2(30),
OBJECT_ID        NUMBER,
DATA_OBJECT_ID   NUMBER,
OBJECT_TYPE      VARCHAR2(19),
CREATED          DATE,
LAST_DDL_TIME    DATE,
TIMESTAMP        VARCHAR2(19),
STATUS           VARCHAR2(7),
TEMPORARY        VARCHAR2(1),
GENERATED        VARCHAR2(1),
SECONDARY        VARCHAR2(1)
)
organization external
( type oracle_datapump
default directory TMP
location( 'allobjects.dat' ));
```

您就可以立即使用 SQL 读取那些卸载的数据了。这就是数据泵文件格式的威力：实现数据在系统间的即时传输，必要时甚至可以通过“人力传递网络”（sneakernet）进行。下次当您想带一部分数据回家在周末测试时，请想想这一点。

即使数据库字符集不同（本例中相同），得益于数据泵格式，Oracle 现在有能力识别不同的字符集并处理它们。可以根据需要动态执行字符集转换，以确保数据在每个数据库的表示中都是“正确”的。

再次说明，我们将在第 15 章回到数据泵文件格式，但本节应让您对其是什么以及文件中可能包含哪些内容有一个整体的了解。

## 平面文件

平面文件自电子数据处理诞生之初就已存在。我们几乎每天都能看到它们。前面描述的文本告警日志就是一个平面文件。我在网上找到了以下对“平面文件”的定义，感觉它基本上概括了其本质：

> *一种剥离了所有特定应用程序（程序）格式的电子记录。这允许数据元素被迁移到其他应用程序中进行操作。这种剥离电子数据的模式可以防止因硬件和专有软件过时而导致的数据丢失。*

平面文件就是一个简单的文件，其中每一“行”是一条“记录”，并且每行都有一些文本分隔符，通常是逗号或竖线（|）。使用传统数据加载工具 `SQLLDR` 或外部表，Oracle 可以轻松读取平面文件。实际上，我将在第 15 章详细介绍这一点。

有时，用户会要求以某种平面文件格式提供数据，例如以下几种：

-   字符（或逗号）分隔值（`CSV`）：这些文件易于导入到电子表格等工具中。
-   超文本标记语言（`HTML`）：用于在网页浏览器中显示页面。
-   JavaScript 对象表示法（`JSON`）：用于表示结构化数据的标准基于文本的格式，并用于在网页浏览器中传输数据。

我将在接下来的章节中简要演示生成每种文件类型。



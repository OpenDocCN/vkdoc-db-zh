# 分析与解决 Oracle 并行函数数据分布问题

## 问题复现与观察

```
        l_key_value_pair.key_item := substr(p_input,i,1);
        l_key_value_pair.value_item := 1;

        pipe row(l_key_value_pair);

      end loop;

      return;

    end result_set;

end single_map_letter_count;
/
```

现在，请通过使用你的 `Reduce` 函数来对单个 Map 的结果进行规约测试，如下所示：

```
select * from table(
reduce_letter_count.result_set(
cursor(select * from table(
single_map_letter_count.result_set('Hello, world!')
))))
order by key_item;
```

```
KEY_ITEM                         VALUE_ITEM
-------------------------------- ----------------------
                                 1
!                                1
,                                1
H                                1
d                                1
e                                1
l                                2
l                                1
o                                1
o                                1
r                                1
w                                1

12 rows selected
```

这看起来不对！字母‘l’和‘o’各有两行数据。可能是什么地方出错了？

## 问题根源分析

让我们查看一下 `Reduce` 代码，并特别注意你在编码时可能做出的任何假设。你应该立即注意到，你假设了传递给 `Reduce` 函数的数据是经过排序或聚集的，即来自 Map 输出的相同字母会连续出现。然而，你使用的 `PARTITION … BY ANY` 选项明确地告诉服务器，你不在乎哪些字母被发送到哪个 Reducer，你也没有告诉服务器，你希望发送到 Reducer 的数据集以某种方式确保数据项是聚集或排序的。

你如何告诉 Oracle 你希望将相同的字母发送到同一个 Reducer？

## 控制数据分布的方法

通过用游标中列的信息（在这个例子中是实际的键/值对）替换通用的 `sys_refcursor`，你现在有多种方式将行分发到并行函数实例。指定分布的完整语法如下：

```
{[ORDER | CLUSTER] BY `column_list`}
PARALLEL_ENABLE({PARTITION p BY [ANY | (HASH | RANGE) column_list]} )
```

使用这些关键字时，*分区 (partitioning)* 指的是将行分配到各个并行函数实例所使用的方法，而 *排序 (ordering)* 和 *聚集 (clustering)* 选项则指的是行如何被推送或“流式传输”到每个并行函数实例中。你应该意识到，对行的分布以及排序/聚集施加更多控制会给你的进程增加开销，并降低你的加速比和扩展性。以下是这些选项的一些描述和示例：

### 排序 (ORDERING) 与 聚集 (CLUSTERING)

这些选项指的是数据在被分区或划分后，如何被传送到每个并行实例。可能通过例子解释最为清楚。

假设你的函数将处理来自数据集的以下数据子集：

`5,4,5,6,8,3,2,3,4,5,6,1,2,4,6`

*   **聚集 (Clustering)** 会以这样的方式传送值：相同的值会直接一个接一个地出现，但不一定是排序的。

    `6,6,6,3,3,8,1,4,4,4,5,5,5,2,2`

*   **排序 (Ordering)** 会对值进行完全排序。

    `1,2,2,3,3,4,4,4,5,5,5,6,6,6,8`

这两个选项都会给处理过程增加开销，因为它们是在并行函数实例接收数据之前执行的。对集合进行排序比聚集相同值需要更多时间。

### PARTITION BY ANY

如前所述，这种方法只是为每个并行函数实例提供源集合中的一组随机行。每个函数实例获得大致相同数量的行，但它们无法对收到的行做任何假设。

从技术上讲，你可以指定希望这些行按特定列值进行 `CLUSTERED`（或分组），甚至可以要求分配给每个并行函数的整个数据集完全 `ORDERED`。然而，由于你不知道哪些行会被发送到哪个函数实例，这些选项的价值有限。在行的随机分发下，你无法假设你的聚集中会包含所有匹配该簇的值，而且虽然你可以要求对行进行排序，但你不知道是否有分配到其他并行实例的行被遗漏了。

### PARTITION BY HASH

这种方法对你列出的列的值进行哈希运算，以创建要分配到各并行函数实例的数据子集。由于值的哈希结果是一致的，来自主数据集的各个值基本上会被分组在一起，并发送到每个函数实例。多个值可能会被发送到一个实例，并且它们可能不是相邻值，但你可以依赖这样一个事实：每个函数实例都将接收到全部的等价值集合。基本上，这就像在划分数据时使用了 `CLUSTERING` 选项。一个或多个完整的值簇将被发送到每个函数实例。这种方法在确保特定数据值保持在一起的同时，也提供了相对均匀的数据分布到每个函数。

数据被划分后，可以进一步进行 `CLUSTERED` 或 `ORDERED`，以供并行函数使用。

### PARTITION BY RANGE

这种方法在将数据分配到各并行函数实例之前先对其进行排序。这就像使用 `ORDERED` 选项来分配数据。它确保数据值会为每个函数分组在一起，并且函数只处理相邻的数据值。这里的风险是，如果你的数据存在任何极端偏斜的分布，你将会遇到不均衡的并行性——一些并行函数实例将比其他实例执行多得多的工作。与 `PARTITION BY HASH` 选项类似，数据可以进一步进行 `CLUSTERED` 或 `ORDERED`，以供并行函数使用。

## 选择并实施解决方案

在审视了这些选项后，你发现你需要确保特定字母的所有映射都被发送到同一个规约函数（为此你将使用 `PARTITION BY RANGE`），并且你希望在处理时字母是分组在一起的。为此，你将使用 `ORDERING` 或 `CLUSTERING`。首先，你将设置 `PARTITION BY RANGE`，并演示仅此一项是不够的。要做到这一点，你需要将你的 `SYS_REFCURSOR` 替换为强类型游标，该游标向数据库提供数据集中可用于区分哪些项可用于分发数据的信息。你将通过在类型包中声明一个类型化的 ref 游标来完成此操作，该游标描述由键/值对组成的游标。

```
create or replace package map_reduce_type as
  type key_value_pair is record (
    key_item    varchar2(32),
    value_item  number
  );
  type key_value_pairs is table of key_value_pair;
  type key_value_pair_cursor is ref cursor
    return key_value_pair;
end map_reduce_type;
/
```

现在，你将更新 `Reduce` 函数的声明，以表明输入的键/值对应划分为范围，这些范围将由你的 `Reduce` 函数使用。

```
create or replace package reduce_letter_count is

  function result_set
      (p_key_value_pairs in map_reduce_type.key_value_pair_cursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (
        partition p_key_value_pairs by range(key_item)
        );

end reduce_letter_count;
/

create or replace package body reduce_letter_count as
```


让我们来试试看吧！

```sql
create or replace package reduce_letter_count is

  function result_set
      (p_key_value_pairs in map_reduce_type.key_value_pair_cursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (
        partition p_key_value_pairs by range(key_item)
        ) is

    l_in1_key_value_pair map_reduce_type.key_value_pair;
    l_out_key_value_pair map_reduce_type.key_value_pair;

    begin

      fetch p_key_value_pairs into l_in1_key_value_pair;
      l_out_key_value_pair.key_item := l_in1_key_value_pair.key_item;
      l_out_key_value_pair.value_item := 0;

      loop
        exit when p_key_value_pairs%notfound;

        if l_out_key_value_pair.key_item = l_in1_key_value_pair.key_item
        then

          l_out_key_value_pair.value_item :=
            l_out_key_value_pair.value_item +
            l_in1_key_value_pair.value_item;

        else

          pipe row(l_out_key_value_pair);

          l_out_key_value_pair.key_item := l_in1_key_value_pair.key_item;
          l_out_key_value_pair.value_item := 1;

        end if;

        fetch p_key_value_pairs into l_in1_key_value_pair;

      end loop;

      pipe row(l_out_key_value_pair);

      return;

    end result_set;

end reduce_letter_count;
/
```

```sql
select * from table(
reduce_letter_count.result_set(
cursor(select * from table(
single_map_letter_count.result_set('Hello, world!')
))))
order by key_item;
```

```
KEY_ITEM                          VALUE_ITEM
-------------------------------- ----------------------
                                  1
!                                 1
,                                 1
H                                 1
d                                 1
e                                 1
l                                 2
l                                 1
o                                 1
o                                 1
r                                 1
w                                 1

12 rows selected
```

现在，尽管你已经适当地分配了 Map 的结果，但你仍然没有确保结果以任何特定的顺序发送给 Reducer。为了实现这一点，你需要添加 `ORDERING`（排序）或 `CLUSTERING`（聚类）选项。我们使用 `ORDERING` 选项，因为它最易于理解；我会让你自己尝试 `CLUSTERING` 选项。

## 使用 ORDERING 选项的示例

```sql
create or replace package reduce_letter_count is

  function result_set
      (p_key_value_pairs in map_reduce_type.key_value_pair_cursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (
        partition p_key_value_pairs by range(key_item))
        order p_key_value_pairs by (key_item);

end reduce_letter_count;
/
```

```sql
create or replace package body reduce_letter_count as

  function result_set
      (p_key_value_pairs in map_reduce_type.key_value_pair_cursor)
    return map_reduce_type.key_value_pairs pipelined
      parallel_enable (
        partition p_key_value_pairs by range(key_item))
        order p_key_value_pairs by (key_item)
        is

    l_in1_key_value_pair map_reduce_type.key_value_pair;
    l_out_key_value_pair map_reduce_type.key_value_pair;

    begin

      fetch p_key_value_pairs into l_in1_key_value_pair;
      l_out_key_value_pair.key_item := l_in1_key_value_pair.key_item;
      l_out_key_value_pair.value_item := 0;

      loop
        exit when p_key_value_pairs%notfound;

        if l_out_key_value_pair.key_item = l_in1_key_value_pair.key_item
        then

          l_out_key_value_pair.value_item :=
            l_out_key_value_pair.value_item +
            l_in1_key_value_pair.value_item;

        else

          pipe row(l_out_key_value_pair);

          l_out_key_value_pair.key_item := l_in1_key_value_pair.key_item;
          l_out_key_value_pair.value_item := 1;

        end if;

        fetch p_key_value_pairs into l_in1_key_value_pair;

      end loop;

      pipe row(l_out_key_value_pair);

      return;

    end result_set;

end reduce_letter_count;
/
```

```sql
select * from table(
reduce_letter_count.result_set(
cursor(select * from table(
single_map_letter_count.result_set('Hello, world!')
))))
order by key_item;
```

```
KEY_ITEM                          VALUE_ITEM
-------------------------------- ----------------------
                                  1
!                                 1
,                                 1
H                                 1
d                                 1
e                                 1
l                                 3
o                                 2
r                                 1
w                                 1

10 rows selected
```

成功了！现在，让我们用你的 100,000 个字符串及其产生的 110 万个键/值 Map 对来测试一下。

## 大规模测试

```sql
drop table documents;
```

```sql
create table documents
as
select rownum doc_id, column_name text from dba_tab_columns;
```

```sql
select /*+ parallel */ * from table(
reduce_letter_count.result_set(
cursor(select /*+ parallel */ * from table(
map_letter_count.result_set(cursor(select * from documents))
))))
order by key_item;
```

```
KEY_ITEM                          VALUE_ITEM
-------------------------------- ----------------------
                                  11
#                                 2522
$                                 457
-                                 4
0                                 2104
1                                 1755
2                                 1679
3                                 993
4                                 798
5                                 536
6                                 348
7                                 254
8                                 241
9                                 225
A                                 83113
B                                 20226
C                                 44506
D                                 47611
E                                 131562
F                                 13999
G                                 19681
H                                 12801
I                                 72349
J                                 4564
K                                 6081
L                                 47635
M                                 45084
N                                 71110
O                                 64789
P                                 40780
Q                                 4763
R                                 64882
S                                 66557
T                                 94510
U                                 33837
V                                 10060
W                                 10161
X                                 7168
Y                                 16269
Z                                 2083
_                                 84576
a                                 1
b                                 1
c                                 1
e                                 5
g                                 1
i                                 2
j                                 1
l                                 1
m                                 1
n                                 1
o                                 1
p                                 1
r                                 2
s                                 1
t                                 1
u                                 1
v                                 1

58 rows selected
```

它成功了！计时测试表明，随着数据库并行运行各个步骤，我能够相对于服务器上的 CPU 数量实现并行加速。从这里开始，你应该能够使用这种技术在 PL/SQL 中编写你自己的并行 MapReduce 函数，或者将现有的 MapReduce 算法转换为在你的 Oracle 数据库上运行。



#### 指导

仅仅因为您有能力在 PL/SQL 中编写并行 MapReduce 算法，并不总是意味着您应该使用它们。在许多情况下，使用单个 SQL 语句（包括使用并行性）来完成相同的结果所需的代码更少。您的整个示例可以写成如下形式：

```sql
select    /*+ parallel */
          letter key_item,
          count(*) value_item
from
(
select    substr(d.text,c.i,1) letter
from      documents d,
          (select level i from dual connect by level <= 30) c
)
where     letter is not null
group by  letter
order by  letter;
```

关键点在于，并非所有解决方案都需要通过像 MapReduce 这样的编程原语来实现。理解这种方法的真正价值在于，当您试图在其他并行编程环境之间来回转换时。通过理解如何在 PL/SQL 中实现并行算法，您可以更轻松地从其他语言和系统转换概念。

#### 并行管道表函数总结

您已经了解了如何编写像表一样工作的 PL/SQL 函数，这些函数以编程方式生成行，以及如何以并行方式调用这些函数。值得注意的是，这些函数可以将数据传递给其他也可以并行运行的 PL/SQL 表函数，从而构建一个数据并行转换和操作链，类似于许多 MapReduce 循环。

### 总结

Oracle 提供了许多不同的方法来使用 PL/SQL 实现并行编程算法。然而，如前所述，必须仔细考虑并行结构是否能为您提供所需的收益。并行选项`总是`比其串行对应版本消耗更多的系统资源，但在您有可用容量的情况下，它们提供了加速长时间运行活动或处理非常大数据集的能力。在将其他系统的并行算法进行转换时，这些功能也可能对您大有裨益。

截至 11gR2，并行管道表函数为您提供了最灵活和最成熟的方法，用于在独立的处理器之间以及在集群数据库环境中的集群节点之间分配工作。11gR2 中新增的 `DBMS_PARALLEL_EXECUTE` 框架也是一个引人注目的选项，标准版用户可以利用它来获得更高程度的编程控制——代价是在为并行化划分和排序数据方面灵活性有所降低。

## 第 4 章

## 警告和条件编译

作者：Torben Holm

当 Oracle 在 RDBMS 内核中实现 Java 时，人们认为这将是 PL/SQL 的终结。然而，在 10.2 版本中，Oracle 重写了 PL/SQL 编译器并实现了各种新功能，证明了 PL/SQL 是有未来的。本章将介绍首次出现在 Oracle 10.2 中的两个特性：PL/SQL 警告和 PL/SQL 条件编译。

### PL/SQL 警告

在 Oracle 10.2 中，Oracle 实现了 PL/SQL 警告功能。此功能可以在编译 PL/SQL 时获得警告，如果某些代码实现了不良实践或与保留字或 Oracle 函数冲突。在 Oracle 10.2 之前，您的 PL/SQL 有时会编译通过——却在运行时抛出错误。PL/SQL 警告的目的是在编译时更容易修复潜在错误时发出警告，从而确保应用于数据库的代码尽可能健壮和最优。

### 基础知识

在您开始了解从 PL/SQL 警告中能获得什么之前，先来看一下该功能的一些基本方面。默认情况下，PL/SQL 警告是禁用的。PL/SQL 警告可以在系统、会话或过程级别启用。PL/SQL 警告分为三类：

> `Severe`（严重）：这些警告针对与 SYS.STANDARD 或 SQL 函数中的函数存在冲突的代码。
>
> `Performance`（性能）：这些警告针对可能对性能产生影响的代码，例如在查询中使用错误的数据类型以及在适当情况下未使用 NOCOPY 编译指令。
>
> `Informational`（信息性）：这些警告与无害和/或甚至可能被移除的代码相关。

当您创建或编译存储过程时，它默认继承您运行时所在会话的设置。PL/SQL 警告级别由参数 `plsql_warnings` 的值决定。您可以按如下方式显示当前参数值：

```sql
SQL@V112> show parameter plsql_warnings

NAME                             TYPE        VALUE
-------------------------------- ----------- ------------------------------
plsql_warnings                   string      DISABLE:ALL

默认值是 DISABLE:ALL
```

如果您无权执行 SHOW PARAMETER 命令，可以使用以下查询检查 `plsql_warnings` 的值：

```sql
SQL@V112> SELECT DBMS_WARNING.GET_WARNING_SETTING_STRING FROM DUAL;

GET_WARNING_SETTING_STRING
----------------------------------------------------------------
DISABLE:ALL
```

要为当前会话启用所有警告，请执行以下操作：

```sql
SQL@V112> alter session set plsql_warnings='ENABLE:ALL';

Session altered.
```

![images](img/square.jpg) `注意` 在大多数情况下，您需要 `ALTER SESSION` 权限才能更改会话参数。但在更改 `plsql_warnings` 参数时并非如此。

此外，`plsql_warnings` 字符串不区分大小写。

要一次启用严重和性能警告并禁用信息性警告，请执行以下语句：

```sql
SQL@V112> alter session set plsql_warnings='ENABLE:SEVERE,ENABLE:PERFORMANCE,
DISABLE:INFORMATIONAL';

Session altered.
```

参数设置不是累积的。如果您先将 `plsql_warnings` 设置为 `ENABLE:SEVERE`，然后再设置 `ENABLE:PERFORMANCE`，那么只有与性能相关的警告会被启用。

如果您想从 PL/SQL 内部设置参数，可以通过调用 `DBMS_WARNINGS.SET_WARNING_SETTING_STRING` 来实现。启用一个设置将会禁用其他设置。此示例将在会话级别仅启用严重警告：

```sql
SQL@V112> EXEC DBMS_WARNING.SET_WARNING_SETTING_STRING('ENABLE:SEVERE', 'SESSION');

PL/SQL procedure successfully completed.
```

启用警告不会影响执行时间，但可能会稍微影响编译时间。如果您编译一个过程 1000 次，您会发现启用警告编译和未启用警告编译之间的差异大约是一秒，未启用的编译更快。

如果一个过程在编译时出现编译警告，该过程仍然可以执行。警告是为了提醒您，您的代码中可能存在问题。但编译器并非全知全能。并非所有警告都是问题。您始终可以自由执行您的代码。

![images](img/square.jpg) `提示` 如果您想检查可能遇到的警告类型，并且可以访问数据库服务器上的 Oracle 二进制文件，可以查看文件 `$ORACLE_HOME/plsql/mesg/plw<LANG>.msg`。此文件包含所有的警告信息。


### 使用警告

让我们来看几个例子。首先，创建一个名为 `T1` 的简单表，其中包含一个类型为 `VARCHAR2` 的 `KEY` 列和一个名为 `VALUE` 的 `VARCHAR2` 列。假设 `KEY` 列虽然定义为 `VARCHAR2`，但只包含数字。不幸的是，你在现实世界的生产系统中有时会看到这种做法。以下是创建表的代码：

```sql
SQL@V112> CREATE TABLE T1 (
2 KEY VARCHAR2(10) CONSTRAINT PK_T1 PRIMARY KEY,
3 VALUE VARCHAR2(10))
4 /

Table Created.
```

现在，向表中加载测试数据。

```sql
SQL@V112> insert into t1 select rownum, substr(text,1,10) from all_source where rownum < 10000;

9999 rows created.
```

然后启用严重和性能相关的 PL/SQL 警告。

```sql
SQL@V112> alter session set plsql_warnings='ENABLE:SEVERE, ENABLE:PERFORMANCE';

Session altered.
```

接下来，创建一个函数，该函数将根据给定的键返回对应的 `VALUE`。

```sql
SQL@V112> CREATE OR REPLACE FUNCTION GET_T1_KEY (P_KEY NUMBER) RETURN VARCHAR2 IS
2   l_value T1.VALUE%TYPE;
3  BEGIN
4     SELECT VALUE INTO l_value FROM T1 WHERE KEY = P_KEY;
5     RETURN l_value;
6     EXCEPTIONS
7     WHEN NO_ROWS_FOUND THEN
8       RETURN 'No value found';
9  END;
10 /

SP2-0804 Procedure created with compilation warnings.
```

该过程编译时带有警告。要在 SQL*Plus 中显示警告，只需执行 `show error`，如下所示：

```sql
SQL@V112> show err
LINE/COL ERROR
-------- -----------------------------------------------------------------
1/1      PLW-05018: unit GET_T1_KEY omitted optional AUTHID clause;
         default value DEFINER used

4/45     PLW-07204: conversion away from column type may result in
         sub-optimal query plan
```

![images](img/square.jpg) **注意** 如果你检查 `USER_ERRORS` 表中的 `ATTRIBUTE` 和 `MESSAGE_NUMBER`，你会发现 `ATTRIBUTE` 要么是 `ERROR` 要么是 `WARNING`，而错误/警告编号则在 `MESSAGE_NUMBER` 列中。

尽管函数编译时带有警告，你仍然可以执行它。

```sql
SQL@V112> var txt varchar2(10);
SQL@V112> exec :txt := GET_T1_KEY(1);

PL/SQL procedure successfully completed.

Elapsed: 00:00:00.14
```

现在，`KEY` 列是否应该或不应该更改为 `NUMBER` 并不是当前的问题。在本例中，你需要修改函数。因此，将 `P_KEY` 参数更改为 `VARCHAR2`，并通过向函数添加 `AUTHID DEFINER` 来消除 `PLW-05018` 警告。以下是函数的新版本：

```sql
SQL@V112> CREATE OR REPLACE FUNCTION GET_T1_KEY (P_KEY VARCHAR2) RETURN VARCHAR2
2  AUTHID DEFINER
3  IS
4    L_value T1.VALUE%TYPE;
5  BEGIN
6     SELECT VALUE INTO l_value FROM T1 WHERE KEY = P_KEY;
7     RETURN l_value;
8     EXCEPTIONS
9     WHEN NO_ROWS_FOUND THEN
10      RETURN 'No value found';
11 END;
12 /

Function created
```

这个新版本的函数创建时没有错误或警告。请注意，由于现在使用了正确的数据类型，并且函数使用索引来查找键，因此执行速度更快（在此例中快了一倍）。以下是一个执行示例：

```sql
SQL@V112> exec :txt := GET_T1_KEY('1');

PL/SQL procedure successfully completed.

Elapsed: 00:00:00.07
SQL@V112>
```

`PLW-7204` 警告告诉你，当从表 `T1` 中选择时，Oracle 将不得不进行数据类型转换。当使用参数值时，Oracle 将不得不进行数据类型转换，因为参数的数据类型是 `NUMBER`，而列的数据类型是 `VARCHAR2`。当 Oracle 进行数据类型转换时（例如 `to_number`、`to_char`、`to_date` 等），可能会导致索引（如果查询列上存在索引的话）被忽略。这种忽略索引的情况解释了前面两个示例之间的时间差异。

你可以对代码进行性能分析，以检查在第一个示例中代码相对于第二个示例在哪些地方花费了更多时间，但由于代码很简单，你只需通过查看 `SELECT` 语句来了解发生了什么。首先为会话设置 `AUTOTRACE ON`，然后根据表中列的数据类型正确地将 `KEY` 值作为字符使用。

```sql
SQL@V112> select value from T1 where key = '1';

VALUE
----------
package ST

Elapsed: 00:00:00.01
SQL@V112>
```

检查执行计划显示索引 `PK_T1` 被使用了。

```sql
-------------------------------------------------------------------------------------
| Id  | Operation                   | Name  | Rows  | Bytes | Cost (%CPU)| Time     |
-------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |       |     1 |    14 |     1   (0)| 00:00:01 |
|   1 |  TABLE ACCESS BY INDEX ROWID| T1    |     1 |    14 |     1   (0)| 00:00:01 |
|*  2 |   INDEX UNIQUE SCAN         | PK_T1 |     1 |       |     1   (0)| 00:00:01 |
-------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - access("KEY"='1')
```

在下一个例子中，`KEY` 值没有根据表中列的数据类型正确使用，但如果我当初忽略了警告，它本应被正确使用。

```sql
SQL@V112> select value from t1 where key = 1;

VALUE
----------
package ST

Elapsed: 00:00:00.20
```

该语句的执行计划现在显示索引未被使用，Oracle 进行了全表扫描。注意谓词信息中的过滤器；Oracle 向 `KEY` 列添加了一个 `TO_NUMBER`，因此索引未被使用。如果 `KEY` 列中存在字符（正如定义所允许的），该函数将因 `ORA-01722: invalid number` 错误而失败。以下是计划输出：

```sql
--------------------------------------------------------------------------
| Id  | Operation         | Name | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |      |     1 |    14 |     2   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| T1   |     1 |    14 |     2   (0)| 00:00:01 |
--------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   1 - filter(TO_NUMBER("KEY")=1)

SQL@V112>
```

正如你在此示例中看到的，正确定义的函数比参数数据类型错误的函数大约快 20 倍。如果警告没有被启用，你可能不会注意到错误的参数数据类型，因此也不会修正该函数。你可能会注意到，一开始你无法区分一个警告是信息性、性能性还是严重性警告。确定警告类别的方法是执行 `dbms_warning.get_category` 函数，如下所示：

```sql
SQL@V112> SELECT DBMS_WARNING.GET_CATEGORY(7204) FROM DUAL;
    DBMS_WARNING.GET_CATEGORY(7204)
    -----------------------------------------------------------------------
    PERFORMANCE
```

当你使用警告时，你会发现严重警告的范围是 5000 到 5999，信息性警告的范围是 6000 到 6249，性能性警告的范围是 7000 到 7249。一旦你知道了这些范围，根据编号对警告进行分类就变得容易了。


### 将警告提升为错误

如前所述，即使您的函数或过程是在有警告的情况下编译的，您仍然可以执行它们。如果您希望确保包含特定警告的存储过程不会进入您的生产系统，您可以将这些警告提升为错误。

以下代码展示了将 `GET_T1_KEY` 函数的参数类型编译为 `NUMBER` 时的情况。请注意生成的警告。

```
SQL@V112> CREATE OR REPLACE FUNCTION GET_T1_KEY (P_KEY NUMBER) RETURN VARCHAR2
  2  AUTHID DEFINER
  3  IS
  4     L_value T1.VALUE%TYPE;
  5  BEGIN
  6      SELECT VALUE INTO l_value FROM T1 WHERE KEY = P_KEY;
  7      RETURN l_value;
  8  EXCEPTIONS
  9  WHEN NO_ROWS_FOUND THEN
 10    RETURN 'No value found';
 11 END;
 12 /
SP2-0806: Function created with compilation warnings

SQL@V112> show err
LINE/COL ERROR
-------- -----------------------------------------------------------------
6/49     PLW-07204: conversion away from column type may result in
         sub-optimal query plan
```

为确保此类代码不被部署——至少不是在无错误的情况下部署——您可以将此警告提升为错误。首先，更改会话以禁用所有警告（在本例中这样做是为了专注于要提升的那个警告）。在同一语句中，将 `PLW-07204` 警告提升为错误而非警告，如下所示：

```
SQL@V112> alter session set plsql_warnings='DISABLE:ALL, ERROR:7204';

Session altered.
```

然后重新编译 `GET_T1_KEY` 函数。

```
SQL@V112> alter function GET_T1_KEY compile;

Warning: Function altered with compilation errors.

SQL@V112> show err
Errors for FUNCTION GET_T1_KEY:

LINE/COL ERROR
-------- -----------------------------------------------------------------
6/45     PLS-07204: conversion away from column type may result in
         sub-optimal query plan
```

现在尝试执行该函数。这将导致错误。

```
SQL@V112> exec :txt := get_t1_key(1);
BEGIN :txt := get_t1_key(1); END;

            *
ERROR at line 1:
ORA-06550: line 1, column 13:
PLS-00905: object ACME.GET_T1_KEY is invalid
ORA-06550: line 1, column 7:
PL/SQL: Statement ignored
```

请注意 `PLW-07204` 变成了 `PLS-07204`。该警告现在已成为错误，您将无法再执行该函数。要能够执行该函数，您必须通过将参数类型更改为 `VARCHAR2` 来消除此错误。

如果您有一个现有的包含 PL/SQL 的应用程序，并且启用了警告并重新编译了 PL/SQL 代码，您可能会得到许多警告。尽管没有任何警告感觉很好，但由于这样或那样的原因，可能仍然无法消除所有警告。对于无法消除的警告，您可以自由选择忽略它们。

![images](img/square.jpg) **注意** 很容易将消除所有警告设定为一个硬性目标。非技术经理尤其容易陷入这种诱惑，因为他们常常不理解警告和错误之间的微妙区别。请记住，警告消息来自机器试图根据他人制定的一套最佳实践来解释您的代码。因此，警告指向的是值得更仔细检查的代码。然而，有时您确实需要执行某个特定操作，如果让编译器迫使您盲目遵循他人的最佳实践，那才真正是*糟糕的做法*。

警告的另一个小问题是它们会弄乱 `USER_ERRORS` 视图。警告和错误记录在同一个视图中。如果您习惯于通过检查 `USER_ERRORS` 视图是否为空来判断您刚刚编译的一段 PL/SQL 代码是否有错误，这可能会令人烦恼。您无法消除这种小困扰。您必须忍受它，但这是为警告功能带来的价值所付出的小小代价。

### 忽略警告

您可能由于某种原因希望忽略特定的警告。在下一个例子中，启用了所有警告。但其中一个警告无法通过更改代码来消除，因为该警告涉及列名与保留字冲突，并且您希望忽略这个特定的信息性警告。要启用所有警告，请将 `plsql_warnings` 设置为 `ENABLE:ALL`，如下所示：

```
SQL@V112> alter session set plsql_warnings='ENABLE:ALL';
```

接下来，创建以下版本的 `GET_T1_KEY` 函数：

```
SQL@V112>CREATE OR REPLACE FUNCTION GET_T1_KEY (P_KEY NUMBER) RETURN VARCHAR2
2  AUTHID DEFINER
3  IS
       l_value T1.VALUE%TYPE;
       l_1 NUMBER :=1;
       l_2 NUMBER :=2;
5  BEGIN
        IF l_1 = l_2 THEN
          SELECT VALUE INTO l_value FROM T1 WHERE KEY = P_KEY;
          RETURN l_value;
       END IF;
12  END;
13  /
SP2-0806: Function created with compilation warnings

SQL@V112> show err
Errors for FUNCTION GET_T1_KEY:

0/0      PLW-06010: keyword "VALUE" used as a defined name
1/1      PLW-05005: subprogram GET_T1_KEY returns without value at line 11
9/11     PLW-06002: Unreachable code
9/51     PLW-07204: conversion away from column type may result in
         sub-optimal query plan
```

让我们逐一查看这些警告。

*   `PLW-06010`：一个 PL/SQL 或 SQL 单词被用作定义的名称。这是合法的，但不推荐。可以在 `V$RESERVED_WORDS` 视图中查看保留字。
*   `PLW-05005`：Oracle 发现您从未进入 `IF` 语句，因此不会从函数中返回值。
*   `PLW-06002`：同样，Oracle 发现您从未进入 `IF` 语句，因此将存在不可达代码。不可达代码本身无害，但这可能表明代码不当。执行 PL/SQL 时，过多的不可达代码会占用内存。
*   `PLW-07204`：嗯，您现在已经知道该怎么处理它了。

那么，修复代码并消除这些警告。假设由于某种原因，您无法更改导致 `PLW-06010` 的列名 `VALUE`；例如，您的应用程序可能很旧，如果更改列名，将有太多代码需要修复。

首先要做的是准备会话。

```
SQL@V112> alter session set plsql_warnings='ENABLE:ALL, ERROR:7204, DISABLE:6010';
```

在这种情况下，启用了所有警告，警告 `7204` 仍被提升为错误，而警告 `6010` 被禁用。现在可以重新应用该函数并进行编译，不再出现错误或警告。

如果您想检查某个存储过程编译时使用了哪些 PL/SQL 警告，可以发出以下查询：

```
SQL@V112> col plsql_warnings for a40
SQL@V112> SELECT NAME, PLSQL_WARNINGS FROM USER_PLSQL_OBJECT_SETTINGS;

NAME                         PLSQL_WARNINGS
------------------------------ ----------------------------------------
GET_T1_KEY                     ENABLE:ALL,DISABLE:  6010,ERROR:  7204
```

这些结果显示了 `GET_T1_KEY` 编译时生效的 `PLSQL_WARNINGS` 参数值。


## 编译与警告

每当您创建或重新编译存储过程时，它们会继承当前会话的设置。因此，如果您在未设置警告的会话中意外编译了存储过程，该过程将正常编译，但设置会丢失。如果您的数据库配置了系统范围的警告（`ALTER SYSTEM SET PLSQL_WARNINGS='ENABLE:ALL, ERROR:7204'`），并且其中一些警告已被提升为错误，结果可能会更麻烦，因为重新编译的过程将会失败。要使用当前设置重新编译过程，请执行以下命令：

```
SQL@V112> ALTER FUNCTION GET_T1_KEY COMPILE REUSE SETTINGS;

Function Compiled.
```

您无需在每次想用特定`PL/SQL`警告编译存储过程时都执行一次`ALTER SESSION`，而是可以在编译时直接指定警告设置。

```
SQL@V112> ALTER FUNCTION GET_T1_KEY COMPILE PLSQL_WARNINGS='ENABLE:ALL,
DISABLE:6010,ERROR: 7204';

Function Compiled.
```

如果您对所有存储过程有通用的警告设置，可以添加一个登录触发器在登录时设置`PLSQL_WARNINGS`。开发者共享同一个数据库用户来开发应用程序的情况并不少见。在这种情况下，触发器可以相当简单，并且仅限于该用户。

在这种情况下，触发器应在登录到`ACME`模式后触发。触发器的作用是执行`ALTER SESSION`，从而设置`PLSQL_WARNINGS`参数。

以`SYS`身份连接并创建触发器。

```
SQL@V112> CREATE OR REPLACE TRIGGER dev_logon_trigger
  2  AFTER LOGON ON ACME.SCHEMA
  3  BEGIN
  4    EXECUTE IMMEDIATE q'"ALTER SESSION SET PLSQL_WARNINGS='ENABLE:SEVERE,ENABLE:PERFORMANCE,ERROR:7204'"';
  5  END;
  6  /

Trigger created.
```

然后以`Acme`用户身份连接。

```
SQL@V112> conn ACME/acme
Connected.
```

检查参数是否已设置。

```
SQL@V112> SELECT DBMS_WARNING.GET_WARNING_SETTING_STRING FROM DUAL;

GET_WARNING_SETTING_STRING
------------------------------------------------------------------------
DISABLE:INFORMATIONAL,ENABLE:PERFORMANCE,ENABLE:SEVERE,ERROR:  7204
```

如果您的应用程序有许多存储过程且它们有不同的设置，构建一个框架来帮助您记住各自的设置并用于重新编译可能是个好主意（请参阅即将在“条件编译”一节中的小框架示例）。

如果您想使用`DBMS_UTILITY.COMPILE_SCHEMA`过程编译模式中的所有存储过程，您应该注意参数`reuse_settings`默认设置为`FALSE`。以下是`DBMS_UTILITY.COMPILE_SCHEMA`过程的描述：

```
PROCEDURE COMPILE_SCHEMA
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 SCHEMA                         VARCHAR2                IN
 COMPILE_ALL                    BOOLEAN                 IN     DEFAULT
 REUSE_SETTINGS                 BOOLEAN                 IN     DEFAULT
```

如果您不应用`reuse_settings`参数并将其设置为`TRUE`，所有已编译的包将丢失任何警告设置。因此，在执行`DBMS_UTILITY.COMPILE_SCHEMA`时，请记得将`reuse_settings`设置为`TRUE`，如下所示：

```
SQL@V112> exec DBMS_UTILITY.COMPILE_SCHEMA (schema=>'ACME', reuse_settings=>true);

PL/SQL procedure successfully completed.
```

![images](img/square.jpg) 注意 如果在`SYSTEM`级别设置`plsql_warnings`，您应谨慎行事。尽管您不能使用`DBMS_UTILITY.COMPILE_SCHEMA`编译`SYS`，但您仍然可以编译单独的包。在`SYSTEM`级别设置`plsql_warnings`，然后只编译几个包，可能会导致数百条警告，或者，如果您已将某些警告提升为错误，则会引发错误。

作为`SYS`或`SYSTEM`，您可以运行实用程序`utlrp`（`@?/rdbms/admin/utlrp`），它将编译所有无效的存储过程。`Utlrp`调用`UTL_RECOMP.RECOMP_PARALLEL`并行编译存储过程。执行此操作时，被提升为错误的警告或被忽略的警告将被剥离，如下例所示。这是一个错误。

首先，设置会话`plsql_warnings`

```
SQL@V112> alter session set plsql_warnings='Enable:all, ERROR:7204, DISABLE:6010';

Session altered.
```

并编译选定的函数。

```
SQL@V112> alter function GET_T1_KEY compile;

Warning: Function altered with compilation errors.
```

接下来，看看函数设置。

```
SQL@V112> SELECT NAME, PLSQL_WARNINGS FROM USER_PLSQL_OBJECT_SETTINGS WHERE NAME =
'GET_T1_KEY';

NAME                         PLSQL_WARNINGS
---------------------------- -------------------------------------------------------------
---------------------------------------
GET_T1_KEY                   ENABLE:ALL,DISABLE:  6010,ERROR:  7204
```

现在，以`SYS`身份连接

```
SQL@V112> conn / as sysdba
Connected.
```

并执行重新编译，通过调用`utlrp.sql`或直接执行`UTL_RECOMP.RECOMP_PARALLEL`。

```
SQL@V112> exec utl_recomp.recomp_parallel(1,'ACME');

PL/SQL procedure successfully completed.
```

现在重新连接

```
SQL@V112> conn ACME/ACME
Connected.
```

并检查对象设置。

```
SQL@V112> SELECT NAME, PLSQL_WARNINGS FROM USER_PLSQL_OBJECT_SETTINGS
WHERE NAME = 'GET_T1_KEY';

NAME                         PLSQL_WARNINGS
---------------------------- -------------------------------------------------------------
---------------------------------------
GET_T1_KEY                   ENABLE:ALL

ACME@V112>
```

请注意，特定的警告设置已被删除；这一定是个错误。这不是灾难，因为一切仍然正常运行，但这很烦人，因为您现在不知道是否设置了特殊的警告。

如果您使用`exp`和`imp`导出和导入模式，任何存储的代码都将带着导出时的设置被导入。

### 关于警告的最后说明

您可能遇到的其他类型警告包括：当您将受益于`NOCOPY`编译器指令时，却没有使用它的警告。在 Oracle 11g 中，对于异常`WHEN OTHERS`没有导致`RAISE`或`RAISE_APPLICATION_ERROR`的情况，有一个新的、非常有用的警告。我个人曾花费数小时疑惑为什么在某些情况下没有得到错误，结果却发现有人写了`WHEN OTHERS THEN NULL`。

### 条件编译

条件编译与`PL/SQL`警告无关，但它与`PL/SQL`警告在同一时间实现。条件编译是通过预处理源代码完成的，您可能从其他编程语言中了解这一点。简而言之，它使您能够在编译时打开或关闭代码，而无需编辑代码。这可能是打开或关闭调试选项，或者启用与某个`Oracle`版本相关的代码，或者您应用程序的两个不同版本，例如标准版和企业版。

![images](img/square.jpg) 注意 条件编译已向后移植到 Oracle 10.1.0.4 和 9.2.0.6，可以通过设置参数`_conditional_compilation`来启用。


### 基础知识

条件编译仅包含少量指令。指令以 `$`（美元符号）开头，用于控制程序流程或引发错误。

系统定义的指令包括：`$IF`、`$THEN`、`$ELSIF`、`$ELSE` 和 `$END`。可以使用 `$ERROR … $END` 指令引发自定义的编译错误。可以查询静态布尔表达式或静态常量。除了创建自定义的查询变量外，Oracle 还提供了一些可用的内置查询变量，后续将会看到。

入门条件编译最简单的方法是从一个例子开始。此示例将展示如何为存储过程启用计时信息日志记录，该过程用于测量运行过程所需的时间以及过程运行的时间。以下是示例代码（行号已标出以便后续引用）：

```sql
1 CREATE OR REPLACE PROCEDURE LEVEL2 AS
2 $IF $$TIMING_ON $THEN
3 $ELSIF NOT $$TIMING_ON $THEN
4 $ELSE
5    $ERROR ' The PLSQL_CCFLAG TIMING_ON, Must be set to either FALSE or TRUE before
compileing procedure ' ||  $$PLSQL_UNIT||'.'
6    $END
7 $END
8 $IF $$TIMING_ON $THEN
9    L_start_time TIMESTAMP;
10 $END
11 BEGIN
12 DBMS_APPLICATION_INFO.SET_MODULE($$PLSQL_UNIT, 'RUN');
13 $IF $$TIMING_ON $THEN
14    L_START_TIME:=CURRENT_TIMESTAMP;
15 $END
16 -- -----------------------------------------------
17 -- Application logic is left out
18 -- -----------------------------------------------
19 $IF $$TIMING_ON $THEN
20   LOGIT (P_PG=>$$PLSQL_UNIT, P_START_TIME=>l_start_time, P_STOP_TIME=>CURRENT_TIMESTAMP);
21 $END
22 DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
23 END LEVEL2;
24 /
```

以下是对代码清单的解释：

*   第 2-7 行：如果查询变量 `$$TIMING_ON` 在编译时未设置（即既不是 `FALSE` 也不是 `TRUE`），则将引发错误，并指出存储过程的名称以及发生错误的位置。注意，允许使用空语句——一个 `$IF … $THEN` 后面可以没有任何操作。
*   第 8 行：如果查询变量 `$$TIMING_ON` 等于 `TRUE`，则声明变量 `l_start_time`。注意，`$END` 后面没有像 PL/SQL 中的 `END IF` 那样的 `IF`，也没有分号。
*   第 12 行：这里调用 `DBMS_APPLICATION_INFO.SET_MODULE` 过程，该过程将设置 `V$SESSION` 视图中的 `MODULE` 列；你使用 Oracle 查询变量 `$$PLSQL_UNIT` 来保存存储过程的名称，或者如果它在包（`PACKAGE`）中被引用，则保存包的名称。在之前的 Oracle 版本中，你必须创建一个函数来执行 `OWA_UTIL.WHO_CALLED_ME` 过程（该过程返回调用者的名称），然后调用该函数来获取名称。调用 `DBMS_APPLICATION_INFO.SET_MODULE` 非常有用，如果你需要查询 `V$SESSION` 来定位你的会话，或者你使用扩展跟踪（Extended Trace）以便在过程启动时开始跟踪。

**注意**：如果使用 SQL*Developer 开发存储过程，可以修改保存 `CREATE PROCEDURE/FUNCTION` 模板的模板，以包含对 `DBMS_APPLICATION_INFO.SET_MODULE($$PLSQL_UNIT, '<action>')` 的调用，确保其始终被包含。这可以通过菜单 “Tools” -> “Preferences” -> “Database:SQL Code Editor Templates” 来完成。

*   第 13 行：如果查询变量 `$$TIMING_ON` 等于 `TRUE`，则在变量 `l_start_time` 中注册开始时间戳。
*   第 16-18 行：这里将是 PL/SQL 代码。
*   第 19 行：如果查询变量 `$$TIMING_ON` 等于 `TRUE`，则调用第 20 行的 `LOGIT` 过程。`LOGIT` 过程是记录计时信息的日志过程。该过程在此不描述，因为它并不重要。
*   第 22 行：清除 `DBMS_APPLICATION_INFO` 的调用。

在将过程添加到数据库之前，需要设置编译器标志 `TIMING_ON`。可以在系统、会话或程序级别设置此标志。在此示例中，它在会话级别设置。在设置标志之前，尝试将代码添加到数据库以触发 `$ERROR` 错误（假设代码保存在名为 `level2.sql` 的文件中）。

`SQL@V112> @level2`

`Warning: Procedure created with compilation errors.`

执行 `show error` 来显示错误，如下所示：

`SQL@V102> show error`

`Errors for PROCEDURE LEVEL2:`

`LINE/COL ERROR`
`-------- -----------------------------------------------------------------`
`5/4      PLS-00179: $ERROR:  The PLSQL_CCFLAG TIMING_ON, Must be set to`
`         either FALSE or TRUE before compileing procedure LEVEL2.`

现在设置标志，如下所示：

`SQL@V112> ALTER SESSION SET plsql_ccflags='TIMING_ON:TRUE';`

`plsql_ccflags` 的数据类型可以是布尔静态表达式、`PLS_INTEGER` 静态表达式或 `VARCHAR2` 静态表达式，具体取决于你分配给它的值。注意，设置 `plsql_ccflags` 不会强制重新编译任何代码；必须发出 `ALTER PROCEDURE … COMPILE` 命令。

`SQL@V102> ALTER PROCEDURE LEVEL2 COMPILE;`

`Procedure altered.`

**注意**：请记住，在设置了会话 `plsql_ccflags` 之后编译的每个存储过程都将继承该 `plsql_ccflags` 设置。

要检查在存储过程编译期间 `plsql_ccflags` 被设置为什么，请执行以下查询：

`SQL@V102> col plsql_ccflags for a80 wrap`
`SQL@V102> select NAME, PLSQL_CCFLAGS FROM USER_PLSQL_OBJECT_SETTINGS;`

`NAME                         PLSQL_CCFLAGS`
`------------------------------ --------------------------------------------`
`LEVEL1`
`LEVEL2                       TIMING_ON:TRUE`
`LOGIT`
`…`

如下例所示，如果一次设置了多个 `plsql_ccflag`，它们必须用逗号分隔。`plsql_ccflag` 值不区分大小写。

`SQL@V112> ALTER SESSION SET plsql_ccflags='TIMING_ON:TRUE,Standard:True';`



#### 代码的哪一部分正在运行？

在“基础”部分刚刚展示的简单过程中，无论 `TIMING_ON` 标志是设置为 `TRUE` 还是 `FALSE`，都很容易了解实际运行的是哪些代码的概览。`USER_SOURCE` 视图会显示存储过程中的所有代码。您创建的条件编译代码可能非常复杂，因此要查看实际编译或未编译的代码，您应该调用 `DBMS_PREPROCESSOR.PRINT_POST_PROCESSED_SOURCE` 过程。此过程会生成实际编译的代码（请记住设置 `serveroutput on`）。

```sql
SQL@V112> CALL DBMS_PREPROCESSOR.PRINT_POST_PROCESSED_SOURCE('PROCEDURE', 'ACME','LEVEL2');
PROCEDURE LEVEL2 AS
L_start_time TIMESTAMP;
BEGIN
DBMS_APPLICATION_INFO.SET_MODULE( 'LEVEL2'          , 'RUN');
L_start_time:=CURRENT_TIMESTAMP;
-- -----------------------------------------------
-- 应用逻辑部分已省略
-- -----------------------------------------------
LOGIT (P_PG=> 'LEVEL2'         , P_START_TIME=>l_start_time, P_STOP_TIME=>CURRENT_TIMESTAMP);
DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
END LEVEL2;

调用完成。
```

呈现的代码不包含您可能应用于代码的任何格式。在预处理后的代码中不存在的数据库对象，在 `USER_DEPENDENCIES` 视图中将不可见。经过包装的代码保持包装状态，您无法使用此过程解包代码。（您可以在第 15 章中阅读更多关于依赖关系的内容。）

如果您想自己处理预处理后的代码，可以调用 `DBMS_PREPROCESSOR.GET_POST_PROCESSED_SOURCE` 函数，该函数将返回一个包含代码的 `VARCHAR2` 类型表。格式丢失的原因是该包使用了 `DBMS_OUTPUT.PUT_LINE`。`PUT_LINE` 过程会移除前导空格。如果您创建一个类似下面这样的过程，在代码左侧添加行号（或任何字符）将确保格式不会丢失。以下过程——`PRINT_SOURCE`——将根据给定的参数，打印带有格式和行号的预处理后代码：

```sql
SQL@V112> CREATE OR REPLACE PROCEDURE PRINT_SOURCE (P_OBJECT_TYPE VARCHAR2, P_SCHEMA_NAME
  VARCHAR2, P_OBJECT_NAME VARCHAR2)  IS
2   l_source_lines DBMS_PREPROCESSOR.source_lines_t;
3  BEGIN
4    l_source_lines := DBMS_PREPROCESSOR.GET_POST_PROCESSED_SOURCE(P_OBJECT_TYPE,
P_SCHEMA_NAME, P_OBJECT_NAME);
5    FOR I IN l_source_lines.FIRST .. l_source_lines.LAST LOOP
6        IF TRIM(SUBSTR(l_source_lines (i),1, length(l_source_lines (i))-1)) IS NOT NULL
THEN
7          DBMS_OUTPUT.PUT_LINE(RPAD(I,4, ' ')||REPLACE(l_source_lines(i), CHR(10),''));
8        END IF;
9    END LOOP;
10  END;
11  /
```

使用给定参数执行该过程会产生以下输出：

```sql
SQL@V112> set serveroutput on
SQL@V112> exec PRINT_SOURCE ('PROCEDURE','ACME','LEVEL2');

1   PROCEDURE LEVEL2 AS
8      l_start_time TIMESTAMP;
10  BEGIN
11  DBMS_APPLICATION_INFO.SET_MODULE( 'LEVEL2'          , 'RUN');
13      l_start_time:=CURRENT_TIMESTAMP;
15  -- -----------------------------------------------
16  -- 应用逻辑部分已省略
17  -- -----------------------------------------------
19    LOGIT (P_PG=> 'LEVEL2'         , P_START_TIME=> l_start_time,
P_STOP_TIME=>CURRENT_TIMESTAMP);
21  DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
22  END LEVEL2;
```

#### 预处理代码的好处

通过预处理代码来排除部分代码的好处之一是，执行代码所使用的内存量可能会减少，从而在共享池中使用更少的内存。例如，假设过程 `LEVEL2` 的编码如下：

```sql
SQL@V112> CREATE OR REPLACE PROCEDURE LEVEL2 (TIMING_ON BOOLEAN := FALSE) AS
2  l_start_time TIMESTAMP;
3  BEGIN
4  DBMS_APPLICATION_INFO.SET_MODULE('LEVEL2' ,NULL);
5     IF TIMING_ON THEN
6        l_start_time:=CURRENT_TIMESTAMP;
7     END IF;
8     -- -----------------------------------------------
   -- 应用逻辑部分已省略
10  -- -----------------------------------------------
11  IF TIMING_ON THEN
12     LOGIT (P_PG=>'LEVEL2', P_START_TIME=>l_start_time,
P_STOP_TIME=>CURRENT_TIMESTAMP);
13  END IF;
14  DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
15  END LEVEL2;
16  /
```

此示例展示了您可能如何编写一个普通的存储过程，其中您会启用一些计时测量。视图 `USER_OBJECT_SIZE` 将显示给定程序所需内存（以字节为单位）的估计值。但是，您无法看到程序在运行时需要多少内存，因为运行时内存使用取决于您如何编写代码、是否进行批量收集以及其他类似因素。以下是 `LEVEL2` 的内存使用估计：

```sql
SQL@V112> SELECT * FROM USER_OBJECT_SIZE WHERE NAME = 'LEVEL2';

NAME           TYPE          SOURCE_SIZE PARSED_SIZE  CODE_SIZE ERROR_SIZE
-------------- ------------- ----------- ----------- ---------- ----------
LEVEL2         PROCEDURE               521         996        510          0
```

`SOURCE_SIZE` 列显示给定过程的源代码大小。源代码在编译期间必须加载到内存中。`PARSED_SIZE` 是当其他已编译的程序引用它时，给定程序所需的内存量。`CODE_SIZE` 是程序执行所需内存的起始大小。最后，`ERROR_SIZE` 列是编译期间错误信息在内存中占用的字节数。在本例中，`ERROR_SIZE` 为 0，因为编译期间没有发生错误。

假设您创建了相同的过程，但让一个 PL/SQL 编译器标志来控制计时测量调用，使代码看起来像这样：

```sql
SQL@V112> CREATE OR REPLACE PROCEDURE LEVEL2 AS
2  $IF $$TIMING_ON $THEN
3     l_start_time TIMESTAMP;
4  $END
5  BEGIN
6  DBMS_APPLICATION_INFO.SET_MODULE($$PLSQL_UNIT ,NULL);
7     $IF $$TIMING_ON $THEN
8       l_start_time:=CURRENT_TIMESTAMP;
9     $END
10     -- -----------------------------------------------
11     -- 应用逻辑部分已省略
12     -- -----------------------------------------------
13     $IF $$TIMING_ON $THEN
14       LOGIT (P_PG=>$$PLSQL_UNIT, P_START_TIME=>l_start_time,
P_STOP_TIME=>CURRENT_TIMESTAMP);
15     $END
16  DBMS_APPLICATION_INFO.SET_MODULE(NULL,NULL);
17  END LEVEL2;
18  /
过程已创建。
```

将 PL/SQL 标志 `TIMING_ON` 设置为 `FALSE` 会导致以下代码大小：

```sql
SQL@V112> SELECT * FROM USER_OBJECT_SIZE WHERE NAME = 'LEVEL2';

NAME          TYPE          SOURCE_SIZE PARSED_SIZE  CODE_SIZE ERROR_SIZE
------------- ------------- ----------- ----------- ---------- ----------
LEVEL2        PROCEDURE               530         857        364          0
```

`SOURCE_SIZE` 与第一个示例中的大小几乎相同，尽管添加了 `$IF` 结构来控制变量 `l_start_time` 的定义和引用。第一个示例在代码中也包含参数解析，所以两者在那方面差异不大。但是 `PARSED_SIZE` 和 `CODE_SIZE` 的值都减小了。

现在让我们看看启用计时时的情况。将编译器标志 `TIMING_ON` 设置为 `TRUE` 来编译过程 `LEVEL2` 会得到以下结果：

```sql
SQL@V112> SELECT * FROM USER_OBJECT_SIZE WHERE NAME = 'LEVEL2';
```



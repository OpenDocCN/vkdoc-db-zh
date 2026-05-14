# TVD$XTAT 使用指南

## 命令行参数

```
usage: tvdxtat [-a no|yes] [-c no|yes] [-f <int>] [-l <int>]
               [-r 7|8|9|10|11] [-s no|yes] [-t <template>]
               [-w no|yes] -i <input> -o <output>
```

- `-c,--cleanup` 移除临时 XML 文件 (`no|yes`)
- `-f,--feedback` 每隔 x 行显示一次进度（整数 >= 0，`0` 表示不显示进度）
- `-h,--help` 显示帮助信息并退出
- `-i,--input` 输入跟踪文件名（有效扩展名为 `.trc`、`.gz` 和 `.zip`）
- `-l,--limit` 限制输出文件中列表的大小（例如语句数量）（整数 >= 0，`0` 表示无限制）
- `-o,--output` 输出文件名（同时会创建一个具有相同名称但扩展名为 `.xml` 的临时 XML 文件）
- `-r,--release` 生成输入跟踪文件的数据库引擎主版本号 (`7|8|9|10|11`)
- `-s,--sys` 报告关于 SYS 递归语句的信息 (`no|yes`)
- `-t,--template` 用于生成输出文件的 XSL 模板名称 (`html.xsl|text.xsl`)
- `-v,--version` 打印产品版本并退出
- `-w,--wait` 报告关于等待事件的详细信息 (`no|yes`)

各参数功能如下：

* `input` 指定输入文件名。输入文件必须是跟踪文件（扩展名 `.trc`）或包含一个或多个跟踪文件的压缩文件（扩展名 `.gz` 或 `.zip`）。但请注意，从 `.zip` 文件中仅提取单个跟踪文件。
* `output` 指定输出文件名。处理过程中，会创建一个与输出文件同名但扩展名为 `.xml` 的临时 XML 文件。请注意，如果存在与输出文件同名的其他文件，它将被覆盖。
* `cleanup` 指定处理过程中生成的临时 XML 文件在结束时是否被移除。通常应设置为 `yes`。此参数仅在开发阶段检查中间结果时重要。
* `feedback` 指定是否显示进度信息。在处理非常大的跟踪文件时，了解分析的当前状态非常有用。该参数指定生成新消息的间隔（行数）。如果设置为 `0`，则不显示进度信息。
* `help` 指定是否显示帮助信息。它不能与其他参数一起使用。
* `limit` 设置输出文件中列表（例如用于 SQL 语句、等待和绑定变量的列表）所包含元素的最大数量。如果设置为 `0`，则没有限制。
* `release` 指定生成输入跟踪文件的 Oracle 数据库引擎的主版本号（即 7、8、9、10 或 11）。
* `sys` 指定输出文件中是否包含用户 SYS 执行的递归 SQL 语句的信息。通常设置为 `no`。
* `template` 指定用于生成输出文件的 XSL 模板名称。默认情况下，提供两个模板：`html.xsl` 和 `text.xsl`。前者生成 HTML 输出文件，后者生成文本输出文件。可以修改默认模板，也可以编写新模板。这样，就可以完全自定义输出文件。模板必须存储在子目录 `templates` 中。
* `version` 指定是否显示 TVD$XTAT 的版本号。它不能与其他参数一起使用。
* `wait` 指定是否显示等待事件的详细信息。启用此功能（即设置此参数为 `yes`）可能会在处理过程中产生显著开销。因此，建议最初将其设置为 `no`。之后，如果基本的等待信息不够，您可以运行另一次分析并将其设置为 `yes`。

## 解释 TVD$XTAT 输出

本节基于之前已与 TKPROF 一起使用的相同跟踪文件。由于 TVD$XTAT 的输出布局基于 TKPROF 的输出布局，因此我将仅在此描述 TVD$XTAT 特有的信息。为生成输出文件，我使用了以下参数。请注意，跟踪文件和 HTML 格式的输出文件可与本章的其他文件一起下载。

```
tvdxtat -i DBM11106_ora_6334.trc -o DBM11106_ora_6334.html -s no -w yes
```

输出文件以关于输入跟踪文件的整体信息开始。这部分最重要的信息是跟踪文件覆盖的时间段以及其中记录的事务数。

```
OVERALL INFORMATION

Database Version
----------------
Oracle Database 11g Enterprise Edition Release 11.1.0.6.0 - 64bit Production
With the Partitioning, Oracle Label Security, OLAP, Data Mining
and Real Application Testing options

Analyzed Trace File
-------------------
/u00/app/oracle/diag/rdbms/dbm11106/DBM11106/trace/DBM11106_ora_6334.trc

Interval
--------
Beginning 29 Feb 2008 07:43:11
End       29 Feb 2008 07:43:17
Duration  5.666

Transactions
------------
Committed 1
Rollbacked 0
```

对输出文件的分析始于查看整体资源使用情况。这里的处理持续了 5.666 秒。大约 44% 的时间用于读取数据文件（`db file scattered read` 和 `db file sequential read`），大约 26% 的时间用于 CPU 运行，大约 25% 的时间用于读写临时文件（`direct path read temp` 和 `direct path write temp`）。总之，大部分时间花费在 I/O 操作上，其余时间花费在 CPU 上。请注意，未计入的时间已被明确给出。

```
Resource Usage Profile
----------------------

                                Total           Number of  Duration per
Component                    Duration        %     Events         Event
--------------------------- --------- -------- ------------- -------------
db file scattered read          1.769   31.224        225         0.008
CPU                             1.458   25.730        n/a           n/a
direct path read temp           1.005   17.731        941         0.001
db file sequential read         0.710   12.530        176         0.004
direct path write temp          0.408    7.195        941         0.000
unaccounted-for                 0.307    5.425        n/a           n/a
SQL*Net message from client     0.009    0.155          2         0.004
log file sync                   0.001    0.010          2         0.000
SQL*Net message to client       0.000    0.000          2         0.000
--------------------------- --------- ---------
Total                           5.666  100.000
```

* * *

**注意** TVD$XTAT 始终根据响应时间对列表进行排序。没有选项可以更改此行为，因为这是调查性能问题的唯一合理顺序。

* * *

了解数据库引擎如何花费时间仅提供了一个概览。为了继续分析，必须找出哪些 SQL 语句是导致该处理时间的原因。为此，在整体资源使用情况之后，提供了一个包含所有非递归 SQL 语句的列表。在此案例中，您可以看到单个 SQL 语句（实际上是一个 PL/SQL 块）约占处理时间的 94%。请注意，在以下列表中，总计不是 100%，因为未计入的时间被省略（即，您根本不知道这些时间去了哪里）：

```
The input file contains 182 distinct statements, 180 of which are recursive.

Only non-recursive statements are reported in the following table.
```




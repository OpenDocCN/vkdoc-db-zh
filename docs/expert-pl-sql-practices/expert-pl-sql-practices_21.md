# 批量 SQL 操作

*作者：Connor McDonald*

本章介绍 PL/SQL 中可用的批量 SQL 操作。PL/SQL 中的批量操作使您能够一次操作和处理多行数据，而不是一次处理一行。在本章中，您将学习批量提取和批量绑定。批量提取是指通过单次调用将一组行从数据库检索到 PL/SQL 结构中的能力，而不是为要检索的每一行都调用数据库。批量绑定则允许您执行相反的操作：将存储在 PL/SQL 结构中的那些行集以高效的方式保存到数据库中。

您将看到使用批量 SQL 操作可能带来的巨大性能优势。要实现这些优势需要您的代码增加一点复杂性，尤其是在错误处理方面，但我会向您展示如何管理这种复杂性，以便在获得性能优势的同时不损害代码的可管理性。

## 五金店之旅

我对批量操作益处的认识，始于一次五金店之旅——不是指计算机硬件店，而是那种可以购买工具、配件、油漆等物品，用于家庭住宅手工项目和维护的 DIY 风格商店。

我**爱**五金店。当然，作为一名信息技术专业人士，我对任何与体力劳动稍微沾边的东西几乎一无所知。所以，每次我到收银台支付一个新工具或一罐新油漆后，大约 5 秒钟内，我就会回到过道去拿一个配套物品；例如，一罐新油漆如果没有油漆刷就没什么用！当我真正拥有完成手头工作所需的所有工具时，我可能已经来回收银台六七次了，每次去收银台时，收银员的脏话都在不断升级。我妻子观察到我这样做——一次只买一件东西——她想出了一个昵称，那就是“白痴”。她说得对；我应该直接推一辆购物车，收集我需要的所有物品，然后一次性付清！

这就是最大的矛盾之处。出于某种奇怪的原因，那些在应对“挑战”（比如去五金店）时能应用简单常识的 IT 专业人士，在处理数据库时却难以应用同样的常识。就像五金店一样，如果你要拿多个（数据）项，使用购物车的编码等价物来收集这些数据会合理得多。

在本章中，我将经常回顾并扩展五金店购物的这个比喻，因为这对构建示例很有用，但也是为了强调一个事实：本章讨论的一切仍然仅仅是我们远离技术的日常生活中所用常识原则的反映。

## 本章示例的设置

本章中的所有示例都可以从 Apress 网站上本书的目录页面下载，每个示例都标注了相应的文件名。对于每个示例，你会看到一个对脚本的初始调用，像这样：

`@@bulk_setup.sql [populate | empty]`

这将创建一个代表“五金店”的表，所有示例都将使用它。

```
SQL> desc HARDWARE
 Name                                      Null?    Type
 ----------------------------------------- -------- ----------------------------
 AISLE                                              NUMBER(5)
 ITEM                                               NUMBER(12)
 DESCR                                              CHAR(50)
```

该表创建时为空，除非传递了'populate'关键字，之后将通过以下 SQL 向表中添加 1,000,000 行：

```
SQL> insert /*+ APPEND */ into  HARDWARE
  2  select trunc(rownum/1000)+1 aisle,
  3         rownum item,
  4         'Description '||rownum descr
  5  from
  6    ( select 1 from dual connect by level <= 1000),
  7    ( select 1 from dual connect by level <= 1000);

1000000 rows created.
```

该脚本会删除并重新创建表，以便示例能在你的数据库上运行，并得到与你在本章中看到的相似的结果。显然，硬件平台和软件版本有如此多的变体，你的结果可能会有所不同，但这些示例都在最常见的默认设置上运行过，即位于 ASSM 表空间中的 8k 块大小。

![images](img/square.jpg) **提示** 使用`DUAL`生成任意行数的技巧可以追溯到多年前 Mikito Harakiri 在`asktom.oracle.com`网站上的一篇帖子。从那时起，它几乎成为新闻组、书籍等中快速合成数据演示的实际标准。在推向极端时，必须注意不要过度消耗内存。详情请参考[`http://blog.tanelpoder.com/2008/06/08/generating-lots-of-rows-using-connect-by-safely/`](http://blog.tanelpoder.com/2008/06/08/generating-lots-of-rows-using-connect-by-safely/)。

## PL/SQL 中的批量操作

Oracle 中的批处理早于 PL/SQL，只是名称不同，即“数组处理”。从我接触 Oracle 开始（大约在 1990 年代初的版本 6），数组处理的概念就已经可用。不是从数据库中获取单行数据，而是将一组行获取到缓冲区中，然后将该缓冲区作为数组传递回客户端。同样地，不是在数据库中修改或创建单行数据，而是填充一个包含行数组的缓冲区并传递给数据库。应用程序程序员可以通过 OCI 原生地进行数组处理，或者当时许多流行的 Oracle 工具（如 Forms）会透明地处理该任务。你很快就会看到这种策略在性能和可扩展性方面的好处。

Oracle 的 PL/SQL 团队在版本 8.1 中引入原生数组处理时，大概面临了一个命名问题。到那时，“数组”一词在 PL/SQL 中已经根深蒂固，指的是`INDEX BY`数组数据结构，也许正是由于这种冲突，才产生了“批量”这个术语。事实上，PL/SQL 中的数组处理甚至早于 8.1 版本；它早在 Oracle 7 中就可以通过`DBMS_SQL`包实现。下一个示例展示了来自版本 7 的 PL/SQL 数组处理（我不会详细讲解代码，因为我很快会介绍更简单的机制）。

以下示例（`bulk_dbms_sql_array.sql`）展示了如何在 8.0 及以下版本中执行数组处理。它一次从`HARDWARE`表中获取 500 行。

```
SQL> set serverout on
SQL> declare
  2    l_cursor   int  := dbms_sql.open_cursor;
  3    l_num_row  dbms_sql.number_table;
  4    l_exec     int;
  5    l_fetched_rows    int;
  6  begin
  7   dbms_sql.parse(
  8           l_cursor,
  9           'select item from hardware where item <= 1200',
 10           dbms_sql.native);
 11   dbms_sql.define_array(l_cursor,1,l_num_row,500,1);
 12   l_exec := dbms_sql.execute(l_cursor);
 13   loop
 14       l_fetched_rows := dbms_sql.fetch_rows(l_cursor);
 15       dbms_sql.column_value(l_cursor, 1, l_num_row);
 16       dbms_output.put_line('Fetched '||l_fetched_rows||' rows');
 17       exit when l_fetched_rows < 500;
 18    end loop;
 19    dbms_sql.close_cursor(l_cursor);
 20  end;
 21  /
Fetched 500 rows
Fetched 500 rows
Fetched 200 rows

PL/SQL procedure successfully completed.
```

遗憾的是，即使在它们被引入十年后，PL/SQL 中的批量操作在现代以 PL/SQL 为中心的应用程序中仍然是一种未被充分利用的功能。许多生产环境中的 PL/SQL 程序仍然以逐行方式处理数据库中的数据。特别是，由于我的大部分工作是在调优领域，我使用集合的主要动机是它鼓励开发者更多地从集合的角度思考，而不是从行的角度思考。虽然从功能上讲，没有理由禁止开发者一次处理一个结果集行，但从效率和性能的角度来看，这通常是坏消息。

同样，PL/SQL 在性能方面经常受到批评，但最常见的原因是逐行处理，而不是 PL/SQL 引擎本身固有的任何问题。PL/SQL 并非唯一受到这种错误指责的对象。开发者未能利用 Pro*C 中的宿主数组，促使 Oracle 添加了`PREFETCH`编译选项，该选项将运行时 Pro*C 代码转换为数组获取，即使开发者是以传统的逐行方式编写的代码。类似地，ODP.NET 通过其`FetchSize`参数，默认情况下对大多数查询使用数组处理。

#### 开始使用 BULK Fetch

将代码迁移到利用 PL/SQL 中的批量操作的一大优势在于，它易于实现，并且与现有代码有直接的映射关系。在 8.1 版本引入批量操作之前，你可以使用以下三种构造之一在 PL/SQL 中检索一行数据。

##### 1. 隐式游标

一个标准的 SQL 查询 (`SELECT-INTO`) 用于从表中将单行数据或该行的某些列检索到目标变量中。如果没有检索到行或检索到的行超过一行，则会引发异常。示例如下 (`bulk_implicit_1.sql`)：

```sql
SQL> declare
  2    l_descr hardware.descr%type;
  3  begin
  4    select descr
  5    into   l_descr
  6    from   hardware
  7    where  aisle = 1
  8    and    item = 1;
  9  end;
 10  /
```
`PL/SQL 过程已成功完成。`

##### 2. 显式提取调用

在 PL/SQL 声明部分显式定义一个游标。然后打开该游标并从中提取数据，通常在一个循环中进行，直到耗尽所有可用行，此时关闭游标。示例如下 (`bulk_explicit_1.sql`)：

```sql
SQL> declare
  2    cursor c_tool_list is
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 500;
  7
  8    l_descr hardware.descr%type;
  9  begin
 10    open c_tool_list;
 11    loop
 12      fetch c_tool_list into l_descr;
 13      exit when c_tool_list%notfound;
 14    end loop;
 15    close c_tool_list;
 16  end;
 17  /
```
`PL/SQL 过程已成功完成。`

##### 3. 隐式提取调用

第三种类型是前两种方法的混合。一个 `FOR` 循环负责游标管理，该游标要么是预先显式定义的，要么是直接在 `FOR` 循环内部编码的。示例如下 (`bulk_implicit_fetch_1.sql`)：

```sql
SQL> begin
  2    for i in (
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 500 )
  7    loop
  8       `<处理每一行的代码>`
  9    end loop;
 10  end;
 11  /
```
`PL/SQL 过程已成功完成。`

将上述每种构造转换为批量集合模型都简单直接。以下是与刚展示的非批量构造相对应的三种批量处理构造。

##### 1. 隐式游标批量模式

通过简单地添加 `BULK COLLECT` 关键字，一个标准的 SQL 查询 (`SELECT-INTO`) 现在可以用来将多行数据检索到一个集合类型中。示例如下 (`bulk_implicit_2.sql`)：

```sql
SQL> declare
  2    `type t_descr_list is table of hardware.descr%type;`
  3    l_descr_list t_descr_list;
  4  begin
  5    select descr
  6    `bulk collect`
  7    into   l_descr_list
  8    from   hardware
  9    where  aisle = 1
 10    and    item between 1 and 100;
 11  end;
 12  /
```
`PL/SQL 过程已成功完成。`

##### 2. 显式提取调用批量模式

唯一需要的更改是定义一个集合类型来保存结果，并向 `FETCH` 命令添加 `BULK COLLECT` 子句。游标结果集中的所有行将在一次调用中被提取到集合类型变量中。示例如下 (`bulk_explicit_2.sql`)：

```sql
SQL> declare
  2    cursor c_tool_list is
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 500;
  7
  8    type t_descr_list is table of c_tool_list%rowtype;
  9    l_descr_list t_descr_list;
 10
 11  begin
 12    open c_tool_list;
 13    fetch c_tool_list `bulk collect` into l_descr_list;
 14    close c_tool_list;
 15  end;
 16  /
```
`PL/SQL 过程已成功完成。`

##### 3. 隐式提取调用批量模式

将混合方法转换为批量收集甚至更容易，因为**无需**进行代码更改。Oracle 10g 引入的最佳特性之一就是针对 `FOR` 循环的“自动批量收集”增强功能。因为游标上的 `FOR` 循环根据定义会提取游标中的**所有**行（除非循环内存在显式的 `EXIT` 命令），所以 PL/SQL 编译器可以安全地采用批量收集优化来尽可能高效地检索这些行。如果你使用的是至少 Oracle 10g 版本，并且数据库参数 `plsql_optimize_level` 设置为至少 2（默认值），你将自动获得此优化。以下示例 (`bulk_implicit_fetch_2.sql`) 展示了数据库优化级别，随后是一个为你自动实现批量处理功能的 `FOR` 循环：

```sql
SQL> select banner from v$version where rownum = 1;

BANNER
----------------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.2.0 - 64bit Production

SQL> select value from v$parameter where name = 'plsql_optimize_level';

VALUE
----------------------------------------------------------------------------
2

SQL> begin
  2    for i in (
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 500 )
  7    loop
  8       null;
  9    end loop;
 10  end;
 11  /
```
`PL/SQL 过程已成功完成。`

在之前的代码中，批量收集的发生并不明显。你很快将看到如何验证代码确实在一次调用中提取了多行数据。


#### 三级集合风格数据类型

前一节最后三个示例以嵌套表数据类型获取行。自 Oracle 8.1 以来，有三种集合风格的数据类型，可以批量获取行。

*   `Varray`
*   `Nested table`
*   `Associative array`

决定使用哪种集合类型通常取决于哪种最适合您的应用程序。这些类型并非互斥。除了在 PL/SQL 程序之间传递集合外，您还可能在 PL/SQL 与驻留在数据库服务器之外的基于 3GL 的应用程序之间来回传递这些集合。许多用于与 Oracle 数据库通信的流行 3GL 技术都有简单有效的接口与 PL/SQL 例程交互，这些例程接受*关联数组*作为参数，但不一定接受*varray*或*嵌套表*。

例如，

*   在 JDBC 中，方法`setPlsqlIndexTable`和`registerIndexTableOutParameter`可用于通过 PL/SQL 关联数组向数据库传递数据和从数据库接收数据。
*   在 Pro*C 中，C 语言中的主机数组可以直接映射到 PL/SQL 参数中的关联数组。
*   在 ODP.NET 中，Oracle 参数可以直接映射到关联数组，即 `Param1.CollectionType = OracleCollectionType.PLSQLAssociativeArray;`

相比之下，如果您选择“对象”路径，创建包含 varrays 和/或嵌套表的用户定义类型，那么通过相同的 3GL 与 PL/SQL 交互可能会变得更加复杂。需要转换工具，例如 Pro*C 的对象类型转换器或 ODP.Net 的自定义元数据映射。因此，如果您有强烈的 3GL 应用程序代码与 PL/SQL 的交互，那么关联数组很可能是最佳选择。或者，鉴于业界当前对面向服务的架构（SOA）的痴迷，其中数据交换媒介是 XML，3GL 应用程序可能更容易将 XML 传递到 PL/SQL 中，并利用 `XMLTYPE` 数据类型内的功能将传入的 XML 转换为 PL/SQL 对象类型。以下是此类转换的示例（`bulk_xml_to_obj.sql`）：

```sql
SQL> create or replace
  2  type COMING_FROM_XML as object
  3     ( COL1 int,
  4       COL2 int)
  5  /

Type created.

SQL> declare
  2    source_xml xmltype;
  3    target_obj coming_from_xml;
  4  begin
  5    source_xml :=
  6       xmltype('<DEMO>
  7                  <COL1>10</COL1>
  8                  <COL2>20</COL2>
  9                </DEMO>');
 10
 11    source_xml.toObject(target_obj);
 12  end;
 13  /

PL/SQL procedure successfully completed.
```

相反，如果您的应用程序以 PL/SQL 为中心（Oracle 自己的 Application Express 就是一个完美的例子），那么使用嵌套表和 varray 类型可能是更好的选择，因为这些数据类型的变量有广泛的集合操作符可用。有关可以对嵌套表类型执行的各种集合操作的详细信息，请参阅《对象关系开发者指南》的第 5 章。

当然，您可以自由混合搭配。3GL 可以通过关联数组将数据传递给您的 PL/SQL，然后您可以在 PL/SQL 中将这些数组转换为更灵活的对象类型以进行后续处理。

#### 为什么我要费心？

正如您所看到的，将逐行获取调用转换为使用批量集合的代码只需要额外几行代码。那么，您为什么要费心添加这几行代码呢？五金店的比喻表明，批量处理行是为了效率，那么让我们比较一下多次前往五金店（`HARDWARE` 表）取每件物品与仅使用一个足够大的手推车（嵌套表）单次前往的结果。以下代码示例（`bulk_collect_ perf_test_1.sql`）比较了逐行代码及其批量收集等效代码的响应时间：

```sql
SQL> declare
  2    cursor c_tool_list is
  3      select descr d1
  4      from   hardware;
  5
  6    l_descr hardware.descr%type;
  7  begin
  8    open c_tool_list;
  9    loop
 10      fetch c_tool_list into l_descr;
 11      exit when c_tool_list%notfound;
 12    end loop;
 13    close c_tool_list;
 14  end;
 15  /

PL/SQL procedure successfully completed.
```

**`Elapsed: 00:00:21.39`**

```sql
SQL> declare
  2    cursor c_tool_list is
  3      select descr d2
  4      from   hardware;
  5
  6    type t_descr_list is table of c_tool_list%rowtype;
  7    l_descr_list t_descr_list;
  8
  9  begin
 10    open c_tool_list;
 11    fetch c_tool_list bulk collect into l_descr_list;
 12    close c_tool_list;
 13  end;
 14  /

PL/SQL procedure successfully completed.
```

**`Elapsed: 00:00:02.20`**

通过减少为获取数据而访问数据库的次数，速度提高了 10 倍。此外，如果启用会话跟踪重复前面的示例，您将从生成的跟踪数据中得到一些有趣的结果。现在让我们启用会话跟踪并重新运行演示：

```sql
SQL> alter session set sql_trace = true;
SQL> [repeat demo]
SQL> alter session set sql_trace = false;
```

在格式化的跟踪文件中，如果搜索列别名“`D1`”，您将找到受单行获取调用影响的 SQL 查询。

```sql
SELECT DESCR D1
FROM   HARDWARE

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          1          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch  1000001      8.17       8.15          0    1000010          0     1000000
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total  1000003      8.17       8.15          0    1000011          0     1000000
```

在跟踪文件的后面部分，您将找到包含 `D2` 别名的查询，该查询是通过批量获取获取的：

```sql
SELECT DESCR D2
FROM   HARDWARE

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          1          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        1      1.85       2.08          0       9001          0     1000000
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        3      1.85       2.09          0       9002          0     1000000
```

请注意，除了减少经过时间外，CPU 消耗也显著减少。使用批量收集，不仅应用程序性能得到提升，其 CPU 占用也会减少，这使得它们更具可扩展性。

![images](img/square.jpg) **提示** 有关如何生成会话跟踪文件并使用 `tkprof` 格式化它们的详细信息，请参阅 Oracle 11.2 数据库文档中的《性能调优手册》第 21 章。




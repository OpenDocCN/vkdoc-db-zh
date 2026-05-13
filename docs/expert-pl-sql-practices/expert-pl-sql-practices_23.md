# 退出循环

当使用传统的逐行机制在游标循环中执行提取时，结果集中没有更多行时会隐式退出。你发出一个提取调用；如果失败，就表示已完成。而当你进行批量提取时，情况会稍微复杂一些——一次提取调用可能检出足以填满数组的行，可能检出零行，也可能检出一些行但不足以完全填满数组。这导致在使用 `LIMIT` 子句时出现一个常见的程序错误。

首先，这是本章前面展示的逐行代码 (`bulk_limit_exit_1.sql`)：

```sql
SQL> set serverout on
SQL> declare
  2    cursor c_tool_list is
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 25;
  7
  8    l_descr hardware.descr%type;
  9  begin
 10    open c_tool_list;
 11    loop
 12      fetch c_tool_list into l_descr;
 13      exit when c_tool_list%notfound;
 14      dbms_output.put_line('Fetched '||l_descr);
 15    end loop;
 16    close c_tool_list;
 17  end;
 18  /
Fetched Description 1
Fetched Description 2
Fetched Description 3
[snip]
Fetched Description 23
Fetched Description 24
Fetched Description 25

PL/SQL procedure successfully completed.
```

注意，提取了 25 行并显示在屏幕上。现在代码被转换为使用 `LIMIT` 子句的批量收集，看起来是直观的方式，但结果证明是错误的方式。这是不正确的解决方案 (`bulk_limit_exit_2.sql`) 及其输出：

```sql
SQL> set serverout on
SQL> declare
  2    cursor c_tool_list is
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 25;
  7
  8    type t_descr_list is table of c_tool_list%rowtype;
  9    l_descr_list t_descr_list;
 10
 11  begin
 12    open c_tool_list;
 13    loop
 14      fetch c_tool_list
 15      bulk collect into l_descr_list limit 10;
 16      exit when c_tool_list%notfound;
 17
 18      for i in 1 .. l_descr_list.count loop
 19        dbms_output.put_line('Fetched '||l_descr_list(i).descr);
 20      end loop;
 21    end loop;
 22
 23    close c_tool_list;
 24  end;
 25  /
Fetched Description 1
Fetched Description 2
Fetched Description 3
[snip]
Fetched Description 18
Fetched Description 19
Fetched Description 20

PL/SQL procedure successfully completed.
```

注意，游标循环过早退出了：应该输出 25 行（与逐行示例完全相同），但代码在输出 20 行后就停止了。问题出在第 16 行：`exit when c_tool_list%notfound`。`LIMIT` 子句指定了每次提取调用获取 10 行。在第三次循环时，只剩下 5 行可供提取。因为所有行都已被提取，`%NOTFOUND` 属性变为真，循环在 `没有` 处理数组中剩余 5 行的情况下退出。

解决方案很简单：要么将游标属性的检查推迟到处理完提取数组中找到的任何行之后，要么将退出条件改为 `<array>.count = 0`（这也意味着需要额外遍历一次提取循环）。

以下是第一种解决方案的示例，代码将游标属性的检查推迟到处理完数组中的任何行之后 (`bulk_limit_exit_3.sql`)：

```sql
SQL> set serverout on
SQL> declare
  2    cursor c_tool_list is
  3      select descr
  4      from   hardware
  5      where  aisle = 1
  6      and    item between 1 and 25;
  7
  8    type t_descr_list is table of c_tool_list%rowtype;
  9    l_descr_list t_descr_list;
 10
 11  begin
 12    open c_tool_list;
 13    loop
 14      fetch c_tool_list
 15      bulk collect into l_descr_list limit 10;
 16
```
注意，因为有可能没有提取到行，所以不能对数组中是否存在行做任何假设。所有处理都必须考虑到这一事实，并且必须遵守数组的 `count` 属性的值，否则可能会因非法引用数组索引而出现错误。在以下代码中，使用 `l_descr_list.count` 确保如果数组中没有条目，甚至不会进入循环：

```sql
 17      for i in 1 .. l_descr_list.count loop
 18        dbms_output.put_line('Fetched '||l_descr_list(i).descr);
 19      end loop;
 20      exit when c_tool_list%notfound;
 21    end loop;
 22
 23    close c_tool_list;
 24  end;
 25  /
Fetched Description 1
Fetched Description 2
Fetched Description 3
[snip]
Fetched Description 23
Fetched Description 24
Fetched Description 25

PL/SQL procedure successfully completed.
```

## 过度批量处理

这种情况虽然罕见，但有时批量处理（bulking up）*并非*从数据库检索数据的合适方式。回到五金店的比喻，假设我口袋里只有几美元就开车去商店。我的购买能力会受限于现金，只有几美元显然买不了多少东西。如果我盲目遵循“一定要推购物车（即批量收集）”的建议，直到结账才停止，结果会怎样？

*   我需要先花时间在商店入口取一辆购物车。
*   我的购物车只装了几件商品，我就不得不停止——因为我超出了消费限额。
*   我为商品付款。
*   我需要花更多时间将商品从购物车卸到袋子里，然后把购物车还回商店入口。

在数据库术语中也是一样。如果你事先知道某些情况下可能不会处理获取到的整批数据，那么费力去获取它们实际上可能会损害而非提升性能。一个非常常见的原因是处理结果集时可能需要提前退出。考虑以下示例（`bulk_early_exit_1.sql`）。首先，我定义一个游标，从`HARDWARE`表中获取一些行，如下所示：

```
SQL> set timing on
SQL> declare
  2    cursor c_might_exit_early is
  3      select aisle, item
  4      from   hardware
  5      where  item between 400000 and 400050
  6        or   descr like '%10000%';
  7  begin
```

根据`HARDWARE`表中的数据，这个游标将返回 72 行。这里使用了隐式获取（fetch）结构，以自动获取批量收集的抓取大小，即 100 行。但是，这个游标被命名为`c_might_exit_early`是有原因的。

```
  8    for i in c_might_exit_early
  9    loop
 10      `if c_might_exit_early%rowcount = 40 then`
 11         `exit;`
 12      `end if;`
```

一旦获取了 40 行，游标循环就会退出。当然，可以在定义游标的 SQL 中添加`rownum <= 40`的谓词，但在现实世界场景中，条件可能更复杂。例如，每获取一行，都可能将其传递给一个网络服务，该服务可能返回一个标志，指示不应再发送更多行。这里的关键是，游标循环存在提前退出的可能性，而这个退出条件不容易整合回游标 SQL 语句中。

```
 13    end loop;
 14  end;
 15  /
```

```
PL/SQL procedure successfully completed.

Elapsed: 00:00:00.38
```

这个例程运行得相当快，但 0.38 秒对于仅处理 40 行来说显得有些迟缓，这归因于自动批量收集。代码实际上执行了检索全部 72 行的工作，因为第一次获取调用试图检索 100 行。PL/SQL 编译器并不知道游标处理可能会提前结束。一个会话跟踪证实所有 72 行都被检索了。

```
SELECT AISLE, ITEM
FROM
 HARDWARE WHERE ITEM BETWEEN 400000 AND 400050 OR DESCR LIKE '%10000%'

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch        1      0.17       0.17          0      10100          0          72
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        3      0.17       0.17          0      10100          0          72
```

识别这类情况并相应编码是开发者的责任；PL/SQL 编译器并不能未卜先知。在上面的例子中，事实证明根本不使用批量收集效率更高。例如（`bulk_early_exit_2.sql`）是使用更传统的逐行方法编写的相同例程。

```
SQL> set timing on
SQL> declare
  2    cursor c_might_exit_early is
  3      select aisle, item
  4      from   hardware
  5      where  item between 400000 and 400050
  6        or   descr like '%10000%';
  7    l_row c_might_exit_early%rowtype;
  8  begin
  9    open c_might_exit_early;
 10    loop
 11      fetch c_might_exit_early
 12      into l_row;
 13      exit when c_might_exit_early%notfound;
 14
 15      if c_might_exit_early%rowcount = 40 then
 16         exit;
 17      end if;
 18
 19    end loop;
 20    close c_might_exit_early;
 21  end;
 22  /
```

```
PL/SQL procedure successfully completed.

Elapsed: 00:00:00.17
```

即使有 40 次单独获取调用的开销，性能也提升了一倍。原因在于本例中的 SQL 被设计成最后几行很难找到；也就是说，它们很可能位于表的高水位标记附近。因此，虽然这是一个稍微有些人工的例子，但它确实表明你不能简单地假设批量收集总会带来好处。对于这个特定的例子，精明的开发者会意识到，通过一些额外的编码，你可以通过明确指定获取大小，而不是交给 PL/SQL 编译器，来兼顾两者的优点。

下面的例子来自`bulk_early_exit_3.sql`；它明确指定了一个获取大小，作为逐行获取和 100 行自动批量收集获取大小之间的折中方案：

```
SQL> set timing on
SQL> declare
  2    cursor c_might_exit_early is
  3      select aisle, item
  4      from   hardware
  5      where  item between 400000 and 400050
  6        or   descr like '%10000%';
  7    type t_rows is table of c_might_exit_early%rowtype;
  8    l_rows t_rows;
  9
 10    l_row_cnt pls_integer := 0;
 11
 12  begin
 13    open c_might_exit_early;
 14    <<cursor_loop>>
 15    loop
 16      fetch c_might_exit_early
```

显然，对于这个演示，我们知道处理会在 40 行后停止，因此 40 的获取大小是最优的。然而，我选择了 20 的获取大小来反映一个现实世界的场景，即无法精确知道处理何时会提前终止，因此开发者会在“从每次获取调用中获得良好价值”和“不获取可能最终不需要的过多行”之间寻求一个折中方案。

```
 17      bulk collect into l_rows `limit 20;`
 18
 19      for i in 1 .. l_rows.count
 20      loop
 21         l_row_cnt := l_row_cnt + 1;
 22         if l_row_cnt = 40 then
 23            exit cursor_loop;
 24         end if;
 25      end loop;
 26
 27      exit when c_might_exit_early%notfound;
 28    end loop;
 29    close c_might_exit_early;
 30  end;
 31  /
```

```
PL/SQL procedure successfully completed.

Elapsed: 00:00:00.09
```

对于这个例子，仅仅通过在代码中加入一些我自己的智慧，而不是依赖 PL/SQL 编译器的智慧，周转时间就从 0.38 秒减少到了 0.09 秒。


### 批量绑定

使用`BULK COLLECT`批量获取数据，完成了优化 PL/SQL 与数据库交互的一半图景。另一半则是尽可能高效地处理从 PL/SQL 发送到数据库表的数据。每当 PL/SQL 程序需要向数据库插入、更新或删除数据时，在大多数情况下，最好通过单次数据库访问来完成。对于某些应用需求（例如，更新已知主键的属性），修改将只针对单行。然而，当需要操作数据库中的多行时，PL/SQL 仍然可以在单次访问中完成该工作。这被称为“批量绑定”，与批量收集一样，都是从逐行处理转向集合处理（通过 PL/SQL 集合）。

![images](img/square.jpg) **注意** 我的妻子慷慨地提出扩展这个比喻：“批量绑定是不是指这一章中你把所有不需要且本不该买的垃圾带回五金店的部分？”如果这有助于你理解批量绑定，那就这样吧。

一个常见的误解是，如果应用需求是修改单行，那么批量绑定就一定不合适。然而，编写高效 PL/SQL 的艺术部分就在于能够识别出看似单行处理的过程可能并非如此。例如，考虑一个显示员工表格列表的网页，操作员可以在其中更新每位员工的详细信息。虽然从操作员的角度来看，每位员工的更新是独立于其他所有人的，但这并不能成为将其实现为一系列单行更新 SQL 语句的理由（无论是直接调用还是在 PL/SQL 块中调用）。

变更后的记录集可以存储在一个数组中，并通过单次调用传递给 PL/SQL 程序，然后执行适当的批量绑定。请始终留意应用程序中可能受益于此方法的部分。

#### 批量绑定入门

与批量收集类似，将现有的常规 DML 代码转换为其等效的批量绑定 DML 是简单而直接的。在常规 DML 中，通常会有一个填充了值的变量，该变量在 DML 语句中使用。以下示例 (`bulk_bind_1.sql`) 展示了一个简单的插入，其中 PL/SQL 变量通过`FOR`循环重复 100 次：

```sql
SQL> declare
  2    l_row hardware%rowtype;
  3  begin
  4    for i in 1 .. 100 loop
  5      l_row.aisle := 1;
  6      l_row.item  := i;
  7      insert into hardware values l_row;
  8    end loop;
  9  end;
 10  /

PL/SQL procedure successfully completed.
```

在批量绑定 DML 中，仍然有一个填充了值的变量，但该变量现在是一个数组，并且 DML 操作被延迟到数组被填满用于批量绑定的值之后才执行。`FORALL`关键字表明这是一个批量绑定 DML。例如 (`bulk_bind_2.sql`) 等效于上面逐行插入的示例，但它通过一次调用插入 100 行，而不是 100 次调用：

```sql
SQL> declare
  2    type t_row_list is table of hardware%rowtype;
  3    l_row t_row_list := t_row_list();
  4  begin
  5    for i in 1 .. 100 loop
  6      l_row.extend;
  7      l_row(i).aisle := 1;
  8      l_row(i).item  := i;
  9    end loop;
 10
 11    forall i in 1 .. 100
 12      insert into hardware values l_row(i);
 13  end;
 14  /

PL/SQL procedure successfully completed.
```

![images](img/square.jpg) **注意** 尽管`FORALL`看起来暗示某种循环处理，但它实际上是对数据库执行 DML 的单次调用。这可以通过会话级别的跟踪轻松确认。

```sql
INSERT INTO HARDWARE
VALUES  (:B1 ,:B2 ,:B3 ,:B4 )

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute      1      0.00       0.00          0          1          6         100
Fetch        0      0.00       0.00          0          0          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total        2      0.00       0.00          0          1          6         100
```



## 测量批量绑定性能

与批量收集一样，使用批量绑定的动机是为了提升性能，这可以通过简单的基准测试来量化。我将从一个简单的 PL/SQL 程序开始，该程序通过逐行插入的方式向表中插入大量行。以下示例 (`bulk_insert_conventional.sql`) 逐行向 `HARDWARE` 表添加了 100,000 行数据：

```
SQL> set timing on
SQL> declare
  2    l_now date := sysdate;
  3  begin
  4   for i in 1 .. 100000 loop
  5     insert into HARDWARE
  6     values (i/1000, i, to_char(i), l_now);
  7   end loop;
  8  end;
  9  /

PL/SQL procedure successfully completed.

Elapsed: 00:00:04.52
```

在转向批量绑定版本之前，值得注意的是，即使是一行一行地插入，Oracle 的插入性能也相当出色。但数据库可以做得更好。让我们看看批量绑定版本 (`bulk_insert_bind.sql`)，它通过一次调用插入那 100,000 行数据。

```
SQL> set timing on
SQL> declare
  2    l_now date := sysdate;
  3
  4    type t_rows is table of hardware%rowtype;
  5
  6    l_rows t_rows := t_rows();
  7  begin
  8   l_rows.extend(100000);
  9   for i in 1 .. 100000 loop
 10     l_rows(i).aisle := i/1000;
 11     l_rows(i).item := i;
 12     l_rows(i).descr := to_char(i);
 13     l_rows(i).stocked := l_now;
 14   end loop;
 15
 16   forall i in 1 .. 100000
 17     insert into hardware values l_rows(i);
 18
 19  end;
 20  /

PL/SQL procedure successfully completed.

Elapsed: 00:00:00.31
```

从 4.52 秒降到了仅仅 0.31 秒！就像批量收集一样，将代码迁移到批量绑定方法来处理数据集合，能带来显著的性能提升。但这里还有其他一些不那么明显的好处。每次开始一个新的 DML 语句时，数据库都必须分配撤销结构，以确保在需要时（无论是由于错误还是显式请求）可以回滚该 DML 语句。使用批量绑定会减少 DML 调用次数，这意味着对撤销基础设施的压力更小。以下示例 (`bulk_bind_undo.sql`) 检查了会话级别的统计信息，以比较使用逐行插入与批量绑定插入时所需的撤销量：

```
SQL> set serverout on
SQL> declare
  2    type t_row_list is table of hardware.descr%type
  3       index by pls_integer;
  4    l_rows t_row_list;
  5
  6    l_stat1 int;
  7    l_stat2 int;
  8    l_stat3 int;
  9
 10  begin
 11    select value
 12    into   l_stat1
 13    from   v$mystat m, v$statname s
 14    where  s.statistic# = m.statistic#
 15    and    s.name = 'undo change vector size';
 16
 17    for i in 1 .. 1000 loop
 18       l_rows(i) := rpad('x',50);
 19       insert into hardware ( descr )  values ( l_rows(i) );
 20    end loop;
 21
 22    select value
 23    into   l_stat2
 24    from   v$mystat m, v$statname s
 25    where  s.statistic# = m.statistic#
 26    and    s.name = 'undo change vector size';
 27
 28    forall i in 1 .. l_rows.count
 29       insert into hardware ( descr )  values ( l_rows(i) );
 30
 31    select value
 32    into   l_stat3
 33    from   v$mystat m, v$statname s
 34    where  s.statistic# = m.statistic#
 35    and    s.name = 'undo change vector size';
 36
 37    dbms_output.put_line('Row at a time: '||(l_stat2-l_stat1));
 38    dbms_output.put_line('Bulk bind:     '||(l_stat3-l_stat2));
 39
 40  end;
 41  /
Row at a time: 64556
Bulk bind:     3296

PL/SQL procedure successfully completed.
```

因此，批量绑定产生的撤销量少得多。类似地，这些撤销结构需要由重做日志条目保护，以使数据库实例可恢复。如果重复前面的示例，但收集的是 `redo size` 统计信息而非 `undo change vector size` (`bulk_bind_redo.sql`)，输出也显示重做量有所减少。

```
Row at a time: 295448
Bulk bind:     69552

PL/SQL procedure successfully completed.
```

![images](img/square.jpg) **提示** 在制作用于基准测试的示例时，你需要确保隔离你的代码，以纯粹检查当前测试。当我最初执行上述基准测试时，我为正在处理的 100,000 行中的每一行都显式引用了 `sysdate`，即：

```
for i in 1 .. 100000 loop
  insert into DEMO values (i, sysdate, to_char(i));
end loop;

for i in 1 .. 100000 loop
   l_rows(i).x := i;
   l_rows(i).y := sysdate;
   l_rows(i).z := to_char(i);
```

![images](img/square.jpg) **注意** 两个测试运行得都非常慢，而且批量绑定测试的结果并没有像我预期的那样，以数量级优势超越单行测试。更仔细的实验表明，事实上，对 `sysdate` 的 100,000 次调用是测试中的主导因素。实际上，如果我在更早版本的 Oracle 上执行该测试（其中对 ‘sysdate’ 的引用会静默地发出 ‘select sysdate from dual’ 语句），那么测试结果可能表明批量绑定毫无益处。在得出结论之前，请务必仔细检查你的测试。




#### 监控内存使用

与批量收集类似，如果你有大量行需要进行批量绑定，这并不一定意味着你应该在将它们传递给数据库之前，将它们全部预存储在数组中。随着绑定行数的增加，性能收益会递减，同时 PGA 内存的成本也在不断增加。你可以分批进行批量绑定，以确保不会耗尽会话内存。下面是一个示例（`bulk_bind_pga.sql`），它与批量收集内存演示类似，展示了随着批量绑定大小的增加，PGA 消耗的情况：

```
SQL> set serverout on
SQL> declare
  2    type t_row_list is table of hardware.descr%type;
  3    l_rows t_row_list;
  4
  5    l_start_of_run timestamp;
  6
  7    l_pga_ceiling  number(10);
  8
  9    type t_bulk_sizes is table of pls_integer;
 10    l_bulk_sizes t_bulk_sizes := t_bulk_sizes(10,100,1000,10000,100000,1000000);
 11
 12    tot_rows pls_integer := 10000000;
 13
 14  begin
 15    select value
 16    into   l_pga_ceiling
 17    from   v$mystat m, v$statname s
 18    where  s.statistic# = m.statistic#
 19    and    s.name = 'session pga memory max';
 20
 21    dbms_output.put_line('Initial PGA: '||l_pga_ceiling);
 22
 23    for i in 1 .. l_bulk_sizes.count
 24    loop
 25
 26      execute immediate 'truncate table hardware';
 27
 28      l_start_of_run := systimestamp;
 29
 30      l_rows := t_row_list();
 31      l_rows.extend(l_bulk_sizes(i));
 32      for j in 1 .. l_bulk_sizes(i)
 33      loop
 34         l_rows(j) := rpad('x',50);
 35      end loop;
 36
 37      for iter in 1 .. tot_rows / l_bulk_sizes(i) loop
 38        forall j in 1 .. l_bulk_sizes(i)
 39          insert into hardware ( descr )  values (l_rows(j));
 40      end loop;
 41
 42      select value
 43      into   l_pga_ceiling
 44      from   v$mystat m, v$statname s
 45      where  s.statistic# = m.statistic#
 46      and    s.name = 'session pga memory max';
 47
 48      dbms_output.put_line('Bulk size: '||l_bulk_sizes(i));
 49      dbms_output.put_line('- Elapsed: '||( systimestamp - l_start_of_run));
 50      dbms_output.put_line('- PGA Max: '||l_pga_ceiling);
 51
 52    end loop;
 53
 54  end;
 55  /
Initial PGA: 3470120
Bulk size: 10
- Elapsed: +000000000 00:00:53.478000000
- PGA Max: 4042888
Bulk size: 100
- Elapsed: +000000000 00:00:14.760000000
- PGA Max: 4042888
Bulk size: 1000
- Elapsed: +000000000 00:00:10.588000000
- PGA Max: 4042888
Bulk size: 10000
- Elapsed: +000000000 00:00:11.872000000
- PGA Max: 4108424
Bulk size: 100000
- Elapsed: +000000000 00:00:16.431000000
- PGA Max: 12890248
Bulk size: 1000000
- Elapsed: +000000000 00:00:17.507000000
- PGA Max: 99266696

PL/SQL procedure successfully completed.
```

与批量收集类似，一旦每个数组绑定的行数超过 1000 行，其好处就微不足道了。如果你的应用程序由成百上千个会话组成，并且每个会话都在服务器上消耗大量内存，那么内存消耗的增加肯定会对可扩展性构成威胁。在这个例子中，当批量绑定大小超过 1000 时，性能实际上变得更差了。

#### 11g 中的改进

本章中的示例代码是针对 11 版本的数据库编写的。如果你在更早的版本上运行批量绑定示例，可能会看到如下错误：

```
SQL> create table DEMO ( x int, y int);

Table created.

SQL> declare
  2    type t_rows is
  3      table of demo%rowtype
  4      index by pls_integer;
  5    l_rows t_rows;
  6  begin
  7    l_rows(1).x := 1;
  8    l_rows(1).y := 1;
  9
 10    l_rows(2).x := 2;
 11    l_rows(2).y := 2;
 12
 13    forall i in 1 .. l_rows.count
 14       insert into DEMO
 15       values ( l_rows(i).x, l_rows(i).y );
 16  end;
 17  /

PL/SQL procedure successfully completed.

 values ( l_rows(i).x, l_rows(i).y );
              *
ERROR at line 15:
PLS-00436: implementation restriction: cannot reference
              fields of BULK In-BIND table of records
```

在 Oracle 11g 之前的版本中，批量绑定操作无法访问关联数组内记录或对象类型的单个元素。然而，并非无计可施——只是需要多写一点代码。你仍然可以对简单数据类型的数组进行批量绑定，因此可以为需要批量绑定的*每个*属性使用一个关联数组。前面的例子可以改写成一个适用于早期 Oracle 版本的版本。

```
SQL>  declare
  2    type t_x_list is table of demo.x%type
  3      index by pls_integer;
  4
  5    type t_y_list is table of demo.y%type
  6      index by pls_integer;
  7
  8    l_x_rows t_x_list;
  9    l_y_rows t_y_list;
 10  begin
 11    l_x_rows(1) := 1;
 12    l_x_rows(2) := 2;
 13
 14    l_y_rows(1) := 1;
 15    l_y_rows(2) := 2;
 16
 17    forall i in 1 .. l_x_rows.count
 18       insert into DEMO
 19       values ( l_x_rows(i), l_y_rows(i) );
 20  end;
 21  /

PL/SQL procedure successfully completed.
```

![images](img/square.jpg) 提示：关于在早期 Oracle 版本上规避此限制的其他技术，请参见 `www.oracle-developer.net/display.php?id=410`





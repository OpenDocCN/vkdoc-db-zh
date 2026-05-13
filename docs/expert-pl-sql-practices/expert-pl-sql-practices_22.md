# 批量收集的性能与内存开销

## 集合类型性能比较

再回到应使用哪种集合类型的问题上，从批量收集性能来看，所有类型似乎都差不多。以下演示（`bulk_which_collection_type.sql`）对三种集合类型的运行结果几乎完全相同：

```sql
SQL> set timing on
SQL> declare
  2    type t_va is varray(1000) of number;
  3    type t_nt is table of number;
  4    type t_aa is table of number index by pls_integer;
  5
  6    va t_va;
  7    nt t_nt;
  8    aa t_aa;
  9  begin
 10    for i in 1 .. 10000 loop
 11
 12      select rownum
 13      --
 14      -- 注释掉你想测试的集合类型
 15      --
 16      bulk collect into va
 17      --bulk collect into nt
 18      --bulk collect into aa
 19
 20      from dual
 21      connect by level <= 1000 ;
 22    end loop;
 23  end;
 24  /
```

```
PL/SQL 过程已成功完成。
```

## 监控批量收集的开销

在你急于将所有 PL/SQL 游标转换为使用批量收集操作之前，了解批量收集将对数据库会话产生的一项重要影响至关重要。如本章开头所述，数组处理是关于将数据检索到`缓冲区`中。该缓冲区必须存放在某处，而那个地方就在你会话的内存中。用 Oracle 的术语来说，你会话的内存是会话的用户全局区（UGA）。根据数据库的配置方式，会话 UGA 可能私有地保存在连接会话的进程全局区（PGA）中，也可能在系统全局区（SGA）中跨进程共享。关于 Oracle 专用服务器与共享服务器架构优缺点的讨论超出了本书范围；只需知道 PL/SQL 集合会消耗会话内存，如果你的集合很大或数据库中有大量并发会话，这很容易成为一个重要的考虑因素。

对于接下来的示例，将假设最常见的配置，即专用服务器连接，因此内存消耗的重点将放在 PGA 上。让我们探讨一下随着为批量收集使用越来越大的集合尺寸，内存消耗的变化情况。我还将介绍可用于`BULK COLLECT`语句的`LIMIT`子句。

![images](img/square.jpg) **注意** 接下来的示例将文本与代码交错呈现。你可以在名为`bulk_collect_memory.sql`的文件中找到完整的代码。

首先，我将重新连接以重置会话级别统计信息（包括 PGA 消耗）。

```sql
SQL> connect
Enter user-name: *****
Enter password:  *****

SQL> set serverout on
SQL> declare
  2    type t_row_list is table of hardware.descr%type;
  3    l_rows t_row_list;
  4
  5    l_pga_ceiling  number(10);
```

接下来，我将定义一个不同获取大小的数组，用于从 HARDWARE 表中检索一组行。因此，在第一次迭代中，将获取 5 行，然后是 10 行，接着是 50 行，依此类推。

```sql
  7    type t_fetch_size is table of pls_integer;
  8    l_fetch_sizes t_fetch_size := t_fetch_sizes(5,10,50,100,500,1000,10000,100000,1000000);
  9
 10    rc      sys_refcursor;
 11  begin
 12    select value
 13    into   l_pga_ceiling
 14    from   v$mystat m, v$statname s
 15    where  s.statistic# = m.statistic#
 16    and    s.name = 'session pga memory max';
 17
 18    dbms_output.put_line('初始 PGA: '||l_pga_ceiling);
 19
 20    for i in 1 .. l_fetch_sizes.count
 21    loop
```

对于每个获取大小，我将使用`LIMIT`子句从表中获取一组行。

```sql
 22      open rc for select descr from hardware;
 23      loop
 24         fetch rc bulk collect into l_rows limit l_fetch_sizes(i);
 25         exit when rc%notfound;
 26      end loop;
 27      close rc;
 28
```

然后，执行获取后，我将捕获会话级别的 PGA 统计信息，以查看该获取操作是否影响了数据库服务器上会话消耗的内存。以下是执行此操作的代码：

```sql
 29      select value
 30      into   l_pga_ceiling
 31      from   v$mystat m, v$statname s
 32      where  s.statistic# = m.statistic#
 33      and    s.name = 'session pga memory max';
 34
 35      dbms_output.put_line('获取大小: '||l_fetch_sizes(i));
 36      dbms_output.put_line('- PGA 最大值: '||l_pga_ceiling);
 37
 38    end loop;
 39
 40  end;
 41  /
```

```
初始 PGA: 3175904
获取大小: 5
- PGA 最大值: 3175904
获取大小: 10
- PGA 最大值: 3175904
获取大小: 50
- PGA 最大值: 3241440
获取大小: 100
- PGA 最大值: 3306976
获取大小: 500
- PGA 最大值: 3306976
获取大小: 1000
- PGA 最大值: 3372512
获取大小: 10000
- PGA 最大值: 4224480
获取大小: 100000
- PGA 最大值: 12482016
获取大小: 1000000
- PGA 最大值: 95122912
```


`PL/SQL 过程已成功完成。`

## 内存消耗与可扩展性问题

一旦提取的大小超过 1,000 行，PGA 消耗会从 3MB 增长到 95MB（当在单个提取调用中批量收集完整的 1,000,000 行数据集时）。如果你有数百个会话，每个会话都消耗数百兆字节的内存，这无疑会成为一个可扩展性威胁。除非你持有内存芯片公司的股票期权，否则使用过大的批量收集大小可能是个坏主意。因此，建议在不知道结果集大小（或近似大小）的情况下，永远不要在没有 `LIMIT` 子句的情况下对结果集执行 `FETCH BULK COLLECT`。

## 极端情况演示

极端情况下，一个失控的集合可能会耗尽服务器上的所有内存，并可能导致其崩溃。例如，下面的演示试图将十亿行数据批量收集到一个 PL/SQL 集合中；这个过程挂起了我的笔记本电脑（这就是为什么在本书的下载目录中找不到这个演示的脚本！）。

```sql
SQL> declare
  2    type t_huge_set is table of number;
  3    l_the_server_slaminator t_huge_set;
  4  begin
  5    select rownum
  6    bulk collect into l_the_server_slaminator
  7    from
  8      ( select level from dual connect by level <= 1000 ),
  9      ( select level from dual connect by level <= 1000 ),
 10      ( select level from dual connect by level <= 1000 );
 11  end;
 12  /
```

```
ERROR at line 1:
ORA-04030: out of process memory when trying to allocate 16396 bytes
```

## 关于 `pga_aggregate_target` 参数的说明

你可能会认为数据库初始化参数 `pga_aggregate_target` 能使你的系统免受此类问题的影响。这种想法并不正确。该参数仅适用于数据库可以按需内部调整的内存分配，例如用于排序或哈希的内存。如果你为 PL/SQL 集合内存请求 100GB 的 PGA，数据库将尝试满足该请求，无论这可能带来多大的麻烦。

## 利用自动优化进行性能权衡

因此，表面上看，在使用批量操作时，需要在性能和内存消耗之间取得平衡。让我们通过利用隐式游标循环中的自动批量收集优化来探讨这个问题（当 `plsql_optimize_level` 设置为 2 或更高时，这在 10g 及以后版本中是默认设置）。我将重新连接以重置会话级别统计信息，然后从 `HARDWARE` 表中提取全部 1,000,000 行，这相当于上一个消耗了 95MB PGA 的示例的最后一次迭代。以下示例（`bulk_collect_perf_test_2.sql`）演示了使用自动批量收集优化提取 1,000,000 行：

```sql
SQL> connect
Enter user-name: *****
Enter password:  *****
SQL> alter session set plsql_optimize_level = 2;

Session altered.

SQL> begin
  2    for i in (
  3      select descr d3
  4      from   hardware )
  5    loop
  6      null;
  7    end loop;
  8  end;
  9  /
```

```
PL/SQL procedure successfully completed.

Elapsed: 00:00:01.78
```

请注意，其性能与完整批量收集示例相当。通过检查会话跟踪，可以确定自动批量收集的提取大小。以下是先前 PL/SQL 块的跟踪输出：

```
SELECT DESCR D3
FROM   HARDWARE

call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          1          0           0
Execute      1      0.00       0.00          0          0          0           0
Fetch    10001      1.03       1.13          0      18907          0     1000000
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total    10003      1.03       1.14          0      18908          0     1000000
```

通过 10,001 次提取调用获取 1,000,000 行，表明提取大小为 100 行。尽管发出了 10,000 次提取调用而不是 1 次，但性能大致相同。一旦你的提取数组大小超过数据库块中的行数，收益就会递减，因此 100 左右的提取大小通常是最佳平衡点。与其仔细基准测试每一段代码来确定最佳批量收集大小，仅仅通过重构游标循环来利用编译器中的自动批量收集优化，其效果就已经接近最优了。

#### 重构代码以使用批量收集

到目前为止，将代码转换为使用批量收集听起来好得令人难以置信。非常简单的代码更改（或者如果你已经在使用隐式提取游标循环，则无需更改）就能带来巨大的性能提升。然而，在重构代码时，你需要防范几个陷阱。


## 未引发异常

传统的隐式游标在结果集中没有行时会引发 `no_data_found` 异常，你的应用程序可能依赖这一事实来采取补救措施。在以下示例 (`bulk_ndf_1.sql`) 中，`no_data_found` 用于指示查询谓词无效：

```sql
SQL> set serverout on
SQL> declare
  2    l_descr hardware.descr%type;
  3  begin
  4    select descr
  5    into   l_descr
  6    from   hardware
  7    where  aisle = 0
  8    and    item = 0;
  9    dbms_output.put_line('Item was found');
 10  exception
 11    when no_data_found then
 12      dbms_output.put_line('Invalid item specified');
 13  end;
 14  /
Invalid item specified

PL/SQL procedure successfully completed.
```

然而，当此示例转换为使用批量收集时（若不加以适当注意），代码中就会引入一个错误。请执行以下版本 (`bulk_ndf_2.sql`) 并在你自己的系统上检查结果：

```sql
SQL> set serverout on
SQL> declare
  2    type t_descr_list is table of hardware.descr%type;
  3    l_descr_list t_descr_list;
  4  begin
  5    select descr
  6    bulk collect
  7    into   l_descr_list
  8    from   hardware
  9    where  aisle = 0
 10    and    item = 0;
 11    dbms_output.put_line('Item was found');
 12  exception
 13    when no_data_found then
 14      dbms_output.put_line('Invalid item specified');
 15  end;
 16  /
Item was found  <==== 错误！

PL/SQL procedure successfully completed.
```

该过程运行成功但返回了错误的结果！这是因为一个批量收集获取调用将 **不会** 引发 `no_data_found` 异常。你可以将这种异常的缺失解释为 PL/SQL 声明它已成功地将可用行（本例中为零行）填充到了目标数组中。即使目标数组不包含任何数据，它也确实被初始化了。这一最终结果与目标数组完全未被初始化有细微差别。

以下示例 (`bulk_ndf_3.sql`) 展示了批量收集将零行返回到集合中，与完全在批量收集调用中不使用集合，这两者之间存在重要区别：

```sql
SQL> set serverout on
SQL> declare
  2    type t_descr_list is table of hardware.descr%type;
  3    l_descr_list t_descr_list;
  4  begin
  5    select descr
  6    bulk collect
  7    into   l_descr_list
  8    from   hardware
  9    where  aisle = 0
 10    and    item = 0;
 11    dbms_output.put_line(l_descr_list.count||' rows found');
 12  end;
 13  /
0 rows found

PL/SQL procedure successfully completed.
```

注释掉 `SELECT` 语句并再次执行相同的代码。批量收集永远不会运行。因此，目标数组永远不会被初始化。这种未初始化会导致错误。例如，`bulk_ndf_4.sql` 现在会在尝试引用集合内容时崩溃。

```sql
SQL> declare
  2    type t_descr_list is table of hardware.descr%type;
  3    l_descr_list t_descr_list;
  4  begin
  5  --  select descr
  6  --  bulk collect
  7  --  into   l_descr_list
  8  --  from   hardware
  9  --  where  aisle = 0
 10  --  and    item = 0;
 11    dbms_output.put_line(l_descr_list.count||' rows found');
 12  end;
 13  /
declare
*
ERROR at line 1:
ORA-06531: Reference to uninitialized collection
ORA-06512: at line 11
```

了解到即使没有返回行，批量收集操作也会初始化目标数组，这使得你可以重构现有代码而不会有太大困难。只需检查数组中的记录数，并在集合为空时显式引发 `no_data_found` 异常，即可保持其余代码像以前一样工作。例如，以下代码 (`bulk_ndf_5.sql`) 添加了一个条件检查，当集合为空时引发 `no_data_found`：

```sql
SQL> set serverout on
SQL> declare
  2    type t_descr_list is table of hardware.descr%type;
  3    l_descr_list t_descr_list;
  4  begin
  5    select descr
  6    bulk collect
  7    into   l_descr_list
  8    from   hardware
  9    where  aisle = 0
 10    and    item = 0;
 11
 12    if l_descr_list.count = 0 then
 13       raise no_data_found;
 14    end if;
 15
 16    dbms_output.put_line('Item was found');
 17  exception
 18    when no_data_found then
 19      dbms_output.put_line('Invalid item specified');
 20  end;
 21  /
Invalid item specified

PL/SQL procedure successfully completed.
```



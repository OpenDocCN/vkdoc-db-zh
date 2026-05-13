# 块清除与重做日志生成

令人惊讶的是，`SELECT` 语句也会生成重做日志（redo）。不仅如此，它还会“弄脏”这些被修改过的数据块，导致 `DBWn` 进程再次将它们写入磁盘。这是由于“块清除”（block cleanout）机制导致的。

## 现象观察

接下来，我将再次运行 `SELECT` 语句以访问每一个数据块，会发现没有生成重做日志。这是符合预期的，因为此时所有数据块都是“干净”的。我们将从创建表开始：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
  2  ( id number primary key,
  3  x char(2000),
  4  y char(2000),
  5  z char(2000));
Table created.

SQL> exec dbms_stats.set_table_stats( user, 'T', numrows=>10000, numblks=>10000 );
PL/SQL procedure successfully completed.
```

我使用 `DBMS_STATS` 来设置表统计信息，以避免后续硬解析可能带来的副作用（Oracle 在硬解析时倾向于扫描没有统计信息的对象，这个副作用会干扰我的示例！）。这样，我的表就创建好了，每个数据块包含一行数据（在我的 8KB 块大小的数据库中）。接下来，我们将检查将要针对此表执行的代码块：

```
SQL> declare
  2    l_rec t%rowtype;
  3  begin
  4    for i in 1 .. 10000
  5    loop
  6      select * into l_rec from t where id=i;
  7    end loop;
  8  end;
  9  /
declare
*
ERROR at line 1:
ORA-01403: no data found
ORA-06512: at line 6
```

这个代码块执行失败了，但这没关系——我们知道它会失败，因为表里还没有数据。我运行这个代码块只是为了提前完成 SQL 和 PL/SQL 的硬解析，这样稍后我们再次运行它时，就不必担心硬解析的副作用被计入测量结果。现在，我们准备将数据加载到表中并提交：

```
SQL> insert into t
  2  select rownum, 'x', 'y', 'z'
  3    from all_objects
  4   where rownum <= 10000;
10000 rows created.

SQL> commit;
Commit complete.
```

最后，我准备测量第一次读取数据时生成的重做日志量：

```
SQL> variable redo number
SQL> exec :redo := get_stat_val( 'redo size' );
PL/SQL procedure successfully completed.

SQL> declare
  2    l_rec t%rowtype;
  3  begin
  4    for i in 1 .. 10000
  5    loop
  6      select * into l_rec from t where id=i;
  7    end loop;
  8  end;
  9  /
PL/SQL procedure successfully completed.

SQL> exec dbms_output.put_line( (get_stat_val('redo size')-:redo) || ' bytes of redo generated...');
802632 bytes of redo generated...
PL/SQL procedure successfully completed.
```

因此，这个 `SELECT` 语句在处理过程中生成了大约 802KB 的重做日志。这代表了它在读取主键索引以及随后读取表 `T` 时所修改的块头信息。`DBWn` 会在未来的某个时间点将这些被修改的数据块写回磁盘（实际上，由于表无法完全放入缓存，我们知道 `DBWn` 至少已经写出了其中的一部分）。

现在，如果我再次运行这个查询：

```
SQL> exec :redo := get_stat_val( 'redo size' );
PL/SQL procedure successfully completed.

SQL> declare
  2    l_rec t%rowtype;
  3  begin
  4    for i in 1 .. 10000
  5    loop
  6      select * into l_rec from t where id=i;
  7    end loop;
  8  end;
  9  /
PL/SQL procedure successfully completed.

SQL> exec dbms_output.put_line( (get_stat_val('redo size')-:redo) || ' bytes of redo generated...');
0 bytes of redo generated...
PL/SQL procedure successfully completed.
```

我看到没有生成重做日志——所有数据块都是“干净”的了。

## 技术细节与影响

如果我们重新运行前面的示例，但将缓冲区缓存（buffer cache）设置为能容纳略多于 100,000 个块，我们会发现任何一次 `SELECT` 都几乎不会生成重做日志——在两次 `SELECT` 语句中，我们都不需要清理脏块。这是因为我们修改的 10,000 多个块（记住索引也被修改了）可以轻松放入缓冲区缓存的百分之十中，而且我们是唯一的用户。没有其他人干扰数据，也没有其他人导致我们的数据被刷新到磁盘或访问那些块。在一个活跃的系统中，至少有时部分块未被清理干净是正常现象。

这种行为在你执行大型的 `INSERT`（如刚才演示的）、`UPDATE` 或 `DELETE` 之后对你影响最大——即影响了数据库中许多块的操作（任何超过缓存大小百分之十的操作肯定会造成影响）。你会注意到，在此之后第一次访问该块的查询会生成一些重做日志并弄脏该块，如果 `DBWn` 已经将其刷出或实例已经关闭并清空了缓冲区缓存，还可能导致它被重写。对此你没有太多办法。这是正常且可预期的。如果 Oracle 不做这种延迟的块清除，那么一个 `COMMIT` 可能需要和事务本身一样长的时间来处理。`COMMIT` 将不得不重新访问每一个块，甚至可能需要再次从磁盘读入它们（因为它们可能已经被刷出了）。

如果你不了解块清除及其工作原理，它们就会成为那些神秘莫测、似乎无缘无故发生的事情之一。例如，假设你 `UPDATE` 了大量数据并 `COMMIT`。现在你运行一个查询来验证结果。这个查询似乎产生了大量的写 I/O 和重做日志。如果你不了解块清除，这看起来是不可能的；我第一次看到时也这么觉得。你去找人来一起观察这个行为，但它在第二次查询时无法重现，因为块现在是“干净”的了。你只能把它当作一个只有你独自一人时才会发生的数据库谜团而置之不理。

在 OLTP 系统中，你可能永远看不到块清除的发生，因为这些系统的特点是小型、短暂的事务，只影响少数几个块。根据设计，所有或大多数事务都是简短而精炼的。修改几个块，它们就全部被清除了。在数据仓库中，你在加载后对数据进行大规模的 `UPDATE` 操作，块清除可能会成为你设计中的一个因素。

某些操作会在“干净”的块上创建数据。例如，`CREATE TABLE AS SELECT`、直接路径加载的数据以及直接路径插入（使用 `/* +APPEND */` 提示）的数据都将创建干净的块。而一个 `UPDATE`、普通的 `INSERT` 或 `DELETE` 可能会创建需要在第一次读取时被清除的块。如果你的处理流程包括以下步骤，这可能会真正影响到你：

*   将大量新数据批量加载到数据仓库中。
*   对刚刚加载的所有数据运行 `UPDATE`（产生需要被清除的块）。
*   让人们查询这些数据。

你必须意识到，如果块需要被清除，那么第一次接触数据的查询将产生一些额外的处理开销。认识到这一点后，你自己应该在 `UPDATE` 后“接触”这些数据。你刚刚加载或修改了大量数据——至少你需要分析它。也许你需要运行一些报告来验证加载结果。这将清除块，使得下一个查询不必再做这件事。更好的是，既然你刚刚批量加载了数据，你现在就需要刷新统计信息。运行 `DBMS_STATS` 工具来收集统计信息很可能会清除所有的块，因为它只是使用 SQL 来查询信息，并且自然会在进行过程中清除块。



# 8. 事务

事务是将数据库与文件系统区分开来的特性之一。在文件系统中，如果你在写入文件的过程中操作系统崩溃，该文件很可能会损坏，尽管存在“日志式”文件系统等可能将文件恢复到某个时间点的机制。然而，如果你需要保持两个文件同步，这样的系统就无济于事了——如果你更新了一个文件，而系统在完成第二个文件更新之前就发生了故障，你的文件将无法保持同步。

这便是*事务*的主要目的——它们将数据库从一个一致的状态转移到下一个一致的状态。这就是它们的功能。当你在数据库中提交工作时，可以确保你的所有更改都已被保存，或者（在失败的情况下）一个更改都没有被保存。此外，还可以确保用于保护数据完整性的各种规则和检查都已被执行。

在上一章中，我们从并发控制的角度讨论了事务，以及作为 Oracle 多版本、读一致性模型的结果，Oracle 事务如何在高度并发的数据访问条件下每次都能提供一致的数据。Oracle 中的事务展现出所有必需的 ACID 特性：

*   *原子性*：一个事务要么全部发生，要么全不发生。
*   *一致性*：事务将数据库从一个一致状态转移到下一个一致状态。
*   *隔离性*：在事务提交之前，其效果可能对其他事务不可见。
*   *持久性*：一旦事务提交，它就是永久的。

特别是在上一章，我们讨论了 Oracle 如何获得*一致性*和*隔离性*。在这里，我们将主要关注*原子性*的概念及其在 Oracle 中的应用。

在本章中，我们将讨论原子性的影响及其如何影响 Oracle 中的语句。我们将涵盖 `COMMIT`、`SAVEPOINT` 和 `ROLLBACK` 等事务控制语句，并讨论事务中如何强制执行完整性约束。我们还将探讨如果你一直在其他数据库中开发，可能会有一些不良的事务习惯。我们将了解分布式事务和两阶段提交（2PC）。最后，我们将审视自治事务，它们是什么以及扮演的角色。

## 事务控制语句

在 Oracle 中，你不需要一个 "begin transaction"（开始事务）语句。事务由第一个修改数据的语句（第一个获取 `TX` 锁的语句）隐式开始。你可以使用 `SET TRANSACTION` 或 `DBMS_TRANSACTION` 包显式地开始一个事务，但这并非必需步骤，这与某些其他数据库不同。发出 `COMMIT` 或 `ROLLBACK` 语句会显式结束一个事务。

**注意**

并非所有的 `ROLLBACK` 语句都是等同的。需要注意的是，`ROLLBACK TO SAVEPOINT` 命令**不会**结束一个事务！只有完整的、正式的 `ROLLBACK` 才会结束事务。

你应该始终使用 `COMMIT` 或 `ROLLBACK` 显式地终止你的事务；否则，你使用的工具或环境将替你选择其一。如果你正常退出 `SQL*Plus` 会话，没有提交或回滚，`SQL*Plus` 会认为你希望提交工作并执行提交。另一方面，如果你只是从 `Pro*C` 程序中退出，则会发生隐式回滚。永远不要依赖隐式行为，因为它将来可能会改变。总是显式地 `COMMIT` 或 `ROLLBACK` 你的事务。

**注意**

作为一个未来可能发生变化的例子，Oracle `11*g*` Release 2 及以上版本的 `SQL*Plus` 包含一个设置 "exitcommit"。该设置控制 `SQL*Plus` 在退出时是发出 `COMMIT` 还是 `ROLLBACK`。所以当你使用 `11*g*` Release 2 时，自 `SQL*Plus` 发明以来一直存在的默认行为很可能会不同！

在 Oracle 中，事务是*原子的*，这意味着构成事务的每条语句要么全部提交（变为永久），要么所有语句全部回滚。这种保护也延伸到单个语句。一条语句要么完全成功，要么完全回滚。请注意，我说的是“语句”被回滚。一条语句的失败**不会**导致先前执行的语句自动回滚。它们的工作会被保留，并且必须由你来提交或回滚。在我们深入探讨语句和事务的原子性究竟意味着什么之前，让我们先看一下可用的各种事务控制语句：

*   `COMMIT`：使用此语句最简单的形式，你只需发出 `COMMIT`。你可以更详细地使用 `COMMIT WORK`，但两者是等效的。`COMMIT` 结束你的事务并使任何更改成为永久（持久）。在分布式事务中，`COMMIT` 语句有扩展功能，允许你用一些有意义的注释标记一个 `COMMIT`（标记事务），并强制提交一个处于不确定状态的分布式事务。还有一些扩展允许你执行异步提交——这实际上打破了*持久性*的概念。我们稍后会看一下这种情况，看看何时适合使用。
*   `ROLLBACK`：使用此语句最简单的形式，你只需发出 `ROLLBACK`。同样，你可以更详细地使用 `ROLLBACK WORK`，但两者是等效的。回滚结束你的事务并撤销任何未提交的更改。它通过读取存储在回滚/撤销段（从现在开始，我将专门称之为*撤销段*）中的信息，并将数据库块恢复到事务开始之前的状态来实现这一点。
*   `SAVEPOINT`：`SAVEPOINT` 允许你在事务中创建一个标记点。你可以在单个事务中拥有多个 `SAVEPOINT`。
*   `ROLLBACK TO <SAVEPOINT>`：此语句与 `SAVEPOINT` 命令配合使用。你可以将事务回滚到该标记点，而无需回滚其之前所做的任何工作。因此，你可以发出两条 `UPDATE` 语句，后跟一个 `SAVEPOINT`，然后再发出两条 `DELETE` 语句。如果在执行 `DELETE` 语句期间发生错误或某种异常情况，并且你捕获该异常并发出 `ROLLBACK TO SAVEPOINT` 命令，事务将回滚到命名的 `SAVEPOINT`，撤销 `DELETE` 语句所做的任何工作，但保留 `UPDATE` 语句所做的工作不变。
*   `SET TRANSACTION`：此语句允许你设置各种事务属性，例如事务的隔离级别，以及它是只读还是读写。你还可以使用此语句指示事务在使用手动撤销管理时使用特定的撤销段，但不推荐这样做。我们将在下一章更详细地讨论手动和自动撤销管理。

就这样——没有其他事务控制语句了。最常用的控制语句是 `COMMIT` 和 `ROLLBACK`。`SAVEPOINT` 语句有一个比较特殊的用途。在内部，Oracle 经常使用它；实际上，Oracle 在你执行任何 `SQL` 或 `PL/SQL` 语句时都会使用它，你可能会在应用程序中发现它的某些用途。

## 原子性

现在，我们准备看看语句、过程和事务的原子性意味着什么。


### 语句级原子性

考虑以下语句：

```
SQL> Insert into t values ( 1 );
```

乍看之下似乎很清楚，如果该语句因约束违反而失败，则该行不会被插入。然而，请考虑以下示例，其中对表 `T` 执行的 `INSERT` 或 `DELETE` 操作会触发一个触发器，该触发器相应地调整表 `T2` 中的 `CNT` 列：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t2 ( cnt int );
表已创建。
SQL> insert into t2 values ( 0 );
已创建 1 行。
SQL> commit;
提交完成。
SQL> create table t ( x int check ( x>0 ) );
表已创建。
SQL> create trigger t_trigger
before insert or delete on t for each row
begin
if ( inserting ) then
update t2 set cnt = cnt +1;
else
update t2 set cnt = cnt -1;
end if;
dbms_output.put_line( 'I fired and updated '  ||
sql%rowcount || ' rows' );
end;
/
触发器已创建。
```

在这种情况下，应该发生什么就不太清楚了。如果错误发生在触发器触发*之后*，触发器的效果应该保留还是取消？也就是说，如果触发器触发并更新了 `T2`，但行没有插入到 `T` 中，结果应该是什么？显然，答案是，如果行实际上没有插入到 `T` 中，我们不希望 `T2` 中的 `CNT` 列递增。幸运的是，在 Oracle 中，来自客户端的原始语句——本例中是 `INSERT INTO T`——要么完全成功，要么完全失败。这个语句是原子的。我们可以确认这一点，如下所示：

```
SQL> set serveroutput on
SQL> insert into t values(1);
I fired and updated 1 rows
已创建 1 行。
SQL> insert into t values(-1);
I fired and updated 1 rows
insert into t values(-1)
*
第 1 行出现错误:
ORA-02290: 违反约束条件 (EODA.SYS_C0013023)
SQL> select * from t2;
CNT
----------
1
```

因此，一行成功插入到 `T` 中，我们正确地收到了消息 `I fired and updated 1 rows`。下一条 `INSERT` 语句违反了我们在 `T` 上设置的完整性约束。`DBMS_OUTPUT` 消息出现了——`T` 上的触发器实际上确实触发了，我们有证据证明这一点。触发器成功地执行了其对 `T2` 的更新操作。我们可能预期 `T2` 现在的值是 2，但我们看到它的值是 1。Oracle 使*原始*的 `INSERT` 操作具有原子性——原始的 `INSERT INTO T` 就是这个语句，而该原始 `INSERT INTO T` 的任何副作用都被视为该语句的一部分。

Oracle 通过为我们对数据库的每次调用静默地包装一个 `SAVEPOINT` 来实现这种语句级原子性。前面的两个 `INSERT` 实际上是这样处理的：

```
Savepoint statement1;
Insert into t values ( 1 );
If error then rollback to statement1;
Savepoint statement2;
Insert into t values ( -1 );
If error then rollback to statement2;
```

对于习惯于 Sybase 或 SQL Server 的程序员来说，这起初可能令人困惑。在那些数据库中，*情况恰恰相反*。那些系统中的触发器独立于触发它的语句执行。如果它们遇到错误，触发器必须显式地回滚自己的工作，然后引发另一个错误来回滚触发语句。否则，触发器所做的工作可能会持续存在，即使触发语句或该语句的其他部分最终失败。

在 Oracle 中，这种语句级原子性会根据需要延伸到任意深度。在前面的例子中，如果 `INSERT INTO T` 触发了一个更新另一个表的触发器，而该表又有一个删除另一个表的触发器（如此层层递进），那么要么*所有*工作都成功，要么*全部*都不成功。你不需要编写任何特殊代码来确保这一点；它就是这样工作的。

### 过程级原子性

有趣的是，Oracle 将 PL/SQL 块也视为语句。考虑以下存储过程和示例表的重置：

```
$ sqlplus eoda/foo@PDB1
SQL> create or replace procedure p
as
begin
insert into t values ( 1 );
insert into t values (-1 );
end;
/
过程已创建。
SQL> delete from t;
I fired and updated 1 rows
已删除 1 行。
SQL> update t2 set cnt = 0;
已更新 1 行。
SQL> commit;
提交完成。
SQL> select * from t;
未选定行
SQL> select * from t2;
CNT
----------
0
```

因此，我们有一个已知会失败的存储过程，在这种情况下，第二个 `INSERT` 总是会失败。让我们看看运行该存储过程时会发生什么：

```
SQL> begin
p;
end;
/
I fired and updated 1 rows
I fired and updated 1 rows
begin
*
第 1 行出现错误:
ORA-02290: 违反约束条件 (EODA.SYS_C0013025)
ORA-06512: 在 "EODA.P", 第 5 行
SQL> select * from t;
未选定行
SQL> select * from t2;
CNT
----------
0
```

如你所见，Oracle 将存储过程调用视为一个原子语句。客户端提交了一个代码块——`BEGIN P; END;`——而 Oracle 在其周围包装了一个 `SAVEPOINT`。由于 `P` 失败了，Oracle 将数据库恢复到了调用它之前的状态。

**注意**

前面的行为——语句级原子性——依赖于 PL/SQL 例程不执行任何提交或回滚操作。我认为在 PL/SQL 中通常不应使用 `COMMIT` 和 `ROLLBACK`；只有 PL/SQL 存储过程的调用者才知道事务何时完成。在你开发的 PL/SQL 例程中发出 `COMMIT` 或 `ROLLBACK` 是不好的编程习惯。

现在，如果我们提交一个稍有不同的代码块，我们将得到完全不同的结果：

```
SQL> begin
p;
exception
when others then
dbms_output.put_line( 'Error!!!! ' || sqlerrm );
end;
/
I fired and updated 1 rows
I fired and updated 1 rows
Error!!!! ORA-02290: 违反约束条件 (EODA.SYS_C0013025)
PL/SQL 过程已成功完成。
SQL> select * from t;
X
----------
1
SQL> select * from t2;
CNT
----------
1
SQL> rollback;
回退完成。
```

在这里，我们运行了一个忽略任何和所有错误的代码块，结果差异巨大。第一次调用 `P` 没有产生任何更改，而这次第一个 `INSERT` 成功了，`T2` 中的 `CNT` 列也相应递增。

Oracle 将客户端提交的代码块视为“语句”。这个语句通过捕获并忽略错误本身而成功了，所以 `If error then rollback...` 没有生效，Oracle 在执行后也没有回滚到 `SAVEPOINT`。因此，`P` 所执行的部分工作得以保留。之所以最初会保留这部分工作，是因为我们在 `P` 内部具有语句级原子性：`P` 中的每个语句都是原子的。当 `P` 提交它的两个 `INSERT` 语句时，`P` 就成为了 Oracle 的客户端。每个 `INSERT` 要么完全成功，要么完全失败。这一点可以从以下事实得到证明：我们可以看到 T 上的触发器触发了两次，并更新了 `T2` 两次，但 `T2` 中的计数只反映了一次 `UPDATE`。在 `P` 中执行的第二个 `INSERT` 周围隐式地包装了一个 `SAVEPOINT`。

**“WHEN OTHERS” 子句**

我认为，几乎所有包含 `WHEN OTHERS` 异常处理器但没有同时包含 `RAISE` 或 `RAISE_APPLICATION_ERROR` 来重新引发异常的代码都是一个缺陷。它会静默地忽略错误，并改变了事务的语义。捕获 `WHEN OTHERS` 并将异常转换为旧式的返回码，改变了数据库应有的行为方式。


事实上，当 Oracle 11g Release 1 还处于规划阶段时，我获准提交三项 PL/SQL 新功能请求。我立即抓住了这个机会，而我的第一个建议就是“从语言中移除 `WHEN OTHERS` 子句”。我的理由很简单：我所见到的由开发者引入的最常见错误根源——*最常见原因*——就是一个 `WHEN OTHERS` 后面没有跟上 `RAISE` 或 `RAISE_APPLICATION_ERROR`。我认为如果没有这个语言特性，世界会更安全。当然，PL/SQL 实现团队无法做到这一点，但他们做了次优的选择。他们让 PL/SQL 在遇到一个 `WHEN OTHERS` 后面没有跟 `RAISE` 或 `RAISE_APPLICATION_ERROR` 调用时，生成一个编译器警告。例如：

```
SQL> alter session set
  2  PLSQL_Warnings = 'enable:all'
  3  /
Session altered.

SQL> create or replace procedure some_proc( p_str in varchar2 )
  2  as
  3  begin
  4     dbms_output.put_line( p_str );
  5  exception
  6     when others
  7     then
  8        -- call some log_error() routine
  9        null;
 10  end;
 11  /
SP2-0804: Procedure created with compilation warnings

SQL> show errors procedure some_proc
Errors for PROCEDURE P:
LINE/COL ERROR
-------- -----------------------------------------------------------------
1/1      PLW-05018: unit SOME_PROC omitted optional AUTHID clause; default
         value DEFINER used
6/10     PLW-06009: procedure "SOME_PROC" OTHERS handler does not end in
         RAISE or RAISE_APPLICATION_ERROR
```

因此，如果你在代码中包含了 `WHEN OTHERS`，且其后没有跟 `RAISE` 或 `RAISE_APPLICATION_ERROR`，那么请注意，你*几乎可以肯定*是在看一段自己开发的代码中的错误，一个由你放置在那里的错误。

包含 `WHEN OTHERS` 异常块的代码与不包含它的代码之间的差异是微妙的，这是你在应用程序中必须考虑的一点。为 PL/SQL 代码块添加异常处理程序会从根本上改变其行为。另一种编码方式——一种能恢复整个 PL/SQL 块的语句级原子性的方式——如下所示：

```
SQL> begin
  2     savepoint sp;
  3     p;
  4  exception
  5     when others then
  6        rollback to sp;
  7        dbms_output.put_line( 'Error!!!! ' || sqlerrm );
  8  end;
  9  /
I fired and updated 1 rows
I fired and updated 1 rows
Error!!!! ORA-02290: check constraint (EODA.SYS_C0013025) violated
PL/SQL procedure successfully completed.

SQL> select * from t;
no rows selected

SQL> select * from t2;
       CNT
----------
         0
```

**注意**
前面的代码代表了一种极其糟糕的做法。一般来说，你既不应该捕获 `WHEN OTHERS`，也不应该显式编码 Oracle 在事务语义方面已经提供的内容。

在这里，通过模拟 Oracle 通常为我们使用 `SAVEPOINT` 所做的工作，我们能够在捕获并“忽略”错误的同时恢复原始行为。我提供这个例子仅用于说明；这是一种极其糟糕的编码实践。

## 事务级原子性

一个事务，即一组作为工作单元一起执行的 SQL 语句，其整个目标是将数据库从一个一致状态带到另一个一致状态。为了实现这个目标，事务也是原子的——事务执行的所有成功的工作要么全部提交并永久化，要么回滚并撤销。就像一条语句一样，事务是一个原子工作单元。在提交事务后收到数据库返回的“成功”消息时，你就知道该事务执行的所有工作都已被持久化。

## DDL 与原子性

值得注意的是，Oracle 中有一类特定的语句是原子的——但仅在语句级别。数据定义语言语句的实现方式使得：

1.  它们首先提交任何未完成的工作，结束你可能已有的任何事务。
2.  它们执行 DDL 操作，例如 `CREATE TABLE`。
3.  如果 DDL 操作成功，它们就提交该操作；否则就回滚该操作。

这意味着，每当你发出一条 DDL 语句（如 `CREATE`、`ALTER` 等）时，你必须预期你现有的事务会立即被提交，并且随后的 DDL 命令会被执行，然后要么被提交并持久化，要么在出现任何错误时被回滚。DDL 并不会以任何方式破坏 ACID 概念，但它会提交的事实是你绝对需要意识到的。

## 持久性

通常，当事务被提交时，它的更改是永久性的；你可以依赖这些更改存在于数据库中，即使数据库在提交完成的瞬间崩溃。然而，在两种特定情况下，这并不成立：

*   你使用了 `COMMIT` 语句中可用的 `WRITE` 扩展。
*   你在非分布式（仅访问单个数据库，没有数据库链接）的 PL/SQL 代码块中发出 `COMMIT`。

我们将依次查看每种情况。



### 写入对提交的扩展

Oracle 允许你在 `COMMIT` 语句中添加一个 `WRITE` 子句。`WRITE` 子句允许提交操作要么 `WAIT`（等待）你生成的重做日志被写入磁盘（默认行为），要么 `NOWAIT`（不等待）重做日志被写入。`NOWAIT` 选项是一种能力——一种必须谨慎、深思熟虑且完全理解其含义后才能使用的能力。

通常，`COMMIT` 是一个同步过程。你的应用程序调用 `COMMIT`，然后 `等待` 整个 `COMMIT` 处理完成。这是 Oracle 10*g* Release 2 之前所有数据库版本中 `COMMIT` 的行为，也是 Oracle 10*g* Release 2 及以上版本的默认行为。

在当前版本的数据库中，你可以选择让提交在后台执行，而无需等待其完成（因为提交涉及对存储在磁盘上的重做日志文件进行物理写入——物理 I/O——这可能需要可计量的时间），而不必等待提交完成。这带来的副作用是 *你的提交不再保证是持久的*。也就是说，你的应用程序可能会从数据库收到响应，表明你提交的异步提交已被接收；其他会话可能能够看到你的更改，但随后可能发现你以为已提交的事务并未提交。这种情况只会发生在非常罕见的情况下，并且总是涉及硬件或软件的严重故障。它需要数据库异常关闭才能导致异步提交不持久，这意味着数据库实例或运行数据库实例的计算机必须遭受完全故障。

那么，如果事务旨在保持持久性，一个可能使它们不再持久的功能有什么潜在用途呢？原始性能。当你在应用程序中发出 `COMMIT` 时，你是在要求 `LGWR` 进程获取你生成的重做日志并确保其被写入联机重做日志文件。执行物理 I/O（此过程所涉及的）速度相对较慢；将数据写入磁盘需要相对较长的时间。因此，`COMMIT` 可能比事务本身的 DML 语句花费的时间还要长！如果你将 `COMMIT` 变为异步，就消除了客户端应用程序等待该物理 I/O 的需要，或许能使客户端应用程序显得更快——尤其是当它执行大量 `COMMIT` 操作时。

这可能会让你想一直使用这个 `COMMIT WRITE NOWAIT`——毕竟，性能难道不是世界上最重要的事情吗？不，并非如此。大多数时候，你需要通过默认的 `COMMIT` 实现的持久性。当你 `COMMIT` 并向最终用户报告“我们已经提交”时，你需要确保更改是永久的。即使提交后数据库/硬件立即发生故障，它也会被记录在数据库中。如果你向最终用户报告“订单 12352 已下达”，你需要确保订单 12352 确实已下达并持久存在。因此，对于几乎所有的应用程序，默认的 `COMMIT WRITE WAIT` 是唯一正确的选项（注意，你只需说 `COMMIT`——默认设置就是 `WRITE WAIT`）。

那么，你何时会想要使用这种无需等待即可提交的能力呢？有三种场景可以考虑：

*   一个自定义的数据加载程序。它必须是自定义的，因为它需要包含额外的逻辑来处理提交可能无法在系统故障中持久化的情况。
*   处理某种实时数据流的应用程序，例如从股票市场插入大量时效性信息的股票行情馈送。如果数据库离线，数据流会继续，系统故障期间生成的数据将永远不会被处理（毕竟，纳斯达克不会因为你的数据库崩溃而关闭！）。这些数据不被处理是可以接受的，因为股票数据时效性极强；几秒钟后，它就会被新数据覆盖。
*   实现自身“排队”机制的应用程序，例如，一个在表中存储数据并带有 `PROCESSED_FLAG` 列的应用程序。新数据到达时，以 `PROCESSED_FLAG='N'`（未处理）的值插入。另一个例程负责读取 `PROCESSED_FLAG='N'` 的记录，执行一些小的、快速的事务，并将 `PROCESSED_FLAG` 从 `'N'` 更新为 `'Y'`。如果它提交了，但该提交后来被撤销（由于系统故障），这是可以接受的，因为处理这些记录的应用程序只会再次处理该记录——它是“可重新处理的”。

如果你看看这些应用程序类别，你会注意到这三类都是后台、非交互式应用程序。它们不直接与人交互。任何与人交互的应用程序——向人报告“提交完成”的应用程序——都应该使用同步提交。异步提交不是为面向在线客户的应用程序准备的调优手段。异步提交仅适用于面向批处理的应用程序，即那些在故障后可以自动重新启动的应用程序。交互式应用程序在故障后无法自动重新启动——必须由人工重新执行该事务。因此，你还有另一个标志告诉你是否可以考虑使用此功能——你拥有的是批处理应用程序还是交互式应用程序？除非它是面向批处理的，否则同步提交是正确的方式。

因此，除了那三类批处理应用程序之外，这个功能——`COMMIT WRITE NOWAIT`——可能不应该被使用。如果你确实使用了它，你需要问自己：如果你的应用程序被告知 *提交已处理*，但后来提交被撤销了，会发生什么？你需要能够回答这个问题并得出结论：如果这种情况发生，结果是可以接受的。如果你无法回答这个问题，或者如果已提交的更改丢失会产生严重后果，你就不应该使用异步提交功能。



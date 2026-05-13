# 11. 设计事务策略

一个正确实施的事务策略将使每个系统受益。本章将提供一套关于该主题的通用指南，并讨论如何提高系统的并发性。

### 事务策略设计注意事项

一致的事务和错误处理策略总是有利于系统。它们有助于减少阻塞，并在阻塞发生时简化故障排除。

正如我们在第 2 章中已经讨论过的，在客户端和服务器端事务管理之间的选择很大程度上取决于数据访问层的架构。基于存储过程的实现可能受益于从存储过程中启动的显式事务。另一方面，使用 ORM 框架或代码生成器的客户端实现则需要在客户端代码中管理事务。

有一种常见的误解，认为自动提交事务可能对系统有益。尽管这种方法可能在一定程度上减少阻塞——毕竟，每个语句都在自己的事务中运行，排他(X)锁的持有时间更短——但这很难说是最佳选择。大量的小事务可能会显著增加事务日志活动并降低系统性能。

更重要的是，当多个相关的数据修改由于错误而部分失败时，自动提交事务可能会引入数据质量问题。当这些问题在生产环境中发生时，极难诊断和解决。在绝大多数情况下，最好在系统中使用显式事务。

#### 提示

避免自动提交事务，改用显式事务。

你可以通过使用`SET XACT_ABORT ON`选项来进一步减少出现数据质量问题的可能性。如你所记，此设置使事务在发生任何错误时变为不可提交状态。这可以防止在部分数据修改未成功完成时，显式事务被提交。



#### 提示

使用 `SET XACT_ABORT ON`。

回顾 `BEGIN TRAN/COMMIT` 语句的嵌套行为。如果您在同一模块中调用 `BEGIN TRAN` 和 `COMMIT`，则无需检查 `@@TRANCOUNT` 变量和活动事务的存在。但请记住，`ROLLBACK` 语句会回滚整个事务，而不考虑 `@@TRANCOUNT` 嵌套级别。在回滚之前，最好检查是否有活动事务。

11-1 清单展示了在启动事务前检查是否存在活动事务的代码示例。由于 `BEGIN TRAN/COMMIT` 语句的嵌套行为，这完全是不必要的，因此您可以从实现中删除 `IF` 语句。

```sql
create proc dbo.Proc1
as
begin
    set xact_abort on
    declare @CurrentTranCount = @@TRANCOUNT;
    if @CurrentTranCount = 0 -- IF is not required and can be removed
        begin tran;
    /* Some logic here */
    if @CurrentTranCount = 0  -- IF is not required and can be removed
        commit;
end
```
清单 11-1
不必要的活动事务检查实现

11-2 清单展示了执行服务器端事务管理和错误处理的存储过程模板。假设调用代码正确处理了异常，此方法无论该存储过程是从外部调用还是在活动事务中调用都有效。

需要注意的是，`CATCH` 块正在检查 `@@TRANCOUNT` 是否大于零。一个常见的错误是使用 `IF @@TRANCOUNT = 1 ROLLBACK` 模式，这与嵌套的 `BEGIN TRAN` 调用不兼容。

```sql
create proc dbo.MyProc
as
begin
    set xact_abort on
    begin try
        begin tran
        /* Some logic here  */
        commit
    end try
    begin catch
        if @@TRANCOUNT > 0 -- Transaction is active
            rollback;
        /* Optional error-handling  code */
        throw;
    end catch;
end;
```
清单 11-2
服务器端事务管理

客户端事务管理的实现取决于系统的技术和架构。然而，使用 `TRY..CATCH` 块并在那里显式提交或回滚事务总是有益的。11-3 清单展示了使用经典 ADO.Net 的这种方法。

```csharp
using (SqlConnection conn = new SqlConnection(connString))
{
    conn.Open();
    using (SqlTransaction tran = conn.BeginTransaction(IsolationLevel.ReadCommitted))
    {
        try
        {
            SqlCommand cmd = conn.CreateCommand("exec dbo.MyProc @Param1");
            cmd.Parameters.Add("@Param1",SqlDbType.VarChar,255).Value = "Param Value";
            cmd.Transaction = tran;
            cmd.ExecuteNonQuery();
            tran.Commit();
        }
        catch (Exception ex)
        {
            tran.Rollback();
            throw;
        }
    }
}
```
清单 11-3
ADO.Net 事务管理

尽管客户端代码需要在 `BeginTransaction()` 和 `ExecuteNonQuery()` 调用之间执行若干操作，但它不会在系统中引入任何低效。SQL Server 将事务视为在第一次数据访问操作时开始，而不是在 `BEGIN TRAN` 调用时开始。此外，在事务完成第一次数据修改之前，它不会在事务日志中记录事务的开始 (`LOP_BEGIN_XACT`)。

您应该记住 `SNAPSHOT` 事务的这种行为，它们处理的是事务开始时的数据“快照”。在实践中，这意味着此类事务将看到第一次数据访问操作（无论是读还是写）时的数据。

## 选择事务隔离级别

选择合适的事务隔离级别并非易事。您需要在阻塞和 `tempdb` 开销与系统所需的数据一致性和隔离级别之间找到适当的平衡。系统必须向客户提供可靠的数据，您不应为了减少阻塞而选择无法保证数据可靠性的隔离级别。

您应选择提供所需数据一致性级别的*最低要求*隔离级别。在许多情况下，默认的 `READ COMMITTED` 隔离级别已经*足够好*，特别是当查询经过优化且不执行不必要的扫描时。除非有正当理由，否则避免在 OLTP 系统中使用 `REPEATABLE READ` 或 `SERIALIZABLE` 隔离级别。这些隔离级别会持有共享（S）锁直到事务结束，这可能导致易失性数据的严重阻塞问题。它们还会在扫描期间触发共享（S）锁升级。

在系统中使用不同的隔离级别是完全正常的。例如，财务管理系统在执行可能影响客户账户余额的操作时，可能需要使用 `REPEATABLE READ` 甚至 `SERIALIZABLE` 隔离级别。然而，其他用例（例如更改客户配置文件信息）可能完全适用于 `READ COMMITTED` 级别。

作为一般规则，最好避免使用 `READ UNCOMMITTED` 隔离级别。尽管许多数据库专业人员尝试通过显式切换或使用 `(NOLOCK)` 提示切换到此隔离级别来减少阻塞，但这很少是正确的选择。首先，`READ UNCOMMITTED` 不能解决写入者引入的阻塞问题。它们在扫描期间仍然获取更新（U）锁。然而，最重要的是，通过使用 `READ UNCOMMITTED`，您是在声明根本不需要数据一致性，而这不仅仅是读取未提交数据的问题。SQL Server 可能选择在大型表上使用*分配映射扫描*的执行计划，这可能导致行丢失和重复读取，尤其是在具有易失性数据的繁忙系统中，这源于页面拆分。

在大多数情况下，乐观隔离级别，特别是 `READ COMMITTED SNAPSHOT`，是比 `READ UNCOMMITTED`、`REPEATABLE READ` 或 `SERIALIZABLE` 更好的选择，即使在 OLTP 系统中也是如此。它在不涉及读写器阻塞的情况下提供语句级数据一致性。历史上，由于其 `tempdb` 开销，我在 OLTP 系统中建议 `RCSI` 时一直非常谨慎；然而，如今，由于现代硬件和基于闪存的磁盘阵列，这已不那么是个问题。尽管如此，您仍应在分析中考虑额外的索引碎片和 `tempdb` 开销。同样值得重复的是，`READ COMMITTED SNAPSHOT` 在 Azure SQL 数据库中默认启用。

作为一般规则，我建议不要在 OLTP 系统中使用 `SNAPSHOT` 隔离级别，因为它会过度使用 `tempdb`，除非绝对需要事务级一致性。对于数据大部分时间是静态的数据仓库和报告系统，它可能是一个不错的选择。

如果在数据库中启用了乐观隔离级别，您必须非常小心事务管理。导致未提交事务的代码中的错误会阻止 `tempdb` 版本存储清理，并导致 `tempdb` 数据文件过度增长。即使您不在系统中使用乐观隔离级别，只要启用了 `READ_COMMITTED_SNAPSHOT` 或 `ALLOW_SNAPSHOT_ISOLATION` 数据库设置，这种情况也可能发生。


### 乐观隔离级别与阻塞减少模式

然而，乐观隔离级别常常会`掩盖`系统中未优化的查询。尽管这些查询是导致系统性能低下的原因，但它们并未参与阻塞条件，因此常被忽视。常见的情况是，人们通过启用`READ COMMITTED SNAPSHOT`来“解决”读写阻塞，但之后并未解决阻塞的根本原因。你应该牢记这一点，并且无论系统中是否存在阻塞，都应进行查询优化。

对于数据仓库系统，事务策略很大程度上取决于数据的更新方式。对于静态只读数据，任何隔离级别都适用，因为读取者不会阻塞其他读取者。你甚至可以将数据库或文件组切换为只读模式，以减少锁开销。否则，乐观隔离级别是更好的选择。它们为报表查询提供事务级或语句级一致性，并能在 ETL 过程中减少阻塞。当此方法可行时，你也应考虑在 ETL 过程中利用表分区并使用分区切换。

## 减少阻塞的模式

当多个会话竞争同一组资源时，就会发生阻塞。会话试图在其上获取不兼容的锁，这会导致锁冲突和阻塞。

正如你所知，SQL Server 在`处理`数据时获取锁。需要修改或返回给客户端的行数并不那么重要。更重要的是 SQL Server 在语句执行期间`访问`了多少行。一个查询可能只选择或更新了一行，但由于执行了过度扫描，它完全可能获取了数千甚至数百万个锁。

适当的查询优化和索引调优可以减少 SQL Server 在查询执行期间需要访问的行数。这反过来又减少了获取的锁数量以及发生锁冲突的机会。

#### 提示

优化查询。这将有助于提高系统中的并发性、性能和用户体验。

减少锁冲突机会的另一种方法是减少锁的持有时间。排他锁（X 锁）总是持有到事务结束。在`REPEATABLE READ`和`SERIALIZABLE`隔离级别下的共享锁（S 锁）也是如此。锁持有时间越长，发生锁冲突和阻塞的可能性就越大。

你需要使事务尽可能短，并避免在事务活动期间执行任何长时间操作以及与用户通过 UI 进行交互。在处理使用 CLR 或链接服务器的外部资源时也需要小心。例如，当链接服务器宕机时，可能需要很长时间才会发生连接超时，而你希望避免锁在此期间一直被保持的情况。

#### 提示

使事务尽可能短。

尽可能在事务接近结束时更新数据。这减少了排他锁（X 锁）的持有时间。在某些情况下，使用临时表作为暂存区是有意义的：先将数据插入其中，然后在事务的最后更新实际表。

#### 提示

尽可能在事务接近结束时修改数据。

这种技术的一个特定变体是，对于一条无法优化或不切实际的`UPDATE`语句。考虑这样一种情况：该语句扫描大量行，但只更新其中一小部分。你可以修改代码，将需要更新的行的聚集索引键值存储在临时表中，然后基于这些收集到的键值运行`UPDATE`。

清单 11-4 展示了一条在执行时可能导致聚集索引扫描的语句示例。SQL Server 将需要获取表中每一行的更新锁（U 锁）。

```
update dbo.Orders
set
Cancelled = 1
where
(PendingCancellation = 1) or
(Paid = 0 and OrderDate < @StockDate)
Listing 11-4
使用临时表减少阻塞：原始语句
```

你可以将代码修改为类似于清单 11-5 所示。`SELECT`语句根据隔离级别获取共享锁（S 锁）或根本不获取行级锁。`UPDATE`语句是经过优化的，只获取少量的更新锁（U 锁）和排他锁（X 锁）。

```
create table #OrdersToBeCancelled
( OrderId int not null primary key );
insert into #OrdersToBeCancelled(OrderId)
select OrderId
from dbo.Orders
where
(PendingCancellation = 1) or
(Paid = 0 and OrderDate < @StockDate);
update dbo.Orders
set Cancelled = 1
where OrderId in (select OrderId from #OrdersToBeCancelled);
Listing 11-5
使用临时表减少阻塞：利用临时表暂存待更新的键值
```

你需要记住，虽然这种方法有助于减少阻塞，但创建和填充临时表可能会带来显著的 I/O 开销，尤其是在涉及大量数据时。在某些情况下，你可以使用 CTE 来避免这种开销，如清单 11-6 所示。

```
;with UpdateIds(OrderId)
as
(
select OrderId
from dbo.Orders
where
(PendingCancellation = 1) or
(Paid = 0 and OrderDate < @StockDate)
)
update o
set o.Cancelled = 1
from UpdateIds u inner loop join dbo.Orders o on
o.OrderId = u.OrderId
Listing 11-6
使用 CTE 减少阻塞
```

与前面的例子类似，`SELECT`语句在扫描期间不获取更新锁（U 锁）。`inner loop`连接提示保证排他锁（X 锁）仅保持在需要修改的行上。记住，连接提示强制语句中的连接顺序。在我们的例子中，需要将 CTE 指定为连接的*左*输入/表，以生成正确的执行计划。

这两种方法都可能以引入额外开销为代价来减少阻塞。此开销会随着待更新数据量的增加而增加，如果你预计要更新表中的大部分行，则不应使用这些方法。记住，在大多数情况下，创建正确的索引是更好的选择。

#### 提示

避免在大表上进行更新扫描。

当`UPDATE`语句修改不同的非聚集索引中的数据时，应避免在同一事务中多次更新数据行。请记住，SQL Server 在更新索引行时是基于每个索引来获取锁的。多次更新会增加其他会话访问已更新行时发生死锁的机会。

#### 提示

不要在单个事务中多次更新数据行。

你需要了解锁升级是否影响你的系统，尤其是在 OLTP 工作负载的情况下。你可以监视对象级阻塞条件和锁等待，然后将其与 `lock_escalation` 扩展事件和跟踪事件关联起来。记住，锁升级有助于减少内存消耗并提高系统性能。在做出任何决定之前，你应该分析锁升级发生的原因及其对系统的影响。在许多情况下，改变代码和工作流程比禁用锁升级更好。

#### 提示

监视系统中的锁升级。

通常，你应该避免在同一事务中混合使用可能导致行级锁和对象级锁的语句，特别是混合 DML 和 DDL 语句。这种模式可能导致意向锁与完整的对象级锁之间的阻塞，以及死锁情况。当服务器具有 16 个或更多逻辑 CPU 时（这启用了锁分区），这一点尤其重要。


## 总结

不要在单个事务中混合使用 DDL 和 DML 语句。

如果你的系统中出现死锁，你需要分析其根本原因。在大多数情况下，查询优化和代码重构有助于解决它们。你还应该考虑在系统中的关键用例周围实施重试逻辑。

找到死锁的根本原因。如果查询优化和代码重构无法解决它们，则实施重试逻辑。

完全消除系统中的所有阻塞是不可能的。幸运的是，理解阻塞的根本原因有助于设计能够缓解该问题的解决方案。

一致的事务和错误处理策略可以减少阻塞并简化并发问题的故障排除。客户端和服务器端实现的选择取决于数据访问层架构；然而，作为一般规则，你应该使用显式事务而非自动提交事务。

业务需求应决定系统中的数据一致性和隔离规则。你应该选择满足这些需求的`最小所需`隔离级别。除非绝对必要，否则不要使用`READ UNCOMMITTED`。

乐观隔离级别是可以接受的，即使是 OLTP 工作负载，只要系统能够处理额外的`tempdb`开销。除非需要事务级一致性，否则最好使用`READ COMMITTED SNAPSHOT`。

在大多数情况下，适当的查询优化和索引调整有助于提高并发性。经过良好优化的查询获取的锁更少，这降低了锁冲突和系统中阻塞的可能性。你还应尽量缩短事务，并在事务结束前修改数据，以减少持有锁的时间。

每个系统都是独特的，无法提供普遍适用的通用建议。然而，对 SQL Server 并发模型的良好理解将帮助你设计正确的事务策略，并解决系统中的任何阻塞和并发问题。


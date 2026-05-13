# 非分布式 PL/SQL 块中的 COMMIT 操作

自 PL/SQL 在 Oracle 第 6 版中首次引入以来，它一直透明地使用异步提交。这种方法之所以有效，是因为在某种程度上所有 PL/SQL 都像批处理程序一样——最终用户在过程完全结束之前并不知道其结果。这也是为什么这种异步提交仅用于非分布式 PL/SQL 代码块；如果我们涉及多个数据库，那么就有两个事物——两个数据库——依赖于提交的持久性。当两个数据库都依赖于提交具有持久性时，我们必须使用同步协议，否则更改可能在一个数据库中提交而在另一个中未提交。

**注意**

当然，流水线 PL/SQL 函数与“普通”PL/SQL 函数有所不同。在普通 PL/SQL 函数中，直到存储过程调用结束才知道结果。流水线函数通常能够在完成之前很早就向客户端返回数据（它们将数据“块”一点一点地返回给客户端）。但由于流水线函数是从`SELECT`语句调用的，并且无论如何都不会提交，因此它们在讨论中不涉及。

因此，PL/SQL 被开发为利用异步提交，允许 PL/SQL 中的`COMMIT`语句不必等待物理 I/O 完成（避免“log file sync”等待）。这并不意味着你不能依赖一个提交并返回控制权给你的应用程序的 PL/SQL 例程对其更改的持久性——PL/SQL 会等待它生成的重做日志写入磁盘后再返回给客户端应用程序——但它只会等待一次，就在它返回之前。

**注意**

以下示例展示了一个不良实践——我称之为“逐行慢速处理”或“逐行处理”，因为在关系数据库中，逐行等同于慢速。这只是为了说明 PL/SQL 如何处理`COMMIT`语句。

首先，我们创建表`T`：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
  2  as
  3  select *
  4  from all_objects
  5  where 1=0;
Table created.
```

现在考虑这个 PL/SQL 过程：

```
SQL> create or replace procedure p
  2  as
  3  begin
  4    for x in ( select * from all_objects )
  5    loop
  6      insert into t values X;
  7      commit;
  8    end loop;
  9  end;
 10  /
Procedure created.
```

该 PL/SQL 代码从`ALL_OBJECTS`一次读取一条记录，将记录插入表`T`，并在每次插入后提交。从逻辑上讲，该代码与此相同：

```
SQL> create or replace procedure p
  2  as
  3  begin
  4    for x in ( select * from all_objects )
  5    loop
  6      insert into t values X;
  7      commit write NOWAIT;
  8    end loop;
  9    -- make internal call here to ensure
 10    -- redo was written by LGWR
 11  end;
 12  /
Procedure created.
```

因此，例程中执行的提交是使用`WRITE NOWAIT`完成的，并且在 PL/SQL 代码块返回给客户端应用程序之前，PL/SQL 确保它生成的最后一部分重做日志已安全记录到磁盘——从而使 PL/SQL 代码块及其更改具有持久性。

## 完整性约束与事务

有趣的是，需要确切了解完整性约束何时被检查。默认情况下，完整性约束在整个 SQL 语句处理完毕后进行检查。还有可延迟约束，允许将完整性约束的验证推迟到应用程序通过发出`SET CONSTRAINTS ALL IMMEDIATE`命令请求验证，或者在发出`COMMIT`时进行验证。

### 即时约束

对于本讨论的第一部分，我们将假设约束处于`IMMEDIATE`模式，这是常规情况。在这种情况下，完整性约束在整个 SQL 语句处理后立即检查。请注意，我使用了“SQL 语句”这一术语，而不仅仅是“语句”。如果我在一个 PL/SQL 存储过程中有许多 SQL 语句，每个 SQL 语句将在其单独执行后立即验证其完整性约束，而不是在存储过程完成时。

那么，为什么约束在 SQL 语句执行*之后*验证，而不是*执行过程中*验证？这是因为单个语句使表中的个别行暂时不一致是非常自然的。查看语句的部分工作会导致 Oracle 拒绝结果，即使最终结果可能是正常的。例如，假设我们有一个这样的表：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t ( x int unique );
Table created.
SQL> insert into t values ( 1 );
1 row created.
SQL> insert into t values ( 2 );
1 row created.
SQL> commit;
Commit complete.
```

然后我们想执行一个多行`UPDATE`：

```
SQL> update t set x=x-1;
2 rows updated.
```

如果 Oracle 在每行更新后检查约束，那么在任何时候我们都有 50%的机会让`UPDATE`失败。`T`表中的行以*某种*顺序被访问，如果 Oracle 先更新`X=1`的行，我们会暂时出现`X`值重复，从而拒绝`UPDATE`。由于 Oracle 会耐心地等到语句结束，因此该语句会成功，因为到完成时，就没有重复值了。



# 事务隔离级别与锁策略

## 可重复读 与锁提示

将事务 1 的隔离级别提升至 `Repeatable Read` 将防止事务 2 在事务 1 执行期间修改数据，从而避免产品价格不一致的问题。

由于意图是直到事务结束前都不释放 `SELECT` 语句获取的 `(S)` 锁，因此将隔离级别设置为 `Repeatable Read` 的效果也可以通过在查询级别使用锁提示来实现。

```sql
DECLARE @Price INT ;

BEGIN TRAN NormalizePrice

SELECT @Price = Price

FROM dbo.MyProduct AS mp WITH (REPEATABLEREAD)

WHERE mp.ProductID = 1 ;

/*Allow transaction 2 to execute*/

WAITFOR DELAY '00:00:10'

IF @Price > 10

UPDATE dbo.MyProduct

SET Price = Price - 10

WHERE ProductID = 1 ;

COMMIT
```

此解决方案防止了 `MyProduct.Price` 的数据不一致，但它为此场景引入了另一个问题。观察事务 2 的结果后，你可能会意识到它可能导致死锁。因此，尽管上述解决方案防止了数据不一致，但它并不是一个完整的解决方案。仔细观察 `Repeatable Read` 隔离级别对事务的影响，你会发现它引入了之前解释过的由 `UPDATE` 语句内部实现所避免的典型死锁问题。`SELECT` 语句获取并保留了一个 `(S)` 锁，而不是 `(U)` 锁，即使它意图在事务后期修改数据。`(S)` 锁允许事务 2 获取一个 `(U)` 锁，但它阻止了 `(U)` 锁向 `(X)` 锁的转换。事务 1 在后期尝试获取 `(U)` 锁导致了循环阻塞，从而造成死锁。

为了防止死锁并同时避免数据损坏，你可以采用与 `UPDATE` 语句内部实现所采用的等效策略。因此，事务 1 可以在执行 `SELECT` 语句时使用 `UPDLOCK` 锁提示来请求一个 `(U)` 锁，而不是请求一个 `(S)` 锁。

```sql
DECLARE @Price INT ;

BEGIN TRAN NormalizePrice

SELECT @Price = Price

FROM dbo.MyProduct AS mp WITH (UPDLOCK)

WHERE mp.ProductID = 1 ;

/*Allow transaction 2 to execute*/

WAITFOR DELAY '00:00:10'

IF @Price > 10

UPDATE dbo.MyProduct

SET Price = Price - 10

WHERE ProductID = 1 ;

COMMIT
```

此解决方案同时防止了数据不一致和死锁的可能性。如果将隔离级别提升至 `Repeatable Read` 没有引入典型的死锁，那么它本可以完成任务。由于因为 `(S)` 锁保留到事务结束而存在发生死锁的机会，通常更倾向于获取一个 `(U)` 锁，而不是持有 `(S)` 锁，如上所述。

www.it-ebooks.info
第 20 章 ■ 阻塞与被阻塞的进程

### 可序列化

`Serializable` 是六个隔离级别中最高的级别。`Serializable` 隔离级别不是仅在要访问的行上获取锁，而是在数据集请求顺序中的该行和下一行上获取范围锁。

例如，在 `Serializable` 隔离级别执行的 `SELECT` 语句会在要访问的行及其顺序中的下一行上获取 `(RangeS-S)` 锁。这阻止了其他事务在第一个事务操作的数据集中添加行，并保护第一个事务在其事务范围内不会在数据集中发现新行。在事务内的数据集中发现新行也称为*幻读*。

为了理解 `Serializable` 隔离级别的必要性，让我们考虑一个例子。假设公司中的一个小组（`GroupID = 10`）有 100 美元的基金，将作为奖金分配给小组中的员工。奖金支付后的基金余额应为 0。考虑以下测试表：

```sql
IF (SELECT OBJECT_ID('dbo.MyEmployees')

) IS NOT NULL

DROP TABLE dbo.MyEmployees ;

GO

CREATE TABLE dbo.MyEmployees

(EmployeeID INT,

GroupID INT,

Salary MONEY

) ;
```



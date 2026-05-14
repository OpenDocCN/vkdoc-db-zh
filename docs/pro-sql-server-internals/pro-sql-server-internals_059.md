# 第 23 章 ■ 架构锁

## 图 23-11. 低优先级锁

■ **重要提示** 必须记住，一旦获取了低优先级锁，其行为将与常规锁相同，会阻止其他会话在该资源上获取不兼容的锁。

图 23-12 展示了来自第 18 章代码清单 18-1 的查询输出。它演示了低优先级锁如何在 `sys.dm_tran_locks` 数据管理视图的输出中显示。值得注意的是，该视图并未提供这些锁的等待时间。

## 图 23-12. `sys.dm_tran_locks` 数据管理视图中的低优先级锁

你可以在 `ALTER INDEX` 和 `ALTER TABLE` 语句中使用 `WAIT_AT_LOW_PRIORITY` 子句来指定锁优先级，如代码清单 23-1 所示。

### 代码清单 23-1. 指定锁优先级

```
alter index PK_Customers on Delivery.Customers rebuild
with
(
    online=on
    (
        wait_at_low_priority
        ( max_duration=10 minutes, abort_after_wait=blockers )
    )
);

alter table Delivery.Orders
switch partition 1 to Delivery.OrdersTmp
with
(
    wait_at_low_priority
    ( max_duration=60 minutes, abort_after_wait=self )
)
```

如你所见，`WAIT_AT_LOW_PRIORITY` 有两个选项。`MAX_DURATION` 设置指定锁等待时间（以分钟为单位）。`ABORT_AFTER_WAIT` 设置定义了如果在指定的时间限制内无法获取锁时，会话的行为。可能的值如下：

`NONE`：低优先级锁将转换为常规锁。转换后，其行为与常规锁相同。在等待获取锁期间，它会阻塞那些想要获取不兼容锁类型的会话。该会话会持续等待，直到锁被获取。

`SELF`：如果在 `MAX_DURATION` 设置指定的时间内无法授予锁，则中止该操作。

`BLOCKERS`：所有持有该资源锁的会话将被中止，而等待低优先级锁的会话将能够获取它。

■ **注意** 省略 `WAIT_AT_LOW_PRIORITY` 子句的效果与指定 `WAIT_AT_LOW_PRIORITY(MAX_DURATION=0 MINUTES, ABORT_AFTER_WAIT=NONE)` 相同。

非常活跃的 OLTP 表总是有大量并发会话访问。因此，即使指定了较长的 `MAX_DURATION`，某个会话仍有可能无法获取低优先级锁。你可以考虑使用 `ABORT_AFTER_WAIT=BLOCKERS` 选项，这将允许操作完成，特别是在客户端应用程序已实现适当的异常处理和重试逻辑的情况下。

#### 总结

SQL Server 使用架构锁来保护元数据，防止其在查询编译和执行期间被更改。SQL Server 中有两种类型的架构锁：架构稳定性锁 (Sch-S) 和架构修改锁 (Sch-M)。

架构稳定性锁 (Sch-S) 在查询编译和执行期间，在查询引用的对象上获取。然而，在某些情况下，SQL Server 可以用意向表锁替代架构稳定性锁 (Sch-S)，后者同样可以保护表结构。架构稳定性锁 (Sch-S) 与任何其他锁类型都兼容，但与架构修改锁 (Sch-M) 不兼容。

架构修改锁 (Sch-M) 与任何其他锁类型都不兼容。SQL Server 在 DDL 操作期间使用它们。如果 DDL 操作需要扫描或修改数据（例如，向表添加受信任的外键约束，或在非空分区上修改分区函数），则架构修改锁 (Sch-M) 将在整个操作期间保持。在大型表上，这可能耗时很长，并可能导致系统中严重的阻塞问题。在设计时需要牢记这一点。



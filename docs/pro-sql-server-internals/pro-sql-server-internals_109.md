# 第 28 章 ■ 系统故障排除

另一种模式是读取并同时处理数据，如清单 28-12 所示。

客户端应用程序逐行消费和处理数据，保持 `SqlDataReader` 处于打开状态。因此，工作线程会等待客户端消费完所有行，从而产生 `ASYNC_NETWORK_IO` 等待类型。

### 清单 28-12. 数据的读取与处理：不正确的实现

```csharp
using (SqlConnection connection = new SqlConnection(connectionString))
{
    SqlCommand command = new SqlCommand(cmdText, connection);
    connection.Open();
    using (SqlDataReader reader = command.ExecuteReader())
    {
        while (reader.Read())
            ProcessRow((IDataRecord)reader);
    }
}
```

处理这种情况的正确方法是，先尽可能快地读取所有行，然后在所有行都读取完毕后再进行处理。清单 28-13 说明了这种方法。

### 清单 28-13. 数据的读取与处理：正确的实现

```csharp
List<Orders> orderRows = new List<Orders>();
using (SqlConnection connection = new SqlConnection(connectionString))
{
    SqlCommand command = new SqlCommand(cmdText, connection);
    connection.Open();
    using (SqlDataReader reader = command.ExecuteReader())
    {
        while (reader.Read())
            orderRows.Add(ReadOrderRow((IDataRecord)reader));
    }
}
ProcessAllOrderRows(orderRows);
```

你可以在 Management Studio 中运行测试，连接到本地的 SQL Server 实例，轻松重现此行为。它将使用共享内存协议，不涉及任何网络流量。你可以使用命令 `DBCC SQLPERF ('sys.dm_os_wait_stats', CLEAR)` 清除服务器上的等待统计信息，然后运行一条读取大量数据的 `SELECT` 语句，并在结果网格中显示。如果在执行后检查等待统计信息，你会看到大量的 `ASYNC_NETWORK_IO` 等待，这是因为网格显示性能缓慢，即使 Management Studio 是运行在本地的 SQL Server 主机上。之后，你应该启用*执行后丢弃结果*配置设置来重复该测试。你应该会看到 `ASYNC_NETWORK_IO` 等待消失了。

如果你在系统中看到很大比例的 `ASYNC_NETWORK_IO` 等待，你应该检查网络性能并分析客户端代码。

## 闩锁与自旋锁

闩锁是轻量级的同步对象，用于保护 SQL Server 内部数据结构的一致性。与保护事务数据一致性的锁相反，闩锁用于防止内存中的数据结构损坏。

考虑一种情况：多个会话需要更新同一页数据上的不同行。这些会话不会相互阻塞，因为它们并未在同一对象上获取不兼容的锁。然而，SQL Server 必须防止多个会话同时更新内存中的数据页结构，导致其不一致并损坏。此外，SQL Server 还需要防止其他会话在修改期间访问该数据页结构。SQL Server 使用闩锁来实现这一点。

SQL Server 中有五种不同的闩锁类型，如下所示：

- `KP` – 保持闩锁确保被引用的结构不会被销毁。除了销毁（`DT`）闩锁外，它与任何其他闩锁类型都兼容。
- `SH` – 共享闩锁在线程需要读取数据结构时是必需的。共享闩锁彼此兼容，也与保持（`KP`）和更新（`UP`）闩锁兼容。
- `UP` – 更新闩锁允许其他线程读取该结构，但防止更新该结构。SQL Server 在某些场景中使用它们来提高并发性，类似于更新（`U`）锁。更新闩锁与保持（`KP`）和共享（`SH`）闩锁兼容，与任何其他类型不兼容。
- `EX` – 独占闩锁在修改数据结构时是必需的。从概念上讲，独占（`EX`）闩锁类似于独占（`X`）锁，它们



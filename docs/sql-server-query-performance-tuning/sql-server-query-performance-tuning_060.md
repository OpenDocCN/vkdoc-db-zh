# 第 6 章 ■ 查询性能指标

这些步骤将生成创建会话并将其输出到文件所需的脚本。

要手动创建此新跟踪，请按以下方式使用 Management Studio：

1.  打开脚本文件或导航到查询窗口。
2.  修改服务器上为创建此会话设置的路径和文件位置。
3.  执行脚本。

会话创建完成后，您可以使用以下命令启动它：

```sql
ALTER EVENT SESSION [Query Performance Metrics]
ON SERVER
STATE = START;
```

您可能希望通过 SQL Agent 自动化执行最后一步，或者甚至可以使用 `sqlcmd.exe` 实用程序从命令行运行脚本。无论您使用哪种方法，最后一步都将启动会话。

要停止会话，只需运行相同的脚本并将 `STATE` 设置为 `stop`。我将在下一节中演示如何操作。

### 使用 T-SQL 定义会话

如果您查看上一节中定义的脚本，您会看到用于定义会话的单个命令 `CREATE EVENT SESSION`。

会话定义后，您可以使用 `ALTER EVENT` 激活它。

一旦会话在服务器上启动，您就不再需要保持 Management Studio 处于打开状态。您可以使用动态管理视图 `sys.dm_xe_sessions` 来识别活动的会话，如下面的查询所示：

```sql
SELECT dxs.name, dxs.create_time
FROM sys.dm_xe_sessions AS dxs;
```

图 6-9 显示了该视图的输出。

**图 6-9.** `sys.dm_xe_sessions` 的输出

返回的行数表示 SQL Server 上活动的会落数量。除了本章中创建的会话外，我还有另外两个会话在运行。您可以通过执行存储过程 `ALTER EVENT SESSION` 来停止特定的会话。

```sql
ALTER EVENT SESSION [Query Performance Metrics]
ON SERVER
STATE = STOP;
```

要验证会话是否已成功停止，请重新执行对目录视图 `sys.dm_xe_sessions` 的查询，并确保该视图的输出不再包含该命名的会话。

[www.it-ebooks.info](http://www.it-ebooks.info/)


# 第五章 通过 SQL 脚本实现 SQL Server 审核

这是审核发生前的等待时间（以毫秒为单位）。你可以将其设置为 `0`、`1000` 或大于 `1000` 的某个值。我将其保留为 `1000`。

## 审核日志失败时的操作 – `ON_FAILURE`
*   **`CONTINUE`** – 如果审核无法捕获语句，它将继续审核。不会因为此选项而导致任何语句失败。你可能会偶尔遗漏个别语句，但我怀疑这种情况即使有，也不常发生。
*   **`FAIL_OPERATION`** – 如果无法进行审核，它将导致该语句执行失败。执行该语句的用户或应用程序将收到错误。
*   **`SHUTDOWN`** – 如果无法进行审核，它将按描述执行；它将关闭服务器。所有用户和应用程序将无法再访问服务器。

我在此选项上选择“继续”。我认为“执行失败操作”和“关闭服务器”都过于极端。如果我不选择“继续”，人们会因为服务器出现问题而对我大喊大叫。如果在法律或金融数据库等审核至关重要的场景中，你可能需要选择“执行失败操作”或“关闭服务器”。

## 路径 – `FILEPATH`
`N'e:\audits\'`
如果你选择了“文件”作为审核目标，你需要指定一个路径。**请确保不要将审核文件放在 C 盘上**。即使我们将在后续步骤中限制审核文件的大小，你也不希望它意外地填满 C 盘。我也不建议将审核文件放在数据驱动器或日志驱动器上。我工作的地方有一个 E 盘用于存放应用程序；那是放置审核文件的好地方。

## 最大文件数 – `MAX_FILES`
`4`

## 最大文件大小 – `MAXSIZE`
`50 MB`
我从不让审核收集无限数量的文件或允许无限的文件大小。这会使查询它们变得困难。我发现，对于收集权限和架构更改，4 个文件、每个 50 MB 对我的需求来说是合适的。文件的数量和大小取决于你的需求。

## 预留磁盘空间 – `RESERVE_DISK_SPACE`
`OFF`
由于我的文件相当小，我将其设置为 `OFF`。

## 启用审核
`ALTER SERVER AUDIT [AuditSpecification] WITH (STATE = ON)`
你需要启用它，它才会开始收集审核数据。

**注意** 不要将 SQL Server 审核设置为收集无限文件或允许无限文件大小。它们会变得巨大无比，几乎无法查询。

如果你想将审核数据存储在应用程序日志中，你需要按照代码清单 5-2 所示设置审核。

### 代码清单 5-2. 配置写入应用程序日志的审核
```sql
USE [master];

CREATE SERVER AUDIT [AuditSpecification]
TO APPLICATION_LOG
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT [AuditSpecification] WITH (STATE = ON);
```

如果你想将审核数据存储在安全日志中，你需要按照代码清单 5-3 所示设置审核。

### 代码清单 5-3. 配置写入安全日志的审核
```sql
USE [master];

CREATE SERVER AUDIT [AuditSpecification]
TO SECURITY_LOG
WITH (QUEUE_DELAY = 1000, ON_FAILURE = CONTINUE);

ALTER SERVER AUDIT [AuditSpecification] WITH (STATE = ON);
```

当你尝试启用配置为使用安全日志的审核时，会遇到如代码清单 5-4 所示的错误。

### 代码清单 5-4. 创建写入安全日志的审核时出现的错误
`Msg 33222, Level 16, State 1, Line 7`
`Audit 'AuditSpecification3' failed to start. For more information, see the SQL Server error log.`

![](img/index-78_1.png)
![](img/index-78_2.jpg)

SQL Server 日志将为你提供有关该错误的更多信息，如图 5-2 所示。

### 图 5-2. 创建写入安全日志的审核时出现的错误的附加信息

要解决错误 `33204`（SQL Server 审核无法写入安全日志），你需要根据这份 Microsoft 文档来配置其他项目：





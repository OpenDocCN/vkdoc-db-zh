# 第 16 章  父子设计模式

即使是包内的单个任务和容器，也可以将它们的属性绑定到参数。这使得父包对子包的执行有了更进一步的控制。父包可以直接将信息传递给任务，覆盖设计时的属性配置。这个过程与将参数绑定到包属性相同。你需要右键单击要参数化的对象，然后选择 **参数化** 选项。

### 日志记录

使用父子设计模式记录执行过程可能有点棘手。开箱即用的包日志记录功能只会简单地将每条消息分开处理。父包的消息和执行事件会包围来自子包的消息和事件，但除了查看它们的时间安排并了解 ETL 过程外，没有真正的方法将它们的执行关联起来。我们在第 13 章会详细讨论日志记录。

### 实现数据驱动的 ETL

父子设计模式最灵活的实现之一是数据驱动的 ETL。实现这一点的最简单方法是将数据存储在表中以便查询。将数据存储在表中允许你快速地向流程中添加或移除包。如果你不想永久地将某些包从流程中移除，它还允许你禁用这些包的执行。清单 16-3 展示了一个可用于驱动你的 ETL 过程的表结构。

#### 清单 16-3. 表 CH16_Apress_PackageExecution

```sql
CREATE TABLE dbo.CH16_Apress_PackageExecution
(
Package NVARCHAR(250) NOT NULL,
PackagePath NVARCHAR(200) NOT NULL,
ParentPackage NVARCHAR(250) NULL,
ExecuteOrder INT NOT NULL,
DisablePackage BIT NOT NULL,
);
GO

CREATE CLUSTERED INDEX CIX_CH16_Apress_PackageExecution ON dbo.CH16_Apress_PackageExecution (ParentPackage);
GO
```

这个表的设计旨在展示 ETL 过程的层次结构视图。你可以看到父子关系，甚至可以编写查询来获取有关流程的详细信息。如果不需要多次执行同一个包，可以在这个表上定义主键。下面简要说明了每列的含义：`Package` 提供当前包的子包名称。包装包将被列为 `ParentPackage`，所有工作包都将列在此列中。根级别的父包将把它的子包列为包装包。

`PackagePath` 提供子包的文件夹路径。

`www.it-ebooks.info`


# 第 12 章：统计信息、数据分布与基数

对于大型表，你可能会发现需要更频繁地更新统计信息。如前所述，你可以使用跟踪标志 `2371` 来修改统计信息自动更新的默认行为。

启用该跟踪标志后，系统会采用一个滑动刻度，随着系统内数据量的增加而更频繁地更新统计信息。

### 异步自动更新统计信息

如果将异步自动更新统计信息设置为开启，SQL Server 中统计信息的基本行为并不会发生根本性改变。当一组统计信息被标记为过时，并且随后有查询使用这些统计信息运行时，统计信息更新**不会**像通常那样中断查询的执行。相反，查询会使用旧的统计信息集完成执行。查询完成后，统计信息才会被更新。这种做法可能吸引人的原因是，当统计信息更新时，存储过程缓存中的查询计划会被移除，而正在运行的查询必须重新编译。因此，与其让查询同时等待统计信息更新和存储过程的重新编译，不如让查询先完成其运行。下次调用同一个查询时，它将使用已更新的统计信息，并且只需进行重新编译即可。

尽管此功能确实使重新编译在一定程度上变快，但它也可能导致本可以从更新的统计信息和新执行计划中受益的查询，继续使用旧的执行计划。在开启此功能之前，需要进行仔细的测试，以确保它不会弊大于利。

**注意**：若要异步更新统计信息，你必须先将 `AUTO_UPDATE_STATISTICS` 设置为 `ON`。

### 手动维护

在以下情况下，你需要干预或协助统计信息的自动维护：

*   当试验统计信息时：一个友好的建议——请将你的生产服务器与你在本书中进行的此类实验隔离开来。
*   从早期版本升级到 SQL Server 2014 之后：由于 SQL Server 2014 的统计信息维护已升级和修改，升级后应立即手动更新整个数据库的统计信息，而不是等待 SQL Server 借助自动统计功能随时间推移进行更新。此外，我建议你对此统计信息更新使用 `FULLSCAN`，以确保它们尽可能准确。据我所知，唯一不适用的版本是从 SQL Server 2008 到 SQL Server 2008 R2 的版本。关于这是否必要存在一些争议，但在大多数情况下，这样做是安全且稳妥的。
*   在执行一系列你不会再次运行的临时 SQL 活动时：在这种情况下，你必须权衡是否愿意支付自动统计信息维护的成本，以在这一次获得更好的计划，同时影响其他 SQL Server 活动的性能。因此，一般来说，你可能不需要关心这种一次性操作。这主要适用于较大的数据库，但如果你认为可能适用，可以在你的环境中进行测试。
*   当你遇到自动统计信息维护问题，并且目前的唯一解决方法是关闭自动统计信息维护功能时：即使在这些情况下，你也可以只针对遇到问题的特定数据库表关闭此功能，而不是为整个数据库禁用它。此类问题可能出现在数据更新频繁但不足以触发更新阈值的大型数据集中。此外，当自动更新的采样级别对于某些数据分布不够充分时，也可以使用此方法。
*   在分析查询性能时，你发现查询引用的少数数据库对象缺少统计信息：这可以从图形和 XML 执行计划中进行评估，如本章前面所述。
*   在分析统计信息的有效性时，你发现它们不准确：当本应良好的统计信息集却生成了糟糕的执行计划时，就可以确定这一点。

SQL Server 允许用户控制其许多自动统计信息维护功能。你可以分别使用 `auto create statistics` 和 `auto update statistics` 设置来启用（或禁用）自动统计信息创建和更新功能，然后你就可以动手操作了。

#### 管理统计信息设置

你可以在数据库级别控制 `auto create statistics` 设置。要禁用此设置，请使用 `ALTER DATABASE` 命令：

```sql
ALTER DATABASE AdventureWorks2012 SET AUTO_CREATE_STATISTICS OFF;
```

你可以在数据库的不同级别控制 `auto update statistics` 设置，包括表上的所有索引和统计信息，或者在单个索引或统计信息级别。要在数据库级别禁用 `auto update statistics`，请使用 `ALTER DATABASE` 命令。

```sql
ALTER DATABASE AdventureWorks2012 SET AUTO_UPDATE_STATISTICS OFF;
```

在数据库级别禁用此设置会覆盖较低级别的单个设置。`异步更新统计信息` 要求 `自动更新统计信息` 首先处于开启状态。然后你才能启用异步更新。

```sql
ALTER DATABASE AdventureWorks2012 SET AUTO_UPDATE_STATISTICS_ASYNC ON;
```

要为当前数据库中表上的所有索引和统计信息配置 `auto update statistics`，请使用 `sp_autostats` 系统存储过程。

```sql
USE AdventureWorks2012;
EXEC sp_autostats 'HumanResources.Department', 'OFF';
```

你也可以使用相同的存储过程为单个索引或统计信息配置此设置。要为 `AdventureWorks2012.HumanResources.Department` 上的 `AK_Department_Name` 索引禁用此设置，请执行以下语句：

```sql
EXEC sp_autostats 'HumanResources.Department', 'OFF', AK_Department_Name;
```

你还可以使用 `UPDATE STATISTICS` 命令的 `WITH NORECOMPUTE` 选项，为当前数据库中表上的所有或单个索引和统计信息禁用此设置。`sp_createstats` 存储过程也有 `NORECOMPUTE` 选项。`NORECOMPUTE` 选项不会禁用数据库的自动统计信息更新，但会针对给定的一组统计信息禁用自动更新。

避免禁用自动统计信息功能，除非你已通过测试确认这能带来性能优势。如果自动统计信息功能被禁用，那么你就有责任手动识别未索引列上缺失的统计信息并进行创建，然后保持现有统计信息为最新状态。一般来说，你只会在非常大的表上禁用自动统计信息功能。

如果你想检查某个表是否已关闭其自动统计信息，可以使用：

```sql
EXEC sp_autostats 'HumanResources.Department';
```

重置索引的自动维护，以便在已关闭的位置将其重新开启。

```sql
EXEC sp_autostats 'HumanResources.Department', 'ON';
EXEC sp_autostats 'HumanResources.Department', 'ON', AK_Department_Name;
```

#### 生成统计信息

要手动创建统计信息，请使用以下选项之一：

*   `CREATE STATISTICS`：你可以使用此选项在表或索引视图的单个或多个列上创建统计信息。与 `CREATE INDEX` 命令不同，`CREATE STATISTICS` 默认使用采样。




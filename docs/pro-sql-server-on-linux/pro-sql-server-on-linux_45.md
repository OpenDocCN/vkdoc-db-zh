# 第六章 性能能力

分离数据库和事务日志文件的另一个重要好处是可恢复性。如果数据库文件丢失但当前的事务日志可用（假设你正在使用完整恢复模型；关于完整恢复模型的更多信息请参见 [`docs.microsoft.com/sql/relational-databases/backup-restore/recovery-models-sql-server#RMov`](https://docs.microsoft.com/sql/relational-databases/backup-restore/recovery-models-sql-server#RMov)），反之亦然，则可以执行高级恢复。

我在前面章节中用于创建数据库的语法没有指定文件名或文件路径。如果你为完整的 WideWorldImporters 示例数据库生成脚本，你将看到以下语法：

```
CREATE DATABASE [WideWorldImporters]
CONTAINMENT = NONE
ON  PRIMARY
( NAME = N'WWI_Primary', FILENAME = N'/var/opt/mssql/data/WideWorldImporters.mdf' , SIZE = 1048576KB , MAXSIZE = UNLIMITED, FILEGROWTH = 65536KB ),
FILEGROUP [USERDATA]  DEFAULT
( NAME = N'WWI_UserData', FILENAME = N'/var/opt/mssql/data/WideWorldImporters_UserData.ndf' , SIZE = 2097152KB , MAXSIZE = UNLIMITED, FILEGROWTH = 65536KB ),
FILEGROUP [WWI_InMemory_Data] CONTAINS MEMORY_OPTIMIZED_DATA  DEFAULT
( NAME = N'WWI_InMemory_Data_1', FILENAME = N'/var/opt/mssql/data/WideWorldImporters_InMemory_Data_1' , MAXSIZE = UNLIMITED)
LOG ON
( NAME = N'WWI_Log', FILENAME = N'/var/opt/mssql/data/WideWorldImporters.ldf' , SIZE = 102400KB , MAXSIZE = 2048GB , FILEGROWTH = 65536KB )
GO
```

请注意 `FILENAME` 参数用于指定文件的路径和物理名称。要分离事务日志，你需要从单独的磁盘挂载一个目录，并指定该磁盘上事务日志的路径。我将在下一节中向你展示如何使用文件组来实现这一点的示例。


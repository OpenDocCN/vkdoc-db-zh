# 3. 连接和查询 Azure SQL

一旦创建并配置了数据库实例，您的下一个任务就是将新开发或现有的应用程序连接到它，并开始执行数据操作或检索命令。

Azure SQL 是一种云原生数据库服务，通过多种进程间通信（IPC）机制（如 TCP/IP 套接字、命名管道或共享内存）与外部应用程序和进程进行通信。`T-SQL`（SQL Server 自己的 SQL 方言）中的命令（如 `SELECT`/`INSERT`/`UPDATE`/`DELETE`）以及从服务返回的结果集都被打包到称为 `TDS`（Tabular Data Stream，[`https://aka.ms/mstds`](https://aka.ms/mstds)）的应用层协议中。

当然，作为应用程序开发人员，您不必在自己的应用程序中针对这些底层协议进行编码。它们通常由全面的驱动程序和库系列抽象出来，这些驱动程序和库几乎涵盖了市场上可用的每一种现代编程语言和框架，并可在 Windows、Linux 和 macOS 操作系统上运行。

本章中的所有示例均使用 Visual Studio Code 编辑器（[`https://code.visualstudio.com/`](https://code.visualstudio.com/)）编辑，并通过命令行工具和相应运行时（如 .NET Core 3.1、OpenJDK 11 和 Python 3.6.6）的 SDK 在 Windows、macOS 或 Linux 操作系统上构建和执行。

## 驱动程序和库

大多数驱动程序都围绕一些基本结构设计，主要代表以下常见实体：

*   与服务器/数据库的连接
*   在该连接上执行的命令
*   用于在返回的结果集上迭代和访问记录的对象

一些库还提供更高级的数据操作功能，例如可以存储检索到的行、跟踪离线修改并提供所包含行的当前/先前版本以用于悲观并发多用户场景的断开连接缓存（例如，`ADO.NET DataSets/DataTable`）。

更高级别的框架和库也可用于覆盖特定场景，例如性能（例如，考虑为本身不提供连接池的库（如 `JDBC` 驱动程序）提供连接池）或生产力工具（如对象关系映射器和微 ORM）以加快开发时间。此表汇总了各种编程语言适用于所有平台（Windows、Linux、macOS）的 Azure SQL 客户端驱动程序：

| **语言** | **驱动程序库** | **版本** |
| --- | --- | --- |
| .NET 语言（C#、F# 等） | Microsoft ADO.NET for SQL Server | V1.1+ |
| Java | Microsoft JDBC driver for SQL Server | V8.2+ |
| PHP | PHP SQL driver for SQL Server | V 5.8+ |
| Node.js | Node.js Tedious driver for SQL Server | V8.0.1+ |
| Python | Python ODBC bridge (pyodbc) | V4.0.30+ |
| Go | Microsoft SQL Server Driver for Go |   |
| Ruby | Ruby driver for SQL Server | V2.1.0+ |
| 原生语言（如 C/C++） | Microsoft ODBC driver for SQL Server | V17.5.1.1+ |



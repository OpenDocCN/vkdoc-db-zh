# 第 7 章 SQLCLR：架构与设计考量

集成特性虽尚未被大规模使用，但其采用率确实在稳步增长。SQL Server 2008 解除了先前的限制，该限制曾将 CLR 用户定义类型（UDT）的数据容量约束为最大仅 8KB，这严重制约了许多潜在应用场景；现在，所有 CLR UDT 单个项最多可容纳 2GB 数据。这为在数据库中存储更多新型复杂对象数据开辟了众多潜在途径，而 `SQLCLR` 比主要基于集合的 `T-SQL` 引擎更胜任此任务。实际上，SQL Server 2008 引入了三种新的系统定义数据类型（`geometry`、`geography` 和 `hierarchyid`），它们出色地展示了 `SQLCLR` 如何扩展 SQL Server，以高效存储和查询超出 SQL 数据库通常关联的标准数值和字符型数据之外的数据类型。

我将在第 10 章和第 12 章详细阐述系统定义的 CLR 数据类型，这两章将分别讨论空间数据和层次数据。然而，本章重点聚焦于在 SQL Server 中利用基于托管代码的用户定义函数的设计与性能考量，以及讨论何时应考虑使用 `SQLCLR` 而非更传统的 `T-SQL` 方法。我认为，`SQLCLR` 集成的主要优势在于能够在各层之间移动和共享代码——因此本章的主要关注点在于可维护性和重用场景。

> **注意：** 本章假设你已熟悉基本的 `SQLCLR` 主题，包括如何创建和部署函数、注册新程序集，以及 C# 编程语言。

## 弥合 SQL/CLR 差距：SqlTypes 库

.NET Framework 和 SQL Server 暴露的原生数据类型在许多情况下相似，但通常不兼容。从数据类型角度看，处理 SQL Server 和 .NET 互操作性时，会出现几个主要问题：

- 首要的是，所有原生 SQL Server 数据类型都是 `可为空` 的——即，任何给定类型的实例可以包含该类型域内的有效值，也可以表示未知（`NULL`）。.NET 中的类型通常不支持这一概念（注意 C# 的 `null` 和 VB .NET 的 `Nothing` 与 SQL Server 的 `NULL` 不同）。尽管 .NET Framework 为值类型变量支持可为空类型，但它们的行为方式与 SQL Server 中的对应类型并不相同。
- 类型系统之间的第二个差异与实现有关。两个系统中涉及的类型的格式、精度和标度差异巨大。例如，.NET 的 `DateTime` 类型支持的范围和精度远大于 SQL Server 的 `datetime` 类型。
- 第三个主要差异与类型结合运算符时的运行时行为有关。例如，在 SQL Server 中，几乎所有涉及至少一个 `NULL` 类型实例的操作都会产生 `NULL`。然而，这与 .NET 中作用于 `null` 值的操作行为并不相同。考虑以下 `T-SQL`：

```sql
DECLARE @a int = 10;
DECLARE @b int = null;
IF (@a != @b)
    PRINT 'test is true';
ELSE
    PRINT 'test is false';
```

在 `T-SQL` 中，任何与 `NULL` 值的比较结果都是未定义的，因此前面的代码将打印 "test is false"。然而，考虑使用 C# 中可为空的 `int` 类型（在类型声明后用 `?` 字符表示）实现的等效函数：

```csharp
int? a = 10;
int? b = null;
if (a != b)
    Console.Write("test is true");
else
    Console.Write("test is false");
```

在 .NET 中，会进行 `10` 和 `null` 的比较，导致代码打印 "test is true"。除了可空性之外，处理溢出、下溢和其他潜在错误的不一致性也可能导致差异。例如，在 .NET 语言中，向值为 `2147483647`（32 位整数最大值）的 32 位整数加 `1` 可能导致值“回绕




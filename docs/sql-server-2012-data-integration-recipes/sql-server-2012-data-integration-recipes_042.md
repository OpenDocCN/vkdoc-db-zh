# 2-9. 从命令行导入平面文件

## 问题

您需要尽可能快地从命令行加载平面文件。

## 解决方案

使用 `BCP` 加载数据。可以像这样操作：

1.  确保存在目标表，其结构映射到源文件。在此示例中，它是 `CarSales_Staging.dbo.Invoice`。
2.  打开命令提示符。
3.  输入以下命令：

```bash
BCP CarSales_Staging.dbo.Invoices IN C:\SQL2012DIRecipes\CH02\Invoices.txt
    –T –SADAM02
```

4.  按 Enter 键从命令提示符运行 `BCP` 命令。
5.  系统提示时，确认数据源中每个字段的数据类型、前缀长度和字段终止符。

## 工作原理

`BCP`（大容量复制程序）自 SQL Server 首次出现以来就已存在。虽然没有人会形容它“用户友好”，但它确实名副其实。也就是说，它能以出色的效率将批量数据复制进（以及复制出）SQL Server。`BCP` 不仅设计用于复制文本数据（您将在第 5 章中看到）。它还具有大量开关，一开始可能让人望而生畏。然而，尽管学习曲线看似陡峭，但掌握它所付出的努力会获得数倍的回报。

我假设可能有些时候您希望从 SQL Server 外部导入文本文件，无论是从命令行交互式导入，还是从命令（`.cmd`）文件导入。无论您的理由是什么，它肯定能完成与 `BULK INSERT`（如配方 2-10 所述）相同的工作——只是从命令提示符执行。如果您选择这条路径，那么您可以使用与 `BULK INSERT` 相同的格式文件，并且本质上，您将使用相同的选项，只是作为命令行标志。`BCP` 最好在以下情况使用：

*   您希望在不调用 T-SQL 的情况下加载数据时（尽管这是可能的——如果您的 SQL Server 安全考虑允许，您也可以使用 `xp_cmdshell` 从 T-SQL 脚本运行 `BCP`）。
*   您想使用批处理文件或基于非 SQL Server 的进程来加载数据时。
*   源平面文件严格统一且无错误时。

在最简化的情况下，`BCP` 需要四个参数：

*   目标表的完全限定的三部分表名（`database.schema.table`）。
*   关键字 `IN`，告诉 `BCP` 数据正被导入到 SQL Server。
*   源数据文件的完整路径。如果路径包含空格，完整路径必须用双引号括起来。
*   任何所需的开关——特别是使用的 SQL Server 安全性（集成安全性或用户和密码）以及 SQL Server 实例名称。

> **注意**：如果服务器上只有一个活动的 SQL Server 实例，则无需指定目标服务器。否则，您必须指定要使用的 SQL Server 实例。始终指定服务器和数据库是良好的实践。

格式文件是可选的，但如果没有格式文件，`BCP` 将要求确认数据源中每个字段的数据类型、前缀长度和字段终止符。您可能已经发现，我发现这很快就会变得极其烦人。诚然，如果您执行一次，`BCP` 会在过程结束时询问您是否要创建一个格式文件以供重用。我的建议是始终选择“是”。这将是一个较旧的文本类型格式文件，因为您必须使用 `-x` 开关创建 XML 格式文件，如配方 2-11 所述。

因此，假设您已经创建了如配方 2-11 所定义的格式文件，或者在运行初始导入时要求 `BCP` 创建一个，以下是使用它的方法：

```bash
BCP CarSales_Staging.dbo.Invoices IN C:\SQL2012DIRecipes\CH02\Invoices.txt –T –f
    C:\SQL2012DIRecipes\CH02\Invoices.Fmt
```

为了让 `BCP` 有任何正常工作的机会，您需要能够连接到服务器。有两个选项：

*   集成安全性
*   SQL Server 身份验证

对于前者，只需在任何使用的 `BCP` 开关中添加 `–T` 开关。

要使用 SQL Server 身份验证，请在任何使用的 `BCP` 开关中添加 `–UyourUserName –PyourPassword`。例如：

```bash
BCP CarSales_Staging.dbo.Invoices IN C:\SQL2012DIRecipes\CH02\Invoices.txt –UAdam –PMe4B0ss
```

这三个开关必须大写。如果您愿意，可以在开关和名称或密码之间添加空格。

如果您连接的不是服务器本身的单个 SQL Server 实例，则需要指定服务器和/或实例。这是使用 `–S` 开关完成的。


例如：
```
BCP CarSales_Staging.dbo.Invoices IN C:\SQL2012DIRecipes\CH02\Invoices.txt ← –T –SmyServer\FirstInstance
```
此开关也必须使用大写字母。

由于`BCP`拥有大量的命令行参数，表 2-5 总结了您可能会觉得有用的主要参数（也称为标志）。

表 2-5. `BCP` 命令行参数

| BCP 标志 | 描述 | 说明 |
| --- | --- | --- |
| `-m` | `最大错误数` | 此标志指定在进程停止并引发错误之前，您希望 `BCP` 允许的最大错误数。 |
| `-f` | `格式文件` | 此标志指定要使用的格式文件（可以是“旧”文本格式或“新” XML 格式）。 |
| `-e` | `错误文件` | 此标志指定错误文件的路径和文件名，错误文件将在该处创建。 |
| `-F` | `起始行` | 此标志指定要导入的第一行。 |
| `-L` | `结束行` | 此标志指定要导入的最后一行。 |
| `-b` | `批处理大小` | 此标志指定源文件中每批提交的行数。如果发生失败，这些行可以回滚。 |
| `-c` | `字符文件` | 使用此标志可确保 `BCP` 不会提示输入数据类型。它假定所有字段都是文本字段。它还预设制表符为字段终止符，换行符 (`CHAR(10)`) 为行终止符。 |
| `-w` | `Unicode 文件` | 使用此标志可确保 `BCP` 不会提示输入数据类型。它假定所有字段都是 Unicode 格式。它还预设制表符为字段终止符，换行符 (`CHAR(10)`) 为行终止符。 |
| `-q` | `带引号的标识符` | 如果数据库和/或文件的名称中包含空格或引号，则必须使用此标志。 |
| `-t` | `字段终止符` | 如果需要覆盖默认的字段分隔符（`\t` 制表符），请使用此标志。 |
| `-r` | `行终止符` | 如果需要覆盖默认的行终止符（`\n` 换行符），请使用此标志。 |
| `-T` | `可信连接` | 此标志指定将使用集成安全性连接到 SQL Server。 |
| `-U` | `用户名` | 如果未使用集成安全性，此标志提供用于连接到 SQL Server 的用户登录名。 |
| `-P` | `密码` | 如果未使用集成安全性，此标志提供用于连接到 SQL Server 的用户密码。 |
| `-S` | `服务器名称` | 此标志提供您要连接的服务器的名称。 |
| `-k` | `保留 NULL 值` | 使用此标志可保留源数据中的 `NULL` 值，而不是使用表定义中指定的默认值。 |
| `-E` | `保留 IDENTITY 值` | 此标志告诉 `BCP` 使用源数据中的值作为 `IDENTITY` 列的值，而不是继续目标表中现有的序列。 |
| `-h` | `提示` | 允许您指定将使用 `CHECK_CONSTRAINTS`、`TABLOCK` 和 `ORDER`。 |

从 T-SQL 运行 `BCP` 非常简单。以下代码片段也将上一个示例中的数据从同一个源文件加载到同一个目标表中：
```sql
DECLARE @BCPVARIABLE VARCHAR(500) = 'BCP CarSales_Staging.dbo.Invoices ← IN C:\SQL2012DIRecipes\CH02\Invoices.txt –T –SADAM02 –f C:\SQL2012DIRecipes\CH02\Invoices.Fmt';
EXECUTE master.dbo.xp_cmdshell @BCPVARIABLE;
```
您需要允许使用 `xp_cmdshell`，这可以通过在 SSMS 中右键单击服务器/实例，从“方面”![image](img/arrow.jpg)“外围应用配置器”中获得，或者（如果您使用的是旧版本 SQL Server）通过“外围应用配置器”实用程序来完成。当然，我假设 DBA 会允许这样做，但实际情况可能并非如此。您也可以运行以下 T-SQL 来启用 `xp_cmdshell`：
```sql
EXECUTE sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
EXECUTE sp_configure 'xp_cmdshell', 1;
GO
RECONFIGURE;
GO
```

提示、技巧和陷阱
*   `BCP` 在每处理 1000 行后显示消息“1000 rows sent to SQL Server”。此消息仅供参考。无论批处理大小如何，都会出现此消息。
*   如果您愿意，可以将 `BCP` 命令文本保存在命令文件 (`.cmd`) 中并执行该文件。
*   `BCP` 选项区分大小写，输入时请小心。
*   如果源数据文件包含列标题，请记得将 `–F`（第一行数据）标志设置为 2。
*   `BCP` 也可以导出平面文件，但这在配方 7-4 中处理。它在 SQL Server 之间传输数据时特别高效，您将在第 4 章中看到。

### 2-10. 使用 T-SQL 导入大型文本文件并侧重于速度

问题
您希望使用基于 T-SQL 的方法尽可能快地导入大型文本文件。

解决方案
使用 T-SQL 的 `BULK INSERT` 命令加载文件。我将解释如何操作。

1.  创建一个目标表来保存您将导入的数据。对于此示例，DDL 为 (`C:\SQL2012DIRecipes\CH02\tblInvoiceBulkLoad.Sql`)：
    ```sql
    CREATE TABLE CarSales_Staging.dbo.InvoiceBulkLoad
    (
     ID int IDENTITY(1,1) NOT NULL,
     InvoiceNumber VARCHAR(50) NULL,
     ClientID INT NULL,
     TotalDiscount numeric(18, 2) NULL,
     DeliveryCharge numeric(18, 2) NULL
    ) ;
    GO
    ```

2.  运行以下代码，将 `C:\SQL2012DIRecipes\CH02\InvoiceBulkLoad.Txt` 文本文件加载到 `dbo.InvoiceBulkLoad` 表中（该表在加载过程开始前已创建）。
    ```sql
    BULK INSERT CarSales_Staging.dbo.InvoiceBulkLoad
       FROM 'C:\SQL2012DIRecipes\CH02\InvoiceBulkLoad.Txt'
       WITH
          (
             FIELDTERMINATOR = ',',
             ROWTERMINATOR = '\n',
             FIRSTROW = 2
          );
    ```

工作原理
当面对庞大而复杂的源文本文件时，开发基于 T-SQL 的解决方案只有一个真正的答案：使用 `BULK INSERT` 来完成繁重的工作。在开始之前，您最好忘记围绕这一特定数据集成选项的一些迷思。与普遍看法相反，它不一定复杂，并且具有以下优点：
*   它是可用的最快解决方案之一。
*   它可以被参数化以处理复杂的数据源，包括仅加载源数据的某些部分。
*   它可以进行调整，以最优方式处理非常大的加载任务。

`BULK INSERT` 命令的大部分功能来自于：
*   您可以设置的多个参数。
*   您创建的格式文件提供的控制，用于控制加载过程中的数据映射。

多年来，格式文件名声不佳；自从它们与历史悠久的 `BCP` 程序（`BULK INSERT` 是其尊贵的继承者）一起出现以来，它们就被认为过于复杂。然而，在配方 2-11 中您会看到，当它们以 XML 文件的形式第二次出现时，它们并不难掌握，并且它们为从文本文件加载大量数据这一潜在艰巨任务带来的可能性，使它们非常值得掌握。

`BULK INSERT` 最适合在以下情况下使用：
*   当目标表结构存在时。
*   当您需要指定全局和/或特定的列和行分隔符时。
*   当速度至关重要时。
*   对于大型加载任务。
*   当您需要对加载过程进行最大程度的控制时。

在此示例中，源文件包含标题信息，因此 `BULK INSERT` 从第二行开始加载，以便仅加载数据——并避免在加载数值数据时出现类型冲突，因为字段名几乎总是字符数据。

现在您已经看到 `BULK INSERT` 可以如此简单，让我们看看它提供的选项，如表 2-6 所示。

[表 2-6.](#9781430247913_Ch02.



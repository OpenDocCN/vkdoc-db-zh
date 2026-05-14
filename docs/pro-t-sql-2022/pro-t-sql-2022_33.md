# 15. 增强

本章继续关注构建可维护的 T-SQL。很多时候，您需要更新数据库架构以添加新的应用程序功能。第一节提供了几个可用于修改现有数据库对象的示例。第二节重点介绍了在逐步淘汰不良数据库设计或已弃用功能时可能遇到的一些挑战。

## 添加新功能

理想情况下，为现有应用程序添加功能是软件开发的常规部分。本节讨论如何在保留现有功能的同时为现有应用程序添加功能的几种选项。在本书中，我将使用“应用程序规则”来指代通过更新存储在数据库表中的值来启用功能的能力。此方法不需要任何应用程序或代码更改。但是，“功能标志”将用于指代通过数据传递给 SQL Server 的方式（包括使用参数）来启用功能的情况。如果您通过向表中添加数据来增加功能，行级安全性或许能让您控制新功能何时上线。如果您需要添加记录数据日志或历史信息的新功能，也可以使用触发器。实现日志记录的其他选项在第 14 章中讨论。


### 应用规则

使用应用规则包括创建一个存储所有可用应用规则及其状态的地方。在第 14 章中，清单 14-9 展示了为应用规则创建表的 T-SQL。作为参考，此代码在清单 15-1 中再次展示。

```sql
CREATE TABLE dbo.ApplicationRule
(
ApplicationRuleID              INT           IDENTITY(1,1)  NOT NULL,
ApplicationRuleDescription     VARCHAR(50)                  NOT NULL,
IsActive                       BIT
CONSTRAINT DF_ApplicationRule_IsActive    DEFAULT 1      NOT NULL,
DateCreated                    DATETIME2(2)
CONSTRAINT DF_ApplicationRule_DateCreated DEFAULT SYSDATETIME() NOT NULL,
DateModified                   DATETIME2(2)
CONSTRAINT DF_ApplicationRule_DateModified DEFAULT SYSDATETIME() NULL,
CONSTRAINT PK_ApplicationRule_ApplicationRuleID
PRIMARY KEY CLUSTERED (ApplicationRuleID)
);
清单 15-1
创建应用规则表
```

您会注意到上面的代码略有不同，因为现在为 `IsActive`、`DateCreated` 和 `DateModified` 列添加了默认约束。既然现在有了存储应用规则的地方，就可以插入您的第一条规则了。清单 15-2 展示了创建应用规则的代码。

```sql
INSERT INTO dbo.ApplicationRule (ApplicationRuleDescription)
VALUES ('仅显示活跃客户');
清单 15-2
填充应用规则表
```

一旦插入了记录，这条应用规则就会存在于 `dbo.ApplicationRule` 表中。表 15-1 展示了您应该期望在数据库中看到的内容。

**表 15-1 应用规则表**

| ApplicationRuleID | ApplicationRuleDescription | IsActive | DateCreated | DateModifed |
| --- | --- | --- | --- | --- |
| 1 | 仅显示活跃客户 | 1 | 2023-04-12 | 2023-04-12 |

这些概念已在第 14 章讨论过，本节将继续梳理整个过程。在第 12 章，清单 12-2 和清单 12-3 中有两个名为 `dbo.GetCustomer` 的存储过程示例。它们之间的区别在于添加了一个 `WHERE` 子句，条件为 `IsActive = 1`。清单 15-3 展示了该存储过程的原始 T-SQL 代码。

```sql
/*-------------------------------------------------------------*\
Name:             dbo.GetCustomer
Author:           Elizabeth Noble
Created Date:     2022-10-30
Description:      获取数据库中所有客户的列表
Sample Usage:
EXECUTE dbo.GetCustomer;
\*-------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE dbo.GetCustomer
AS
SELECT
CustomerID,
FirstName,
LastName,
Address,
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateDisabled
FROM dbo.Customer;
清单 15-3
原始存储过程
```

当我们在第 12 章讨论修改清单 15-3 中的存储过程时，我们只包括在查询末尾添加一个硬编码的行项，以更改存储过程使其仅允许活跃客户。然而，您可能希望在应用程序准备好进行此更改之前部署这个变更。幸运的是，有一种方法可以在存储过程内切换此功能。清单 15-4 展示了更新后的存储过程，以便应用规则表决定是显示所有客户还是仅显示活跃客户。

```sql
/*-------------------------------------------------------------*\
Name:             dbo.GetCustomer
Author:           Elizabeth Noble
Created Date:     2022-10-30
Description:      获取数据库中所有客户的列表
Updated Date      2023-04-12
Description:      更新为使用 IsActive 应用规则
Sample Usage:
EXECUTE dbo.GetCustomer;
\*-------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE dbo.GetCustomer
AS
DECLARE @OnlyActive BIT;
SELECT @OnlyActive = IsActive
FROM dbo.ApplicationRule
-- 这是'仅显示活跃客户'的硬编码值
---- 如果 ApplicationRuleID 1 启用，则 @OnlyActive = 1
---- 如果 ApplicationRuleID 1 禁用，则 @OnlyActive = 0
WHERE ApplicationRuleID = 1;
SELECT
CustomerID,
FirstName,
LastName,
Address,
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateDisabled
FROM dbo.Customer
-- 存储过程将始终显示活跃客户
WHERE IsActive = 1
-- 如果应用规则 1 启用，则使用 1
---- 这将只返回活跃客户
-- 如果应用规则 1 禁用，则使用 0
---- 这将返回非活跃客户
AND IsActive = @OnlyActive;
清单 15-4
使用应用规则的存储过程
```

在未来的某个时刻，您可能会决定不再需要该应用规则。当这种情况发生时，您可以更新存储过程以移除对应用规则的依赖。在此场景下，让我们将存储过程更新为仅显示活跃客户，如清单 15-5 所示。

```sql
/*-------------------------------------------------------------*\
Name:             dbo.GetCustomer
Author:           Elizabeth Noble
Created Date:     2022-10-30
Description:      获取数据库中所有客户的列表
Updated Date      2023-04-12
Description:      更新为使用 IsActive 应用规则
Updated Date      2023-06-13
Description:      更新为仅显示 IsActive 客户
Sample Usage:
EXECUTE dbo.GetCustomer;
\*-------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE dbo.GetCustomer
AS
SELECT
CustomerID,
FirstName,
LastName,
Address,
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateDisabled
FROM dbo.Customer
-- 存储过程将始终显示活跃客户
WHERE IsActive = 1;
清单 15-5
使用应用规则的存储过程
```

应用规则可以应用于其他不支持输入参数的 T-SQL 数据库对象，例如视图。清单 15-6 展示了如何使用应用规则来控制视图的功能。

```sql
CREATE OR ALTER VIEW dbo.vwCustomer
AS
SELECT
CustomerID,
FirstName,
LastName,
Address,
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateDisabled
FROM dbo.Customer
-- 此视图将始终显示活跃客户
WHERE
-- 这是
-- '仅显示活跃客户'的硬编码值
IsActive = 1
-- 如果应用规则 1 启用，则使用 1
---- 这将只返回活跃客户
-- 如果应用规则 1 禁用，则使用 0
---- 这将返回非活跃客户
AND IsActive =
(
SELECT IsActive
FROM dbo.ApplicationRule
WHERE ApplicationRuleID = 1
);
清单 15-6
使用应用规则的视图
```

这是一种切换功能的方法。另一种选择是使用功能标志，如第 12 章所讨论。它们在概念上与应用规则相似。然而，在许多情况下，可能存在一段外部代码来决定是否应启用或关闭新功能。

### 特性标志

使用特性标志时，你可以向存储过程添加一个输入参数，以此控制 T-SQL 代码的行为。代码清单 15-7 展示了如何更新代码清单 15-3 中的存储过程 `dbo.Customer` 以使用特性标志。

```sql
/*-------------------------------------------------------------*\
Name:             dbo.GetCustomer
Author:           Elizabeth Noble
Created Date:     2022-10-30
Description:      Get a list of all customers in the databases
Updated Date      2023-04-12
Description:      Updated to use IsActive feature flag
Sample Usage:
EXECUTE dbo.GetCustomer
\*-------------------------------------------------------------*/
CREATE OR ALTER PROCEDURE dbo.GetCustomer
@OnlyActive BIT = 0 –- Defaults to show all customers
AS
SELECT
CustomerID,
FirstName,
LastName,
Address,
City,
PostalCode,
Country,
IsActive,
DateCreated,
DateModified,
DateDisabled
FROM dbo.Customer
-- Stored procedure will always show Active customers
WHERE IsActive = 1
-- This will use 1 if ApplicationRule 1 is enabled
---- This will return only active customers
-- This will use 0 if ApplicationRule 1 is disabled
---- This will return inactive customers
AND IsActive = @OnlyActive;
```
代码清单 15-7
使用特性标志的存储过程

在上面的存储过程中，`@OnlyActive` 值被传递给存储过程。使用参数来切换存储过程功能的一个好处是，你可以为该参数设置默认值，以维持现有功能。

还可能有一些情况是，你希望向数据库添加数据，但在未来的某个日期之前不希望应用程序能够使用这些数据。当你正在将一个新的业务或应用程序迁移或集成到现有数据库时，就可能发生这种情况。如果你发现自己需要填充表中的数据，但又不希望这些数据被访问，那么可以尝试使用行级安全性。关于行级安全性的更多信息，可以参考第 21 章。

你可能会遇到想要记录表上的操作行为的情况。虽然让应用程序直接记录信息是最理想的，但这可能无法实现。在这种情况下，你可能希望用一个触发器来记录这些信息。例如，你可以使用触发器来记录产品何时被添加、修改或删除。你可能想要记录新产品的添加时间以及添加产品的登录用户。代码清单 15-8 展示了一个用于记录 `dbo.Product` 表上行为的日志表。

```sql
CREATE TABLE dbo.ProductLog(
ProductLogID     INT                        IDENTITY(1,1)   NOT NULL,
ProductID        INT                                        NOT NULL,
ProductName      VARCHAR(25)                                NOT NULL,
ProductPrice     DECIMAL(6,2)                               NOT NULL,
IsActive         BIT
CONSTRAINT DF_ProductLog_IsActive     DEFAULT (1)     NOT NULL,
DateCreated      DATETIME2(2)
CONSTRAINT DF_ProductLog_DateCreated  DEFAULT (SYSDATETIME()) NOT NULL,
DateModified     DATETIME2(2)
CONSTRAINT DF_ProductLog_DateModified DEFAULT (SYSDATETIME()) NOT NULL,
DateDisabled     DATETIME2(2)                                   NULL,
CONSTRAINT PK_ProductLog PRIMARY KEY CLUSTERED (PK_ProductLogID ASC)
);
```
代码清单 15-8
创建 ProductLog 表

现在你已经创建了表，可以给 `dbo.Product` 表添加一个触发器，这样当在该表中插入或更新记录时，新数据的记录将被保存到 `dbo.ProductHistory` 表中。由于表上已经会有触发器，你还可以添加逻辑来更新 `dbo.Product` 表中的 `DateModified` 字段。代码清单 15-9 展示了如何创建一个在产品被插入或更新时记录产品更改的触发器。

```sql
CREATE TRIGGER dbo.LogProductHistory
ON dbo.Product
AFTER INSERT, UPDATE
AS
INSERT INTO dbo.ProductHistory
(
ProductID,
ProductPrice,
ProductName
)
SELECT
ProductID,
ProductPrice,
ProductName
FROM inserted;
UPDATE prd
SET DateModified = SYSDATETIME()
FROM dbo.Product prd
INNER JOIN deleted del
ON prd.ProductID = del.ProductID;
```
代码清单 15-9
创建触发器以记录产品历史

除了在记录被添加或更改后向 `dbo.ProductHistory` 表插入一条记录外，此触发器还会更新 `dbo.Product` 表上的 `DateModified` 字段，以便记录该记录最近一次被更改的时间。

像编写任何其他代码一样，你应该对此进行测试并确认触发器是否按预期工作。要测试此触发器，请执行代码清单 15-10 中的 T-SQL。

```sql
UPDATE dbo.Product
SET IsActive = 1
WHERE ProductID = 5;
INSERT INTO dbo.Product (ProductName, ProductPrice)
VALUES ('Kayak',1299.00);
```
代码清单 15-10
测试触发器的插入和更新功能

在代码清单 15-10 中，你将 `ProductID` 为 5 的产品更新为激活状态。你还运行了一个单独的查询，将一个新产品“Kayak”插入到 `dbo.Product` 表中。由于触发器 `dbo.LogProductHistory` 已被添加到 `dbo.Product` 表，你预期会在 `dbo.ProductHistory` 表中找到两条新记录。

第一个查询不仅应该向 `dbo.ProductHistory` 表添加一条记录，还应该更新 `dbo.Product` 表中的 `DateModified` 字段。第二个查询应该向 `dbo.ProductHistory` 表添加一条记录，因为创建了一个新产品。请参考代码清单 15-11。

```sql
SELECT ProductLogID,
ProductID,
ProductName,
ProductPrice,
DateCreated
FROM dbo.ProductHistory;
SELECT ProductID,
ProductName,
DateCreated,
DateModified,
DateDisbled
FROM dbo.Product
WHERE ProductID = 5;
```
代码清单 15-11
检查触发器的运行结果

运行这些查询后，你可以检查结果。如前所述，你应该在 `dbo.ProductHistory` 表中找到两条新记录。此外，`dbo.Product` 表中 `ProductID` 为 5 的记录的 `DateModified` 字段应该已被更新。表 15-2 显示了第一个查询的结果。

表 15-2
产品历史结果

| 产品日志 ID | 产品 ID | 产品名称 | 产品价格 | 创建日期 |
| --- | --- | --- | --- | --- |
| 1 | 5 | 飞盘高尔夫盘 - 大号 | 12.99 | 202304-12 11:58 AM |
| 2 | 7 | 皮划艇 | 1299.00 | 2023-04-12 11:58 AM |

一旦你确认了触发器的日志记录部分按预期工作，就可以检查 `dbo.Product` 表中的 `DateModified` 字段是否已更新。上面第二个查询的结果如表 15-3 所示。

表 15-3
产品表更新情况

| 产品 ID | 产品名称 | 创建日期 | 修改日期 | 禁用日期 |
| --- | --- | --- | --- | --- |
| 5 | 飞盘高尔夫盘 - 大号 | 2023-02-12 8:27 AM | 2023-04-12 11:58 AM | NULL |

这可以让你确认 `dbo.Product` 表中的 `DateModified` 字段已按预期更新。

使用应用程序规则、特性标志、行级安全性或触发器，在尝试为数据库实现新功能时可能很有用。这可以包括更改存储过程或其他数据库代码的运行方式、限制特定用户对信息的访问，或为数据表添加日志记录。然而，有时你可能希望从数据库中移除现有功能。请继续阅读下一节，了解更多关于如何使用 T-SQL 来管理应用程序增强功能的想法。


## 逐步淘汰旧技术

你也可能遇到这样的情况：你想要重构或修改现有的表或数据库代码。与添加新功能类似，本节将介绍一些技巧，使你能够以不影响应用程序当前功能的方式更改数据库模式。当试图改进数据库规范化时，你或许可以临时使用视图，让应用程序的表现就像仍在使用之前的表模式一样。如果需要借助存储过程创建更动态的功能，你可以使用输入参数，让应用程序能够使用先前的功能。这可以通过应用程序规则或功能标志来实现。当你准备好将应用程序从当前表迁移出去时，你可能还不准备停止将这些表用于其他目的。你可以使用触发器来确保数据也会被保存到遗留数据表中。

淘汰旧技术的主要挑战在于，确保你的应用程序在能够更新以使用新的 T-SQL 代码之前，继续按预期工作。然而，“添加新功能”一节中提到的许多相同技巧也可以用来逐步淘汰现有功能。


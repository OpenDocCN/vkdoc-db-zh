# 13. 功能性

在本书中，我们涵盖了许多不同的主题。其中大多数章节都涉及最佳实践。这些最佳实践包括命名规范、`T-SQL` 代码格式化或数据库对象设计。尽管你已尽最大努力，但有时仍可能遇到超出最佳实践范围的情况。这些时候往往与高压期或紧迫的截止日期相关。本章的目标是帮助你为必须就复杂主题快速做出决策的情况做好准备。

为允许用户插入或更新数据的应用程序设计数据库代码可能具有挑战性。你可能会发现自己需要处理需要禁用权限或功能的应用程序。在处理遗留应用程序代码时，可能会遇到额外的挑战，特别是如果这些应用程序使用了 `ORM`（对象关系映射）软件。随着组织的成长，可能需要额外的报表功能，但可能没有足够的时间来开发数据仓库。在 `SQL Server` 中，你可以让 `T-SQL` 代码更加灵活，但必须确保不以牺牲功能为代价来换取灵活性。

在本章中，你将回顾常见的编程需求和可用于满足这些需求的 `T-SQL` 编码方法。对于每种方法，你将审视其优缺点，以帮助你快速做出决策。



## 插入与更新数据

T-SQL 代码需要处理的一个常见情况是如何插入或更新数据。本节首先讨论使用 `MERGE` 语句最简单但也最麻烦的解决方案。然后将讨论另一种使用 `IF` 语句的方法；它允许 SQL 判断记录是否存在。如果存在，则更新该记录；否则，将插入该记录。

在许多场景下，你希望能够传入一组值。如果记录不存在，你可能希望插入信息；但如果记录已存在，你可能希望更新该记录。通常，当我们有压力时，我们希望使用最快、最简单的方式行事。其他时候，我们可能更关注 T-SQL 代码的可读性，而非数据库引擎的最佳性能。虽然确保你的数据库代码能被他人理解很重要，但在编写 T-SQL 时，考虑数据库代码的性能同样重要。

使用 T-SQL 插入或更新数据库记录时，情况也是如此。对于应用程序来说，无论用户是插入新记录还是更新现有记录，都执行相同的存储过程可能更简单、更直接。这类操作也可以被称为 `upsert`（更新插入）。你是如何陷入这种境地的并不重要；重要的是设计出可靠且高效工作的 T-SQL 代码。

### MERGE 语句的问题

有一些 T-SQL 数据库代码看似能够简化在单个查询中执行插入或更新的过程。此功能称为 `MERGE` 语句。使用 `MERGE` 语句的好处是逻辑直接明了。然而，`MERGE` 语句存在许多已知问题。`MERGE` 语句最显著的问题涉及从使用索引视图的表中删除数据。当从表中删除数据时，关联的索引视图会间歇性地更新。也有报告称，将 `MERGE` 语句与时态表一起使用时，效果并不总是如预期。其他问题包括：当 `MERGE` 语句修改属于全文索引一部分的字符串列时，分区表不会更新全文索引；当使用非筛选唯一索引通过 `MERGE` 语句向表中 `DELETE` 或 `INSERT` 数据时，可能会发生唯一键冲突错误。

除非你愿意接受这些潜在问题，否则我建议不要使用 `MERGE` 语句。清单 13-1 展示了一个 `MERGE` 语句的示例。

```sql
CREATE PROCEDURE dbo.CustomerOrderMerge
@CustomerOrderID INT,
@CustomerID INT,
@OrderNumber VARCHAR(15),
@OrderDate DATETIME2(2),
@ShipDate DATETIME2(2),
@IsActive BIT,
@DateCreated DATETIME2(2),
@DateModified DATETIME2(2),
@DateDisabled DATETIME2(2)
AS
MERGE dbo.CustomerOrder AS [Target]
USING
(
VALUES
(
@CustomerOrderID,
@CustomerID,
@OrderNumber,
@OrderDate,
@ShipDate,
@IsActive,
@DateCreated,
@DateModified,
@DateDisabled
)
) AS [Source]
(
CustomerOrderID,
CustomerID,
OrderNumber,
OrderDate,
ShipDate,
IsActive,
DateCreated,
DateModified,
DateDisabled
)
ON ([Target].CustomerOrderID = [Source].CustomerOrderID)
WHEN MATCHED THEN
UPDATE SET
[Target].CustomerID = [Source].CustomerID,
[Target].OrderNumber = [Source].OrderNumber,
[Target].OrderDate = [Source].OrderDate,
[Target].ShipDate = [Source].ShipDate,
[Target].IsActive = [Source].IsActive,
[Target].DateCreated = [Source].DateCreated,
[Target].DateModified = [Source].DateModified,
[Target].DateDisabled = [Source].DateDisabled
WHEN NOT MATCHED BY TARGET THEN
INSERT (
CustomerOrderID,
CustomerID,
OrderNumber,
OrderDate,
ShipDate,
IsActive,
DateCreated,
DateModified,
DateDisabled
)
VALUES (
[Source].CustomerOrderID,
[Source].CustomerID,
[Source].OrderNumber,
[Source].OrderDate,
[Source].ShipDate,
[Source].IsActive,
[Source].DateCreated,
[Source].DateModified,
[Source].DateDisabled
);
```
*清单 13-1 包含 Merge 语句的存储过程*

此 `MERGE` 语句会更新客户订单（如果该订单已存在于 `dbo.CustomerOrder` 表中）。这个 `MERGE` 语句的逻辑易于遵循，其格式也易于编写。但在你的环境中实施 `MERGE` 语句之前，你需要意识到这些问题。

我曾经在充分理解 `MERGE` 语句所带来的问题之前就实施了它们。当我试图替换这些 `MERGE` 语句时，有人担心逻辑会变得过于困难。要替换清单 13-1 中的逻辑，你可以编写一个如清单 13-2 所示的查询。

### 使用 IF…ELSE 进行更新插入

要替换 `MERGE` 语句的逻辑，你可以使用 `IF… ELSE` 语句来控制 SQL Server 查询中的操作流程。在清单 13-2 中，如果订单存在于 `dbo.CustomerOrder` 表中，记录将更新除 `DateCreated` 外的所有字段。否则，将在 `dbo.CustomerOrder` 中插入一条新记录。

```sql
CREATE PROCEDURE dbo.CustomerOrderUpsert
@CustomerOrderID INT,
@CustomerID INT,
@OrderNumber VARCHAR(15),
@OrderDate DATETIME2(2),
@ShipDate DATETIME2(2),
@IsActive BIT,
@DateCreated DATETIME2(2),
@DateModified DATETIME2(2),
@DateDisabled DATETIME2(2)
AS
IF EXISTS (SELECT CustomerOrderID FROM dbo.CustomerOrder WHERE CustomerOrderID = @CustomerOrderID)
BEGIN
UPDATE dbo.CustomerOrder
SET   CustomerID = @CustomerID,
OrderNumber = @OrderNumber,
OrderDate = @OrderDate,
ShipDate = @ShipDate,
IsActive = @IsActive,
DateModified = @DateModified,
DateDisabled = @DateDisabled
WHERE CustomerOrderID = @CustomerOrderID;
END;
ELSE
BEGIN
INSERT dbo.CustomerOrder
(
CustomerOrderID,
CustomerID,
OrderNumber,
OrderDate,
ShipDate,
IsActive,
DateCreated,
DateModified,
DateDisabled
)
VALUES
(
@CustomerOrderID,
@CustomerID,
@OrderNumber,
@OrderDate,
@ShipDate,
@IsActive,
@DateCreated,
@DateModified,
@DateDisabled
)
END;
```
*清单 13-2 用于插入或更新的存储过程*

### 结论与建议

`MERGE` 语句不仅限于向目标表插入或更新数据。它可以扩展到处理其他场景，例如插入或删除数据。也可以创建 `MERGE` 语句，使其在单个语句中完成插入缺失记录、更新更改记录以及删除孤立记录。虽然数据修改可以合并为单个语句的一部分，但在使用 `MERGE` 语句时应谨慎，因为如前所述，它们容易导致性能问题和数据不一致。由于 `MERGE` 语句易于编写，有可能以一种对 SQL Server 效率不高的方式来比较两个表之间的数据。尽管表可能只被比较一次，但如果比较操作效率低下，可能会对被比较表的性能产生显著影响。也出现过在使用筛选索引时发生唯一键违规的情况。在特定情况下，`MERGE` 语句还可能导致外键约束违规。这包括使用两个具有外键约束的表，其中外键设置为 `NOCHECK` 然后被回滚。我建议研究 `MERGE` 语句的已知问题，并进行充分的测试以确认 T-SQL 代码能按预期工作。清单 13-2 中的代码表明，编写能够处理插入或更新的代码并非过于复杂。在 SQL Server 2019 之前，清单 13-1 或 13-2 中的代码都可能受到参数嗅探的影响。既然 SQL Server 2019 已支持自适应联接，这个问题应该会有所缓解。



## 禁用功能

在开发应用程序时，一个常见的挑战是如何在不影响业务连续性的情况下修改后端表设计。本节将讨论如何更新数据，以便在应用程序当前不支持软删除（即记录被标记为非活动状态而非从数据库中移除）的情况下，仍能从数据库中删除记录。这些方法可能包括禁用外键约束或创建虚拟记录。之后，记录就应该可以被删除。

本节还将介绍如何使用软删除，以及一些决定何时切换“允许用户查找所有记录”或“仅查找未软删除记录”的方法。你也可以运用“应用程序规则”的概念来实现功能的切换。这同样可以与功能标志（Feature Flags）结合使用，正如第 13 章所讨论的那样。

### 启用/禁用功能的挑战

在设计应用程序时，首要目标通常是满足既定需求。在许多情况下，产品负责人或业务部门提出的是他们当前所需的功能。然而，这些人并不具备你对应用程序和数据库开发的了解。因此，他们可能不会主动要求具备按需启用或禁用特定功能的能力。这里，“功能”可以指用户或角色执行某项操作的能力，也可以指应用程序以某种特定方式运行的能力。他们可能指望着你能预见到未来他们可能会如何使用这些应用程序。

设计新应用程序时，最容易被遗忘的功能之一，就是为用户角色启用或禁用特定功能的能力。如果你遇到这种情况：表中一条记录的存在就意味着它对应用程序可用，那么你就属于这种情况。例如，`dbo.Customer`表是使用清单 13-3 中的 T-SQL 语句创建的。

```sql
CREATE TABLE [dbo].[Customer] (
[CustomerID]       INT            IDENTITY (1, 1) NOT NULL,
[FirstName]        VARCHAR (40)                   NOT NULL,
[LastName]         VARCHAR (100)                  NOT NULL,
[Address]          VARCHAR (100)                  NOT NULL,
[City]             VARCHAR (100)                  NOT NULL,
[PostalCode]       VARCHAR (20)                       NULL,
[Country]          VARCHAR (75)                   NOT NULL,
[DateCreated]      DATETIME2 (2)                  NOT NULL
CONSTRAINT [DF_Customer_DateCreated] DEFAULT (SYSDATETIME()),
[DateModified]     DATETIME2 (2)     NOT NULL
CONSTRAINT [DF_Customer_DateModified] DEFAULT (SYSDATETIME()),
CONSTRAINT PK_Customer PRIMARY KEY CLUSTERED (CustomerID)
);
```
清单 13-3: 创建 `dbo.Customer` 表

在表 13-1 中，你可以查看 `dbo.Customer` 表中的记录。

**表 13-1: 客户表示例数据**

| CustomerID | FirstName | LastName | DateCreated |
| --- | --- | --- | --- |
| 1 | Myra | Acharya | 2022-08-30 |
| 2 | José | Gomez | 2022-08-30 |
| 12 | Selena | Tiësto | 2022-08-30 |
| 27 | Ian | West | 2022-08-30 |
| 401405 | Marty` | Bethel | 2022-10-02 |

例如，如果因为一条订单在 `dbo.Customer` 表中，它就会在应用程序中显示，那么在这种情况下，你无法控制应用程序如何与客户订单交互。要防止应用程序使用某个客户订单，你唯一的选择就是删除该客户订单记录。此操作被称为*硬删除*。

### 硬删除及其局限性

你需要从哪些表中删除记录，取决于该删除操作是否会因外键关系而被阻止。在这些场景下，你只有几个可用选项。一个选项是删除外键，然后删除指定的记录。对于此表设计，如果你想从数据库中移除客户 Ian，你需要删除任何引用 `dbo.CustomerOrder` 表的外键，然后删除记录。清单 13-4 中的查询用于删除外键。

```sql
ALTER TABLE [dbo].[CustomerOrder]
DROP CONSTRAINT [FK_CustomerOrder_Customer];
```
清单 13-4: 删除外键

一旦删除了所有引用 `dbo.CustomerOrder` 表的外键，你就需要移除客户记录以确保其不再可用。清单 13-5 展示了删除特定客户的语句。

```sql
DELETE FROM dbo.Customer
WHERE FirstName = 'Ian';
```
清单 13-5: 删除客户记录

生成的表包含如表 13-2 所示的记录。

**表 13-2: 删除 Ian 后的客户表**

| CustomerID | FirstName | LastName | DateCreated |
| --- | --- | --- | --- |
| 1 | Myra | Acharya | 2022-08-30 |
| 2 | José | Gomez | 2022-08-30 |
| 12 | Selena | Tiësto | 2022-08-30 |
| 401405 | Marty` | Bethel | 2022-10-02 |

这种方法存在一个显著问题。当你需要决定是否删除表之间的外键关系时，应格外谨慎。这不仅可能对查询性能产生负面影响，还可能影响数据质量。外键是帮助确保表间数据一致性的最后防线之一。

另一种可能的方法是向存在外键关系的表中插入一条虚拟记录。你可以向 `dbo.Customer` 表添加一个名为 Inactive Deactivated 的客户。可以使用清单 13-6 中的查询插入此记录。

```sql
SET IDENTITY_INSERT dbo.Customer ON;
INSERT INTO dbo.Customer
(
CustomerID,
FirstName,
LastName,
[Address],
City,
PostalCode,
Country,
DateCreated,
DateModified,
DateDisabled
)
VALUES
(
0,
'Inactive',
'Deactivated',
'',
'',
'',
'',
GETDATE(),
GETDATE(),
NULL
);
SET IDENTITY_INSERT dbo.Customer OFF;
```
清单 13-6: 插入一个虚拟客户记录

将这条虚拟记录插入 `dbo.Customer` 表只是第一步。你需要将 `dbo.CustomerOrder` 表中 Ian 的记录更新为使用 Inactive Deactivated 的 `CustomerID`。更新操作如清单 13-7 所示。

```sql
UPDATE dbo.CustomerOrder
SET CustomerID = 0,
DateModified = GETDATE()
WHERE CustomerID = 27;
```
清单 13-7: 在客户订单表中更新非活动客户

除非你的应用程序已配置为始终忽略该值，否则这可能无法阻止应用程序显示特定值。如果这两种选项都无法使用，还有另一种选择。对于这种表设计，另一个选择是移除与客户 Ian 关联的所有订单。

由于这些限制，硬删除的概念可能并不适合你的环境。你还有其他可用选项。你可以设计表结构，使得能够切换客户是否处于活动状态。这使你无需删除数据记录即可启用或禁用功能。

### 使用 `IsActive` 标志实现软删除

要使用此选项，你需要 `dbo.Customer` 表包含一个类似 `IsActive` 的列。要修改清单 13-3 中创建的表，请执行清单 13-8 中的查询。

```sql
ALTER TABLE dbo.Customer
ADD IsActive BIT
CONSTRAINT DF_Customer_IsActive DEFAULT ((1)) NOT NULL;
```
清单 13-8: 向 `dbo.Customer` 表添加 `IsActive` 列



这通常是一个表示真或假的值。对于 `SQL Server`，常用 1 表示真，0 表示假。若将 `IsActive` 设为真并运行筛选 `IsActive` 值为真的结果的查询，你将只获得活跃客户的列表。一旦将 `IsActive` 列添加到 `dbo.Customer` 表，你就可以修改 `T-SQL` 代码，根据编写方式显示活跃或非活跃客户的订单。

你可以选择仅根据分配给 `IsActive` 的值来显示结果。当 `IsActive` 值设为 1 时，表示值为真。若只想显示活跃客户，需在 `WHERE` 子句中修改 `T-SQL` 代码以使用 `IsActive = 1`。这也让你能够实现某些屏幕仅显示活跃客户，而其他屏幕仅显示非活跃或禁用客户的功能。

除了为用户或角色启用或禁用功能外，有时你可能希望控制应用程序的运行方式。公司通常希望随时间改变其业务运营方式。这通常可以通过在业务所使用的应用程序中添加、更改或移除功能来实现。

例如，如果一家企业希望扩展到当前市场之外，可能会选择为其现有应用程序添加新功能。添加新功能最简单的方法是增加新的应用程序代码。然而，企业可能会发现这个新市场或新业务线并未如预期运作，公司决定回滚这项新功能。根据代码的编写方式，这可能是一个需要重新编写代码的复杂变更，也可能是一个允许应用程序使用代码不同部分的简单变更。

根据你的业务环境，你可能会发现通过数据库管理功能变更比通过应用程序代码更容易。当功能需要基于季节性或地区进行变化时，这种可能性更大。这让你能够实施可快速更改的精细控制。你需要仔细考虑管理此功能的最佳位置。我能提供的最佳建议是：当高层管理需要对任何变更进行审批时，将业务逻辑保留在应用程序中。对于那些能够快速变更且监督较少的业务逻辑，最好将其保留在数据库中。

如果你想从数据库内部管理部分应用程序业务逻辑，则需要创建一个表来管理某些功能是否启用。这个表在概念上类似于第 12 章为功能标志创建的表。如果你想在数据库中管理功能，可能需要创建一个如代码清单 13-9 所示的表。

```sql
CREATE TABLE dbo.ApplicationRule
(
ApplicationRuleID               IDENTITY(1,1) INT      NOT NULL,
ApplicationRuleDescription      VARCHAR(50)            NOT NULL,
IsActive                        BIT                    NOT NULL,
DateCreated                     DATETIME               NOT NULL,
DateModified                    DATETIME                   NULL,
CONSTRAINT PK_ApplicationRule_ApplicationRuleID
PRIMARY KEY CLUSTERED (ApplicationRuleID)
);
```
代码清单 13-9
创建应用规则表

如果你是为一个单一、集中的应用程序创建规则，这个表设计将满足你的需求。如果你在管理多个应用程序，请考虑应用程序如何使用这些规则。如果大多数或所有规则对于每个应用程序都是唯一的，你可能需要为此表添加一列来指明受影响的应用程序。

可能存在业务原因，使你希望切换应用程序以仅显示活跃客户或显示所有客户。如果应用程序需要显示所有客户或仅活跃客户存在合理的业务理由，就可能发生这种情况。此规则的值可以存储在表 13-3 中。

表 13-3

应用规则表

| Application RuleID | ApplicationRule Description | IsActive | DateCreated | DateModifed |
| --- | --- | --- | --- | --- |
| 1 | 仅显示活跃客户 | 1 | 2023-04-12 | 2023-04-12 |

此表中的条目表示，启用此应用规则时将仅显示活跃客户。若希望同一应用程序显示所有客户，你需要将 `IsActive` 值更新为假。虽然你可以更新所有数据库代码和存储过程，以根据此应用规则的启用状态使用 `IF... THEN` 语句，但我建议在应用程序内部处理此功能。

这正是将代码开发得足够灵活变得重要的地方。尽管本书的主题不是应用程序代码开发，但你将需要一个能与你的应用程序代码协作来处理这些场景的数据库。从概念上讲，这可以与第 12 章讨论的功能标志类似地工作。你可以创建一个表来存储对特定功能的引用。在某些环境中，这可能被称为应用规则。




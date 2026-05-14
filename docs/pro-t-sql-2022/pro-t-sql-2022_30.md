# 动态 SQL

有时，编写可重用于众多且可能不同场景的 T-SQL 代码是理想的做法。在本节中，你将了解动态 SQL 如何用于实现代码重用。本节内容应谨慎阅读，因为动态 SQL 存在许多问题，包括可能使你的数据库遭受攻击或损坏的风险。例如，在表单输入中包含 `;DELETE FROM dbo.Customer` 就可能导致这种情况发生。当这种情况发生时，它被称为 `SQL 注入`，因为 SQL 代码被注入到了数据库中。

根据你要编写的 T-SQL 代码类型，你可能会发现难以将数据库代码编写为基于集合的设计。在这些情况下，以更具迭代性的方式编写代码可能很有诱惑力。你可能希望编写高度可变、并能根据传入的参数和值进行修改的 T-SQL 代码。动态 SQL 使你可以在代码执行时，混合使用变量和 T-SQL 代码来生成 SQL 语句。使用动态 SQL 似乎正是你一直想要的解决方案。但在大多数情况下，动态 SQL 的弊端往往远大于其好处，不过在某些时候，动态 SQL 可能是正确的解决方案之一。

在第 7 章中，我介绍了如何读取执行计划。使用动态 SQL 时，每次执行查询都需要计算一个新的执行计划。这会给数据库引擎带来额外的压力。

## 使用动态 SQL 的场景

大多数需要使用动态 SQL 的情况都涉及需要额外灵活性的时刻。许多希望使用动态 SQL 的情况与数据库管理活动有关。这些活动大多涉及在 SQL Server 实例的多个数据库上执行相同的操作。动态 SQL 提供了一些额外的功能，这使得使用它特别具有吸引力。

在执行标准 T-SQL 时，不能将数据库名或表名用作变量。而使用动态 SQL 则提供了编写可在多个数据库对象上执行的查询的选项。

## 动态 SQL 示例

清单 13-21 中的查询展示了一个动态 SQL 的示例。

```sql
CREATE PROCEDURE dbo.TableByDynamicValues
@TableName SYSNAME,
@ColumnName SYSNAME,
@ColumnValue NVARCHAR(25)
AS
DECLARE @ObjectID INT;
DECLARE @ColumnList NVARCHAR(MAX);
DECLARE @Query NVARCHAR(MAX);
SET @ObjectID = OBJECT_ID(@TableName);
SELECT @ColumnList = STRING_AGG (cast([name] as nvarchar(max)), ',')
FROM sys.columns
WHERE [object_id] = @ObjectID
AND [name] NOT IN (N'IsActive',N'DateCreated',N'DateModified',N'DateDisabled') ;
SET @Query =
N'SELECT ' + @ColumnList +
N' FROM ' + @TableName +
N' WHERE ' + @ColumnName + N' = @ColumnValue';
EXECUTE sp_executesql @Query,
N'@ColumnName SYSNAME, @ColumnValue NVARCHAR(25)',
@ColumnName = @ColumnName,
@ColumnValue = @ColumnValue;
```

**清单 13-21 动态地从表中检索数据**

此查询允许用户或应用程序传入一个表名。这将是查询其值的目标表。另外两个参数允许将列名和列值作为查询的一部分传入。此列名将用于过滤数据。清单 12-22 展示了此代码的功能。

```sql
EXECUTE dbo.TableByDynamicValues N'Product', N'ProductID', N'1';
EXECUTE dbo.TableByDynamicValues N'Customer', N'FirstName', N'Marty`';
```

**清单 13-22 使用不同值执行存储过程**

清单 13-22 中有两次执行同一个存储过程。第一次执行返回 `Product` 表的结果。第二次执行返回 `Customer` 表的结果。执行清单 13-22 中的查询，你会得到表 13-4 中的结果。

**表 13-4 动态 SQL 结果**

| DateDisabled | ProductID | ProductName | ProductPrice |
| --- | --- | --- | --- |
| NULL | 1 | Telescope | 599.00 |

| Address | City | Country | Customer ID | Date Disabled | First Name | LastName | Postal Code |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 750 Cherry Rd | Memphis | United States | 410405 | 2022-08-30 | Marty` | Bethel | 38117 |

返回的第一组数据是 `Product` 表中 `ProductID` 为 1 的列。清单 13-21 中的存储过程排除了 `IsActive`、`DateCreated`、`DateModified` 和 `DateDisabled` 这些列。表中所有其他列都被返回。第二组数据中的值是 `Customer` 表中 `FirstName` 为 Marty` 的记录。从软件开发的角度来看，在编写数据库代码时，这似乎是一种理想的方法。

## 动态 SQL 的缺点

然而，这种使用 T-SQL 的方法没有考虑数据库引擎如何执行查询。这将导致 SQL Server 在每次执行存储过程时生成一个新的执行计划，因为每次执行时请求的数据可能发生巨大变化。虽然动态 SQL 似乎有助于让你的应用程序代码更灵活，但编写 T-SQL 的这种方法是有代价的。我曾在尝试创建 ETL 流程时使用动态 SQL，那时我希望以编程方式确定哪些表有主键，哪些没有。然后我可以轻松地生成一个脚本，只为那些有主键的表删除主键。总的来说，当需要编写 T-SQL 代码时，我会在大多数场景中避免使用动态 SQL。尝试在没有动态 SQL 的情况下编写代码，将促使你实践如何利用 SQL Server 数据库引擎的优势来编写数据库代码。

### SQL 注入风险

使用动态 SQL 也会增加执行超出你原始预期的额外 T-SQL 代码的风险。这种行为被称为 `SQL 注入`。SQL 注入的概念是，额外的 T-SQL 代码被插入或注入到原始语句中。被插入的额外 T-SQL 代码实现了查询执行时原本未打算实现的功能。这可能包括查看此用户本不应访问的数据。同一用户还可能通过 SQL 注入修改数据或数据库对象。如果你想使用动态 SQL，可以通过使用参数将值传递给动态 SQL 来最小化 SQL 注入的风险。对动态 SQL 进行参数化的方法将使用户更难以你不期望的方式查看或修改数据。

## 总结与建议

编写 T-SQL 代码时，你需要编写功能有效且高效的代码。这意味着要编写可读性强且易于维护的代码。编写能够伴随业务发展的代码也是一项关键技能。如果你正在编写代码来处理插入、更新或各种数据修改，请确保你编写的代码易于理解并便于后续支持。在应用程序开发过程中，考虑你希望如何处理禁用权限或应用程序功能。如果不考虑，你需要考虑一些策略，允许你以不破坏数据库中表之间关系的方式停用数据值。你可能还需要支持遗留代码。支持遗留代码的挑战之一是在不破坏现有功能的情况下进行更改。有时你可能需要从事务型数据库中提取数据用于报告。尝试设计你的查询以实现灵活性和可重用性。虽然灵活性是好的，但也要谨慎考虑在 T-SQL 代码中实现多大程度的灵活性。在改进了数据库代码的功能后，你可能会发现你想要确定如何跟踪数据的变更。你可以在第 14 章中学习有关日志记录的内容。



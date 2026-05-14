# 使用数据集

如果你正在更改应用程序代码的工作方式，例如插入新表或修改值的填充方式，你可能会发现自己需要执行批量更新。虽然你可以通过使用硬编码值逐条更新记录来完成批量更新，但让我们看看是否有一种方法能够使用相同的代码逻辑来更新数据集中的所有记录。这正是数据集真正能帮助你的地方。

你可以编写一个查询来查找所有需要更新的记录，并系统地执行所有这些更新操作。除了在性能和功能方面的优势外，这种处理方式还能确保你的数据得到一致性的处理。

总的来说，作为数据集更新数据的过程与之前讨论的选择和插入数据的方法非常相似。在表 5-19 中，你可以看到存储在 `dbo.Customer` 表中的信息。

### 表 5-19：客户与地址信息

| 名字 | 姓氏 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Myra | Acharya | 30 Magrath Road | Bengaluru | 560025 | India |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |

虽然这是客户当前的保存方式，但你可能会决定要改变地址信息的保存方式。例如，你可能希望将国家字段改为使用国家代码缩写，而不是完整的国家名称。在这种情况下，你可以如表 5-20、表 5-21、表 5-22 和表 5-23 所示，逐条更新记录。

### 表 5-23：为 Josè 更新国家

| 名字 | 姓氏 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | EGY |

### 表 5-22：为 Stacy 更新国家

| 名字 | 姓氏 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | PRT |

### 表 5-21：为 Josè 更新国家

| 名字 | 姓氏 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | MEX |

### 表 5-20：为 Myra 更新国家

| 名字 | 姓氏 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Myra | Acharya | 30 Magrath Road | Bengaluru | 560025 | IND |

要使用 `T-SQL` 逐条执行这些更新，请运行下面的代码清单 5-4。

```sql
UPDATE dbo.Customer
SET Country = 'IND'
WHERE CustomerID = 401408;
UPDATE dbo.Customer
SET Country = 'MEX'
WHERE CustomerID = 401406;
UPDATE dbo.Customer
SET Country = 'PRT'
WHERE CustomerID = 401409;
UPDATE dbo.Customer
SET Country = 'EGY'
WHERE CustomerID = 401407;
```
*清单 5-4：逐个更新国家代码*

你可以创建一些查询来批量更新所有国家为印度、墨西哥、埃及或葡萄牙的记录。代码清单 5-5 展示了这些查询。

```sql
UPDATE dbo.Customer
SET Country = 'IND'
WHERE Country = 'India';
UPDATE dbo.Customer
SET Country = 'MEX'
WHERE CustomerID = 'Mexico';
UPDATE dbo.Customer
SET Country = 'PRT'
WHERE CustomerID = 'Portugal';
UPDATE dbo.Customer
SET Country = 'EGY'
WHERE CustomerID = 'Egypt';
```
*清单 5-5：按国家更新国家代码*

这段代码可以工作，并创建了一些集合。但还有更高效的方法来更新这些数据，特别是当你不想为每个国家创建一个查询时。你可以创建一个包含完整国家名称和三位国家代码的引用表。然后编写一个简单的查询，同时更新所有记录，如表 5-24 所示。

### 表 5-24：按国家分组的客户数据集

| 名字 | 姓氏 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Myra | Acharya | 30 Magrath Road | Bengaluru | 560025 | IND |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | MEX |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | PRT |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | EGY |

此表显示了你希望使用单个查询更新的整个数据集。在执行更新之前，你应该先创建引用表。创建该表的 `SQL` 语句在代码清单 5-6 中。

```sql
CREATE TABLE dbo.Country
(
CountryCode VARCHAR(3),
Country     VARCHAR(75)
);
INSERT INTO dbo.Country (CountryCode, Country)
VALUES
('EGY','Egypt'),
('IND','India'),
('MEX','Mexico'),
('PRT','Portugal');
```
*清单 5-6：创建国家引用表*

现在 `Country` 表已经创建好了，你可以编写一个单一的查询，只要国家名称存在于引用表中，它就会将所有国家名称更新为你指定的国家代码。代码清单 5-7 中的 `SQL` 会更新所有客户记录，如果国家名称存在于 `dbo.Country` 表中，则使用对应的国家代码。

```sql
UPDATE cus
SET Country = cty.CountryCode,
    DateModified = GETDATE()
FROM dbo.Customer cus
INNER JOIN dbo.Country cty
    ON cus.Country = cty.Country;
```
*清单 5-7：将客户国家更新为国家代码*

虽然这些更新可以通过硬编码值来完成，但有时将你正在更新的表与其他相关表连接起来可能更好。表之间的这种交互可能使执行更新操作变得更加容易。

你可以使用客户的地址来决定如何更新国家代码。思考一下你想要使用的逻辑会很有帮助。如果你想在地址中添加所属大洲，你首先需要确定每个客户所在的国家。下一步是获取一个包含国家及其对应大洲的表。一旦你拥有了这两部分信息，你就可以创建一个临时表来将国家映射到特定的大洲。虽然对于仅仅几处更改来说，这似乎需要大量的工作，但其威力体现在你需要长期维护这些数据，并且可能需要更频繁地进行此类更新时。

总的来说，我发现每当我需要逐条处理数据时，我通常必须执行某种手动操作。而手动操作最有可能在插入、更新或删除数据时出现数据错误的问题。我更喜欢在数据集中编写代码，这样我更能确保我的逻辑是一致的。



## 为数据集编写代码

在前文中，我讨论了如何开始将你的数据视为数据集而非单条记录。在选择、更新或删除数据时，我经常使用数据集。虽然大多数应用程序代码一次只插入单条记录，但在许多常见场景下，以数据集的形式插入数据可能会很有帮助。在接下来的部分，我将介绍几种你可能会希望将数据视为集合而非单条记录的场景。

数据集最常用于你希望查看部分数据时。理想情况下，这些数据集是基于特定条件选择的。这不仅有利于性能，也有助于确保你只查看你想要的特定数据。在大多数场景下，一次检索和显示一条单个记录甚至没有意义。通常，如果你发现自己处于一次只选择一条记录的情况，这通常是一个很好的迹象，表明你可能需要查看该 `T-SQL` 代码是否可以重写以使用基于集合的逻辑。

查询可以有更复杂的选择方式。在某些情况下，你可能会发现自己想要合并或比较两个不同的数据集。如果你想将两个数据集连接在一起，你可以选择使用 `UNION` 或 `UNION ALL`。它们之间只有一个微小但重要的区别。回顾表 5-2，我识别了两组不同的数据，一组基于城市名称，另一组基于部分地址。在下面的例子中，我将展示一种你可以用来查找同时包含这两个数据集结果的方法。当你对数据进行 `UNION` 时，返回的每条记录在连接在一起的 `SELECT` 语句之间都将是唯一的。代码清单 5-8 展示了一个在两个查询之间的 `UNION` 操作。

```sql
SELECT FirstName, LastName
FROM dbo.Customer
WHERE City = 'Memphis'
UNION
SELECT FirstName, LastName
FROM dbo.Customer
WHERE [Address] LIKE '%de%';
```
代码清单 5-8
两个查询的联合

表 5-25 显示了当前存储的所有客户数据。共有六位客户。

**表 5-25**
**所有客户**

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Myra | Acharya | 30 Magrath Road | Bengaluru | 560025 | India |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

表 5-26 显示了第一个查询的结果值。

**表 5-26**
**Memphis 的客户**

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

第二个查询的数据如表 5-27 所示。

**表 5-27**
**地址中包含 "de" 的客户**

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

联合查询提供了如表 5-28 所示的结果集。

**表 5-28**
**表 5-26 和 5-27 结果的联合**

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

然而，对于 `UNION ALL`，无论多个查询中是否存在重复值，两个查询中返回的每条记录都将被返回。如果你运行相同的查询但使用 `UNION ALL`，`T-SQL` 代码如代码清单 5-9 所示。

```sql
SELECT FirstName, LastName
FROM dbo.Customer
WHERE City = 'Memphis'
UNION ALL
SELECT FirstName, LastName
FROM dbo.Customer
WHERE [Address] LIKE '%de%';
```
代码清单 5-9
两个查询的联合所有

你返回了所有数据，即使它是重复的。`UNION ALL` 的结果如表 5-29 所示。

**表 5-29**
**表 5-26 和 5-27 结果的联合所有**

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | Mexico |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | Portugal |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |
| Karim | Khalil | 153 Road Mit Rahina Al Shabab | Memphis | 3364932 | Egypt |
| Marty` | Bethel | 750 Cherry Rd | Memphis | 38117 | United States |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

如你所见，`UNION ALL` 返回了两行完全相同的数据。在某些时候，这些场景中的每一种都可能是可取的。

有时候，查询逻辑可能足够复杂，以至于我可能无法快速理解所有的 `T-SQL` 代码，或者在其他时候，我正在对非常大的数据集进行故障排除。如果我编写一个更简单的查询来返回我想要的所有数据，我通常会在那两个查询之间写一个 `INTERSECT` 来查找记录匹配的地方。使用与之前相同的通用查询，我想向你展示使用 `INTERSECT` 与其他选项相比，返回的数据会是什么样子。在代码清单 5-10 中，你可以看到查找两个查询交集的查询。

```sql
SELECT FirstName, LastName
FROM dbo.Customer
WHERE City = 'Memphis'
INTERSECT
SELECT FirstName, LastName
FROM dbo.Customer
WHERE [Address] LIKE '%de%';
```
代码清单 5-10
两个查询的交集

你已经从前面的图表中知道了每个单独查询返回的结果。前面 `T-SQL` 代码的实际结果如表 5-30 所示。

**表 5-30**
**表 5-26 和 5-27 结果的交集**

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Ruth | Johnson | 4711 Ponce de Leon | Memphis | 38110 | United States |

此外，如果你试图查找缺失的记录或验证是否存在匹配的记录，你可以在两个查询之间使用 `EXCEPT`。代码清单 5-11 展示了如果你从一个查询的结果中排除另一个查询的结果时的 `T-SQL` 代码。

```sql
SELECT FirstName, LastName
FROM dbo.Customer
WHERE City = 'Memphis'
EXCEPT
SELECT FirstName, LastName
FROM dbo.Customer
WHERE [Address] LIKE '%de%';
```
代码清单 5-11
两个查询的除外



在这种情况下，第一条查询返回所有城市名包含“Memphis”一词的结果。然而，`EXCEPT`语句表明，如果某条记录同时出现在第一条和第二条查询中，则该记录将从结果集中排除。此查询的结果见表 5-31。

表 5-31
表 5-26 和 5-27 结果的 EXCEPT 运算

| 名 | 姓 | 地址 | 城市 | 邮政编码 | 国家 |
| --- | --- | --- | --- | --- | --- |
| Josè | Gomez | 790 Calle Cinco de Mayo | Cancun | 53778 | 墨西哥 |
| Stacy | Parks | 123 Rua de Santa Catarina | Porto | 1234-567 | 葡萄牙 |

使用`EXCEPT`语句可能不是从数据集中排除结果的首选方法，但它是我更习惯使用的方法。这正是 T-SQL 代码的益处与挑战所在：对于几乎任何场景，编写 T-SQL 代码的方式几乎肯定不止一种。你编写数据库代码的原因将决定你在编码时拥有的灵活度。如果你只是执行一次查询，或许可以使用效率较低的方法来访问这些数据。然而，如果你是为某个应用程序编写代码，你就需要平衡 T-SQL 代码的可读性与提升数据返回速度。

代码清单 5-12 展示了在插入数据时如何使用数据集。

```sql
CREATE TABLE #TempProductArchive
(
ProductID      INT            NOT NULL,
ProductName    VARCHAR(25)    NOT NULL,
ProductPrice   DECIMAL(6,2)   NOT NULL,
IsActive       BIT            NOT NULL,
DateCreated    DATETIME2(2)   NOT NULL,
DateModified   DATETIME2(2)   NOT NULL,
DateDisabled   DATETIME2(2)   NULL,
DateInserted   DATETIME2(2)   NOT NULL
);
INSERT INTO #TempProductArchive
(
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled,
DateInserted
)
SELECT
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled,
GETDATE()
FROM dbo.Product
WHERE DateDisabled < '2022-08-01';
```
代码清单 5-12
作为集合插入数据

代码清单 5-13 展示了如果是逐条记录插入，插入语句的样子。

```sql
CREATE TABLE #TempProductArchive
(
ProductID         INT               NOT NULL,
ProductName       VARCHAR(25)       NOT NULL,
ProductPrice      DECIMAL(6,2)      NOT NULL,
IsActive          BIT               NOT NULL,
DateCreated       DATETIME2(2)      NOT NULL,
DateModified      DATETIME2(2)      NOT NULL,
DateDisabled      DATETIME2(2)      NULL,
DateInserted      DATETIME2(2)      NOT NULL,
);
INSERT INTO #TempProductArchive
(
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled,
DateInserted
)
VALUES
(
14,
'Water shoes - Large',
25.00,
0,
'2018-01-01',
'2022-07-29',
'2022-07-29',
GETDATE()
);
INSERT INTO #TempProductArchive
(
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled,
DateInserted
)
VALUES
(
15,
'Water shoes - Medium',
25.00,
0,
'2018-01-01',
'2022-07-29',
'2022-07-29',
GETDATE()
);
INSERT INTO #TempProductArchive
(
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled,
DateInserted
)
VALUES
(
16,
'Water shoes - Small',
25.00,
0,
'2018-01-01',
'2022-07-29',
'2022-07-29',
GETDATE()
);
INSERT INTO #TempProductArchive
(
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled,
DateInserted
)
VALUES
(
17,
'Water shoes - Child',
15.00,
0,
'2018-01-01',
'2022-07-29',
'2022-07-29',
GETDATE()
);
```
代码清单 5-13
逐条记录插入数据

如你所见，逐条记录插入数据会占用相当多的代码，并且可能繁琐得多。

我最喜欢的数据集用途之一是更新数据。在许多情况下，我不是在更新特定记录，而是在更新具有相同特征的多条记录。如果我想将表中的所有产品设置为非活动状态，我有几种不同的方法可以做到。例如，我可以为每条记录编写一个更新语句，将`IsActive`值设置为零，如代码清单 5-14 所示。

```sql
UPDATE dbo.Product
SET IsActive = 0,
DateModified = GETDATE(),
DateDisabled = GETDATE()
WHERE ProductID = 1;
UPDATE dbo.Product
SET IsActive = 0,
DateModified = GETDATE(),
DateDisabled = GETDATE()
WHERE ProductID = 2;
UPDATE dbo.Product
SET IsActive = 0,
DateModified = GETDATE(),
DateDisabled = GETDATE()
WHERE ProductID = 3;
```
代码清单 5-14
逐条记录更新数据

这需要三次独立的事务。此外，SQL Server 必须访问这些记录所在的数据页三次。如代码清单 5-15 所示，我也可以编写一个查询，一次性更新所有三条记录。

```sql
UPDATE dbo.Product
SET IsActive = 0,
DateModified = GETDATE(),
DateDisabled = GETDATE()
WHERE ProductID BETWEEN 1 AND 3;
```
代码清单 5-15
按范围更新数据

事实上，如果这三条记录都位于同一个数据页上，我只需要访问该页面一次。一个更可能的场景是请求停用产品名称中包含“water”一词的所有产品。这正是你希望发现的模式。如果存在这样的模式，你可以编写如代码清单 5-16 所示的查询。

```sql
UPDATE dbo.CustomerOrder
SET IsActive = 0,
DateDisabled = GETDATE()
WHERE IsActive = 1
AND CustomerID = 234;
```
代码清单 5-16
作为数据集更新数据

这个查询实现了两个目标。它让你能在一次给定的查询中更新多条记录。`WHERE`子句包含了`IsActive`和`CustomerID`列，允许查询使用索引`IX_CustomerOrder_Isactive_CustomerID_OrderID_DateCreated`。

使用数据集更新多条记录没有限制。我经常需要更新成百上千条记录。虽然我可以编写查询来单独识别每条记录，然后手动提取这些 ID 来逐条或批量更新它们，但有更简单的方法来更新这些多条记录，并最大限度地减少因人为干预而导致的错误风险。代码清单 5-17 展示了如何在连接表的同时更新数据。

```sql
UPDATE prd
SET DateModified = GETDATE(),
IsActive = 0
FROM dbo.Product prd
INNER JOIN dbo.ProductArchive pch
ON prd.ProductID = pch.ProductID
WHERE pch.DateInserted >= CAST(GETDATE() AS DATE);
```
代码清单 5-17
通过连接创建数据集来更新数据

在更新操作中使用连接时，我建议你确认正在更新的数据是什么。这可以通过将更新语句转换为选择语句来完成，如代码清单 5-18 所示。

```sql
SELECT
ProductID,
ProductName,
ProductPrice,
IsActive,
DateCreated,
DateModified,
DateDisabled
FROM dbo.Product prd
INNER JOIN dbo.ProductArchive pch
ON prd.ProductID = pch.ProductID
WHERE pch.DateInserted >= CAST(GETDATE() AS DATE);
```
代码清单 5-18
使用 SELECT 语句验证数据集

你可以检查记录并验证记录数。在执行更新之前，我建议将`UPDATE`语句包裹在`BEGIN TRAN…ROLLBACK`中以验证记录数。

就像处理更新一样，你也可以在删除数据时使用数据集。我发现使用数据集删除数据可以显著提高效率，但如果数据在删除前未经验证，也可能存在一定风险。



在编写 T-SQL 代码时，最大的问题之一是未能充分利用数据集。这通常是因为编写过程式代码更得心应手，或者尚未培养以数据集方式思考的能力。在本书中，我的目标是通过多种方式，指导你编写出拥抱并善用数据集的代码。


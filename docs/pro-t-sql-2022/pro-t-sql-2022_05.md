# 3. 标准化 T-SQL

制定一种编写代码的方法，有助于提高你和他人理解代码意图的速度。你是否曾多次写下代码，过后再看却不明其意？我们的目标是，如果你遵循良好的格式、命名和注释 T-SQL 的实践，理解和掌握 T-SQL 的意图就会更轻松、更迅速。虽然确定这些标准前期会花费一些额外时间，但长远来看，它所节省的成本将是值得的。

编写易于理解的 T-SQL 对你和你的公司都有益。许多其他编程语言都有标准或最佳实践，我认为 T-SQL 也不例外。编写 T-SQL 的主要目标或许是实现某个功能，但次要目标——确保你的 T-SQL 清晰易懂——同样重要。随着时间的推移，代码会发生变化，或者会发现缺陷。你的 T-SQL 代码可读性和可理解性越高，修改或排查问题就越容易。

## 格式化 T-SQL

T-SQL 书写时的外观与其内容同等重要。和其他应用程序代码一样，未来很可能有人需要查看你的代码，或者你需要查看别人的代码。如果我不是在编写新代码，那我就是在查看已有代码，以理解其目的、调试代码、调整查询性能或更新业务逻辑。根据我审查代码的原因，我通常会确定当时什么对我最重要。

如果我查看 T-SQL 代码是为了理解代码功能，我会审阅任何注释和涉及的表，以理解哪个应用程序可能正在使用此 T-SQL 代码。我并不太关心表是如何连接的，因为我预期它们能正确工作——尽管表连接逻辑错误也可能是查询返回非常意外结果的原因。接下来，我会查看用于筛选 T-SQL 代码结果的条件。只有在我分析的最后阶段，我才会审阅查询返回的列。通常，只有涉及特殊业务逻辑时，我才会关注列。将这种思维模式应用于编写简单查询时，你可以看到我在清单 3-1 中将列名列在了一行。

```
SELECT ProductID, ProductName, DateCreated, DateModified
FROM dbo.Product;
```
清单 3-1
基础查询

我以这种格式编写代码，是因为我希望能够快速看到通过 `FROM` 子句和 `WHERE` 条件发生的所有操作项。如果我为每个列创建单独一行，我将更难看清表之间的关系以及应用于这些关系的条件。在清单 3-2 中，我修改了 `SELECT` 子句中列的显示方式。

```
SELECT cus.FirstName,
       cus.LastName,
       prd.ProductName,
       dtl.QuantitySold AS QuantitySold
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
  ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
  ON dtl.ProductID = prd.ProductID
INNER JOIN dbo.Customer cus
  ON ord.CustomerID = cus.CustomerID
  AND cus.Country = 'India';
```
清单 3-2
带连接的查询

对于此查询，列是逐行罗列的。这是因为我对实际检索的列进行了某种修改。如果我为列设置了别名或向列添加了特殊逻辑，我会改变 `SELECT` 语句中列的格式化方式。对于这些场景，我会根据逻辑的复杂性，为每个列创建一行或多行。请注意查询最后两行上的两个连接条件。我通常将第一个连接条件之后的所有条件进行缩进，因为我希望它能立即显现出对一个连接应用了多个条件。

我审查 T-SQL 代码常常是为了排查它为何返回错误结果。当返回的结果出现问题时，我会从一个描述发生了什么错误的用户故事开始。在这种情况下，我会直接查看 `WHERE` 子句，仔细检查逻辑并确认其正确性。一旦确认了该逻辑，我就会查看连接条件，以确认表是否连接正确。我使用与理解 T-SQL 代码相同的流程来排查代码问题；我最后查看 `SELECT` 语句，重点关注任何带有特殊逻辑的主要列。查看清单 3-3 中的查询，我可以先快速浏览 `WHERE` 子句，然后再看 `FROM` 子句。



### 清单 3-3 使用子查询的查询

```sql
SELECT
(
    SELECT prd.ProductName
    FROM dbo.CustomerOrder ord
    INNER JOIN dbo.OrderDetail dtl
        ON ord.CustomerOrderID = dtl.CustomerOrderID
    INNER JOIN dbo.Product prd
        ON dtl.ProductID = prd.ProductID
    WHERE ord.CustomerID = bord.CustomerID
    GROUP BY prd.ProductName
) AS ProductName,
bord.CustomerOrderID,
bord.OrderNumber,
bord.OrderDate,
bdtl.QuantitySold,
bprd.ProductPrice
FROM dbo.CustomerOrder bord
INNER JOIN dbo.OrderDetail bdtl
    ON bord.CustomerOrderID = bdtl.CustomerOrderID
INNER JOIN dbo.Product bprd
    ON bdtl.ProductID = bprd.ProductID
WHERE bprd.ProductName LIKE '%board%';
```

这使我能够立即判断该查询正在处理名称中包含 `board` 一词的产品。`SELECT` 语句中的列各自独立成行列出，我能立刻意识到查询的这部分存在特殊逻辑。我还对子查询部分的逻辑进行了缩进，这有助于让该子查询更加突出。现在我正在尝试排查可能不准确的结果，可以快速识别可能导致问题的原因。根据报告的错误，问题极有可能出现在 `WHERE` 子句或 `SELECT` 语句返回的第一列。分析清单 `3-4` 中创建的视图则会得出不同的结论。

### 清单 3-4 创建视图

```sql
CREATE OR ALTER VIEW dbo.CustomerProduct
AS
SELECT cus.FirstName,
       cus.LastName,
       prd.ProductName,
       SUM(dtl.QuantitySold) AS QuantitySold
FROM dbo.CustomerOrder ord
INNER JOIN dbo.OrderDetail dtl
    ON ord.CustomerOrderID = dtl.CustomerOrderID
INNER JOIN dbo.Product prd
    ON dtl.ProductID = prd.ProductID
INNER JOIN dbo.Customer cus
    ON ord.CustomerID = cus.CustomerID
GROUP BY cus.FirstName, cus.LastName, prd.ProductName;
```

在这个视图的 `T-SQL` 代码中，我仍然首先查看 `FROM` 子句。我立即识别出存在多个联接。此外，没有 `WHERE` 子句，并且我还能快速判断 `SELECT` 语句中没有特殊逻辑，因为所有列并未各自独立成行。将该视图的信息与我正在研究的任何潜在错误进行匹配，我知道此查询最复杂的部分是联接逻辑。如果联接正确，并且该视图返回的结果过多，我可以快速排除 `SELECT` 语句的问题。在创建函数时也可以遵循类似的模式，如清单 `3-5` 所示。

### 清单 3-5 创建函数

```sql
CREATE FUNCTION dbo.ProductsByCustomer (@CustomerID INT)
RETURNS TABLE
AS
RETURN
(
    SELECT cus.CustomerID, dtl.ProductID
    FROM dbo.Customer cus
    INNER JOIN dbo.CustomerOrder ord
        ON cus.CustomerID = ord.CustomerID
    INNER JOIN dbo.OrderDetail dtl
        ON ord.CustomerOrderID = dtl.CustomerOrderID
    WHERE cus.CustomerID = @CustomerID
    GROUP BY cus.CustomerID, dtl.ProductID
);
```

在上述函数中，我可以快速识别 `FROM` 语句中的多个联接以及 `WHERE` 子句中的一个条件。如果函数仅返回所提供客户的结果，那么任何已发现的错误极有可能与联接条件有关。

我对查询进行性能调优的处理方式有所不同，我将在本书的 `第二部分` 进一步讨论这些差异。当作为性能调优的一部分审查 `T-SQL` 时，我重点关注使用了哪些表。如果涉及多个表，我还会查看这些表是如何联接在一起的。我最后关注的是使用了哪些列以及它们与现有索引的关系。

在更新 `T-SQL` 代码内部的逻辑时，我也会审查 `T-SQL` 代码。由于新增、更改或删除了功能，我需要修改 `T-SQL` 代码以反映这些变更。根据修改的性质，可能简单到查看 `SELECT` 子句中的字段并更改要显示的字段或计算方式。其他时候，我可能需要查看 `FROM` 子句，并在联接条件中添加或删除表。在某些情况下，我可能需要更新 `WHERE` 子句中的条件以处理新的业务需求。清单 `3-6` 就是这种情况，它展示了表值参数的创建。

### 清单 3-6 创建表值参数

```sql
CREATE PROCEDURE dbo.UpdateProductPricing
    @ProductList ProductPricing READONLY
AS
SET NOCOUNT ON
UPDATE prd
SET ProductName = pr.ProductName,
    ProductPrice = pr.ProductPrice
FROM dbo.Product prd
INNER JOIN @ProductList pr
    ON prd.ProductID = pr.ProductID;
```

我首先注意到的一点是此存储过程缺少 `WHERE` 子句。这也是在处理用户定义表类型时增加复杂性的地方。很可能用户定义的表被用于在联接时过滤数据。然而，仅看代码很难判断应用程序如何使用此存储过程。由于用户定义的表类型，增强此存储过程逻辑所需的工作量显著增加。我需要了解数据如何通过表值参数传入，但同时还需要考虑传递给此表值参数的数据随时间可能发生的变化。由于数据库管理员通常是在应用程序部署后长期管理 `T-SQL` 代码的人，我发现最好将 `T-SQL` 代码设计得易于后续支持。如你在清单 `3-7` 中所见，在创建公用表表达式时，我使用了相同的方法，但我会对公用表表达式内部的查询进行缩进。我再次使用这种缩进来表明在特定部分存在特殊逻辑。

### 清单 3-7 创建公用表表达式

```sql
WITH cte_customer AS
(
    SELECT cus.CustomerID,
           cus.FirstName,
           cus.LastName,
           dtl.ProductID,
           MIN(ord.OrderDate) AS FirstOrder
    FROM dbo.CustomerOrder ord
    INNER JOIN dbo.OrderDetail dtl
        ON ord.CustomerOrderID = dtl.CustomerOrderID
    INNER JOIN dbo.Customer cus
        ON ord.CustomerID = cus.CustomerID
    GROUP BY cus.CustomerID, cus.FirstName, cus.LastName, dtl.ProductID
)
SELECT cus.CustomerID,
       cus.FirstName,
       cus.LastName,
       cus.FirstOrder,
       SUM(dtl.QuantitySold) AS QuantitySold
FROM cte_customer cus
INNER JOIN dbo.CustomerOrder ord
    ON cus.CustomerID = ord.CustomerID
INNER JOIN dbo.OrderDetail dtl
    ON ord.CustomerOrderID = dtl.CustomerOrderID
    AND cus.ProductID = dtl.ProductID
GROUP BY cus.CustomerID,
         cus.FirstName,
         cus.LastName,
         cus.FirstOrder
HAVING SUM(dtl.QuantitySold) > 5;
```

在定义我个人的风格时，我了解到我的总体目标是让查询适合在足够小的区域内显示，以便我能快速高效地找到我正在试图审查的那部分 `T-SQL` 代码。在设计你自己的标准时，你需要思考你的总体目标。

在许多公司，会招聘初级团队成员。其中一些初级团队成员对 `SQL Server` 可能是新手，任何新员工都需要一些时间来了解你公司业务中的应用程序如何运作。在设计内部的 `T-SQL` 编码标准时，你需要考虑应遵循哪些格式约定，以帮助新员工快速了解你公司的系统和数据流。




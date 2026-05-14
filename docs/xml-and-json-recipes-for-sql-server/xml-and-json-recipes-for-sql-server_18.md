# 8. 构造 JSON

欢迎阅读本书第二部分。第二部分将介绍 SQL Server 中的 JSON，该功能于 SQL Server 2016 引入。JSON 与 XML 有许多相似之处，但它们并不相同。JSON 可以被视为 XML 的“无脂”版本。要运行第二部分的示例，请确保 SQL Server 2016 中的数据库兼容级别设置为 130。第二部分的示例数据库是 WideWorldImporters。请使用以下链接下载数据库：[`https://github.com/Microsoft/sql-server-samples/releases/tag/wide-world-importers-v1.0?utm_source=MyTechMantra.com`](https://github.com/Microsoft/sql-server-samples/releases/tag/wide-world-importers-v1.0?utm_source=MyTechMantra.com) 。

### 解决方案

您可以为选择性 XML 索引添加或删除路径。清单 7-25 创建了我们之前在解决方案 7-5 中创建的演示表。它用 XML 数据填充该表，并在表上创建了一个选择性 XML 索引。这是此解决方案的先决条件。

```sql
-- These settings are important when creating a table with XML columns and XML indexes
SET NUMERIC_ROUNDABORT OFF;
SET ARITHABORT ON;
SET ANSI_NULLS ON;
SET ANSI_PADDING ON;
SET ANSI_WARNINGS ON;
SET CONCAT_NULL_YIELDS_NULL ON;
SET QUOTED_IDENTIFIER ON;
GO
-- Drop table SQL Server 2016 syntax
DROP TABLE IF EXISTS dbo.ProductXML
-- Create demo table with an XML column
CREATE TABLE dbo.ProductXML
(
ProductID INT NOT NULL,
Name NVARCHAR(50) NOT NULL,
ProductNumber NVARCHAR(25) NOT NULL,
ProductDetails XML NULL
CONSTRAINT PK_ProductXML_ProductID PRIMARY KEY CLUSTERED
(
ProductID ASC
)
);
GO
-- Populate table with sample XML data
INSERT INTO dbo.ProductXML
(
ProductID,
Name,
Number,
ProductDetails
)
SELECT Product2.ProductId,
Product2.Name,
Product2.ProductNumber,
(
SELECT ProductCategory.Name AS "Category/CategoryName",
(
SELECT DISTINCT Location.Name "text()", ', cost rate $',
Location.CostRate "text()"
FROM Production.ProductInventory Inventory
INNER JOIN Production.Location Location
ON Inventory.LocationID = Location.LocationID
WHERE Product.ProductID = Inventory.ProductID
FOR XML PATH('LocationName'), TYPE
) AS "Locations/node()",
Subcategory.Name AS "Category/Subcategory/SubcategoryName",
Product.Name AS "Category/Subcategory/Product/ProductName",
Product.Color AS "Category/Subcategory/Product/Color",
Inventory.Shelf AS "Category/Subcategory/Product/ProductLocation/Shelf",
Inventory.Bin AS "Category/Subcategory/Product/ProductLocation/Bin",
Inventory.Quantity AS "Category/Subcategory/Product/ProductLocation/Quantity"
FROM Production.Product Product
LEFT JOIN Production.ProductInventory Inventory
ON Product.ProductID = Inventory.ProductID
LEFT JOIN Production.ProductSubcategory Subcategory
ON Product.ProductSubcategoryID = Subcategory.ProductSubcategoryID
LEFT JOIN Production.ProductCategory
ON Subcategory.ProductCategoryID = Production.ProductCategory.ProductCategoryID
WHERE Product.ProductID = Product2.ProductId
ORDER BY ProductCategory.Name, Subcategory.Name, Product.Name
FOR XML PATH('Categories'), ROOT('Products'), ELEMENTS, TYPE
)
FROM Production.Product Product2;
GO
-- Create the selective XML index
CREATE SELECTIVE XML INDEX IX_SELECTIVE_XML_ProductXML
ON dbo.ProductXML
(
ProductDetails
)
FOR
(
Quantity = '/Products/Categories/Category/Subcategory/Product/ProductLocation/Quantity',
ProductName = '/Products/Categories/Category/Subcategory/Product/ProductName'
);
GO
```

清单 7-25.
创建带有选择性 XML 索引的演示表

清单 7-26 演示了如何从此索引中移除 `Quantity` 路径并添加 `CategoryName` 路径。

```sql
ALTER INDEX IX_SELECTIVE_XML_ProductXML
ON dbo.ProductXML
FOR
(
ADD CategoryName = '/Products/Categories/Category/CategoryName',
REMOVE Quantity
);
```

清单 7-26.
更改选择性 XML 索引

### 工作原理

`ALTER INDEX` 语句可用于选择性 XML 索引来更改索引内容。可以使用 `REMOVE` 关键字删除现有路径，并可以在 `FOR()` 子句中使用 `ADD` 关键字添加新路径，如清单 7-26 所示。您可以像在清单 7-26 中那样组合使用 `ADD` 和 `REMOVE` 关键字，也可以在单独的 `ALTER INDEX` 语句中发出，如清单 7-27 所示。

```sql
ALTER INDEX IX_SELECTIVE_XML_ProductXML
ON dbo.ProductXML
FOR
(
ADD CategoryName = '/Products/Categories/Category/CategoryName'
);
GO
ALTER INDEX IX_SELECTIVE_XML_ProductXML
ON ProductXML
FOR
(
REMOVE Quantity
);
GO
```

清单 7-27.
使用单独的 `ALTER INDEX` 语句更改选择性索引

每个 `ALTER INDEX` 语句都会重建整个索引。因此，出于效率考虑，在一个 `ALTER INDEX` 语句中实现更改列表比多次运行 `ALTER INDEX` 语句更实用。

要删除选择性 XML 索引，请使用 `DROP INDEX` 语句，如清单 7-28 所示。

```sql
DROP INDEX IX_SELECTIVE_XML_ProductXML
ON dbo.ProductModelXML;
```

清单 7-28.
删除选择性 XML 索引

> **注意**
>
> 请记住，与主 XML 索引类似，当选择性 XML 索引被删除时，所有关联的辅助选择性 XML 索引也会自动被删除。

## 总结

本章完成了本书的 XML 部分。我希望你从 XML 部分提供的解决方案中学到了很多。

如果你实践了 XML 章节中的解决方案，那么你应该能够构建 XML、分解 XML 实例、从不同来源加载 XML 文档、通过 `XQuery` 语句利用基于 XML 的搜索，以及创建 XML 索引。

接下来的部分将介绍 JSON，并演示 SQL Server 2016 与新的 JSON 功能的集成。




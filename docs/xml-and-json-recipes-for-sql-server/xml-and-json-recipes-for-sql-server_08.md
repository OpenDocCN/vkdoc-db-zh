# 2-9. 简化自定义 XML 生成

### 问题

`EXPLICIT` 模式允许对生成的 XML 格式进行细粒度控制，但使用起来很复杂。你想要生成自定义格式的 XML，但希望有一个更简单、同时能达到与 `EXPLICIT` 模式类似效果的替代方案。

### 解决方案

`PATH` 模式可以提供与使用 `EXPLICIT` 模式生成的 XML 相似的结果。然而，用于生成 `PATH` 模式 XML 的机制比 `EXPLICIT` 模式简单得多，这是一个很大的优点。`PATH` 模式的一个限制是它没有 `EXPLICIT` 模式那么多的指令选项。清单 2-27 展示了配方 2-9 中使用 `PATH` 模式实现的查询。

```sql
SELECT ProductCategory.Name AS "Category/CategoryName",
Subcategory.Name AS "Category/Subcategory/SubcategoryName",
Inventory.Shelf AS "Category/Subcategory/Product/ProductName/@Shelf",
Inventory.Bin AS "Category/Subcategory/Product/ProductName/@Bin",
Inventory.Quantity AS "Category/Subcategory/Product/ProductName/@Quantity"
Product.Name AS "Category/Subcategory/Product/ProductName",
Product.Color AS "Category/Subcategory/Product/Color",
FROM Production.Product Product
INNER JOIN Production.ProductInventory Inventory
ON Product.ProductID = Inventory.ProductID
INNER JOIN Production.ProductSubcategory Subcategory
ON Product.ProductSubcategoryID = Subcategory.ProductSubcategoryID
INNER JOIN Production.ProductCategory
ON Subcategory.ProductCategoryID = Production.ProductCategory.ProductCategoryID
ORDER BY ProductCategory.Name, Subcategory.Name, Product.Name
FOR XML PATH('Categories'), ELEMENTS XSINIL, ROOT('Products');
```
清单 2-27. 使用 `PATH` 模式生成自定义 XML

### 工作原理

比较展示 `EXPLICIT` 模式的清单 2-23 和展示 `PATH` 模式的清单 2-27。很明显，`PATH` 模式没有使用多个 `SELECT` 语句和 `UNION ALL` 来生成嵌套 XML 数据。相反，XML 层次结构是通过 XML 路径语言（XPath）风格的列别名定义的，其中路径中的步骤用正斜杠分隔。

要使用 `PATH` 模式生成 XML，请遵循以下规则：

1.  位置应反映预期的 XML 层次结构，即子元素列在父元素下。例如：
    ```sql
    SELECT ProductCategory.Name AS "Category/CategoryName",
    Subcategory.Name AS "Category/Subcategory/SubcategoryName",
    ```

2.  每个子级别都在 XPath 别名中建立，通过用斜杠将其与父元素分隔，并添加子元素名称。例如：
    ```sql
    Name AS "Category/CategoryName",
    Name AS "Category/Subcategory/SubcategoryName",
    Name AS "Category/Subcategory/Product/ProductName",
    ```

3.  当别名中出现“@”符号时，它会将值呈现为属性；否则定义为元素。此代码段展示了“@”在定义 XML 输出属性中的应用：
    ```sql
    SELECT ...
    Inventory.Shelf AS "Category/Subcategory/Product/ProductName/@Shelf",
    Inventory.Bin AS "Category/Subcategory/Product/ProductName/@Bin",
    Inventory.Quantity AS "Category/Subcategory/Product/ProductName/@Quantity",
    ...
    ```

4.  `ORDER BY` 子句在 `PATH` 模式下的效果与在 `EXPLICIT` 模式下不同，因此可以完全省略。但是，根据 XML 结构对查询进行排序是一个好习惯。例如：
    ```sql
    ORDER BY ProductCategory.Name , Subcategory.Name, Product.Name
    ```

5.  建议的命名约定是在括号内为 `PATH` 提供一个元素名称。默认情况下，`PATH` 模式生成一个 `<row>` 元素，但这并不是最佳的 XML 设计。例如：
    ```sql
    FOR XML PATH('Categories')
    ```

6.  当 `XSINIL` 指令在 `PATH` 模式中实现时，该指令会自动应用于所有 XML 元素。然而，在 `EXPLICIT` 模式中，`ELEMENTXSINIL` 指令仅影响指定指令的元素。
7.  指定根元素是良好的 XML 设计实践。我强烈建议在所有 XML 查询中使用 `ROOT(‘ElementName')` 选项。

总结这个配方，我建议分析你期望的 XML，然后决定是使用 `EXPLICIT` 还是 `PATH` 模式。两种模式各有优缺点：`EXPLICIT` 模式使用起来更复杂，但它让你能更好地控制 XML 输出。`PATH` 模式则相反，它易于编写代码；然而，你对元素和属性的控制力稍弱一些。


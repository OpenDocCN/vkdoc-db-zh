# 2-3\. 生成以元素为中心的 XML

### 问题

`RAW` 和 `AUTO` 模式默认都返回以属性为中心的 XML。但是，当业务规则需要时，您希望生成以元素为中心的 XML 数据。

### 解决方案

带有 `ELEMENTS` 选项的 `FOR XML` 子句以以元素为中心的格式返回 XML 结果。清单 2-10 展示了如何向 `FOR XML AUTO` 子句添加 `ELEMENTS` 指令。

```sql
SELECT Product.Name,
Product.ProductNumber AS Number,
Product.ListPrice AS Price
FROM  Production.Product AS Product
WHERE Product.ListPrice > 0
AND Product.SellEndDate IS NULL
ORDER BY Product.Name
FOR XML AUTO, ELEMENTS;
清单 2-10.
带有 ELEMENTS 指令的 FOR XML AUTO 查询
```

带有 `ELEMENTS` 指令的 `FOR XML AUTO` 查询的结果如清单 2-11 所示。

```xml
<Product>
  <Name>Chain</Name>
  <Number>CH-0234</Number>
  <Price>20.2400</Price>
</Product>
<Product>
  <Name>Classic Vest, L</Name>
  <Number>VE-C304-L</Number>
  <Price>63.5000</Price>
</Product>
清单 2-11.
带有 ELEMENTS 指令的 FOR XML AUTO 查询的结果
```

### 工作原理

`ELEMENTS` 选项将您的 XML 结果格式化，将列作为每个行元素的子元素进行嵌套。这种格式被称为以元素为中心的 XML。`ELEMENTS` 选项可以与 `FOR XML` 子句的 `RAW`、`AUTO` 和 `PATH` 模式一起指定。`FOR XML EXPLICIT` 模式不支持 `ELEMENTS` 指令。

`ELEMENTS` 指令必须与 `RAW`、`AUTO` 和 `PATH` 模式关键字用逗号分隔。例如：`FOR XML AUTO, ELEMENTS`。

这个简单的更改允许您以以元素为中心的格式返回 XML，而不是默认的以属性为中心的格式。


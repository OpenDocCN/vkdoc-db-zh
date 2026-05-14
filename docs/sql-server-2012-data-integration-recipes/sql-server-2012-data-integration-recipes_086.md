# 7-13. 创建 XML 模式

## 问题

你想根据 `SQL Server` 表中的 `XML` 数据创建一个 `XML` 模式文件并保存以供后用。

## 解决方案

使用 `T-SQL` 和 `FOR XML AUTO, XMLSCHEMA`。

1.  使用以下代码片段从 `SQL Server` 表中的 `XML` 数据创建一个 `XML` 模式文件，与数据本身一起保存（`C:\SQL2012DIRecipes\CH07\CreateXMLSchema.sql`）：

```sql
SELECT
  [ID]
  ,[ClientName]
  ,[Address1]
  ,[Address2]
  ,[Town]
  ,[County]
  ,[PostCode]
  ,[Country]
FROM [CarSales].[dbo].[Client]
WHERE 1 = 0
FOR XML AUTO,XMLSCHEMA;
```

2.  将模式输出到行集后，只需单击它，它就会出现在新的查询窗口中，然后你可以将其保存为 `.xsd` 文件。

## 工作原理

`XMLSCHEMA` 与 `FOR XML AUTO` 一起使用会自动生成一个 `XML` 模式（`.xsd`）。`WHERE 1 = 0` 子句用于防止实际的 `XML` 数据和模式被输出。当然，如果你希望同时获得模式和数据，可以省略此子句。


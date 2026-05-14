# 2-13. 通过 T-SQL 快速加载文本文件

## 问题

你需要从文本文件中快速执行数据加载。

## 解决方案

使用 T-SQL 中的 `OPENROWSET (BULK)`。以下是实现该操作的代码片段（`C:\SQL2012DIRecipes\CH02\BulkInsertWithOpenrowset.sql`）：

```sql
INSERT INTO     CarSales_Staging.dbo.Invoices
SELECT          ID, InvoiceNumber, ClientID
FROM            OPENROWSET(BULK 'C:\SQL2012DIRecipes\CH02\Invoices.Txt',
                FORMATFILE = 'C:\SQL2012DIRecipes\CH02\Invoicebulkload.Xml') AS MyDATA;
```

## 工作原理

假设你已掌握格式文件的基本知识，现在可以继续使用 `OPENROWSET (BULK)` T-SQL 命令，该命令强制要求使用格式文件。前面的代码片段使用预定义的格式文件加载文本文件。

#### 提示、技巧与陷阱

*   必须提供别名（本例中为 `MyDATA`），否则会收到错误消息。
*   与 `OPENROWSET (BULK)` 一起使用的 T-SQL 可以使用 `WHERE`、`ORDER BY`、`CAST`、`CONVERT` 和别名进行扩展。这在加载数据时提供了极大的灵活性。


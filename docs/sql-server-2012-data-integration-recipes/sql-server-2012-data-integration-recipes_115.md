# 9-8. 使用 T-SQL 连接源数据

## 问题

您希望将来自多个列的数据连接到一个列中。数据已加载到 SQL Server 表中。

## 解决方案

使用 `FOR XML` “黑盒”技术来连接来自两列或多列的数据。

以下代码片段将为 Client 表中的每个客户连接 Invoice 表中的所有发票 ID (`C:\SQL2012DIRecipes\CH09\Concatenate.sql`)：

```sql
SELECT ID, LEFT(CA. CnCatLst, LEN(CA. CnCatLst)-1) AS InvoiceIDs
FROM dbo.Client C
CROSS APPLY (
    SELECT          CAST(InvoiceNumber AS VARCHAR(50)) + ',' AS [text()]
    FROM            dbo.Invoice I
    WHERE           I.ClientID = C.ID
    ORDER BY        I.ID
    FOR XML PATH('')
) CA (CnCatLst)
ORDER BY        ID;
```

## 工作原理

有时，您需要将源数据连接成某种形式的列表——很可能是以分隔符分隔的。以这种方式连接数据是 SQL 处理的另一个领域，关于执行此操作的“最佳”方式可能会引发激烈的讨论。事实上，仅这个主题就有一些出色的论文，我强烈建议您查阅。在这里，我不希望继续讨论，因此我将简单地建议一种自多年前首次在互联网上发表以来我一直使用的技术：XML“黑盒”方法。

这种技术似乎是复杂性和效率之间的一个很好的权衡，前提是连接的文本不超过两 GB——但我认为，如果超过，很可能在成为阻碍问题之前就有其他问题需要处理。还有其他方法可以移除连接文本中的最终逗号。如果您愿意，可以使用 `STUFF` 函数。


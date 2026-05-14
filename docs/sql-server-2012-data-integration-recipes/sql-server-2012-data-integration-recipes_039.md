# 2-8. 将文本文件作为链接服务器表访问

## 问题

您希望避免从文本文件导入数据，但仍能在 T-SQL 查询中使用它。

## 解决方案

将文本文件作为链接服务器表访问。以下说明如何操作，使用 `C:\SQL2012DIRecipes\CH02\Invoices.Txt` 文件作为数据源。

1.  运行以下 T-SQL 片段（此代码连同本配方中所有 T-SQL 都可在 `C:\SQL2012DIRecipes\CH02\FlatFileLinkedServer.sql` 中找到）：
    ```sql
    EXECUTE master.dbo.sp_addlinkedserver TXT_INVOICES, ' ', 'Microsoft.ACE.OLEDB.12.0',
                 'C:\SQL2012DIRecipes\CH02', NULL, 'Text';
    ```
2.  定义安全性——或者更准确地说，定义其缺失：
    ```sql
    EXECUTE master.dbo.sp_addlinkedsrvlogin 'TXT_INVOICES', false, NULL, 'admin';
    ```
3.  创建一个 `Schema`。




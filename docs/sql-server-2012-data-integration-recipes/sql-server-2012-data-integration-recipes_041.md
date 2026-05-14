# 提示、技巧和陷阱

有关如何设置 ODBC DSN 的详细信息，请参阅配方 2-6。

*   如果您创建了 `Schema.ini` 文件，链接服务器将使用它。
*   上一个示例在传递查询中特意使用了 `SELECT *`，以表明传递查询和 T-SQL 可以略有不同。
*   如果您希望使用稍微冗长但更易于调试的代码来创建链接服务器，可以尝试类似这样的方法——这里是针对 Jet 的——使用完全参数化的语句：

```sql
EXECUTE master.dbo.sp_addlinkedserver
    @SERVER = 'TXTACCESS',
    @PROVIDER = 'Microsoft.Jet.OLEDB.4.0',
    @SRVPRODUCT = 'Jet',
    @DATASRC = 'C:\SQL2012DIRecipes\CH02\',
    @PROVSTR = 'Text'
```


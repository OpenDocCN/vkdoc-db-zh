# 6-13. 从 64 位 SQL Server 链接到 32 位数据源

## 问题

您正在 64 位环境中工作，需要实现一个链接服务器，以访问只有 32 位提供程序可用的数据。

## 解决方案

安装第二个 32 位`SQL Server`实例，并设置一个（32 位）链接服务器连接到您的源数据。然后，您可以将 64 位版本的`SQL Server`链接到此实例。虽然执行起来很痛苦且加载数据缓慢，但以下是操作方法（此处使用`Access`作为示例 32 位数据源）。

1.  安装第二个 32 位`SQL Server`实例。
2.  将两台服务器都配置为允许即席分布式查询。这在指南 1-4 中有描述。
3.  在 32 位`SQL Server`实例上选择——或者最好创建——一个数据库，用于将来自链接的`Access`数据库的查询路由到 64 位实例。我将该数据库命名为`CarSales32`。
4.  在 32 位`SQL Server`实例上，创建一个链接服务器以连接到`Access`数据库。以下代码片段可以完成此操作。当然，在这里您需要配置您正在使用的源数据作为链接服务器：
    ```sql
    EXECUTE master.dbo.sp_addlinkedserver @server = N'Access', @srvproduct = N'Access', @provider = N'Microsoft.Jet.OLEDB.4.0', @datasrc = C:\SQL2012DIRecipes\CH01\CarSales.mdb';
    ```
5.  在 32 位`SQL Server`实例上，在您步骤 3 中创建的数据库中，创建一个视图来查询源数据数据库。可以很简单，如下所示（`C:\SQL2012DIRecipes\CH01\vw_SQL32.Sql`）：
    ```sql
    CREATE VIEW vw_SQL32 
     AS 
     SELECT theText from Access. . .Client; 
     GO
    ```
6.  在`SQL Server` 64 位实例上，创建一个链接服务器连接到 32 位`SQL Server`实例。
    ```sql
    EXECUTE master.dbo.
    ```




# 7-20. 安全导出数据以便从 T-SQL 访问

## 问题

你希望将数据安全地导出到 Access——也就是说，在 T-SQL 代码中不暴露安全信息。

## 解决方案

创建一个指向 Access 数据库的链接服务器，然后从 T-SQL 导出数据。

以下步骤说明了如何使用指向 Excel 的链接服务器来导出数据。

1.  添加一个链接服务器：
    ```sql
    EXECUTE master.dbo.sp_addlinkedserver
    @server = N'Access'
    ,@srvproduct = N'Access'
    ,@provider = N'Microsoft.ACE.OLEDB.12.0'
    ,@datasrc = NC:\SQL2012DIRecipes\CH07\TestAccess.mdb'
    ```
2.  设置无安全上下文的数据访问：
    ```sql
    EXECUTE master.dbo.sp_addlinkedsrvlogin
    @rmtsrvname = N'Access', @useself = N'False', @locallogin = NULL,
    @rmtuser = NULL,@rmtpassword = NULL
    ```
3.  使用四部分名称法导出数据，如下所示：
    ```sql
    INSERT INTO   Access...ClientExport (ID, ClientName)
    SELECT        ID,  ClientName FROM dbo.Client
    ```

#### 提示、技巧与陷阱

*   导出数据时，Access 数据库不必关闭。事实上，表本身可以是打开的。
*   Access 将强制执行已应用于目标表的任何约束（例如唯一的主键）。
*   Access 表的数据类型必须能够存储导出的数据类型。
*   如果 Access 数据库受密码保护，那么你需要设置链接服务器登录，使用类似下面的语句：
    ```sql
    EXECUTE master.dbo.sp_addlinkedsrvlogin @rmtsrvname = N'Access', @locallogin = NULL , @useself = N'False', @rmtuser = N'YourAccessUserName', @rmtpassword = N'AccessPassword'
    ```
    ![image](img/sq.jpg) **注意** 你需要将此处显示的用户名和密码替换为你自己的用户名和密码。


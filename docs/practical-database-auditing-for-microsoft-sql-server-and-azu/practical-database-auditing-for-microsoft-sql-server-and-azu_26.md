# 第 6 章 什么是扩展事件？

## 扩展事件用例

以下列表为你介绍了一些使用扩展事件进行审计的场景：

### 审计用户的所有操作

- 要捕获的事件：`rpc_completed` 和 `sql_batch_completed`
- 全局字段：
    - `client_app_name`
    - `client_hostname`
    - `database_name`
    - `server_instance_name`
    - `server_principal_name`
    - `sql_text`

这需要在你选择的每个事件上对 `server_principal_name` 设置筛选器。

### 审计特定数据库中的所有活动

- 要捕获的事件：`rpc_completed` 和 `sql_batch_completed`
- 全局字段：
    - `client_app_name`
    - `client_hostname`
    - `database_name`
    - `server_instance_name`
    - `server_principal_name`
    - `sql_text`

这需要在你选择的每个事件上对 `database_name` 设置筛选器，但请谨慎使用，因为它会产生大量审计数据。我通常只在试图确认某个数据库是否不再使用时才这样做。

### 审计数据库服务器上的所有活动

- 要捕获的事件：`rpc_completed` 和 `sql_batch_completed`
- 全局字段：
    - `client_app_name`
    - `client_hostname`
    - `database_name`
    - `server_instance_name`
    - `server_principal_name`
    - `sql_text`

这不需要设置筛选器，但请谨慎使用，因为它会产生大量审计数据。我通常只在试图确认某个 SQL Server 是否不再使用时才这样做。

### 审计使用存储过程或表的所有人

- 要捕获的事件：`rpc_completed`
- 全局字段：
    - `client_app_name`
    - `client_hostname`
    - `database_name`
    - `server_instance_name`
    - `server_principal_name`
    - `sql_text`

这需要在你选择的每个事件上对 `object_name` 设置筛选器。

在下一章中，你将学习如何在 SQL Server Management Studio 中设置和配置扩展事件。你还将学习如何查询扩展事件以了解数据库服务器上正在发生的事情。


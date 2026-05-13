# 数据库

*   `DATABASE_PERMISSION_CHANGE_GROUP` – 捕获当对语句权限发出 `GRANT`（授予）、`REVOKE`（撤销）或 `DENY`（拒绝）时。
*   `DATABASE_PRINCIPAL_CHANGE_GROUP` – 捕获当主体（如用户）在数据库中被创建、修改或删除时。
*   `DATABASE_ROLE_MEMBER_CHANGE_GROUP` – 捕获当登录名被添加到数据库角色或从数据库角色中移除时。
*   `APPLICATION_ROLE_CHANGE_PASSWORD_GROUP` – 捕获当应用程序角色的密码被更改时。
*   `DBCC_GROUP` – 捕获当用户执行 `DBCC` 命令时。
*   `SCHEMA_OBJECT_CHANGE_GROUP` – 捕获当对架构执行 `CREATE`（创建）、`ALTER`（修改）或 `DROP`（删除）操作时。
*   `SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP` – 捕获当架构对象的所有权权限发生变化时。
*   `SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP` – 捕获当对架构对象发出授予、拒绝或撤销权限时。

以下操作类型可用于捕获数据库或数据库服务器中发生的所有活动。我仅会在将审计筛选为仅针对一个用户，或检查某个架构或数据库是否不再使用时，才使用这些操作类型。*请务必谨慎*使用这些操作，它们可能迅速失控，收集到海量的审计数据，以至于你永远无法梳理清楚。

*   `DATABASE_OBJECT_ACCESS_GROUP` – 捕获当在被审计的数据库中采取任何操作时。
*   `SCHEMA_OBJECT_ACCESS_GROUP` – 捕获当在被审计的架构中采取任何操作时。

## 第三章 什么是 SQL Server 审计？

**注意** 还有其他可用的数据库审计操作组。每个组的描述可在以下链接中找到：[`docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15#database-level-audit-action-groups`](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15#database-level-audit-action-groups)

当处理数据库审计时，有些操作在服务器审计级别不可用。这些操作将捕获在数据库架构和架构对象（如表、视图、存储过程和函数）上执行的动作。以下列表概述了这些操作组：

*   `SELECT` – 捕获 `SELECT` 语句。
*   `UPDATE` – 捕获 `UPDATE` 语句。
*   `INSERT` – 捕获 `INSERT` 语句。
*   `DELETE` – 捕获 `DELETE` 语句。
*   `EXECUTE` – 捕获 `EXECUTE` 语句。

**注意** 还有其他可用的数据库审计操作。每个操作的描述可在以下链接中找到：[`docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15#database-level-audit-actions`](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15#database-level-audit-actions)

#### SQL Server 审计示例

H 以下是一些具体示例，展示了如何实现我在本章中概述的用例。



## 第三章 什么是 SQL Server 审计？

如果您想以相同的方式审计服务器和所有数据库的架构及权限更改，请实施包含以下操作的服务器审计规范：

* `AUDIT_CHANGE_GROUP`
* `SERVER_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `SERVER_OBJECT_PERMISSION_CHANGE_GROUP`
* `SERVER_OPERATION_GROUP`
* `SERVER_PERMISSION_CHANGE_GROUP`
* `SERVER_PRINCIPAL_CHANGE_GROUP`
* `SERVER_ROLE_MEMBER_CHANGE_GROUP`
* `SERVER_STATE_CHANGE_GROUP`
* `LOGIN_CHANGE_PASSWORD_GROUP`
* `DATABASE_CHANGE_GROUP`
* `DATABASE_OBJECT_ACCESS_GROUP`
* `DATABASE_OBJECT_CHANGE_GROUP`
* `DATABASE_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `DATABASE_OBJECT_PERMISSION_CHANGE_GROUP`
* `DATABASE_OWNERSHIP_CHANGE_GROUP`
* `DATABASE_PERMISSION_CHANGE_GROUP`
* `DATABASE_PRINCIPAL_CHANGE_GROUP`
* `DATABASE_ROLE_MEMBER_CHANGE_GROUP`
* `APPLICATION_ROLE_CHANGE_PASSWORD_GROUP`
* `DBCC_GROUP`
* `SCHEMA_OBJECT_CHANGE_GROUP`
* `SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP`

如果您想审计服务器上的架构和权限更改，但不审计任何数据库（因为您为每个数据库实施了不同的数据库审计），请实施包含以下操作的服务器审计规范：

* `AUDIT_CHANGE_GROUP`
* `SERVER_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `SERVER_OBJECT_PERMISSION_CHANGE_GROUP`
* `SERVER_OPERATION_GROUP`
* `SERVER_PERMISSION_CHANGE_GROUP`
* `SERVER_PRINCIPAL_CHANGE_GROUP`
* `SERVER_ROLE_MEMBER_CHANGE_GROUP`
* `SERVER_STATE_CHANGE_GROUP`
* `LOGIN_CHANGE_PASSWORD_GROUP`

如果您想审计特定用户执行的所有操作，您将需要使用一些额外的审计操作。除非您在服务器上筛选较小的活动子集，否则我喜欢尽量减少这些操作的使用。您将使用前面列出的所有用于审计所有架构和权限更改的操作，此外，还将包括以下审计操作：

* `DATABASE_OBJECT_ACCESS_GROUP`
* `SCHEMA_OBJECT_ACCESS_GROUP`

如果您想审计每个人查询或修改表的操作，则需要为此使用数据库审计规范。在此审计中，您将使用以下审计操作：

* `SELECT`
* `UPDATE`
* `INSERT`
* `DELETE`
* `EXECUTE`

如果您想审计每个人在服务器级别和仅某个特定数据库中进行的架构和权限更改，您将需要一个服务器审计规范和一个数据库审计规范。您的服务器审计将仅包括如下列出的服务器审计操作：

* `AUDIT_CHANGE_GROUP`
* `SERVER_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `SERVER_OBJECT_PERMISSION_CHANGE_GROUP`
* `SERVER_OPERATION_GROUP`
* `SERVER_PERMISSION_CHANGE_GROUP`
* `SERVER_PRINCIPAL_CHANGE_GROUP`
* `SERVER_ROLE_MEMBER_CHANGE_GROUP`
* `SERVER_STATE_CHANGE_GROUP`
* `LOGIN_CHANGE_PASSWORD_GROUP`

对于此情况，您的数据库审计将仅在您希望审计的数据库上包括以下审计操作：

* `DATABASE_CHANGE_GROUP`
* `DATABASE_OBJECT_ACCESS_GROUP`
* `DATABASE_OBJECT_CHANGE_GROUP`
* `DATABASE_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `DATABASE_OBJECT_PERMISSION_CHANGE_GROUP`
* `DATABASE_OWNERSHIP_CHANGE_GROUP`
* `DATABASE_PERMISSION_CHANGE_GROUP`
* `DATABASE_PRINCIPAL_CHANGE_GROUP`
* `DATABASE_ROLE_MEMBER_CHANGE_GROUP`
* `APPLICATION_ROLE_CHANGE_PASSWORD_GROUP`
* `DBCC_GROUP`
* `SCHEMA_OBJECT_CHANGE_GROUP`
* `SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP`
* `SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP`

#### 多重审计设置

如果您需要在同一服务器上进行多个不同的审计，可以通过配置使它们几乎没有重叠。不过，根据审计要求，您可能需要一些重叠，但您希望尽可能避免重叠。以下是一些可以相互配合使用的审计规范设置：

* 审计服务器级别的 DDL 和权限更改




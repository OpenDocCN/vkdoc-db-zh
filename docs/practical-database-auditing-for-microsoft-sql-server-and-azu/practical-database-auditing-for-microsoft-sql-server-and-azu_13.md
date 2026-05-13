# 第 3 章 SQL Server 审计是什么？

在“安全” ➤ “审计”下设置审计规范，可以决定保留多少数据、是否只捕获特定类型的数据，以及审计日志记录失败时的处理方式。每个数据库可以拥有多个审计规范，每个审计规范可以包含一个服务器审计规范和一个数据库审计规范。

## 服务器审计规范

服务器审计规范用于配置希望在服务器级别，以及可选地在数据库级别收集的审计内容。可以在 SQL Server Management Studio 的“安全” ➤ “服务器审计规范”下设置服务器审计规范。可以拥有多个服务器审计规范，但它们各自需要一个单独的审计规范，因为每个审计规范只能包含一个服务器审计规范。此服务器审计可以审计服务器级对象，但也可以以相同方式审计所有数据库。如果对每个数据库有更具体的审计需求，则应在服务器审计规范中仅使用服务器级别的操作。本章稍后将提供更多关于审计操作的信息。

## 数据库审计规范

数据库审计规范用于配置希望在数据库级别收集的审计内容。可以在每个数据库的“安全” ➤ “数据库审计规范”下设置服务器审计规范。每个数据库上可以拥有多个数据库审计规范，但它们各自需要一个单独的审计，因为每个审计规范每个数据库只能包含一个数据库审计规范。

设置和配置审计规范将在第 4 章《通过 GUI 实现 SQL Server 审计》中介绍。

> **注意**：设置 SQL Server Audit 不需要 `sysadmin` 权限，但需要一些自定义权限来设置。对于审计和服务器审计规范，需要 `CONTROL SERVER` 或 `ALTER ANY SERVER AUDIT` 权限。对于数据库审计规范，需要 `ALTER ANY DATABASE AUDIT` 或 `CONTROL SERVER` 权限。当然，`sysadmin` 权限也可以用来设置所有审计组件。

#### SQL Server 审计使用案例

服务器审计通常适用于审计服务器级更改和/或同时审计所有数据库。数据库审计则适用于审计单个数据库或其中一部分活动。以下列表介绍了使用多个审计规范以及服务器和数据库审计进行审计的一些场景：

*   **以相同方式审计服务器和所有数据库的架构与权限更改**
    这需要一个审计规范和一个服务器审计规范。
*   **审计特定用户执行的所有操作**
    这需要一个带筛选器以仅捕获该用户活动的审计规范，以及一个服务器审计规范。
*   **审计所有人查询或修改某个表**
    这需要一个审计规范和一个数据库审计规范，该规范需指定此表以及希望捕获的相关操作。
*   **审计服务器级别的架构和权限更改以及仅特定数据库**
    这需要一个审计规范、一个服务器审计规范和一个数据库审计规范。

#### 审计类别

审计操作分为三类：服务器、数据库和审计。

*   **服务器级操作** – 捕获权限更改和数据库创建。包括任何不以 `SCHEMA_` 或 `DATABASE_` 开头的审计操作。
*   **数据库级操作** – 捕获 DML 和 DDL 更改，这包括数据库级别的审计操作。包括以 `SCHEMA_` 或 `DATABASE_` 开头的审计操作。

## 审计级别操作与操作组

### 审计级别操作
审计级别操作是指在审计过程中捕获的操作，例如创建或删除审计规范。这对应于 `AUDIT_CHANGE_GROUP` 选项。

#### 审计操作组
在每个审计类别中，都有一些你可以捕获的操作组。有些操作是自动审计的，你无需设置任何审计操作来捕获它们，例如“服务器审计状态更改”（将 `State` 设置为 `ON` 或 `OFF`）。这对应于你启动或停止审计的时刻。

##### 服务器审计操作组
对于服务器审计，以下是我认为在服务器级别审计变更时最有用的操作组。它们只能在服务器审计规范中选择。下面列出了这些组：

- **`AUDIT_CHANGE_GROUP`** – 捕获审计创建、修改或删除时的操作。
- **`DBCC_GROUP`** – 捕获用户执行 `DBCC` 命令时的操作。
- **`SERVER_OBJECT_OWNERSHIP_CHANGE_GROUP`** – 捕获服务器对象所有者变更时的操作。
- **`SERVER_OBJECT_PERMISSION_CHANGE_GROUP`** – 捕获对服务器对象权限执行 `GRANT`、`REVOKE` 或 `DENY` 时的操作。
- **`SERVER_OPERATION_GROUP`** – 捕获类似更改设置、资源、外部访问或授权等变更。
- **`SERVER_PERMISSION_CHANGE_GROUP`** – 捕获在服务器级别设置权限的 `GRANT`、`REVOKE` 或 `DENY` 时的操作。
- **`SERVER_PRINCIPAL_CHANGE_GROUP`** – 捕获服务器主体创建、修改或删除时的操作。
- **`SERVER_ROLE_MEMBER_CHANGE_GROUP`** – 捕获登录名被添加到或从固定服务器角色（如 `sysadmin`）中移除时的操作。
- **`SERVER_STATE_CHANGE_GROUP`** – 捕获 `SQL Server` 服务状态被修改时的操作，例如在打补丁后重启。
- **`LOGIN_CHANGE_PASSWORD_GROUP`** – 捕获登录密码被更改时的操作。

> **注意**：还有其他可用的服务器审计操作组。每个组的描述可在以下链接中找到：
> [`docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15#server-level-audit-action-groups`](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions?view=sql-server-ver15#server-level-audit-action-groups)

##### 数据库审计操作组
对于数据库审计，以下是我认为在捕获数据库级别的架构和权限变更时最有用的操作组。它们可以在服务器审计规范中选择，这将允许以相同方式审计服务器上的所有数据库。如果你只想用这些操作审计一个数据库，请不要在服务器审计中使用它们，而只在数据库审计中设置。下面列出了这些组：

- **`DATABASE_CHANGE_GROUP`** – 捕获数据库创建、修改或删除时的操作。
- **`DATABASE_OBJECT_ACCESS_GROUP`** – 捕获访问数据库对象（如证书和非对称密钥）时的操作。
- **`DATABASE_OBJECT_CHANGE_GROUP`** – 捕获在数据库对象（如架构）上执行 `CREATE`、`ALTER` 或 `DROP` 语句时的操作。
- **`DATABASE_OBJECT_OWNERSHIP_CHANGE_GROUP`** – 捕获数据库内对象所有者发生变更时的操作。
- **`DATABASE_OBJECT_PERMISSION_CHANGE_GROUP`** – 捕获对数据库对象（如程序集和架构）发出 `GRANT`、`REVOKE` 或 `DENY` 时的操作。
- **`DATABASE_OWNERSHIP_CHANGE_GROUP`** – 捕获你使用 `ALTER AUTHORIZATION` 语句更改数据库所有者时的操作。



# 第 5 章 通过 SQL 脚本实施 SQL Server 审计

第二个可选组件是数据库审计。清单 5-6 展示了如何通过脚本创建数据库审计规范。

## 清单 5-6. 创建数据库审计规范

```sql
USE [dbname];

CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpecification_dbname]
FOR SERVER AUDIT [AuditSpecification]
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
ADD (AUDIT_CHANGE_GROUP),
ADD (DBCC_GROUP),
ADD (DATABASE_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_OBJECT_PERMISSION_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_CHANGE_GROUP),
ADD (DATABASE_OBJECT_CHANGE_GROUP),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_CHANGE_GROUP),
ADD (APPLICATION_ROLE_CHANGE_PASSWORD_GROUP),
ADD (DATABASE_OWNERSHIP_CHANGE_GROUP),
ADD (DATABASE_OBJECT_OWNERSHIP_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP),
ADD (USER_CHANGE_PASSWORD_GROUP)
WITH (STATE = ON);
```

关于如何配置你的数据库审计规范的建议：

• `数据库上下文` – `USE [dbname];`
所有数据库审计规范都在你想要审计的数据库中创建。

• `名称` – `CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpecification_dbname]`
我总是用下划线加数据库名来命名，因为它有助于识别它审计的是哪个数据库，例如 `DatabaseAuditSpecification_Auditing`。这完全取决于你希望名称具有多大描述性。不过，你可以创建多个审计，所以如果你计划添加更多审计，使用更具描述性的名称可能更有意义。需要注意的一点是，如果你不在数据库审计的名称中包含数据库名称，你将无法轻松查询存储审计信息的系统表。如果你的审计命名具有描述性，这些表会非常有帮助。本章稍后将更详细地介绍查询系统表。

• `审计` – `FOR SERVER AUDIT [AuditSpecification]`
你需要将其与你的审计关联起来。你需要这种关联，因为这是你的审计数据存放的地方。

• `操作` – `ADD (AUDIT_ACTIONS)`
我正在数据库级别捕获权限和架构更改。第 [3](https://doi.org/10.1007/978-1-4842-8634-0_3) 章“什么是 SQL Server 审计？”中有一个关于数据库审计操作组的部分，可帮助你确定每个操作审计的内容。

> `如果你已经在服务器审计级别获取权限和架构更改，请不要使用此功能。这将产生重复的审计记录。`

• `启用审计` – `WITH (STATE = ON);`
你需要启用它，以便它开始收集审计数据。

数据库审计的亮点在于你想审计对象时。通过数据库审计，你可以审计对当前数据库中对象（如表、视图和存储过程）执行的插入、更新、删除、选择和执行等操作。你还可以审计整个架构或数据库。清单 5-7 展示了如何通过脚本创建数据库审计规范来审计 DML 操作。

## 清单 5-7. 创建数据库审计规范以审计 DML 操作

```sql
USE [Auditing];

CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpecification_AuditingTables]
FOR SERVER AUDIT [AuditSpecification_AuditingTables]
ADD (INSERT ON OBJECT::[dbo].[testing] BY [public]),
ADD (EXECUTE ON OBJECT::[dbo].[SelectTestingTable] BY [public]),
ADD (SELECT ON OBJECT::[dbo].[TestingTop10] BY [public]),
ADD (DELETE ON SCHEMA::[dbo] BY [auditing]),
ADD (UPDATE ON DATABASE::[Auditing] BY [public])
WITH (STATE = ON);
```

关于在审计特定对象、架构或数据库时如何配置数据库审计规范的建议：

• `数据库上下文` – `USE [dbname];`
所有数据库审计规范都在你想要审计的数据库中创建。

• `名称` – `CREATE DATABASE AUDIT SPECIFICATION [DatabaseAuditSpecification_AuditingTables]`




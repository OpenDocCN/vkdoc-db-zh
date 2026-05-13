# 第 5 章 通过 SQL 脚本实现 SQL Server 审计

[`docs.microsoft.com/en-us/sql/relational-databases/security/auditing/write-sql-server-audit-events-to-the-security-log?view=sql-server-ver15`](https://docs.microsoft.com/en-us/sql/relational-databases/security/auditing/write-sql-server-audit-events-to-the-security-log?view=sql-server-ver15)

如果您为审计选择了文件目标，则在启用审计后，审计文件会放置在磁盘上，如图 5-3 所示。这里就是与该审计相关联的服务器和数据库审计数据将存储的位置。

***图 5-3.** 磁盘上的审计文件*

随着数据收集，此文件将增长至审计中指定的大小。然后，它将创建另一个文件，直至达到配置中指定的文件数量。一旦最后一个文件写满，它将删除最旧的文件并创建一个新的文件。您需要了解文件写满的速度，以便在文件被删除前不会错过收集其中的数据。

#### 设置服务器审计规范

两个可选审计中的第一个是服务器审计规范。清单 5-5 展示了如何通过脚本创建服务器审计规范。

***清单 5-5.** 创建服务器审计规范*

```sql
USE [master];

CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpecification]
FOR SERVER AUDIT [AuditSpecification]
ADD (DATABASE_OBJECT_ACCESS_GROUP),
ADD (SCHEMA_OBJECT_ACCESS_GROUP),
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (AUDIT_CHANGE_GROUP),
ADD (DBCC_GROUP),
ADD (DATABASE_PERMISSION_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP),
ADD (SERVER_OBJECT_PERMISSION_CHANGE_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_CHANGE_GROUP),
ADD (DATABASE_OBJECT_CHANGE_GROUP),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_CHANGE_GROUP),
ADD (SERVER_OBJECT_CHANGE_GROUP),
ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
ADD (SERVER_OPERATION_GROUP),
ADD (APPLICATION_ROLE_CHANGE_PASSWORD_GROUP),
ADD (LOGIN_CHANGE_PASSWORD_GROUP),
ADD (SERVER_STATE_CHANGE_GROUP),
ADD (DATABASE_OWNERSHIP_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP),
ADD (SERVER_OBJECT_OWNERSHIP_CHANGE_GROUP),
ADD (USER_CHANGE_PASSWORD_GROUP)
WITH (STATE = ON);
```

关于如何配置服务器审计规范的建议：

*   `数据库上下文` – `USE [master];`
    **所有服务器审计规范都是在 master 数据库中创建的。**
*   `名称` – `CREATE SERVER AUDIT SPECIFICATION [ServerAuditSpecification]`
    我倾向于将其命名为`ServerAuditSpecification`或`ServerAuditSpecification_servername`。这完全取决于您希望名称具有多大的描述性。不过，您可以创建多个服务器审计，因此，如果您计划添加额外审计，使其名称更具描述性可能更有意义。
*   `审计` – `FOR SERVER AUDIT [AuditSpecification]`
    您需要将其与您的审计关联起来。您需要此关联，因为这是您的审计数据将存放的地方。
*   `操作` – `ADD (AUDIT_ACTIONS)`
    我在服务器级别以及服务器上的所有数据库中捕获权限和架构更改。我还会捕获是否有人更改了密码、更改了审计，或者是否有人发出了`DBCC`命令。第 3 章“什么是 SQL Server 审计？”中有一个关于服务器审计操作组的小节，可以帮助您确定每个操作审计的内容。
*   `启用审计` – `WITH (STATE = ON);`
    您需要启用它，以便它收集审计数据。

#### 设置数据库审计规范





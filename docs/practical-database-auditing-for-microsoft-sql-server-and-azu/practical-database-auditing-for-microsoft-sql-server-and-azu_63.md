# 第 15 章 其他云提供商审计选项

## 审计组
要配置审计，可以使用 `ADD` 语句添加各种操作组：

- `ADD (DATABASE_OBJECT_ACCESS_GROUP)`
- `ADD (SCHEMA_OBJECT_ACCESS_GROUP)`
- `ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP)`
- `ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP)`
- `ADD (AUDIT_CHANGE_GROUP)`
- `ADD (DBCC_GROUP)`
- `ADD (DATABASE_PERMISSION_CHANGE_GROUP)`
- `ADD (SCHEMA_OBJECT_PERMISSION_CHANGE_GROUP)`
- `ADD (SERVER_OBJECT_PERMISSION_CHANGE_GROUP)`
- `ADD (SERVER_PERMISSION_CHANGE_GROUP)`
- `ADD (DATABASE_CHANGE_GROUP)`
- `ADD (DATABASE_OBJECT_CHANGE_GROUP)`
- `ADD (DATABASE_PRINCIPAL_CHANGE_GROUP)`
- `ADD (SCHEMA_OBJECT_CHANGE_GROUP)`
- `ADD (SERVER_OBJECT_CHANGE_GROUP)`
- `ADD (SERVER_PRINCIPAL_CHANGE_GROUP)`
- `ADD (SERVER_OPERATION_GROUP)`
- `ADD (APPLICATION_ROLE_CHANGE_PASSWORD_GROUP)`
- `ADD (LOGIN_CHANGE_PASSWORD_GROUP)`
- `ADD (SERVER_STATE_CHANGE_GROUP)`
- `ADD (DATABASE_OWNERSHIP_CHANGE_GROUP)`
- `ADD (SCHEMA_OBJECT_OWNERSHIP_CHANGE_GROUP)`
- `ADD (SERVER_OBJECT_OWNERSHIP_CHANGE_GROUP)`
- `ADD (USER_CHANGE_PASSWORD_GROUP)`

`WITH (STATE = ON)`

清单 15-2 中的服务器审计将仅捕获架构和权限变更。这将使您的审计文件更小，数量更少。

## 查询 SQL Server 审计数据
在每个文件达到其最大大小之前，它将存储在数据库实例上。这些审计文件存储在 `D:\rdsdbdata\SQLAudit\` 中。一旦文件达到最大大小，它将被上传到 S3。此时，该文件会被移入名为 `transmitted` 的保留文件夹。然后，您的审计文件将存储在 `D:\rdsdbdata\SQLAudit\transmitted` 中。

要查询审计数据，请使用清单 15-3 中的脚本。

**清单 15-3.** 查询尚未传输到 S3 的审计数据

```sql
SELECT DISTINCT
    event_time,
    aa.name as audit_action,
    statement,
    succeeded,
    database_name,
    server_instance_name,
    schema_name,
    session_server_principal_name,
    server_principal_name,
    object_Name,
    file_name,
    client_ip,
    application_name,
    file_name
FROM msdb.dbo.rds_fn_get_audit_file('D:\rdsdbdata\SQLAudit\*.sqlaudit', default, default) af
INNER JOIN sys.dm_audit_actions aa
    ON aa.action_id = af.action_id
WHERE event_time > DATEADD(HOUR, -1, GETDATE())
ORDER BY event_time DESC;
```

**提示：** AWS 在审计功能方面做得很好，尽可能地模仿了 SQL Server 的行为。您可以使用 `*.sqlaudit` 一次性查询所有审计文件。在 Azure SQL 数据库托管实例中查询审计数据则较为困难，因为您必须知道文件的确切名称。在 Azure SQL 数据库托管实例中，您无法使用 `*.sqlaudit` 一次性查询所有文件。

即使按照清单 15-1 和 15-2 设置了审计，它仍会捕获大量在后台发生的操作。您需要对其进行过滤。可能会存在类似 `WORKGROUP\EC2AMAZ-QG7G9L3$` 的用户，具体取决于您的实例。

## 过滤审计数据
您需要过滤掉该用户，以避免审计数据过多的问题。您可能还会看到大量 `sys` 架构的活动，而这些您无需审计。此外，`rdsa` 可能会被大量审计，其中很多是您不需要审计的后台操作。您需要的过滤器将取决于您看到的、使审计变得杂乱的内容。清单 15-4 提供了一种过滤审计的方法。

**清单 15-4.** 过滤您的审计

```sql
USE [master];
ALTER SERVER AUDIT [AuditSpecification] WITH (STATE = OFF);

ALTER SERVER AUDIT [AuditSpecification]
WHERE [database_name] <> 'rdsadmin'
    AND session_server_principal_name <> 'WORKGROUP\EC2AMAZ-QG7G9L3$'
    AND schema_name <> 'sys'
    AND server_principal_name <> 'rdsa';

ALTER SERVER AUDIT [AuditSpecification] WITH (STATE = ON);
```

如果您需要查询已经移动到 S3 的审计数据，则需要像清单 15-5 那样更改您的查询。与清单 15-3 的唯一区别在于文件名路径。

**清单 15-5.** 查询已传输到 S3 的审计数据

```sql
SELECT DISTINCT
    event_time,
    aa.name as audit_action,
    statement,
    succeeded,
```



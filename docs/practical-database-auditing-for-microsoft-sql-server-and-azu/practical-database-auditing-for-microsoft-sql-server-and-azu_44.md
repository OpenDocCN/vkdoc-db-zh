# 第 11 章 集中审计数据

您不必使用 `_servername` 来命名审计。这只是我个人选择的方式。如果您愿意，可以采用不同的命名方式，但我倾向于这样命名，以便在跨多个服务器查询系统表时能够区分它们。

#### 创建集中审计数据库和用户

在您选作集中审计数据库服务器的服务器上，需要如**清单 11-4**所示设置审计数据库。

### 清单 11-4. 设置审计数据库

```
CREATE DATABASE [Auditing];
```

接下来，您需要创建一个表来存储审计数据，如**清单 11-5**所示。

### 清单 11-5. 设置审计表

```
USE [Auditing];

CREATE TABLE [dbo].AuditChanges NOT NULL,
    [statement] nvarchar NULL,
    [server_instance_name] nvarchar NULL,
    [database_name] nvarchar NULL,
    [schema_name] nvarchar NULL,
    [session_server_principal_name] nvarchar NULL,
    [server_principal_name] nvarchar NULL,
    [object_Name] nvarchar NULL,
    [file_name] nvarchar NOT NULL,
    [client_ip] nvarchar NULL,
    [application_name] nvarchar NULL,
    [host_name] nvarchar NULL,
    [succeeded] [bit] NULL,
    [audit_action] nvarchar NULL
) ON [PRIMARY];

CREATE CLUSTERED INDEX [CIX_EventTime_User_Server_DB] ON [dbo].[AuditChanges]
(
    [event_time] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF,
       DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY];
```

然后，您需要一个拥有审计数据库权限的审计用户，如**清单 11-6**所示。这样，被审计服务器上的链接服务器才能访问审计数据库。

### 清单 11-6. 设置审计用户

```
USE [master];

CREATE LOGIN [sqlauditing] WITH PASSWORD=N'testing1234!',
DEFAULT_DATABASE=[master], CHECK_EXPIRATION=OFF, CHECK_POLICY=ON;

USE [Auditing];

CREATE USER [sqlauditing] FOR LOGIN [sqlauditing];

ALTER ROLE [db_datareader] ADD MEMBER [sqlauditing];
ALTER ROLE [db_datawriter] ADD MEMBER [sqlauditing];
```

**注意** 要设置**清单 11-6**中的 `sqlauditing` 用户，需要将服务器身份验证设置为“SQL Server 和 Windows 身份验证模式”。有关身份验证方法的更多信息，请访问 https://docs.microsoft.com/zh-cn/sql/relational-databases/security/choose-an-authentication-mode?view=sql-server-ver16。

请务必将脚本中的密码从其默认值 `‘testing1234!’` 更新为您选择的密码。

#### 创建链接服务器

您需要在每个被审计的服务器上设置一个链接服务器，以将审计数据发送到集中服务器。**清单 11-7**提供了创建此链接服务器的脚本。

### 清单 11-7. 设置链接服务器

```
USE [master];

EXEC master.dbo.sp_addlinkedserver @server = N'YourCentralizedAuditServerName', @srvproduct=N'SQL Server';

EXEC master.dbo.sp_addlinkedsrvlogin @rmtsrvname = N'YourCentralizedAuditServerName', @locallogin = NULL , @useself = N'False',
@rmtuser = N'sqlauditing', @rmtpassword = N'CreateStrongPasswordHere';
```

请务必将 `@server` 变量的值更新为您的中央服务器名称。同时，务必将 `@rmtpassword` 更新为您为 `sqlauditing` 用户设置的密码。

#### 用于收集和清理审计数据的 SQL Agent 作业

您需要在每个被审计的服务器上设置一个 SQL Server Agent 作业，以将审计数据发送到集中服务器。



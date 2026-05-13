# 第 12 章 从审计数据创建报告

将审计数据集中后，您可能想要对其进行查询和报告。本章将展示如何使用 SQL Server Agent 或 PowerShell 创建和发送 HTML 报告。

#### 使用 SQL Server Agent 创建 HTML 报告

由于目标是使用 SQL Server Agent 通过电子邮件发送 HTML 报告，因此需要首先配置数据库邮件。

**提示** 有关设置数据库邮件的更多信息，请访问 [`docs.microsoft.com/en-us/sql/relational-databases/database-mail/configure-database-mail?view=sql-server-ver15`](https://docs.microsoft.com/en-us/sql/relational-databases/database-mail/configure-database-mail?view=sql-server-ver15)。



[配置数据库邮件?view=sql-server-ver15](https://docs.microsoft.com/en-us/sql/relational-databases/database-mail/configure-database-mail?view=sql-server-ver15)

你可以通过执行清单 12-1 中的脚本来检查是否已经设置了数据库邮件。

## ***清单 12-1.*** 验证任何当前的数据库邮件设置

```sql
EXEC msdb.dbo.sysmail_help_account_sp;
```

如果你已经设置了数据库邮件，请记下清单 12-1 中查询返回的名称。在本章稍后设置代理作业时会需要它。

如果你尚未设置数据库邮件，则需要使用清单 12-2 中的脚本来启用数据库邮件 XPs。请注意，在启用数据库邮件之前，你需要将 `show advanced options` 设置为 1。

© Josephine Bush 2022

J. Bush, *Microsoft SQL Server 和 Azure SQL 的实用数据库审计*, [`doi.org/10.1007/978-1-4842-8634-0_12`](https://doi.org/10.1007/978-1-4842-8634-0_12#DOI)

第 12 章 从审计数据创建报告

## ***清单 12-2.*** 启用数据库邮件

```sql
USE master
GO
sp_configure 'show advanced options',1
reconfigure
GO
sp_configure 'Database Mail XPs',1
reconfigure
GO
```

现在你已启用数据库邮件，你需要对其进行配置。清单 12-3 提供了一个配置脚本。

## ***清单 12-3.*** 配置数据库邮件

```sql
DECLARE @profilename varchar(30);
DECLARE @emailaddress varchar(100);
DECLARE @displayname varchar(50);
DECLARE @mailserver varchar(50);

/* 仅更改这四个变量 */
SET @profilename = 'yourdbmailprofilename';
SET @emailaddress = 'youremail@domain.com';
SET @displayname = 'yourservername SQL Server Alerting';
SET @mailserver = 'smtp.domain.com';

IF NOT EXISTS(SELECT * FROM msdb.dbo.sysmail_profile WHERE name =
'yourdbmailprofilename')
BEGIN
    EXECUTE msdb.dbo.sysmail_add_profile_sp
        @profile_name = @profilename,
        @description = '';
END

IF NOT EXISTS(SELECT * FROM msdb.dbo.sysmail_account
              WHERE name = @profilename)
BEGIN
    EXECUTE msdb.dbo.sysmail_add_account_sp
        @account_name = @profilename,
        @email_address = @emailaddress,
        @display_name = @displayname,
        @replyto_address = @emailaddress,
        @description = '',
        @mailserver_name = @mailserver,
        @mailserver_type = 'SMTP',
        @port = '25',
        @username = NULL ,
        @password = NULL ,
        @use_default_credentials = 0 ,
        @enable_ssl = 0 ;
END

IF NOT EXISTS(SELECT *
              FROM msdb.dbo.sysmail_profileaccount pa
              INNER JOIN msdb.dbo.sysmail_profile p
                  ON pa.profile_id = p.profile_id
              INNER JOIN msdb.dbo.sysmail_account a
                  ON pa.account_id = a.account_id
              WHERE p.name = @profilename
                  AND a.name = @profilename)
BEGIN
    EXECUTE msdb.dbo.sysmail_add_profileaccount_sp
        @profile_name = @profilename,
        @account_name = @profilename,
        @sequence_number = 1 ;
END
```

你只需要更新清单 12-3 脚本顶部的变量：

- `@profilename` – 这是你希望你的数据库邮件配置文件被命名的名称。
- `@emailaddress` – 这将是同时作为发送至和发送自的电子邮件地址。
- `@displayname` – 这将是使用你的数据库邮件配置文件发送的电子邮件中显示为发件人的名称。例如，你将看到 `@displayname` 作为发件人，例如 yourservername SQL Server Alerting (youremail@domain.com)，而不是看到 user@domain.com 作为发件人。
- `@mailserver` – 你需要用你的邮件服务器地址填写此项，通常是类似 `stmp.domain.com` 的地址。

你将在发送每日报告的作业步骤上使用 HTML 格式。作业的完整脚本在清单 12-4 中。

## ***清单 12-4.*** 用于发送每日审计报告的 SQL Server 代理作业

```sql
DECLARE @profilename varchar(30);
DECLARE @mailrecipients varchar(100);

/* 仅更改这两个变量 */
SET @profilename = '''yourdbmailprofilename''';
SET @mailrecipients = '''youremail@domain.com''';
```



# SQL Server 证书、权限传播与加密

## 权限传播：通过证书创建代理登录

在`DATABASE`中进行权限传播需要先从证书创建代理登录，然后使用同一个证书创建数据库用户。为此，证书在创建后必须备份，并在要创建用户的数据库中还原。一旦数据库用户创建完毕，应用权限的流程就与传播数据库级别权限时相同。

首先，在`master`数据库中创建证书。与之前的示例不同，此证书必须用密码加密，而不是用数据库主密钥加密，以确保其私钥在从数据库中移除时保持加密状态。证书创建完成后，可以按如下方式用它来创建代理登录：

```sql
USE MASTER;
GO

CREATE CERTIFICATE alter_db_certificate
ENCRYPTION BY PASSWORD = 'stR()Ng_PaSSWoRDs are?BeST!'
WITH SUBJECT = 'ALTER DATABASE permission';
GO

CREATE LOGIN alter_db_login FROM CERTIFICATE alter_db_certificate;
GO
```

顾名思义，此登录将用于传播`ALTER DATABASE`权限。下一步是向该登录授予适当的权限：

```sql
GRANT ALTER ANY DATABASE TO alter_db_login;
GO
```

此时，您必须将证书备份到文件。然后可以从文件将证书还原到您选择的数据库中，并从那里创建一个数据库用户，该用户将拥有与服务器登录相同的权限，因为它是使用同一个证书创建的。

```sql
BACKUP CERTIFICATE alter_db_certificate
TO FILE = 'C:\alter_db.cer'
WITH PRIVATE KEY
(
    FILE = 'C:\alter_db.pvk',
    ENCRYPTION BY PASSWORD = 'YeTan0tHeR$tRoNGpaSSWoRd?',
    DECRYPTION BY PASSWORD = 'stR()Ng_PaSSWoRDs are?BeST!'
);
GO
```

备份后，可以将证书还原到任何用户数据库。出于本示例的目的，我们将专门为此创建一个新的数据库：

```sql
CREATE DATABASE alter_db_example;
GO

USE alter_db_example;
GO

CREATE CERTIFICATE alter_db_certificate
FROM FILE = 'C:\alter_db.cer'
WITH PRIVATE KEY
(
    FILE = 'C:\alter_db.pvk',
    DECRYPTION BY PASSWORD = 'YeTan0tHeR$tRoNGpaSSWoRd?',
    ENCRYPTION BY PASSWORD = 'stR()Ng_PaSSWoRDs are?BeST!'
);
GO
```

> **注意** 有关`CREATE CERTIFICATE`语句的更多信息，请参见第 6 章。

值得注意的是，此时证书的物理文件可能应该被删除或备份到安全的存储库。尽管私钥是用密码加密的，但坚定的攻击者肯定有可能通过暴力破解来破解它。而且由于证书被用来授予`ALTER DATABASE`权限，这样的攻击最终可能会造成一些损害——所以安全地处理这些文件。

在数据库中创建证书后，其余过程与之前相同。创建一个需要权限提升的存储过程（在本例中，该存储过程会将数据库设置为`MULTI_USER`访问模式），基于证书创建用户，并使用证书对存储过程进行签名：

```sql
CREATE PROCEDURE SetMultiUser
AS
BEGIN
    ALTER DATABASE alter_db_example
    SET MULTI_USER;
END;
GO

CREATE USER alter_db_user
FOR CERTIFICATE alter_db_certificate;
GO

ADD SIGNATURE TO SetMultiUser
BY CERTIFICATE alter_db_certificate
WITH PASSWORD = 'stR()Ng_PaSSWoRDs are?BeST!';
GO
```

现在可以测试权限了。为了使服务器级别权限的传播有效，执行存储过程的用户必须与有效的服务器登录相关联，并且必须模拟服务器级别的登录，而不是数据库用户。因此，这次`CREATE USER WITHOUT LOGIN`将不起作用：

```sql
CREATE LOGIN test_alter WITH PASSWORD = 'iWanT2ALTER!!';
GO

CREATE USER test_alter FOR LOGIN test_alter;
GO

GRANT EXECUTE ON SetMultiUser TO test_alter;
GO
```

最后，可以模拟`test_alter`登录并执行存储过程：

```sql
EXECUTE AS LOGIN='test_alter';
GO

EXEC SetMultiUser;
GO
```

命令成功完成，表明`test_alter`登录能够行使通过存储过程授予的`ALTER DATABASE`权限。这个例子显然相当简单，但它应该可以作为一个基本模板，在您需要向数据库用户提供服务器级别权限提升时，可以根据需要进行调整。

## 总结

SQL Server 的模拟功能允许开发人员创建安全、细粒度的授权方案。

通过保持授权分层并遵循最小权限原则，可以使数据库中的资源更加安全，要求攻击者做更多工作才能检索他们本不应访问的数据。存储过程层可用于控制安全性，根据高权限代理用户系统委派必要的权限。

当需要将数据库逻辑上分解为范围相似的对象组时，应使用架构。架构还可用于简化权限分配任务，因为随着架构中对象的变化，可能不需要随时间维护权限。

`EXECUTE AS`子句可能是基于存储过程传播权限的一种非常有用且简单的方法，但证书提供了更大的灵活性和控制力。也就是说，您应该尽量保持系统简单易懂，以避免造成可维护性噩梦。

最后一点：在安全方面不要过度。本章介绍的许多技术对于大多数应用程序可能*并非*必要。如果您的应用程序不存储敏感数据，请不要过度创建复杂的权限提升方案；它们只会让您的应用程序更难处理。


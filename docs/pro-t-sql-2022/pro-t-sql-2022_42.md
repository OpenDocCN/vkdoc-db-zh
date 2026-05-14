# 透明数据加密

另一种数据加密选项是**透明数据加密**。它会对磁盘上的整个数据库进行加密，而不仅仅是一个列或列组。由于数据文件被加密，备份文件也随之加密。当数据读入缓冲池时，SQL Server 会对其进行解密，用户无法感知这一过程，因此得名。本节将介绍如何实现透明数据加密。一旦启用，SQL Server 将尝试加密数据。本节还将演示如何暂停加密以及查看数据库加密的当前状态。此外，您将学习如何验证证书备份并备份证书。本节最后将说明如何移除透明数据加密。

在开始实现透明数据加密之前，请制定证书管理计划。这是实现的关键步骤。**如果没有备份证书，当透明数据加密已启用时，您将无法还原数据库备份。** 设置透明数据加密时，首先需要创建主密钥和证书，如代码清单 19-2 所示。

```
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'oUtd@or*R3cre&tion!';
CREATE CERTIFICATE DataEncryption
WITH SUBJECT = 'OutdoorRecreation Certificate';
Listing 19-2
创建主密钥和证书
```

主密钥和证书应在 master 数据库中创建。创建主密钥和证书后，需要创建数据库加密密钥。如代码清单 19-3 所示，请确保在 `OutdoorRecreation` 数据库中创建密钥。

```
USE OutdoorRecreation;
GO
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE DataEncryption;
Listing 19-3
创建数据库加密密钥
```

加密是使用代码清单 19-2 中的服务器证书创建的。加密密钥的算法可以是 AES 128、AES 192 或 AES 256。如果尚未备份加密密钥，您将收到以下警告信息：

> 警告
> 用于加密数据库加密密钥的证书尚未备份。您应立即备份证书及其关联的私钥。如果证书变得不可用，或者您必须在另一台服务器上还原或附加数据库，则必须同时拥有证书和私钥的备份，否则将无法打开数据库。

现在数据库已准备好启用透明数据加密，如代码清单 19-4 所示。

```
ALTER DATABASE OutdoorRecreation
SET ENCRYPTION ON;
Listing 19-4
启用透明数据加密
```

一旦在数据库上启用透明数据加密，后台加密操作将开始运行。

> 注意
> 为任何用户数据库启用透明数据加密，也会在 `tempdb` 上启用透明数据加密。

如果您想暂停此过程，可以运行代码清单 19-5 中的 T-SQL 代码。

```
ALTER DATABASE OutdoorRecreation
SET ENCRYPTION SUSPEND;
Listing 19-5
暂停后台加密操作
```

了解如何管理数据库的加密状态很有帮助。了解如何监控数据库加密状态可能更有用。代码清单 19-6 中的查询可用于查找有关数据库加密状态的信息。

```
SELECT db.[name] AS DatabaseName,
encryptor_type,
encryption_state_desc,
encryption_scan_state_desc
FROM sys.dm_database_encryption_keys ky
INNER JOIN sys.databases db
ON ky.database_id = db.database_id;
Listing 19-6
查找加密状态
```

此查询提供数据库名称、加密类型、加密状态描述和加密扫描描述。当加密进行时，您应得到类似表 19-4 的结果。

**表 19-4**
加密状态（已暂停）

| 数据库名称 | 加密程序类型 | 加密状态描述 | 加密扫描状态描述 |
| --- | --- | --- | --- |
| tempdb | ASYMMETRIC KEY | ENCRYPTED | COMPLETE |
| OutdoorRecreation | CERTIFICATE | ENCRYPTION_IN_PROGRESS | SUSPENDED |

在 `OutdoorRecreation` 数据库的场景中，加密正在进行中。此查询是在我运行了代码清单 19-5 中的代码之后执行的，因此扫描状态为“已暂停”是预期的。

要恢复透明数据加密加密，可以执行代码清单 19-7 中的代码。

```
ALTER DATABASE OutdoorRecreation
SET ENCRYPTION RESUME;
Listing 19-7
恢复数据加密扫描
```

这将允许加密过程在 `OutdoorRecreation` 数据库上继续。表 19-5 显示了在恢复加密扫描后，运行代码清单 19-6 的结果。

**表 19-5**
加密状态（进行中）

| 数据库名称 | 加密程序类型 | 加密状态描述 | 加密扫描状态描述 |
| --- | --- | --- | --- |
| tempdb | ASYMMETRIC KEY | ENCRYPTED | COMPLETE |
| OutdoorRecreation | CERTIFICATE | ENCRYPTION_IN_PROGRESS | RUNNING |

`OutdoorRecreation` 数据库相对较小，因此加密过程不应耗时过长。如果再次执行代码清单 19-6 中的 T-SQL，您将得到表 19-6 中的结果。

**表 19-6**
加密状态（已完成）

| 数据库名称 | 加密程序类型 | 加密状态描述 | 加密扫描状态描述 |
| --- | --- | --- | --- |
| tempdb | ASYMMETRIC KEY | ENCRYPTED | COMPLETE |
| OutdoorRecreation | CERTIFICATE | ENCRYPTED | COMPLETE |

表 19-6 表明数据库 `OutdoorRecreation` 已被加密。既然数据库已启用透明数据加密，请确保将您的证书备份存放在安全位置。

> 注意
> 如果您习惯于使用密码加密证书，请不要对启用透明数据加密的证书这样做。实例重启后，您的数据库将无法访问。

代码清单 19-8 中的查询显示了如何查找证书最近的备份日期。

```
USE master;
GO
SELECT crt.pvt_key_last_backup_date AS LastBackupDate,
DB_NAME(dky.database_id) AS EncryptedDatabase,
crt.[name] AS CertificateName
FROM sys.certificates crt
INNER JOIN sys.dm_database_encryption_keys dky
ON crt.thumbprint = dky.encryptor_thumbprint;
Listing 19-8
检查证书的最后备份日期
```

执行此查询将返回所有有关联数据库的证书的最后备份日期。此查询的结果如表 19-7 所示。

**表 19-7**
证书最后备份日期的查询结果

| 最后备份日期 | 加密数据库 | 证书名称 |
| --- | --- | --- |
| NULL | OutdoorRecreation | DataEncryption |

此表表明 `OutdoorRecreation` 数据库的 `DataEncryption` 证书尚未备份。要备份证书，您可以执行代码清单 19-9 中的 T-SQL。

```
BACKUP CERTIFICATE DataEncryption
TO FILE = N'C:\Certificates\DataEncryption.cer'
WITH PRIVATE KEY (
FILE = N'C:\Certificates\DataEncryption.pvk',
ENCRYPTION BY PASSWORD = 'pR0!5Ql#'
);
Listing 19-9
备份加密证书
```

在此示例中，此证书已备份到 `C` 盘的 `Certificates` 文件夹。


### 证书备份与透明数据加密（TDE）管理

在撰写本书时，您需要指定私钥以更新证书的最后备份日期。

备份证书后，您可以重新运行清单 19-8 中的查询。此查询的结果如表 19-8 所示。

**表 19-8 证书最后备份日期的查询结果**

| 上次备份日期 | 加密数据库 | 证书名称 |
| --- | --- | --- |
| 2023-04-01 | OutdoorRecreation | DataEncryption |

以上结果表明证书最近已备份。

> **注意：**
> 启用 TDE 后，备份文件也会使用数据库加密密钥进行加密。

既然您已经有了备份的证书，就应该将证书备份保存到安全的位置。您还应该确保拥有用于备份证书的密码。如果您需要将使用 TDE 加密的数据库还原到新服务器，这两者都是必需的。例如，如果我想将此数据库还原到新服务器，我需要将证书备份复制到新服务器。在此示例中，我将把 `DataEncryption.cer` 和 `DataEncryption.pvk` 复制到新服务器的 `D:\Certificates` 目录。然后，我可以执行清单 19-10 中的代码，使用先前的证书和加密密钥创建新证书。

```sql
USE master;
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'oUtd@or*R3cre&tion#';
CREATE CERTIFICATE DataEncryption
FROM FILE = N'D:\Certificates\DataEncryption.cer'
WITH PRIVATE KEY (
FILE = N'D:\Certificates\DataEncryption.pvk',
DENCRYPTION BY PASSWORD = 'pR0!5Ql#'
);
```
**清单 19-10 备份加密证书**

创建新的主密钥和证书后，您将能够还原使用 TDE 加密的数据库。

如果您发现需要删除 TDE，首先需要通过执行清单 19-11 中的 T-SQL 代码来禁用加密。

```sql
ALTER DATABASE OutdoorRecreationTDE
SET ENCRYPTION OFF;
```
**清单 19-11 移除 TDE**

这将导致后台解密过程运行，因此建议在维护窗口期间执行此操作。继续之前，您必须通过执行清单 19-8 中的查询来确认解密过程已完成。一旦解密过程完成，您可以使用清单 19-12 中的代码删除数据库加密密钥。

```sql
DROP DATABASE ENCRYPTION KEY;
```
**清单 19-12 删除数据库加密密钥**

删除加密密钥后，TDE 即已从数据库中移除。

本节介绍了透明数据加密（TDE）的用途。您还学习了实施 TDE 所需的步骤。本节还提供了如何暂停或恢复加密/解密过程以及查看后台过程状态的说明。您还学习了如何备份和还原证书，以及查找证书的最近备份日期。最后，本节介绍了如何删除 TDE。



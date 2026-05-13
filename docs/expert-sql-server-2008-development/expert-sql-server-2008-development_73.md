# 第六章：加密

如前所述，默认情况下，非对称密钥对的私钥和证书由 DMK 保护，DMK 会在任何被授予相关安全对象权限的用户需要使用这些密钥时自动将其打开。

以下代码清单展示了如何使用 RSA 算法创建一个 1024 位的非对称密钥：

```sql
CREATE ASYMMETRIC KEY AsymKey1
WITH Algorithm = RSA_1024;
GO
```

使用非对称密钥加密数据的过程与对称加密略有不同，因为在加密或解密之前无需显式打开非对称密钥。

相反，你需要将非对称密钥的 ID 和要加密的明文传递给 `ENCRYPTBYASYMKEY` 方法，如下列代码清单所示：

```sql
DECLARE @Secret nvarchar(255) = 'This is my secret message';
DECLARE @Encrypted varbinary(max);
SET @Encrypted = ENCRYPTBYASYMKEY(ASYMKEY_ID(N'AsymKey1'), @Secret);
GO
```

`DECRYPTBYASYMKEY`（以及等效的 `DECRYPTBYCERT`）函数可用于返回任何非对称加密数据的 `varbinary` 表示形式，通过提供密钥 ID 和加密后的密文，如下所示：

```sql
DECLARE @Secret nvarchar(255) = 'This is my secret message';
DECLARE @Encrypted varbinary(max);
SET @Encrypted = ENCRYPTBYASYMKEY(ASYMKEY_ID(N'AsymKey1'), @secret);
SELECT
CAST(DECRYPTBYASYMKEY(ASYMKEY_ID(N'AsymKey1'), @Encrypted) AS nvarchar(255));
GO
```

你可能在运行前面的代码示例时已经注意到，即使处理的数据量非常小，非对称加密和解密方法也比等效的对称函数慢得多。这就是为什么，尽管非对称加密比对称加密提供更强的保护，但不建议将其用于加密任何将在查询中用于筛选行的列，或用于一次需要解密多于一两条记录的情况。相反，非对称加密最有用的方法是作为保护其他加密密钥的手段，本章稍后将对此进行讨论。

### 透明数据加密

透明数据加密（TDE）是 SQL Server 2008 开发版和企业版中提供的一项新功能。它是“透明”的，因为与之前列出的所有方法都不同，它不需要显式更改数据库架构，也不需要使用特定的方法重写查询来加密或解密数据。此外，一般来说，不需要处理与 TDE 相关的任何密钥管理问题。那么 TDE 是如何工作的呢？

透明加密不是操作选定的值或数据列，而是对从文件系统层传递到 SQL Server 进程的 *所有* 数据提供自动的对称加密和解密。换句话说，保存数据库信息的 MDF 和 LDF 文件以加密状态存储。当 SQL Server 请求这些文件中的数据时，它会在 I/O 层自动解密。因此，查询引擎可以像处理任何其他类型的数据一样对其进行操作。当数据写回磁盘时，TDE 会再次自动将其加密。

TDE 基于存储在用户数据库中的 `数据库加密密钥`（DEK），为启用此功能的任何数据库中的（几乎所有）所有数据提供加密。DEK 是一个对称密钥，由存储在 master 数据库中的服务器证书保护，而该证书又由 master 数据库的 DMK 和 SMK 保护。

要在数据库上启用 TDE，首先需要在 master 数据库中创建一个服务器证书（假设 master 数据库已有一个 DMK）：

```sql
USE MASTER;
GO
CREATE CERTIFICATE TDE_Cert
WITH SUBJECT = 'Certificate for TDE Encryption';
GO
```

然后，在相应的用户数据库中使用 `CREATE DATABASE ENCRYPTION KEY` 语句创建 DEK：

```sql
USE ExpertSqlEncryption;
GO
CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_128
```



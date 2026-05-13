# 第 6 章：加密

`ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;`

`GO`

**注意** 上面的代码清单将生成一个警告信息，建议您备份用于加密 DEK 的服务器证书。在生产环境中，我建议您仔细阅读此信息并遵循其建议。如果证书变得不可用，您将无法解密 DEK，并且关联数据库中的所有数据都将丢失。

创建必要的密钥后，可以使用单条 T-SQL 语句启用 TDE，如下所示：
```sql
ALTER DATABASE ExpertSqlEncryption
SET ENCRYPTION ON;
GO
```

要检查服务器上所有数据库的加密状态，可以查询 `sys.dm_database_encryption_keys` 视图，如下所示：
```sql
SELECT
DB_NAME(database_id) AS database_name,
CASE encryption_state
WHEN 0 THEN 'Unencrypted (No database encryption key present)'
WHEN 1 THEN 'Unencrypted'
WHEN 2 THEN 'Encryption in Progress'
WHEN 3 THEN 'Encrypted'
WHEN 4 THEN 'Key Change in Progress'
WHEN 5 THEN 'Decryption in Progress'
END AS encryption_state,
key_algorithm,
key_length
FROM sys.dm_database_encryption_keys;
GO
```

结果显示，用户数据库 `ExpertSqlEncryption` 和系统数据库 `tempdb` 现在都已加密：

| database_name | encryption_state | key_algorithm | key_length |
|---------------|------------------|---------------|------------|
| tempdb | 已加密 | AES | 256 |
| ExpertSqlEncryption | 已加密 | AES | 128 |

这展示了一个重要观点：由于 `tempdb` 是所有用户数据库共享的系统资源，如果 SQL Server 实例上的**任何**用户数据库使用了 TDE 加密，那么 `tempdb` 数据库也必须被加密，如此案例所示。

禁用 TDE 的过程与启用它一样简单：
```sql
ALTER DATABASE ExpertSqlEncryption
SET ENCRYPTION OFF;
GO
```

请注意，即使在所有用户数据库上关闭了 TDE，`tempdb` 数据库仍将保持加密状态，直到它被重新创建，例如在服务器下次重启时。

**注意** 有关 `sys.dm_database_encryption_keys` 中可用列的更多信息，请参阅 http://msdn.microsoft.com/en-us/library/bb677274.aspx。

TDE 的优点如下：

-   静态数据被加密（嗯，并非**所有**静态数据；参见下一节）。
    -   如果您的主要目标是防止数据物理被盗，TDE 确保您的数据库文件（包括事务日志和备份）无法在没有必要 DKM 的情况下被盗并还原到另一台机器上。
-   TDE 可以应用于现有数据库，无需任何应用程序重新编码。因此，在无法重新设计现有生产应用程序以合并单元级加密的情况下，可以应用 TDE。
-   在 I/O 级别执行加密和解密是高效的，对数据库性能的整体影响很小。Microsoft 估计启用 TDE 导致的平均性能下降在 3%到 5%的范围内。

TDE 无疑是对 SQL Server 提供的单元级加密方法的一个有用补充，如果使用得当，可以加强数据保护。然而，认为仅仅通过启用 TDE 就能保护数据将是一个严重的错误。尽管表面上 TDE 简化了加密过程，但在实践中，它隐藏了确保真正安全解决方案所必需的复杂性。

几乎没有业务需求是加密整个数据库中的每一项数据。然而，由于启用 TDE 非常容易，这意味着在某些情况下，开启 TDE 比对每个数据元素进行适当分析是更简单的选择。这种行为表明缺乏针对安全加密策略的规划，并且不太可能提供高安全性应用程序所需的保护级别。

TDE 与先前讨论的单元级方法之间还存在一些重要区别：

-   TDE 仅保护静态数据。当 SQL Server 请求数据时，它是



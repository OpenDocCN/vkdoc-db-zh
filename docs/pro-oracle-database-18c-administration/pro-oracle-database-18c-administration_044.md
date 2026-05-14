# 配置加密

您可能被要求加密备份。某些场所尤其需要对包含敏感数据并且存储在异地的备份进行加密。要在备份时使用加密，您必须使用 Oracle 企业版并拥有高级安全选项的许可。

如果您已配置了安全钱包（详情请参阅可从 Oracle 网站技术网络区域（[`http://otn.oracle.com`](http://otn.oracle.com)）免费下载的《Oracle 高级安全管理员指南》），您可以为备份配置透明加密，如下所示：

```
RMAN> configure encryption for database on;
```

现在您进行的任何备份都将被加密。如果您需要从备份中恢复，它会自动解密（假设与您加密备份时相同的安全钱包仍在使用）。要禁用加密，请使用 `CLEAR` 命令：

```
RMAN> configure encryption for database off;
```

加密的表空间在备份集中将保持加密状态。备份加密的配置不需要开启，只有那些被加密的表空间才会保持加密。

您也可以使用 `CLEAR` 清除加密设置：

```
RMAN> configure encryption for database clear;
```

您可以查询 `V$RMAN_ENCRYPTION_ALGORITHMS` 来查看您数据库版本可用的加密算法详情。


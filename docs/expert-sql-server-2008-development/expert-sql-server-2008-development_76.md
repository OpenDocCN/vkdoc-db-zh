# 第 6 章：加密

## 创建证书与共享对称密钥

接下来，为每个用户创建证书，每个证书使用各自独立的密码：

```sql
CREATE CERTIFICATE FinanceCertificate
AUTHORIZATION FinanceUser
ENCRYPTION BY PASSWORD = '#F1n4nc3_P455w()rD#'
WITH SUBJECT = 'Certificate for Finance',
EXPIRY_DATE = '20101031';

CREATE CERTIFICATE MarketingCertificate
AUTHORIZATION MarketingUser
ENCRYPTION BY PASSWORD = '-+M@Rket1ng-P@s5w0rD!+-'
WITH SUBJECT = 'Certificate for Marketing',
EXPIRY_DATE = '20101105';
GO
```

我们还将创建一个示例表，并授予两个用户对该表的选择与插入权限：

```sql
CREATE TABLE Confidential (
    EncryptedData varbinary(255)
);
GO

GRANT SELECT, INSERT ON Confidential TO FinanceUser, MarketingUser;
GO
```

现在基础结构已设置好，我们将创建一个共享对称密钥，该密钥将由财务和营销两个证书加密保护。但是，不能直接以此方式创建密钥。我们将首先通过第一个证书保护来创建密钥，然后打开并修改该密钥以添加第二个证书的加密保护：

```sql
-- 创建受第一个证书保护的对称密钥
CREATE SYMMETRIC KEY SharedSymKey
WITH ALGORITHM = AES_256
ENCRYPTION BY CERTIFICATE FinanceCertificate;
GO

-- 然后打开并修改密钥以添加第二个证书的加密保护
OPEN SYMMETRIC KEY SharedSymKey
DECRYPTION BY CERTIFICATE FinanceCertificate
WITH PASSWORD = '#F1n4nc3_P455w()rD#';

ALTER SYMMETRIC KEY SharedSymKey
ADD ENCRYPTION BY CERTIFICATE MarketingCertificate;

CLOSE SYMMETRIC KEY SharedSymKey;
GO
```

最后，我们需要为每个用户授予对该对称密钥的权限：

```sql
GRANT VIEW DEFINITION ON SYMMETRIC KEY::SharedSymKey TO FinanceUser;
GRANT VIEW DEFINITION ON SYMMETRIC KEY::SharedSymKey TO MarketingUser;
GO
```

## 加密与解密数据

要使用共享对称密钥加密数据，任一用户必须首先通过指定保护该密钥的证书名称和密码来打开密钥。然后，他们可以使用`ENCRYPTBYKEY`函数，如本章前面所示。以下代码清单展示了`FinanceUser`如何将加密数据插入`Confidential`表，使用共享对称密钥进行加密：

```sql
EXECUTE AS USER = 'FinanceUser';

OPEN SYMMETRIC KEY SharedSymKey
DECRYPTION BY CERTIFICATE FinanceCertificate
WITH PASSWORD = '#F1n4nc3_P455w()rD#';

INSERT INTO Confidential
SELECT ENCRYPTBYKEY(KEY_GUID(N'SharedSymKey'), N'This is shared information
accessible to finance and marketing');

CLOSE SYMMETRIC KEY SharedSymKey;

REVERT;
GO
```

要解密此数据，营销用户或财务用户都可以在解密前显式打开`SharedSymKey`密钥，类似于加密时使用的模式，或者他们可以利用`DECRYPTBYKEYAUTOCERT`函数，该函数允许你在执行解密的内联函数中自动打开受证书保护的对称密钥。

为了演示`DECRYPTBYKEYAUTOCERT`函数，以下代码清单展示了`MarketingUser`如何解密由`FinanceUser`插入到`Confidential`表中的值，使用他们自己的证书名称和保护对称密钥的密码来打开共享对称密钥：

```sql
EXECUTE AS USER = 'MarketingUser';

SELECT
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID(N'MarketingCertificate'),
            N'-+M@Rket1ng-P@s5w0rD!+-',
            EncryptedData)
        AS nvarchar(255))
FROM Confidential;

REVERT;
GO
```

结果显示，`MarketingUser`可以使用`MarketingCertificate`访问由`FinanceUser`使用共享对称密钥加密的数据：

    This is shared information accessible to finance and marketing

## 扩展混合模型

为了进一步扩展此示例，假设除了存储财务和营销之间共享的加密数据外，`Confidential`表还保存着只应授予财务用户访问权限的数据。使用此处描述的混合模型，这很容易实现——只需创建一个新的对称密钥，由现有的财务证书保护，并授予财务用户对该密钥的权限。

```sql
CREATE SYMMETRIC KEY FinanceSymKey
WITH ALGORITHM = AES_256
ENCRYPTION BY CERTIFICATE FinanceCertificate;
GO

GRANT VIEW DEFINITION ON SYMMETRIC KEY::FinanceSymKey TO FinanceUser;
GO
```

由于这个新的对称密钥使用现有的`FinanceCertificate`进行保护，财务用户可以使用与共享密钥完全相同的语法打开它，并通过向`ENCRYPTBYKEY`方法指定适当的`KEY_GUID`来使用该密钥加密数据：

```sql
EXECUTE AS USER = 'FinanceUser';

OPEN SYMMETRIC KEY FinanceSymKey
DECRYPTION BY CERTIFICATE FinanceCertificate
WITH PASSWORD = '#F1n4nc3_P455w()rD#';

INSERT INTO Confidential
SELECT ENCRYPTBYKEY(
    KEY_GUID(N'FinanceSymKey'),
    N'This information is only accessible to finance');

CLOSE SYMMETRIC KEY FinanceSymKey;

REVERT;
GO
```

`Confidential`表现在包含两行数据：一行使用共享对称密钥`SharedSymKey`加密，`MarketingUser`和`FinanceUser`都可以访问；第二行使用`FinanceSymKey`加密，只有`FinanceUser`可以访问。

这种方法的妙处在于，由于某个用户有权访问的所有密钥都由单一证书保护，因此可以使用`DECRYPTBYKEYAUTOCERT`方法解密整列数据，无论单个值是用哪个密钥加密的。

例如，以下代码清单演示了`FinanceUser`如何通过结合使用`DECRYPTBYKEYAUTOCERT`和`FinanceCertificate`来自动解密`EncryptedData`列中的所有值：

```sql
EXECUTE AS USER = 'FinanceUser';

SELECT
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID(N'FinanceCertificate'),
            N'#F1n4nc3_P455w()rD#',
            EncryptedData
        ) AS nvarchar(255))
FROM Confidential;

REVERT;
GO
```

结果显示，`FinanceUser`可以使用`DECRYPTBYKEYAUTOCERT`解密由`FinanceSymKey`和`SharedSymKey`加密的值，因为它们都受同一个`FinanceCertificate`保护：

    This is shared information accessible to finance and marketing
    This information is only accessible to finance

如果`MarketingUser`尝试使用`MarketingCertificate`执行完全相同的查询，他们将只能解密那些使用`SharedSymKey`加密的值；尝试解密使用`FinanceSymKey`加密的值的结果将是`NULL`：

```sql
EXECUTE AS USER = 'MarketingUser';

SELECT
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID(N'MarketingCertificate'),
            N'-+M@Rket1ng-P@s5w0rD!+-',
            EncryptedData) AS nvarchar(255))
FROM Confidential;

REVERT;
GO
```

    This is shared information accessible to finance and marketing
    NULL

我希望这已经展示了加密混合模型如何在大多数需要加密的场景中实现安全性与可维护性之间的平衡，以及如何扩展它以应对各种不同的使用场景。

在下一节中，我将向您展示如何针对以此模型保存的加密数据编写高效查询。

## 加密对查询设计的影响

讨论了不同加密架构决策的相对优劣之后，本章剩余部分将重点介绍优化处理加密数据应用程序性能的方法。

加密的安全性总是伴随着相关的性能成本。如前所述，加密数据的读写操作资源消耗更大，通常需要更长时间执行。然而，加密数据的特殊性质带来的影响范围超出了简单的性能下降，需要特别注意任何涉及对加密数据进行排序、筛选或连接的操作。



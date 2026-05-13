# 第 6 章 „ 加密

DECLARE `@salt` varbinary(255);

SET `@salt` = (

SELECT `DECRYPTBYASYMKEY`(

`ASYMKEY_ID`('HMACSubStringASymKey'),

`HMACKey`,

N'~Y3T_an0+h3r_5tR()ng_K4y~'

)

FROM `HMACKeys`

WHERE `HMACKeyID` = 2);

-- 使用盐值更新 Last4HMAC 值

UPDATE `CreditCards`

SET `CreditCardNumber_Last4HMAC` = (SELECT `dbo.GenerateHMAC`(

'SHA256',

CAST(RIGHT(`CreditCardNumber_Plain`, 4) AS varbinary(max)), `@salt`));

GO

并且，为了支持针对新子串 HMAC 的查询，我们将添加一个新的索引：

```sql
CREATE NONCLUSTERED INDEX idxCreditCardNumberLast4HMAC
ON CreditCards (CreditCardNumber_Last4HMAC)
INCLUDE (CreditCardNumber_Sym);
```

GO

假设我们想要查询所有以数字 `0019` 结尾的信用卡。这种模式匹配查询现在可以通过比较这些数字的 HMAC 与存储在 `CreditCardNumber_Last4HMAC` 列中的 HMAC 值来发出，如下所示：

```sql
-- 选择要搜索的信用卡后 4 位
DECLARE @CreditCardLast4ToSearch nchar(4) = '0019';

-- 检索秘密盐值
DECLARE @salt varbinary(255);
SET @salt = (
    SELECT DECRYPTBYASYMKEY(
        ASYMKEY_ID('HMACSubStringASymKey'),
        HMACKey,
        N'~Y3T_an0+h3r_5tR()ng_K4y~'
    )
    FROM HMACKeys
    WHERE HMACKeyID = 2
);

-- 生成要搜索的后 4 位的 HMAC
DECLARE @HMACToSearch varbinary(255);
SET @HMACToSearch = dbo.GenerateHMAC(
    'SHA256',
    CAST(@CreditCardLast4ToSearch AS varbinary(max)),
    @salt
);

-- 从 CreditCards 表中检索匹配的行
SELECT
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID('CreditCard_Cert'),
            N'#Ch0o53_@_5Tr0nG_P455w0rD#',
            CreditCardNumber_Sym
        )
        AS nvarchar(32)
    ) AS CreditCardNumber_Decrypted
FROM
    CreditCards
WHERE
    CreditCardNumber_Last4HMAC = @HMACToSearch
    AND
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID('CreditCard_Cert'),
            N'#Ch0o53_@_5Tr0nG_P455w0rD#',
            CreditCardNumber_Sym
        )
        AS nvarchar(32)
    ) LIKE '%' + @CreditCardLast4ToSearch;
```

GO

在我的系统上获得的结果列出了以下十张信用卡，包括我们最初搜索的信用卡号码 `4005550000000019`（你的列表会有所不同，因为 `CreditCards` 表中的记录是随机生成的，但你应该仍然能找到预期的匹配值）：

对如图 6-9 所示的查询执行计划的检查表明，只要所选的子串具有足够的选择性（如此例中所示），则可以使用高效的索引查找（index seek）来实现对加密数据的通配符搜索。

**图 6-9. HMAC 子串的索引查找**

在我的机器上，此查询的性能与针对 `CreditCard_HMAC` 列的完全相等匹配几乎相同，耗时 `301ms` 的 CPU 时间，需要九次逻辑读取。

虽然此技术展示了可以对加密数据执行通配符搜索，但它不是一个非常灵活的解决方案。本例中使用的 `CreditCardNumber_Last4HMAC` 列只能用于满足指定信用卡最后四位的查询。如果你想搜索信用卡号码的最后六位，例如，你将需要创建一个全新的 HMAC 列来支持该类型的查询。

你的应用程序在搜索条件上要求的灵活性越高，此解决方案就变得越不可行，并且必须创建的附加数据列就越多。这不仅会成为维护难题，还意味着更大的存储需求，并增加了数据暴露的风险。即使在存储相对安全的 HMAC 哈希值时，每个附加列都会为黑客提供更多关于你数据的信息，这些信息可能被用于发动定向攻击。

### 范围搜索

本章我将考虑的最后一个搜索加密数据的场景是范围查询。例如，我们如何识别信用卡号码范围在 `4929100012347890` 和 `4999100012349999` 之间的那些数据行？

这些类型的搜索可能是最难用加密数据实现的，并且没有明显的解决方案：HMAC 解决方案或其子串衍生物在这里没有帮助——顺序的明文数字显然不会导致顺序的密文值。事实上，当你处理单元级加密时，无法避免扫描并解密表中的每一行来完成范围查询。

如果要搜索的范围非常有限（例如，所提供值的 +/-10），则可能可以放弃加密，并在新列中存储部分明文值，例如信用卡的最后两位。通过一些查询重设计，查询可以被设计为基于此列促进某些范围查询，但这显然涉及巨大的风险，因为它甚至披露了敏感信息的部分明文。

如果必须支持范围查询，唯一其他的替代方法是依赖在 I/O 级别执行的加密形式之一，例如使用 `TDE`、`EFS` 或 `Windows BitLocker` 的数据库加密。在这种情况下，解密是在非常低的级别执行的，以至于数据库引擎可以在不考虑数据在处理前可能已受到的任何加密的情况下操作数据。

## 总结

加密构成了任何安全策略的重要组成部分，并且可以在阻止攻击者获取敏感或机密数据方面提供关键的最后一道防线。然而，它是有代价的——加密需要精心规划的策略和安全政策，并且几乎总是需要在数据库或应用程序级别进行设计更改才能成功实施。增加的安全性也必然会对性能产生负面影响。

由非确定性算法产生的加密数据不具备与正常数据相同的结构或模式。这意味着许多标准的数据库任务，如排序、过滤和连接，必须重新设计才能有效工作。尽管有方法可以绕过处理加密数据的一些限制，例如使用 HMAC 进行搜索和匹配，但它们并非在所有情况下都令人满意，如果可能，应设计应用程序以避免直接查询加密数据。


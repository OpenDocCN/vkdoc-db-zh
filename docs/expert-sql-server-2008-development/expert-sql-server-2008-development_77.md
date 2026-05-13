# 第 6 章 加密

索引、排序和筛选数据依赖于识别数据内部的有序模式。大多数数据可以被赋予逻辑顺序：例如，`varchar` 或 `char` 数据可以按字母顺序排列；`datetime` 数据可以按时间顺序排列；而 `int`、`decimal`、`float` 和 `money` 数据可以按数字顺序排列。然而，加密的根本目的正是要消除任何可能暴露底层数据信息的逻辑模式，这就使得为加密数据分配顺序变得非常棘手。这给高效的查询设计带来了一些问题。

为了演示这些问题，让我们创建一个包含一些示例机密数据的表。下面的代码清单将创建一个表并填充 100,000 行随机生成的 16 位数字，代表虚拟信用卡号。

在随机数据中，我们还将插入一个预定值，代表信用卡号 `4005-5500-0000-0019`。我们将使用这个已知值来测试在加密数据中搜索值的各种方法。

```sql
CREATE TABLE CreditCards (
    CreditCardID int IDENTITY(1,1) NOT NULL,
    CreditCardNumber_Plain nvarchar(32)
);
GO

WITH RandomCreditCards AS (
    SELECT
        CAST(9E+15 * RAND(CHECKSUM(NEWID())) + 1E+15 AS bigint) AS CardNumber
)
INSERT INTO CreditCards (CreditCardNumber_Plain)
SELECT TOP 100000
    CardNumber
FROM
    RandomCreditCards,
    MASTER..spt_values a,
    MASTER..spt_values b
UNION ALL SELECT
    '4005550000000019' AS CardNumber;
GO
```

**注意** 在您冲出去疯狂购物之前，我很遗憾地告诉您，本示例中列出的信用卡号 `4005-5500-0000-0019` 是 VISA 用于测试支付处理系统的测试卡号，不能用于购买商品。当然，那 100,000 行随机生成的数据*可能*碰巧包含有效的信用卡详细信息，但祝您好运能找到它们！

为了保护这些信息，我们将遵循前一节描述的混合加密模型。为此，首先需要创建一个由密码加密的证书：

```sql
CREATE CERTIFICATE CreditCard_Cert
ENCRYPTION BY PASSWORD = '#Ch0o53_@_5Tr0nG_P455w0rD#'
WITH SUBJECT = 'Secure Certificate for Credit Card Information',
EXPIRY_DATE = '20101031';
GO
```

然后创建一个由该证书保护的对称密钥：

```sql
CREATE SYMMETRIC KEY CreditCard_SymKey
WITH ALGORITHM = AES_256
ENCRYPTION BY CERTIFICATE CreditCard_Cert;
GO
```

由于我们只关心测试不同方法的性能，这次我们不必费心创建具有密钥访问权限的不同用户——我们将以 `dbo` 用户身份完成所有操作。

首先，向 `CreditCards` 表添加一个新列 `CreditCardNumber_Sym`，并使用 `CreditCard_SymKey` 对称密钥加密的值来填充它：

```sql
ALTER TABLE CreditCards ADD CreditCardNumber_Sym varbinary(100);
GO

OPEN SYMMETRIC KEY CreditCard_SymKey
DECRYPTION BY CERTIFICATE CreditCard_Cert
WITH PASSWORD = '#Ch0o53_@_5Tr0nG_P455w0rD#';

UPDATE CreditCards
SET CreditCardNumber_Sym =
    ENCRYPTBYKEY(KEY_GUID('CreditCard_SymKey'),CreditCardNumber_Plain);

CLOSE SYMMETRIC KEY CreditCard_SymKey;
GO
```

现在，假设我们正在设计一个需要搜索 `CreditCardNumber_Sym` 列中已加密的特定信用卡号的应用程序。首先，让我们创建一个索引来支持搜索：

```sql
CREATE NONCLUSTERED INDEX idxCreditCardNumber_Sym
ON CreditCards (CreditCardNumber_Sym);
GO
```

让我们尝试对加密数据执行一个简单的搜索。我们将通过解密 `CreditCardNumber_Sym` 列并将结果与搜索字符串进行比较来搜索我们选定的信用卡号：

```sql
DECLARE @CreditCardNumberToSearch nvarchar(32) = '4005550000000019';

SELECT * FROM CreditCards
WHERE DECRYPTBYKEYAUTOCERT(
    CERT_ID('CreditCard_Cert'),
    N'#Ch0o53_@_5Tr0nG_P455w0rD#',
    CreditCardNumber_Sym) = @CreditCardNumberToSearch;
GO
```

快速浏览一下*图 6-7* 所示的执行计划就会发现，尽管存在索引，但查询必须扫描整个表，解密每一行以查看它是否满足谓词条件。

*图 6-7. 在加密数据上执行的查询扫描*

需要全表扫描的原因在于加密数据的非确定性本质。由于 `CreditCard_Sym` 列中的值是加密的，我们无法在不解密每个值的情况下判断哪些行匹配搜索条件。我们也不能简单地加密搜索值 `4005550000000019` 并在 `CreditCard_Sym` 列中搜索相应的加密值，因为结果是非确定性的，每次都会不同。

即使我们使用的是对称加密（更快的单元格加密和解密方法），但需要进行整个表扫描意味着该查询的性能不佳。我们可以通过检查 `sys.dm_exec_query_stats` 来快速评估整体性能，如下所示：

```sql
SELECT
    st.text,
    CAST(qs.total_worker_time AS decimal(18,9)) / qs.execution_count / 1000
        AS Avg_CPU_Time_ms,
    qs.total_logical_reads
FROM
    sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.plan_handle) st
WHERE
    st.text LIKE '%CreditCardNumberToSearch%';
```

我得到的结果如下：

```
text                                        Avg_CPU_Time_ms  total_logical_reads
DECLARE @CreditCardNumberToSearch nvarchar(32)
    = '4005550000000019'; --*SELECT------------- 1389.079         71623
```

该 `SELECT` 查询仅查询 100,000 行加密数据就花费了近 1.5 秒的 CPU 时间。另请注意，`sys.dm_exec_sql_text` 的 `text` 列中包含的查询文本被清空了，所有引用加密对象的查询都是如此。

解决加密数据搜索问题的适当解决方案因数据的查询方式而异——无论是搜索与搜索字符串完全匹配的单行（相等匹配）、使用通配符的模式匹配（即 `LIKE %`），还是范围搜索（即使用 `BETWEEN`、`>` 或 `<` 运算符）。

## 使用哈希消息认证码进行相等匹配

一种促进高效搜索和过滤加密数据单行的方法是利用哈希消息认证码，它是由哈希函数创建的，如本章前面所述。请记住，哈希函数的输出是确定性的——任何给定的输入总是导致相同的输出，这是允许对加密数据高效搜索任何给定搜索值的关键属性。存储在数据库中的二进制哈希值可以被索引，并且可以通过创建搜索条件的哈希值并在包含预计算哈希值的列中搜索该值来执行数据过滤。

但是，我们不想存储由 `HASHBYTES` 返回的每个信用卡号的简单哈希值——如前所述，哈希值可以使用字典或彩虹表进行攻击，特别是当它们（如信用卡号的情况）遵循预定格式时。这将削弱使用对称加密实现的现有解决方案的强度。

HMAC 设计用于验证消息的真实性，它通过将哈希函数与密钥（盐值）结合来克服这个问题。通过在哈希前将明文与强盐值组合，得到的哈希值几乎可以免疫字典攻击，因为任何查找都必须将每个可能的源值与每个可能的盐值组合，从而产生一个黑客必须与之比较的、不可行的庞杂可能哈希值列表。

**消息认证码**

HMAC 是一种验证消息真实性以证明其在传输过程中未被篡改的方法。其工作原理如下：

1.  发送方和接收方约定一个只有他们知道的秘密值。例如，假设他们约定单词 `salt`。



## 第 6 章 „ 加密

2.  发送方创建消息时，他们会将秘密值附加到消息末尾。然后，他们会对组合了盐值的消息进行哈希运算：
    `HMAC = HASH( “This is the original message” + “salt”)`
3.  发送方将原始消息和 `HMAC` 一起传输给接收方。
4.  接收方收到消息后，会将约定的盐值附加到收到的消息上，然后自行对结果进行哈希运算。接着，他们会检查结果是否与提供的 `HMAC` 值相同。

`HMAC` 是一种相对简单但非常有效的方法，用于确保目标接收方可以验证消息的真实性。如果有人试图拦截并篡改消息，它将与提供的 `HMAC` 值不匹配。攻击者也无法生成新的匹配 `HMAC` 值，因为他们不知道需要应用的秘密盐值。

实施基于 `HMAC` 的解决方案的第一步，是定义一种安全存储盐值（或 `密钥`）的方法。在本例中，我们将盐值存储在一个单独的表中，并使用安全的非对称密钥对其进行加密。

```sql
CREATE ASYMMETRIC KEY HMACASymKey
WITH ALGORITHM = RSA_1024
ENCRYPTION BY PASSWORD = N'4n0th3r_5tr0ng_K4y!';
GO

CREATE TABLE HMACKeys (
    HMACKeyID int PRIMARY KEY,
    HMACKey varbinary(255)
);
GO

INSERT INTO HMACKeys
SELECT
    1,
    ENCRYPTBYASYMKEY(ASYMKEY_ID(N'HMACASymKey'), N'-->Th15_i5_Th3_HMAC_kEy!');
GO
```

现在我们可以基于密钥创建一个 `HMAC` 值。你可以通过在 `T-SQL` 中简单地使用 `HASHBYTES` 并将盐值附加到原始消息后面来创建一个简单的 `HMAC`，如下所示：

```sql
DECLARE @HMAC varbinary(max) = HASHBYTES('SHA1', 'PlainText' + 'Salt')
```

然而，这种方法可以通过多种方式得到改进：

*   `HASHBYTES` 函数仅基于任何输入的前 8000 字节创建哈希值。如果消息长度等于或超过此值，则附加在消息内容后的任何盐值都将被忽略。因此，最好先提供盐值，*然后*再提供要哈希的消息。
*   根据联邦信息处理标准 (`FIPS PUB 198`, `http://csrc.nist.gov/publications/fips/fips198/fips-198a.pdf`) 定义的 `HMAC` 规范，实际上对明文进行了两次哈希运算，如下所示：
    `HMAC = HASH(Key1 + HASH(Key2 + PlainText))`
    根据 `FIPS` 标准，`Key1` 和 `Key2` 是通过将提供的盐值填充到哈希函数密钥的块大小来计算的。`Key1` 用字节 `0x5c` 填充，`Key2` 用字节 `0x36` 填充。这两个值特意选择具有较大的 `汉明距离`——也就是说，生成的填充密钥将彼此显著不同，从而最大化使用两个密钥进行哈希的增强效果。
*   `HMAC` 标准并未指定使用哪种哈希算法来执行哈希运算，但最终生成的 `HMAC` 的强度与其所基于的底层哈希函数的加密强度直接相关。`HASHBYTES` 函数仅支持使用 `MD2`、`MD4`、`MD5`、`SHA` 和 `SHA1` 算法进行哈希。相比之下，`.NET Framework` 提供的哈希方法则支持更安全的 `SHA2` 系列算法，以及 160 位的 `RIPEMD160` 算法。

看来 `SQL Server` 的基础 `HASHBYTES` 函数还有待改进，但幸运的是有一个简单的解决方案。为了创建一个符合 `FIPS` 标准、基于强哈希算法的 `HMAC`，我们可以采用一个 `CLR` 用户定义函数，该函数使用 `.NET` `System.Security` 命名空间提供的方法。以下代码清单说明了创建一个可重用类所需的 `C#` 代码，该类可用于为所有支持的哈希算法和给定密钥生成 `HMAC` 值：

```csharp
[SqlFunction(IsDeterministic = true, DataAccess = DataAccessKind.None)]
public static SqlBytes GenerateHMAC
(
    SqlString Algorithm,
    SqlBytes PlainText,
    SqlBytes Key
)
{
```



## 第 6 章 - 加密

```csharp
if (Algorithm.IsNull || PlainText.IsNull || Key.IsNull) {
    return SqlBytes.Null;
}

HMAC HMac = null;

switch (Algorithm.Value) {
    case "MD5":
        HMac = new HMACMD5(Key.Value);
        break;
    case "SHA1":
        HMac = new HMACSHA1(Key.Value);
        break;
    case "SHA256":
        HMac = new HMACSHA256(Key.Value);
        break;
    case "SHA384":
        HMac = new HMACSHA384(Key.Value);
        break;
    case "SHA512":
        HMac = new HMACSHA512(Key.Value);
        break;
    case "RIPEMD160":
        HMac = new HMACRIPEMD160(Key.Value);
        break;
    default:
        throw new Exception("Hash algorithm not recognised");
}

byte[] HMacBytes = HMac.ComputeHash(PlainText.Value);
return new SqlBytes(HMacBytes);
```

**注意**：为求完整，`GenerateHMAC`函数支持`Security.Cryptography`命名空间内所有可用的哈希算法。实际上，不建议在生产应用中使用`MD5`算法，因为它已被证明易受哈希碰撞攻击。

生成程序集并在 SQL Server 中注册。然后，在`CreditCards`表中创建一个新列，并使用`GenerateHMAC`函数填充`HMAC-SHA1`值。我将`HMAC`列创建为`varbinary(255)`类型——存储在此列中的`HMAC`值的实际长度将取决于所使用的哈希算法：`MD5`为 128 位（16 字节）；`RIPEMD160`和`SHA1`为 160 位（20 字节）；`SHA512`最长可达 64 字节。以下代码清单演示了这些步骤：

```sql
ALTER TABLE CreditCards
ADD CreditCardNumber_HMAC varbinary(255);
GO

-- 从 MACKeys 表中检索 HMAC 盐值
DECLARE @salt varbinary(255);
SET @salt = (
    SELECT DECRYPTBYASYMKEY(
        ASYMKEY_ID('HMACASymKey'),
        HMACKey,
        N'4n0th3r_5tr0ng_K4y!'
    )
    FROM HMACKeys
    WHERE HMACKeyID = 1
);

-- 使用盐值更新 HMAC 值
UPDATE CreditCards
SET CreditCardNumber_HMAC = (
    SELECT dbo.GenerateHMAC(
        'SHA256',
        CAST(CreditCardNumber_Plain AS varbinary(max)),
        @salt
    )
);
GO
```

**注意**：如果您在生产应用中实现此处提出的`HMAC`解决方案，您需要创建触发器以确保在对`CreditCards`表进行`INSERT`和`UPDATE`操作时，`CreditCardNumber_HMAC`列的完整性得以维护。由于我仅专注于评估`HMAC`的性能，因此不会进行此步骤。

我们现在可以发出基于`HMAC`搜索信用卡的查询；但在执行之前，让我们创建一个索引来帮助支持搜索。针对`CreditCards`表发出的查询将根据`CreditCardNumber_HMAC`列过滤行，但我们希望返回`CreditCardNumber_Sym`列，以便解密原始信用卡号。

为确保这些查询能被索引覆盖，我们将基于`CreditCardNumber_HMAC`创建一个新的非聚集索引，并将`CreditCard_Sym`作为非键列包含在索引中。

```sql
CREATE NONCLUSTERED INDEX idxCreditCardNumberHMAC
ON CreditCards (CreditCardNumber_HMAC)
INCLUDE (CreditCardNumber_Sym);
GO
```

**注意**：一个**覆盖索引**在一个索引中包含了查询所需的所有列。这意味着数据库引擎可以仅基于索引就满足查询，而无需再对数据页执行额外的查找或查找操作以检索其他列。

让我们通过搜索`CreditCards`表中已知的信用卡来测试新的基于`HMAC`的解决方案的性能：

```sql
-- 选择要搜索的信用卡
DECLARE @CreditCardNumberToSearch nvarchar(32) = '4005550000000019';

-- 检索秘密盐值
DECLARE @salt varbinary(255);
SET @salt = (
    SELECT DECRYPTBYASYMKEY(
        ASYMKEY_ID('HMACASymKey'),
        MACKey,
        N'4n0th3r_5tr0ng_K4y!'
    )
    FROM MACKeys
);

-- 生成要搜索的信用卡的 HMAC
DECLARE @HMACToSearch varbinary(255);
SET @HMACToSearch = dbo.GenerateHMAC(
    'SHA256',
    CAST(@CreditCardNumberToSearch AS varbinary(max)),
    @salt
);

-- 从 CreditCards 表中检索匹配的行
SELECT
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID('CreditCard_Cert'),
            N'#Ch0o53_@_5Tr0nG_P455w0rD#',
            CreditCardNumber_Sym
        ) AS nvarchar(max)
    ) AS CreditCardNumber
FROM CreditCards
WHERE CreditCardNumber_HMAC = @HMACToSearch;
```


## 第 6 章 „ 加密

```sql
SELECT
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID('CreditCard_Cert'),
            N'#Ch0o53_@_5Tr0nG_P455w0rD#',
            CreditCardNumber_Sym) AS nvarchar(32)) AS CreditCardNumber_Decrypted
FROM CreditCards
WHERE CreditCardNumber_HMAC = @HMACToSearch
AND
    CAST(
        DECRYPTBYKEYAUTOCERT(
            CERT_ID('CreditCard_Cert'),
            N'#Ch0o53_@_5Tr0nG_P455w0rD#',
            CreditCardNumber_Sym) AS nvarchar(32)) = @CreditCardNumberToSearch;
GO
```

此查询先为提供的搜索值生成 HMAC 哈希，然后使用 `idxCreditCardNumberHMAC` 索引在 `CreditCardNumber_HMAC` 列中搜索该值。找到匹配行后，它会解密 `CreditCardNumber_Sym` 列中的值，以确保该值与最初提供的搜索值相匹配（以防止哈希冲突的风险，即由于某个错误结果恰好与我们的搜索字符串具有相同的哈希值而被返回的情况）。

图 6-8 所示的执行计划表明，整个查询可以通过单次索引查找完成。

*图 6-8. 在 HMAC 哈希列上执行聚集索引查找*

`sys.dm_exec_query_stats` 中包含的性能计数器显示，该查询比直接查询 `CreditCardNumber_Sym` 列的效率显著提高，仅消耗 253 毫秒的 CPU 时间，并且只需要九次逻辑读取。

### 使用 HMAC 子串进行通配符搜索

在许多情况下，数据库需要支持搜索条件具有一定灵活性的查询。这适用于加密数据的搜索，也适用于任何其他类型的数据。例如，您可能需要搜索所有信用卡在特定月份到期的客户，或者基于其社会安全号码的后四位数字进行搜索。

前面提出的 HMAC 解决方案无法用于这些情况，因为 HMAC 哈希值对于精确值是唯一的。由部分字符串生成的哈希将与由完整字符串生成的哈希完全不同，如下所示：

```sql
SELECT
    HASHBYTES('SHA1', 'The quick brown fox jumped over the lazy dog')
UNION SELECT
    HASHBYTES('SHA1', 'The quick brown fox jumped over the lazy dogs');
```

尽管输入字符串仅相差一个字符，但生成的哈希值却大不相同：

`0xF6513640F3045E9768B239785625CAA6A2588842`
`0xFBADA4676477322FB3E2AE6353E8BD32B6D0B49C`

为了支持部分字符串的模式匹配，我们仍然可以使用 HMAC，但我们需要基于字符串中某个在所有搜索中保持一致的片段来创建哈希。换句话说，要支持查找 `LIKE 'The quick brown fox jumped over the lazy%'` 值的查询，就需要搜索所有与截断字符串 `The quick brown fox jumped over the lazy` 的哈希值相匹配的行。这个确切的子字符串必须在所有查询中一致使用，并且它应具有足够的选择性以确保查询效率。

回到本章使用的示例，假设我们想要创建一种方法，允许用户基于信用卡号的后四位数字搜索 `CreditCards` 表中的一行数据。为此，我们将在表中添加一个新列，并使用 `GenerateHMAC` 函数填充该列，该函数基于每个信用卡号的后四位创建 HMAC。这个新列 `CreditCardNumber_Last4HMAC` 将仅用于支持通配符搜索，而现有的 `CreditCardNumber_HMAC` 列将继续用于对整个信用卡号的等值搜索。

首先，让我们向 `HMACKeys` 表添加一个新密钥：

```sql
CREATE ASYMMETRIC KEY HMACSubStringASymKey
    WITH ALGORITHM = RSA_1024
    ENCRYPTION BY PASSWORD = N'~Y3T_an0+h3r_5tR()ng_K4y~';
GO

INSERT INTO HMACKeys
SELECT
    2,
    ENCRYPTBYASYMKEY(
        ASYMKEY_ID(N'HMACSubStringASymKey'),
        N'->Th15_i$_Th3_HMAC_Sub5Tr1ng_k3y');
GO
```

现在，让我们使用新密钥为每个信用卡号的后四位字符创建一个 HMAC：

```sql
ALTER TABLE CreditCards
ADD CreditCardNumber_Last4HMAC varbinary(255);
GO

-- 从 MACKeys 表中检索 HMAC 盐值
```



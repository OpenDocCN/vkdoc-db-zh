# 第六章：加密

## 哈希函数

所谓“确定性”，我指的是哈希函数对给定的输入总会产生相同的输出。哈希本身可以说并不是一种真正的加密方法，因为一旦一个值被哈希，就无法反向过程来获得原始输入。然而，哈希方法常与加密结合使用，本章后续将会演示。

SQL Server 2008 提供了 `HASHBYTES` 函数，它使用多种标准哈希算法之一为任何提供的数据生成二进制哈希值。对于任何给定的输入 `x`，`HASHBYTES(x)` 总是会产生相同的输出 `y`，但没有提供方法从结果值 `y` 中检索 `x`。图 6-3 展示了 `HASHBYTES` 函数的实际应用。

`HASHBYTES` 在需要比较两个安全值是否相同，而不关心其实际值的情况下非常有用：例如，用于验证登录应用程序的用户提供的密码是否与该用户的存储密码匹配。在这种情况下，不是直接比较两个值，而是可以比较每个值的哈希值；如果任意两个给定值相等，那么使用给定算法生成的它们的哈希值也会相同。

尽管哈希算法是确定性的，即给定输入值总会生成相同的哈希值，但从理论上讲，两个不同的源输入有可能共享相同的哈希值。这种被称为**哈希碰撞**的情况相对罕见，但从安全角度来看必须牢记。在一个完全安全的环境中，不能仅仅因为两个哈希值相等，就断定生成这些哈希值的原始值是相同的。

`HASHBYTES` 函数支持消息摘要（Message Digest，MD）算法 MD2、MD4、MD5，以及安全哈希算法（Secure Hash Algorithm，SHA）算法 SHA 和 SHA1。其中，SHA1 是最强的，除非有充分理由，否则应在所有情况下指定该算法。以下代码清单展示了使用 SHA1 算法对纯文本字符串进行哈希的 `HASHBYTES` 函数的结果：
```sql
SELECT HASHBYTES('SHA1', 'The quick brown fox jumped over the lazy dog');
GO
```
这种情况下生成的哈希值如下：
```
0xF6513640F3045E9768B239785625CAA6A2588842
```
*图 6-3. `HASHBYTES` 函数的实际应用*

这个结果是完全可重复的——你可以根据需要多次执行前面的查询，并且总会得到相同的输出。哈希函数的这种确定性特性既是其最大的优点，也是其最大的缺点。

获得一致输出的优点是，从数据架构的角度来看，哈希数据可以像任何其他二进制数据一样被存储、索引和检索，这意味着对哈希数据的查询可以设计成以最小的查询重新设计来高效运行。然而，存在一种风险，即潜在的攻击者可以编译已知哈希值的字典（针对不同算法），并利用这些哈希值对你的数据进行反向查找，以尝试猜测源值。

当攻击者可以做出合理的假设以减少可能的源值列表时，这种风险变得更加现实。例如，在未强制执行密码强度的系统中，没有安全意识的用户通常会选择简短、简单的密码或常见单词的易预测变体。如果黑客获得了一个使用 MD5 算法生成的此类密码哈希值的数据集，他们只需查找 `0x2AC9CB7DC02B3C0083EB70898E549B63` 的出现，即可识别所有选择使用“Password1”作为密码的用户。

通过在对纯文本进行哈希之前添加一个秘密的**盐值**，可以使哈希更安全，本章后续将展示这一点。

### 对称密钥加密

对称密钥加密方法使用同一个密钥来执行数据的加密和解密。如图 6-4 所示。

*图 6-4. 对称密钥加密*

与确定性的哈希不同，对称加密是一个非确定性过程。也就是说，即使每次使用相同的密钥加密，对同一数据项在不同场合进行加密也会得到不同的结果。

SQL Server 2008 支持多种常见的对称加密算法，包括三重数据加密标准（Triple DES），以及基于高级加密标准（Advanced Encryption Standard, AES）的 128 位、192 位和 256 位密钥。

支持的最强对称密钥是 AES256。

对称密钥本身可以由证书、密码、非对称密钥或另一个对称密钥来保护，也可以由外部的 EKM 提供程序在 SQL Server 外进行保护。

以下示例展示了如何创建一个新的、受密码保护的对称密钥 `SymKey1`：
```sql
CREATE SYMMETRIC KEY SymKey1
WITH ALGORITHM = AES_256
ENCRYPTION BY PASSWORD = '5yMm3tr1c_K3Y_P@$$w0rd!';
GO
```
> **注意** 默认情况下，使用 `CREATE SYMMETRIC KEY` 语句创建的对称密钥是随机生成的。

如果要生成特定的、可重现的密钥，则必须在生成密钥时显式指定 `KEY_SOURCE` 和 `IDENTITY_VALUE` 选项，如下所示：
```sql
CREATE SYMMETRIC KEY StaticKey
WITH
    KEY_SOURCE = '#K3y_50urc£#',
    IDENTITY_VALUE = '-=1d3nt1ty_VA1uE!=-',
    ALGORITHM = TRIPLE_DES
ENCRYPTION BY PASSWORD = 'P@55w0rD';
```

要使用对称密钥加密数据，必须首先打开该密钥。由于 `SymKey1` 受密码保护，要打开此密钥，必须使用 `OPEN SYMMETRIC KEY DECRYPTION BY PASSWORD` 语法提供关联的密码。打开密钥后，可以通过将要加密的纯文本连同对称密钥的 GUID 一起提供给 `ENCRYPTBYKEY` 方法来加密数据。

使用完密钥后，应使用 `CLOSE SYMMETRIC KEY` 再次关闭它（如果未能显式关闭任何打开的对称密钥，它们将在会话结束时自动关闭）。这些步骤在下面的代码清单中说明：
```sql
-- 打开密钥
OPEN SYMMETRIC KEY SymKey1
DECRYPTION BY PASSWORD = '5yMm3tr1c_K3Y_P@$$w0rd!';

-- 声明要加密的明文
DECLARE @Secret nvarchar(255) = 'This is my secret message';

-- 加密消息
SELECT ENCRYPTBYKEY(KEY_GUID(N'SymKey1'), @secret);

-- 再次关闭密钥
CLOSE SYMMETRIC KEY SymKey1;
GO
```
上述代码清单的结果是一个加密的二进制值，例如：
```
0x007937851F763944BD71F451E4E50D520100000097A76AED7AD1BD77E04A4BE68404AA3B48FF6179A
D9FD74E10EE8406CC489D7CD8407F7EC34A879BB34BA9AF9D6887D1DD2C835A71B760A527B0859D47B3
8EED
```
> **注意** 请记住，加密是一个非确定性过程，因此你获得的结果会与刚才显示的不同。

使用对称密钥解密已加密的数据遵循与加密类似的过程，但使用 `DECRYPTBYKEY` 方法而不是 `ENCRYPTBYKEY` 方法。另一个需要注意的区别是，`DECRYPTBYKEY` 唯一需要的参数是要解密的密文；无需指定用于加密数据的对称密钥的 GUID，因为该值作为加密数据的一部分被存储。

以下代码清单说明了如何解密上一个示例中用对称密钥加密的数据：
```sql
OPEN SYMMETRIC KEY SymKey1
DECRYPTION BY PASSWORD = '5yMm3tr1c_K3Y_P@$$w0rd!';

DECLARE @Secret nvarchar(255) = 'This is my secret message';
DECLARE @Encrypted varbinary(max);
SET @Encrypted = ENCRYPTBYKEY(KEY_GUID(N'SymKey1'), @secret);

SELECT CAST(DECRYPTBYKEY(@Encrypted) AS nvarchar(255));

CLOSE SYMMETRIC KEY SymKey1;
GO
```
这将检索到原始的纯文本消息：
```
This is my secret message
```



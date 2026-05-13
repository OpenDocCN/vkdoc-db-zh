# 使用 EXECUTE AS 与所有权链结合

```sql
ALTER AUTHORIZATION ON KevinsData TO Kevin;
GO
CREATE USER Hilary
WITHOUT LOGIN;
GO
CREATE TABLE HilarysData
(
    SomeOtherData int
);
GO
ALTER AUTHORIZATION ON HilarysData TO Hilary;
GO
```

用户 Kevin 和 Hilary 都拥有表。可能需要创建一个访问这两个表的存储过程，但使用所有权链将不起作用：如果过程由 Kevin 拥有，所有权链只会允许执行用户访问 `KevinsData`，而不会授予对 `HilarysData` 的任何权限。同样，如果过程由 Hilary 拥有，则所有权链将不允许访问 `KevinsData` 表。

在这种情况下，一种解决方案是结合使用 `EXECUTE AS` 与所有权链，创建一个由其中一个用户拥有、但在另一个用户的上下文中执行的存储过程。以下存储过程展示了如何实现：

```sql
CREATE PROCEDURE SelectKevinAndHilarysData
WITH EXECUTE AS 'Kevin'
AS
BEGIN
    SET NOCOUNT ON;
    SELECT *
    FROM KevinsData
    UNION ALL
    SELECT *
    FROM HilarysData;
END;
GO
ALTER AUTHORIZATION ON SelectKevinAndHilarysData TO Hilary;
GO
```

因为 Hilary 拥有该存储过程，所有权链将生效并允许从 `HilarysData` 表中选择行。但由于存储过程在 Kevin 用户的上下文中执行，权限也会级联到 `KevinsData` 表。通过这种方式，可以结合使用两套权限集。

不幸的是，这几乎是使用 `EXECUTE AS` 所能实现的极限。对于更复杂的权限场景，有必要考虑使用证书对存储过程进行签名。

### 使用证书对存储过程签名

如前所述，可以基于证书创建代理登录名和用户。

基于证书创建代理是应用存储过程权限最灵活的方式，因为通过证书授予的权限是**相加的**。可以使用一个或多个证书对存储过程进行签名，每个证书都会在已有的权限之上添加其权限，而不是像使用 `EXECUTE AS` 执行模拟那样替换权限。

要为证书创建代理用户，必须首先确保数据库具有数据库主密钥。如果数据库中还没有主密钥，可以使用以下代码创建：

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '5Tr()nG_|)MK_p455woRD';
```

> **注意**：数据库主密钥是 SQL Server 加密密钥层次结构的重要组成部分，这将在下一章讨论。

然后创建证书，接着使用 `FOR CERTIFICATE` 语法创建关联的用户：

```sql
CREATE CERTIFICATE Greg_Certificate
WITH SUBJECT='Certificate for Greg';
GO
CREATE USER Greg
FOR CERTIFICATE Greg_Certificate;
GO
```

一旦创建了代理用户，就可以像任何其他数据库用户一样，授予其对数据库中资源的权限。但基于证书创建用户的一个副作用是，证书本身也可以用于传播授予该用户的权限。这就是存储过程签名的用武之地。

## 证书签名的工作原理

为了说明这个概念，创建一个表并授予 `Greg` 用户 `SELECT` 访问权限：

```sql
CREATE TABLE GregsData
(
    DataColumn int
);
GO
GRANT SELECT ON GregsData
TO Greg;
GO
```

然后可以创建一个从 `GregsData` 表中选择的存储过程。但在这个示例中，为了打破可能因在相同默认架构中创建表和存储过程而产生的所有权链，该存储过程将由名为 Steve 的用户拥有：

```sql
CREATE PROCEDURE SelectGregsData
AS
BEGIN
    SET NOCOUNT ON;
    SELECT *
    FROM GregsData;
END;
GO
CREATE USER Steve
WITHOUT LOGIN;
GO
ALTER AUTHORIZATION ON SelectGregsData TO Steve;
GO
```

请注意，此时 Steve 无法从 `GregsData` 中选择——Steve 仅仅拥有试图从该表中选择数据的存储过程。即使授予执行此存储过程的权限，任何用户（除了 `Greg`）都将无法成功执行，因为存储过程不会将权限传播到 `GregsData` 表：

```sql
CREATE USER Linchi
WITHOUT LOGIN;
GO
GRANT EXECUTE ON SelectGregsData TO Linchi;
GO
EXECUTE AS USER='Linchi';
GO
EXEC SelectGregsData;
GO
```

此尝试将失败并显示以下错误：

```
Msg 229, Level 14, State 5, Procedure SelectGregsData, Line 6
The SELECT permission was denied on the object 'GregsData', database
'OwnershipChain', schema 'dbo'.
```

为了使存储过程对 `Linchi` 用户生效，必须通过存储过程将权限传播到 `GregsData` 表。这可以通过使用创建 `Greg` 用户的同一证书对过程来完成。使用 `ADD SIGNATURE` 命令对存储过程进行签名（在执行以下代码之前，请确保使用 `REVERT` 退出 `Linchi` 上下文）：

```sql
ADD SIGNATURE TO SelectGregsData
BY CERTIFICATE Greg_Certificate;
GO
```

一旦过程用证书签名，该过程就拥有与 `Greg` 用户相同的权限；在这种情况下，这意味着任何有权执行该过程的用户在运行存储过程时都将能够从 `GregsData` 表中选择行。

当考虑到可以用任意数量的证书对给定的存储过程进行签名，而每个证书又可以关联到不同的用户，从而关联到不同的权限集时，证书签名的灵活性就变得显而易见。这意味着即使在一个拥有众多安全角色的极其复杂的系统中，仍然可以编写存储过程来聚合跨越安全边界的数据。

使用证书时请记住，任何时候修改存储过程，SQL Server 都会自动撤销所有签名。因此，重要的是要将签名与存储过程一起编写为脚本，这样在修改过程时，可以轻松地保持权限同步。

## 查找存储过程的证书签名

同样重要的是要知道如何找出哪些证书（以及因此哪些用户）与给定的存储过程相关联。可以查询 SQL Server 的目录视图来获取此信息，但要获得正确的查询并不明显。以下查询可以作为一个起点，它返回所有存储过程、对其进行签名的证书以及与证书关联的用户：

```sql
SELECT
    OBJECT_NAME(cp.major_id) AS signed_module,
    c.name AS certificate_name,
    dp.name AS user_name
FROM sys.crypt_properties AS cp
INNER JOIN sys.certificates AS c ON c.thumbprint = cp.thumbprint
INNER JOIN sys.database_principals dp
    ON SUBSTRING(dp.sid, 13, 32) = c.thumbprint;
```

这个查询有些难以理解，因此值得解释一下。`sys.crypt_properties` 视图包含有关哪些模块已由证书签名的信息。每个证书都有一个 32 字节的加密哈希值，即其*指纹*，用于通过 `sys.certificates` 视图找出是哪个证书对模块进行了签名。最后，每个数据库主体都有一个安全标识符，如果主体是基于证书创建的，则其最后 32 字节就是该证书的指纹。

执行此查询时，结果将显示刚刚创建的已签名模块，如下所示：

```
signed_module    certificate_name    user_name
SelectGregsData  Greg_Certificate    Greg
```


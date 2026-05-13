# 第 7 章 SQLCLR：架构与设计考虑

`ALTER DATABASE AdventureWorks2008`

`SET TRUSTWORTHY ON;`

`GO`

遗憾的是，尽管启用此选项的操作很简单，但其带来的影响却远非如此。

本质上，这意味着在可信数据库上下文中运行的代码，比在未标记为可信的数据库中运行的代码，能更轻松地访问数据库外部的资源。这包括访问文件系统、远程数据库服务器，甚至同一服务器上的其他数据库——所有这些访问都受控于这一个选项，因此请务必谨慎。

关闭 `TRUSTWORTHY` 选项意味着恶意代码将更难访问数据库外部的资源，但这也意味着，作为开发人员，你将不得不花费更多时间处理安全问题。话虽如此，我强烈建议除非有非常充分的理由，否则应保持 `TRUSTWORTHY` 选项处于关闭状态。在非可信数据库中处理访问控制并非难事；应采用第 5 章讨论的模块签名技术，这样访问控制权就完全掌握在你手中，而不应获得特定资源访问权限的代码也就无法轻易得逞。

在 SQLCLR 环境中，如果你在非可信数据库中注册一个使用 `EXTERNAL_ACCESS` 或 `UNSAFE` 权限集引用了另一个程序集的程序集，将会看到部署时异常。

以下是我尝试注册包含 `GetConvertedAmount` 方法的程序集时遇到的异常（在此前，我已将数据库设置为非可信模式）：

```
CREATE ASSEMBLY for assembly 'CurrencyConversion' failed because
assembly 'SafeDictionary' is not authorized for PERMISSION_SET = UNSAFE.
The assembly is authorized when either of the following is true: the database
owner (DBO) has UNSAFE ASSEMBLY permission and the database has the TRUSTWORTHY
database property on; or the assembly is signed with a certificate or an asymmetric
key that has a corresponding login with UNSAFE ASSEMBLY permission.
If you have restored or attached this database, make sure the database owner is
mapped to the correct login on this server. If not, use sp_changedbowner to fix
the problem.
```

这段相当详细的异常信息实属难得，值得珍视：它确切地描述了如何解决问题！遵循第 5 章描述的流程，你可以使用证书来授予 `UNSAFE ASSEMBLY` 权限。首先，在 `master` 数据库中创建一个证书和对应的登录名，并授予该登录名 `UNSAFE ASSEMBLY` 权限：

```
USE master;
GO
CREATE CERTIFICATE Assembly_Permissions_Certificate
ENCRYPTION BY PASSWORD = 'uSe_a STr()nG PaSSW0rD!'
WITH SUBJECT = 'Certificate used to grant assembly permission';
GO
CREATE LOGIN Assembly_Permissions_Login
FROM CERTIFICATE Assembly_Permissions_Certificate;
GO
GRANT UNSAFE ASSEMBLY TO Assembly_Permissions_Login;
GO
```

接下来，将证书备份到文件：

```
BACKUP CERTIFICATE Assembly_Permissions_Certificate
TO FILE = 'C:\assembly_permissions.cer'
WITH PRIVATE KEY
(
    FILE = 'C:\assembly_permissions.pvk',
    ENCRYPTION BY PASSWORD = 'is?tHiS_a_VeRySTronGP4ssWoR|)?',
    DECRYPTION BY PASSWORD = 'uSe_a STr()nG PaSSW0rD!'
);
GO
```

现在，在你正在工作的数据库中（我的例子中是 `AdventureWorks2008`）恢复证书并基于它创建一个本地数据库用户：

```
USE AdventureWorks2008;
GO
CREATE CERTIFICATE Assembly_Permissions_Certificate
FROM FILE = 'C:\assembly_permissions.cer'
WITH PRIVATE KEY
(
    FILE = 'C:\assembly_permissions.pvk',
    DECRYPTION BY PASSWORD = 'is?tHiS_a_VeRySTronGP4ssWoR|)?',
    ENCRYPTION BY PASSWORD = 'uSe_a STr()nG PaSSW0rD!'
);
GO
CREATE USER Assembly_Permissions_User
FOR CERTIFICATE Assembly_Permissions_Certificate;
GO
```

最后，用证书对程序集进行签名，从而授予访问权限并允许引用该程序集：

```
ADD SIGNATURE TO ASSEMBLY::SafeDictionary
BY CERTIFICATE Assembly_Permissions_Certificate
WITH PASSWORD='uSe_a STr()nG PaSSW0rD!';
GO
```


### 强命名

你可能遇到的另一个问题与强命名程序集有关。强命名是一项 .NET 安全特性，允许你对程序集进行数字签名，分配版本号，并向用户确保其有效性。对于大多数 SQLCLR 代码来说，强命名可能有些过度——在安全、受管理的数据库中运行的代码可能不需要强命名提供的额外保证。然而，对于考虑分发包含 SQLCLR 组件的应用程序的供应商来说，他们绝对会希望研究强命名。

在对包含 `ReadFileLines` 方法的程序集进行签名，并重新部署它和包含 `CAS_Exception` 存储过程的程序集之后，当我调用该过程时，我收到了以下错误：

```
Msg 6522, Level 16, State 1, Procedure CAS_Exception, Line 0
A .NET Framework error occurred during execution of user-defined routine or
aggregate "CAS_Exception":
System.Security.SecurityException: That assembly does not allow partially trusted
callers.
System.Security.SecurityException:
at System.Security.CodeAccessSecurityEngine.ThrowSecurityException(Assembly asm,
PermissionSet granted, PermissionSet refused, RuntimeMethodHandle rmh,
SecurityAction action, Object demand, IPermission permThatFailed)
at udf_part2.CAS_Exception()
.
```

解决方案是在代码中添加一个 `AllowPartiallyTrustedCallersAttribute`（在文章中常简称为 APTCA）。这个特性应该被添加到程序集中的一个单独文件中，位于 `using` 声明之后，在任何类或命名空间定义之前。在 `FileLines` 程序集的例子中，添加该特性后的文件如下所示：

```
using System;
using System.Data;
using System.Data.SqlClient;
using System.Data.SqlTypes;
using Microsoft.SqlServer.Server;
using System.Collections.Generic;
using System.Security.Permissions;

[assembly: System.Security.AllowPartiallyTrustedCallers]

public partial class FileLines
{
```

一旦添加了这个特性，任何调用者都可以使用 `FileLines` 类中的方法，而不会收到异常。请记住，指定此特性是有原因的，使用它你可能允许调用者绕过安全检查。如果程序集执行的操作不应被所有用户访问，请确保实施其他安全措施，例如创建具有不同所有者的程序集组，以确保非组内程序集无法引用敏感方法。


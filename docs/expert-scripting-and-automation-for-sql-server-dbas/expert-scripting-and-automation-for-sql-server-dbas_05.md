# 5. 使用 DbaTools

正如我们在第 4 章讨论的，由 Microsoft 维护的 `sqlserver` PowerShell 模块是一个很棒的工具，它允许数据库管理员轻松地通过脚本或 PowerShell 交互方式与 SQL Server 交互；但是，它支持的功能有限。由社区维护的 `DbaTools` 模块填补了功能上的许多空白，为从 PowerShell 与 SQL Server 交互提供了 `sqlserver` 模块之外的另一种选择。

截至本文撰写时，`DbaTools` 模块拥有 707 个 cmdlet 和函数，并且这个数字还在持续增长。因此，我们无法详细讨论每一个命令。本章将在初步了解 `Invoke-DbaQuery` 函数（它等同于 `Invoke-SqlCmd`）之后，主要聚焦于第 4 章讨论过的 `sqlserver` 模块的功能缺口。具体来说，我们将探讨如何端到端地管理登录（Logins）和用户（Users），以及如何管理 SQL Server 代理（SQL Server Agent）对象。不过，我建议您查阅 `DbaTools` 中可用的命令列表。完整的命令列表可以在 `dbatools.io/commands/` 找到，或者您可以使用代码清单 5-1 中的脚本来安装 `DbaTools` 模块并列出所有可用的命令。

> **提示**
>
> 请务必访问 `dbatools.io/commands/` 并查看 `DbaTools` 中完整的可用命令列表。

```powershell
## 安装 DbaTools
Install-Module dbatools
## 列出 DbaTools 模块中的命令
Get-Command -Module dbatools
```

**代码清单 5-1：安装 DbaTools 模块**

`DbaTools` 提供以下类别的命令：

*   可用性组
*   备份和还原
*   社区工具
*   连接字符串
*   数据库
*   数据屏蔽
*   计算机管理
*   配置
*   支持工具
*   更新监视器
*   DBCC
*   分离和附加
*   诊断和性能
*   端点
*   导出
*   文件系统和存储
*   文件流
*   查找器
*   常规
*   日志传送
*   登录和用户管理
*   邮件和日志记录
*   最大内存
*   迁移
*   镜像
*   网络和连接
*   基于策略的管理
*   已注册的服务器
*   复制
*   资源调控器
*   安全和加密
*   服务器管理
*   服务主体名称
*   服务
*   快照
*   Sp_configure
*   SQL Agent
*   SQL 客户端配置
*   SQL 管理对象
*   SQL Server Integration Services
*   系统启动
*   Tempdb
*   数据屏蔽
*   跟踪、探查器和扩展事件
*   实用工具
*   Windows Server 故障转移群集
*   写入 SQL 表

## 使用 `Invoke-DbaQuery`

`Invoke-DbaQuery` 提供了与第 4 章讨论的 `Invoke-SqlCmd` 命令等效的功能。因此，为了探究命令之间的差异，我们将使用相同的例子；即，我们希望检索一个数据库列表，然后将其传入循环中，根据索引碎片级别动态重建索引。

### `Invoke-DbaQuery` 基础

`Invoke-DbaQuery` 最基本的调用涉及传递两个参数：`SqlInstance`（定义查询应针对哪个 SQL Server 实例运行）和 `Query`（提供要执行的查询）。代码清单 5-2 演示了使用此基本调用从本地服务器的默认实例返回数据库名称列表。`SqlInstance` 参数可能的格式包括 `ServerName` 和 `ServerName\InstanceName`。如果您希望针对本地服务器的默认实例运行查询，则可以使用 `localhost`、`.` 或完全省略 `SqlInstance` 参数。

> **提示**
>
> 如果您想跟随本章的示例操作，但尚未为 SQL Server 实例配置安全的 TLS 连接，那么您可以运行以下命令来绕过使用 `DbaTools` 时的证书失败问题（您不应在生产环境中这样做）：`Set-DbatoolsConfig -FullName sql.connection.encrypt -Value $false`。

```powershell
$Params = @{
    SqlInstance   = "localhost"
    Query         = "SELECT name FROM sys.databases"
}
Invoke-DbaQuery @Params
```

**代码清单 5-2：`Invoke-DbaQuery` 的基本调用**

要运行相同的查询，但使用二级 SQL 登录名（SQL Login）向 SQL Server 进行身份验证，可以传递 `SqlCredential` 参数。如代码清单 5-3 所示。

```powershell
$UserName = 'Pete'
$SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$Credential = [PSCredential]::new($UserName, $SecurePassword)
$Params = @{
    SqlInstance   = "localhost"
    Query         = "SELECT name FROM sys.databases"
    sqlCredential = $Credential
}
Invoke-DbaQuery @Params
```

**代码清单 5-3：使用 SQL 登录名进行身份验证**

### 使用参数

`Invoke-DbaQuery` 和 `Invoke-SqlCmd` 在常见命令用法中最大的区别在于参数的处理方式。正如第 4 章所讨论的，使用 `Invoke-SqlCmd` 时，参数是作为键值对的字符串数组传递给 `Variable` 参数的。然而，对于 `Invoke-DbaQuery`，变量是以哈希表的形式传递给名为 `SqlParameters` 的参数。这是 `Invoke-DbaQuery` 的一个重要优势，因为它在处理多个参数时可以产生更整洁、更易读且更易于维护的代码。

代码清单 5-4 中的脚本是一个 T-SQL 脚本，它将根据从调用 `Invoke-DbaQuery` 传递过来的碎片级别动态重建索引。请注意，参数的格式与原生 T-SQL 参数相同，而不是 SQLCMD 参数格式。这可以使测试和调试脚本更容易，因为它们可以直接运行而无需切换到 SQLCMD 模式。此脚本随后将由代码清单 5-5 中的 PowerShell 脚本调用。

> **注意**
>
> 如果单独运行，T-SQL 脚本将执行失败，因为未定义 `@Fragmentation` 和 `@PageSpace` 参数。

```sql
DECLARE @SQL NVARCHAR(MAX)
SET @SQL =
(
    SELECT 'ALTER INDEX '
        + QUOTENAME(i.name)
        + ' ON ' + s.name
        + '.'
        + QUOTENAME(OBJECT_NAME(i.object_id))
        + ' REBUILD ; '
    FROM sys.dm_db_index_physical_stats(DB_ID(),NULL,NULL,NULL,'DETAILED') ps
    INNER JOIN sys.indexes i
        ON ps.object_id = i.object_id
        AND ps.index_id = i.index_id
    INNER JOIN sys.objects o
        ON ps.object_id = o.object_id
    INNER JOIN sys.schemas s
        ON o.schema_id = s.schema_id
    WHERE index_level = 0
        AND (avg_fragmentation_in_percent > @Fragmentation
            OR avg_page_space_used_in_percent < @PageSpace)
    FOR XML PATH('')
) ;
EXEC(@SQL) ;
```

**代码清单 5-4：重建索引的 T-SQL 脚本**

代码清单 5-5 中的 PowerShell 脚本将返回一个表列表，并将其传入一个 `foreach` 循环，针对每个数据库运行索引重建脚本。该脚本将两个参数传递给存储在文件中的 T-SQL 查询。T-SQL 脚本的位置通过 `File` 参数传递。然而，关键点在于参数是如何使用哈希表传递的。然后这个哈希表被传递给 `SqlParameters` 参数。

```powershell
$Variables = @{
    "Fragmentation" = "25"
    "PageSpace"     = "70"
}
$Params = @{
    SqlInstance = "localhost"
    Query       = "SELECT name FROM sys.databases"
}
$Databases = Invoke-DbaQuery @Params
foreach ($Database in $Databases) {
    $Params = @{
        SqlInstance   = "localhost"
        Database      = $Database.name
        File          = "c:\scripts\IndexRebuild.sql"
        SqlParameters = $Variables
    }
    Invoke-DbaQuery @Params
}
```

**代码清单 5-5：使用参数**


### 错误处理

`DbaTools` 系列中的命令默认会尝试帮助你处理错误。具体来说，当发生错误时，`DbaTools` 会捕获并解析错误，然后返回一个友好的错误信息。例如，考虑代码清单 5-6 中的命令，它会抛出一个除零错误。

```
Invoke-DbaQuery -SqlInstance localhost -Query "SELECT 1/0"
Listing 5-6
生成友好错误
```

此错误的输出如下所示：

```
WARNING: [10:56:33][Invoke-DbaQuery] [localhost] Failed during execution | Divide by zero error encountered.
```

虽然这肯定比抛出标准的 PowerShell 异常更容易阅读，但在编写脚本时可能会导致问题，因为它可能导致脚本静默失败。因此，在许多情况下，建议关闭此功能。这可以通过使用 `–EnableException` 参数来实现，如代码清单 5-7 所示。

```
Invoke-DbaQuery -SqlInstance localhost -Query "SELECT 1/0" –EnableException
Listing 5-7
使用 EnableException 参数
```

这将导致抛出一个标准异常，如下所示：

```
Exception:
Line |
97974 |          throw $records[0]
|          ~~~~~~~~~~~~~~~~~
| Divide by zero error encountered.
```

### 针对多个实例运行查询

使用 `Invoke-DbaQuery`，针对多个 SQL Server 实例运行查询非常简单。你只需要定义一个实例名称列表，然后将其通过管道传递给 `Invoke-DbaQuery` 命令即可。代码清单 5-8 展示了这一点，它返回了每个实例上的数据库列表。

**注意**
该脚本假设你有一个默认的 SQL Server 实例和一个名为 `poshscripting` 的命名实例。

```
"localhost", "localhost\poshscripting" | Invoke-DbaQuery -Query "SELECT name FROM sys.databases"
Listing 5-8
针对多台服务器运行查询
```

观察下面的结果，你会注意到一个明显的问题：无法确定每个数据库属于哪个实例：

```
master
tempdb
model
msdb
AdventureWorks2022
master
tempdb
model
msdb
```

当然，你可以通过将 `@@SERVERNAME` 添加到 `SELECT` 列表中来解决这个问题。然而，`Invoke-DbaQuery` 提供了另一种方法。你可以简单地指定 `–AppendServerInstance` 参数。在处理本地服务器时，两种技术的差异显而易见。使用 `@@SERVERNAME` 技术将返回实际的服务器名称。而使用 `–AppendServerInstance` 技术，则会返回你传递给 `Invoke-DbaQuery` 的任何值。这可能是服务器名称、`localhost` 或一个 IP 地址。例如，考虑代码清单 5-9 中的命令。

```
$query = "SELECT name, @@SERVERNAME AS SQLInstance FROM sys.databases"
"localhost", "localhost\poshscripting" | Invoke-DbaQuery -Query $query –AppendServerInstance
Listing 5-9
返回服务器名称
```

结果（如下所示）说明了两种技术在结果上可能存在的差异：

```
name               SQLInstance                      ServerInstance
--------------     ---------------------------    -----------------------
master             WIN-J38I01U06D7                  localhost
tempdb             WIN-J38I01U06D7                  localhost
model              WIN-J38I01U06D7                  localhost
msdb               WIN-J38I01U06D7                  localhost
AdventureWorks2022 WIN-J38I01U06D7                  localhost
master             WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
tempdb             WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
model              WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
msdb               WIN-J38I01U06D7\POSHSCRIPTING    localhost\poshscripting
```

#### 将结果写入文件

与 `Invoke-SqlCmd` 一样，没有直接允许你将查询结果发送到文件的参数；但是，可以通过将 `Invoke-SqlCmd` 的结果通过管道传递给 `Out-File` cmdlet 来达到相同的目的。代码清单 5-10 展示了这一点，它将数据库列表写入到一个名为 `c:\scripts\DatabasesOut.csv` 的文件中。

```
$Params = @{
SqlInstance  = "localhost"
Query        = "SELECT name FROM sys.databases"
}
Invoke-DbaQuery @Params | ConvertTo-Csv | Out-File -FilePath "c:\scripts\DatabasesOut.csv"
Listing 5-10
将结果输出到文件
```

## 管理安全对象

在第 4 章中，我们讨论了如何使用 `SqlServer` 模块来创建登录名并将其添加到角色中。那时，我们想开始管理数据库用户；然而，我们遇到了 `SqlServer` 模块功能上的空白，不得不直接使用 SMO 来执行所需的任务。

`DbaTools` 模块拥有更丰富的命令集来处理安全对象。因此，在接下来的章节中，我们将逐步介绍如何使用 `DbaTools` 来端到端地管理登录名和用户。具体来说，我们将探讨如何创建登录名、创建数据库用户、将登录名和用户都添加到角色中、重命名登录名、删除登录名以及移除孤立用户。


### 创建登录名

`New-DbaLogin` 命令可用于创建登录名。该命令最简单的调用方式是，通过 `–Login` 参数传递要创建的登录名，并通过 `-SqlInstance` 参数传递应创建登录名的 SQL Server 实例名称。如代码清单 5-11 所示。

> 注意
> 请将服务器\登录名更改为与你自己的服务器名称以及服务器上存在的本地用户相匹配。

```
$params = @{
    SqlInstance     = "localhost"
    Login           = "WIN-J38I01U06D7\Adam"
    DefaultDatabase = "AdventureWorks2022"
}
New-DbaLogin @params
```
*代码清单 5-11 创建一个登录名*

此脚本为现有的 Windows 用户创建登录名。如果要创建 SQL 登录名，则需要额外以 `SecureString` 形式传递登录名的密码。创建 SQL 登录名时，我们还可以选择强制执行域密码策略和过期策略，就像使用 `SqlServer` 模块一样。代码清单 5-12 展示了这一点。

```
$Password = 'Pa$$w0rd'
# 转换为 SecureString
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
$params = @{
    SqlInstance      = "localhost"
    Login            = "Adam"
    SecurePassword   = $SecurePassword
    DefaultDatabase  = "AdventureWorks2022"
}
New-DbaLogin @params -PasswordPolicyEnforced –PasswordExpirationEnabled
```
*代码清单 5-12 创建一个 SQL 登录名*

创建 SQL 登录名时，我们还可以强制使用特定的 SID（安全标识符）。这在诸如配置管理（在第 9 章讨论）等场景中非常有用，因为我们希望确保登录始终存在，并在意外删除时自动重新创建。

SQL Server 使用 SID 将数据库用户映射到登录名；因此，即使 SQL 登录名具有相同的名称和密码，如果 SID 发生变化，数据库用户也会成为孤立用户。使用特定的 SID 可以消除此问题。

> 提示
> 对于为 Windows 帐户创建的登录名，不会出现此问题。这是因为 SID 存储在域中（对于本地 Windows 帐户则存储在本地服务器中）。每次为 Windows 用户创建登录名时，即使在不同的实例上，它都将具有相同的 SID。

你可以通过 `sys.syslogins` 表来识别 SQL 登录名的 SID。代码清单 5-13 中的 T-SQL 查询演示了如何检索我们在代码清单 5-12 中创建的名为 `Adam` 的 SQL 登录名的 SID。

```
SELECT
    name
    , sid
FROM sys.syslogins
WHERE name = 'Adam'
```
*代码清单 5-13 检索登录名的 SID*

代码清单 5-14 演示了如何创建具有特定 SID 的登录名。为此，我们使用 `New-DbaLogin` 命令的 `–Sid` 参数。

> 注意
> 登录名 `Adam` 已存在。因此，我们使用了 `–Force` 参数，这将导致登录名被删除并重新创建，从而避免失败。

```
$Password = 'Pa$$w0rd'
# 转换为 SecureString
[securestring]$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
$params = @{
    SqlInstance     = "localhost"
    Login           = "Adam"
    SecurePassword  = $SecurePassword
    DefaultDatabase = "AdventureWorks2022"
    SID             = "0xC8105F39AEDBF947ACA0FA99544EBF33"
}
New-DbaLogin @params –Force
```
*代码清单 5-14 使用特定 SID 创建登录名*

### 管理登录名

DbaTools 提供了用于管理登录名的有用命令。在以下部分中，我们将探讨如何将登录名添加到服务器角色、重命名以及删除登录名。

#### 将登录名添加到服务器角色

通过将登录名添加到服务器角色，可以为其授予实例范围内的权限。这可以使用 DbaTools 的 `Add-DbaServerRoleMember` 命令实现。例如，代码清单 5-15 中的命令将 `Adam` 登录名添加到 `dbcreator` 服务器角色，允许他在实例上创建新数据库。

```
Add-DbaServerRoleMember -SqlInstance localhost -ServerRole dbCreator -Login Adam
```
*代码清单 5-15 将登录名添加到服务器角色*

较新版本的 SQL Server 还允许你创建自己的自定义服务器角色，此功能也通过 DbaTools 得到支持，即使用 `New-DbaServerRole` 命令；但是，在撰写本文时，此命令功能非常有限，可以创建服务器角色，但无法为其分配权限。因此，我们将不对其进行详细探讨。

#### 重命名登录名

假设你接手了一个应用程序的职责，并发现连接是通过一个名为 `AppLogin` 的登录名建立的。我们都经历过这种情况。这不仅仅令人有点沮丧，而且你已将应用程序迁移到一个托管多个应用程序数据库的 SQL 实例。因此，这个名称根本不合适，因为故障排除将变得混乱和复杂。

你可以使用 `Rename-DbaLogin` 命令轻松更改登录名的名称。代码清单 5-16 说明了如何将 `AppLogin` 的名称更改为 `SalesApp`，从而允许你区分实例上的多个应用程序登录名。`-Login` 参数指定原始登录名，`-NewLogin` 参数用于指定登录名的所需名称。

> 注意
> 请记住，如果在 SQL Server 实例中更改了应用程序登录名的名称，还需要在应用程序中进行更新。

```
$params = @{
    SqlInstance = "localhost"
    Login       = "AppLogin"
    NewLogin    = "SalesApp"
}
Rename-DbaLogin @params
```
*代码清单 5-16 重命名登录名*

#### 删除登录名

可以使用 `Remove-DBALogin` 命令删除登录名。要使用此命令，只需使用 `–Login` 参数指定登录名即可。代码清单 5-17 演示了这一点，我们在其中删除了 `SalesApp` 登录名。

```
$params = @{
    SqlInstance = "localhost"
    Login       = "SalesApp"
}
Remove-DbaLogin @params
```
*代码清单 5-17 删除登录名*


### 创建数据库用户

DbaTools 可用于创建数据库用户和实例登录名。`New-DbaDbUser` 命令可用于此目的。在最简单的调用方式中，你可以通过 `–Database` 参数指定数据库名称，并通过 `–Login` 参数指定用户应映射到的登录名，如代码清单 5-18 所示，我们在 AdventureWorks2022 数据库中创建了一个关联到 `Adam` 登录名的数据库用户。

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
Login       = "Adam"
}
New-DbaDbUser @params
Listing 5-18
创建数据库用户
```

你也可以选择性地使用 `–User` 参数来指定数据库用户的名称。你可以选择这样做作为一种代码风格选项，或者当你希望数据库用户的名称与登录名不同时，这也非常有用。例如，代码清单 5-19 中的脚本在 `AdventureWorks2022` 数据库中创建了一个名为 `PeteCarter` 的数据库用户，该用户映射到 `Pete` 登录名。

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
Login       = "Pete"
User        = "PeteCarter"
}
New-DbaDbUser @params
Listing 5-19
创建具有特定名称的数据库用户
```

此外，该命令可用于复制用户，可以在同一实例的不同数据库之间进行。例如，代码清单 5-20 中的命令将 AdventureWorks2022 数据库中的用户 Pete（基于 Windows 登录名 Pete）复制到同一实例的 WideWorldImporters 数据库中。

`Get-DbaDbUser` 命令用于获取 AdventureWorks2022 数据库中所有数据库用户的详细信息。然后通过管道传递给 `Where-Object`，对结果进行过滤，以便仅包含 Pete 数据库用户。这随后存储在一个名为 `$user` 的变量中，该变量由 `New-DbaDbUser` 命令使用。用户将在我们指定的数据库中创建。

> 注意
>
> 需要注意的是，虽然数据库用户被复制，但与该用户关联的权限不会随之复制。

```
$user = Get-DbaDbUser -SqlInstance localhost -Database AdventureWorks2022 | Where-Object name -eq "Pete"
$params = @{
Login = $user.Login
Database = 'WideWorldImporters'
SqlInstance = 'Localhost'
}
New-DbaDbUser @params
Listing 5-20
复制数据库用户
```

### 管理数据库用户

DbaTools 提供了许多用于管理数据库用户的有用命令。在以下部分中，我们将探讨如何将用户添加到数据库角色。我们还将讨论如何删除用户以及如何修复已成为孤立状态的用户。

#### 将用户添加到数据库角色

可以使用 `Add-DbaDbRoleMember` 命令将用户添加到数据库角色。你需要通过 `–Database` 参数传递数据库名称，通过 `–Role` 参数传递角色名称，并通过 `–User` 参数传递要添加到角色的用户名称。代码清单 5-21 演示了这一点，其中 `PeteCarter` 登录名被添加到 `AdventureWorks2022` 数据库中的 `db_owner` 数据库角色。

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
User        = "PeteCarter"
Role        = "db_owner"
}
Add-DbaDbRoleMember @params
Listing 5-21
将用户添加到角色
```

> 提示
>
> 为避免被提示确认操作，你可以在命令后附加 `-Confirm:$false`。

存在一个名为 `New-DbaDbRole` 的命令，可用于创建你自己的数据库角色。然而，与其服务器角色对应物类似，其功能仅限于创建角色。它不允许向该角色分配权限。因此，我们将不在这里介绍它，我建议使用 `Invoke-DbaQuery` 来创建服务器和数据库角色。

#### 删除用户及将其从角色中移除

可以使用 `Remove-DbaDbRoleMember` 命令将用户从数据库角色中移除。代码清单 5-22 演示了这一点，它将 `PeteCarter` 用户从 `AdventureWorks2022` 数据库的 `db_owner` 数据库角色中移除。

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
User           = "PeteCarter"
Role           = "db_owner"
}
Remove-DbaDbRoleMember @params
Listing 5-22
从数据库角色中移除用户
```

如果需要完全删除一个数据库用户，可以通过使用 `Remove-DbaDbUser` 命令来实现。该命令需要传递数据库和用户，如代码清单 5-23 所示。

```
$params = @{
SqlInstance = "localhost"
Database    = "AdventureWorks2022"
User        = "PeteCarter"
}
Remove-DbaDbUser @params
Listing 5-23
删除数据库用户
```

### 管理服务器代理

在第 4 章中，我们讨论了 SqlServer 模块如何使你能够创建凭据并检索有关 SQL Server 代理作业的信息，但由于没有提供与 SQL Server 代理对象交互或管理其的命令，因此其用处有限。因此，在以下部分中，我们将讨论如何端到端地管理 SQL Server 代理。具体来说，我们将探讨如何创建凭据、创建映射到该凭据的代理以及创建作业。我们还将讨论如何检索作业信息以及如何使用这些信息来排查问题。

> 注意
>
> 本章假设你熟悉 SQL Server 代理及其对象，例如作业、作业步骤和代理。对 SQL Server 代理的完整讨论和解释可以在 Apress 的书籍 *Pro SQL Server 2022 Administration* 中找到，链接为：link.springer.com/book/10.1007/978-1-4842-8864-1。

#### 创建凭据

`New-DbaCredential` 提供了与本书第 4 章讨论的 `SqlServer` 模块的 `New-SqlCredential` 命令等效的功能。我们将使用该命令创建一个名为 `SQLAgentCredential` 的凭据，该凭据映射到 `Pete` Windows 用户，并将用作运行 SQL Server 代理作业的安全上下文。

该命令需要通过 `–Name` 参数传递凭据的名称。它还需要将要使用的标识（在本例中为 Windows 用户）传递给 `–Identity` 参数，并将密钥（在本例中为 Windows 用户的密码）传递给 `–SecurePassword` 参数。

代码清单 5-24 中的脚本说明了如何使用 `New-DbaCredential` 命令来创建描述的凭据对象。

```
$SecurePassword = ConvertTo-SecureString 'Pa$$w0rd' -AsPlainText -Force
$params = @{
SqlInstance     = "localhost"
Name            = "SqlAgentCredential"
Identity        = "Pete"
SecurePassword  = $SecurePassword
}
New-DbaCredential @params
Listing 5-24
创建凭据
```



### 创建代理

SQL Server Agent 代理定义了作业步骤得以运行的安全上下文。本质上，它提供了作业步骤到凭据的映射。在此示例中，我们将创建一个名为 `SQLAgentProxy` 的代理，该代理映射到 `SQLAgentCredential` 凭据，并为其分配运行配置为 `PowerShell` 或 `CmdExec` 作业步骤类型的作业步骤的权限。

为此，我们将使用 `New-DbaAgentProxy` 命令，并通过 `-Name` 参数传入代理的名称，通过 `-ProxyCredential` 参数传入其将映射到的凭据，最后通过 `–SubSystem` 参数指定它可以运行的作业步骤类型，如代码清单 5-25 所示。

提示

如果直接在命令中传递 `–SubSystem` 参数，我们可以简单地传递一个逗号分隔的列表。因为我们使用的是拼接技术，所以将两个值定义为字符串数组，然后传递给命令。

此代码仅适用于基于 Windows 的机器。

```
$params = @{
SqlInstance     = "localhost"
Name            = "SqlAgentProxy"
ProxyCredential = "SQLAgentCredential"
SubSystem       = @("PowerShell","CmdExec")
}
New-DbaAgentProxy @params
代码清单 5-25
创建代理
```

有效的子系统完整列表如下：

*   ActiveScripting
*   AnalysisCommand
*   AnalysisQuery
*   CmdExec
*   Distribution
*   LogReader
*   Merge
*   PowerShell
*   QueueReader
*   Snapshot
*   Ssis

注意

`ActiveScripting` 子系统与运行 ActiveX 脚本有关。这在 SQL Server 中已弃用，不应使用。

### 创建作业

使用 DbaTools 创建 SQL Server Agent 作业涉及三个方面。首先，您需要创建作业本身。然后，您需要创建由该作业运行的作业步骤，最后附加一个作业计划，该计划将定义作业何时运行。

在本节中，我们将创建一个名为 `DbaMaintenance` 的作业，该作业对实例上的每个数据库运行 `DBCC CHECKDB` 和智能索引重建过程。由于作业内置了逻辑，它们将是 PowerShell 脚本，而不是 T-SQL 脚本。这将在本书的第 10 章和第 11 章中进一步讨论。

使用 `New-DbaAgentJob` 命令创建作业。其基本调用需要通过 `–Job` 参数提供作业名称，并通过 `–OwnerLogin` 参数提供作业所有者。创建作业如代码清单 5-26 所示。

```
New-DbaAgentJob -SqlInstance localhost -Job DbaMaintenance -OwnerLogin sa
代码清单 5-26
创建 SQL Agent 作业
```

可以使用 `New-DbaAgentJobStep` 命令创建作业步骤。让我们先看看一个简单的调用，然后再看一个更复杂的例子。

代码清单 5-27 中的脚本创建了所需的两个作业步骤。这里，我们传入步骤的名称、应运行的命令、相关的子系统、代理名称以及将运行步骤的作业名称，还有每个步骤的成功/失败操作。

我们还传递了 `StepID`。这是一个在作业内顺序递增的数字，用于定义步骤运行的顺序。如果没有此参数，每次添加一个步骤时，它都会移动到作业中的第一个位置，这会导致一个有点奇怪的情况：步骤的运行顺序与代码中定义的顺序相反。

```
New-DbaAgentJobStep -SqlInstance localhost -Job DbaMaintenance -StepName CheckDB -ProxyName SQLAgentProxy -Command 'PowerShell "c:\scripts\CheckDB.ps1"' -Subsystem PowerShell -StepId 1 -OnSuccessAction GoToNextStep -OnFailAction GoToNextStep
New-DbaAgentJobStep -SqlInstance Localhost -Job DbaMaintenance -StepName IndexRebuild -ProxyName SQLAgentProxy -Command 'PowerShell "c:\scripts\IndexRebuild.ps1"' -Subsystem PowerShell -StepId 2 -OnSuccessAction GoToNextStep -OnFailAction GoToNextStep
代码清单 5-27
创建作业步骤
```

虽然这种方法可行，但它相当繁琐，并且会导致大量代码重复。考虑到此示例仅使用两个作业步骤，这种情况被放大了。想象一下，如果有 20 个步骤会怎样！然而，我们可以通过在 PowerShell 中结合使用拼接和循环来解决这个问题，使我们能够仅定义一次大多数参数值，并且只声明一次命令调用。代码清单 5-28 说明了这种技术。在这里，我们首先使用拼接技术定义通用参数集。然后，我们使用第二个拼接，该拼接由每个步骤定义的数组组成。最后，一个 `foreach` 循环遍历每个步骤定义，依次为每个步骤调用 `New-DbaAgentJobStep`。结果在功能上是等效的，但这减少了代码重复，并产生了更易于维护的代码。

```
$commonParams = @{
SqlInstance     = "localhost"
Job             = "DbaMaintenance"
Subsystem       = "PowerShell"
ProxyName       = "SQLAgentProxy"
OnSuccessAction = "GoToNextStep"
OnFailAction    = "GoToNextStep"
}
$steps = @(
@{ StepName = 'CheckDB';          Command = 'powershell "C:\scripts\CheckDB.ps1"';            StepID = 1 }
@{ StepName = 'RebuildIndexes'; Command = 'powershell "C:\scripts\RebuildIndexes.ps1"';   StepID = 2 }
)
foreach ($step in $steps) {
New-DbaAgentJobStep @step @commonParams
}
代码清单 5-28
改进作业步骤创建的代码



## 为作业添加调度

为作业添加调度的最后一步是定义其运行时间。这可以通过使用 `New-DbaAgentSchedule` 命令来实现。可以传递给此命令的参数定义在表 5-1 中。

### 表 5-1：New-DbaAgentSchedule 参数

| 参数 | 描述 | 允许的值 |
| --- | --- | --- |
| `Job` | 要附加调度的 SQL Agent 作业的名称 | |
| `Schedule` | 调度的名称 | |
| `Disabled` | 指定调度应启用还是禁用 | `True`，`False`（默认） |
| `FrequencyType` | 指定作业执行的频率周期 | `Once`，`OneTime`，`Daily`，`Weekly`，`Monthly`，`MonthlyRelative`，`AgentStart`，`AutoStart`，`IdleComputer`，`OnIdle` |
| `FrequencyInterval` | 在频率周期内，应执行作业的间隔。对于 `FrequencyType` 为每周的作业，这可以是一周中的某几天；对于 `FrequencyType` 为每月的作业，这可以是一个月中的某一天 | `Sunday`，`Monday`，`Tuesday`，`Wednesday`，`Thursday`，`Friday`，`Saturday`，`Sunday`，`Weekdays`，`Weekend`，`Everyday`，或 1 到 31（月度）或 1 到 365（每日）的数值 |
| `FrequencySubdayType` | 对于一天内执行多次的作业，指定每次执行之间的时间单位。对于一天内只执行一次的作业，请使用 `Time`。`Once` 指定作业在其生命周期内仅执行一次 | `Once`，`Time`，`Seconds`，`Second`，`Minutes`，`Minute`，`Hours`，`Hour` |
| `FrequencySubdayInterval` | 指定每次执行之间应发生的 `FrequencySubdayType` 次数的整数。例如，如果 `FrequencySubdayType` 设置为 `Minutes` 且 `FrequencySubdayInterval` 设置为 `5`，则作业将每 5 分钟执行一次 | |
| `FrequencyRecurrenceFactor` | 如果 `FrequencyType` 设置为 `Weekly`、`Monthly` 或 `MonthlyRelative`，则可以向此参数传递一个整数，以指定每次执行之间应间隔多少周或月 | |
| `FrequencyRelativeInterval` | 当 `FrequencyInterval` 配置为 `MonthRelative` 时，此参数用于指定在月内何时执行作业 | `First`，`Second`，`Third`，`Fourth`，`Last` |
| `StartDate` | 调度应开始的日期，格式为 yyyyMMdd | |
| `EndDate` | 调度应结束的日期，格式为 yyyyMMdd | |
| `StartTime` | 调度应开始的时间。可接受的格式是 24 小时制的 HHMMSS | |
| `EndTime` | 调度应结束的时间。可接受的格式是 24 小时制的 HHMMSS | |

在我们的场景中，假设我们希望作业每天每五分钟运行一次。我们希望调度立即开始并无限期运行。因此，我们不指定 `StartDate`、`StartTime`、`EndDate` 和 `EndTime`，而是使用 `–Force` 参数，它会将当前日期添加为开始日期，当前日期/时间添加为开始时间，结束日期和时间设置为 9999 年 12 月 31 日 00:00:00。清单 5-29 中的脚本演示了如何创建此调度。

> 注意
>
> 实际上，这样的维护作业可能会安排每天运行一次，或者在某些环境中甚至可能每周运行一次。此处的调度仅用于说明目的。

```powershell
$params = @{
    SqlInstance = "localhost"
    Job = "DbaMaintenance"
    Schedule = "DailyEvery5Mins"
    FrequencyType = "Daily"
    FrequencyInterval = "Everyday"
    FrequencySubdayType = "Minutes"
    FrequencySubdayInterval = 5
}
New-DbaAgentSchedule @params -force
```
*清单 5-29：创建作业调度*

## 检索作业信息

DbaTools 提供了多个用于检索作业信息的命令。这些信息可以提供非常快速简便的机制，帮助 DBA 故障排除问题。在本节中，我们将通过一个示例来说明。

首先，假设我们正在对服务器执行晨检。我们需要检查 SQL Agent 作业，因此我们将使用 `Get-DbaAgentJob` 命令返回作业列表。默认情况下，此命令将为每个作业返回一个对象，如清单 5-30 所示。

```powershell
Get-DbaAgentJob -SqlInstance localhost
```
*清单 5-30：返回 SQL Agent 作业列表*

此命令的结果当然会因你在实例上拥有的作业而异。当我运行此命令时，收到以下结果：

```
ComputerName    InstanceName SqlInstance     Name                    Category                OwnerLoginName CurrentRunStatus CurrentRunRetryAttempt Enabled LastRunDate         LastRunOutcome HasSchedule OperatorToEmail CreateDate
------------    ------------ -----------     ----                    --------                -------------- ---------------- ---------------------- ------- -----------         -------------- ----------- ---------------
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance          [Uncategorized (Local)] sa                    Executing                      0    True 18/01/2021 08:37:00         Failed        True                 17/01/2021 16:36:29
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 syspolicy_purge_history [Uncategorized (Local)] sa                         Idle                      0    True 18/01/2021 07:09:47      Succeeded        True                 11/08/2020 13:10:41
```

从这些结果中明显可以看出，`DbaMaintenance` 作业在上次执行时失败了。因此，我们将使用 `Get-DbaAgentJobHistory` 命令检索有关失败作业的更多信息，以返回作业历史输出。我们将此命令的结果通过管道传递给 `Where-Object` 命令，以便筛选失败的作业，最后再通过管道传递给 `Format-Table` 命令，使结果更易于阅读。清单 5-31 演示了这一点。

```powershell
Get-DbaAgentJobHistory -SqlInstance localhost | Where-Object status -eq "Failed" | Format-Table
```
*清单 5-31：返回作业历史记录*

运行此命令的结果如下所示：

```
ComputerName    InstanceName SqlInstance     Job            StepName      RunDate             StartDate               EndDate                 Duration Status OperatorEmailed Message
------------    ------------ -----------     ---            --------      -------             ---------               -------                 -------- ------ ---------------
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:41:00 2021-01-18 08:41:00.000 2021-01-18 08:42:01.000 00:01:01 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:39:00 2021-01-18 08:39:00.000 2021-01-18 08:40:02.000 00:01:02 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:37:00 2021-01-18 08:37:00.000 2021-01-18 08:38:02.000 00:01:02 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
WIN-J38I01U06D7 MSSQLSERVER  WIN-J38I01U06D7 DbaMaintenance (Job outcome) 18/01/2021 08:35:00 2021-01-18 08:35:00.000 2021-01-18 08:36:02.000 00:01:02 Failed                 The job failed.  JobManager tried to run a non-existent step (3) for job DbaMain...
```



### 从结果看失败任务

从这些结果中，我们可以看出哪个任务总是在同一位置失败。但你也会注意到，错误信息被截断了。我们可以通过将 `Where-Object` 命令的结果通过管道传递给 `Select-Object` 命令来解决这个问题，后者可以展开 `Message` 属性。这在代码清单 5-32 中有演示。

```
Get-DbaAgentJobHistory -SqlInstance localhost | Where-Object status -eq "Failed" | Select-Object -ExpandProperty Message
```
代码清单 5-32
展开 Message 属性

此命令的结果如下所示：

```
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
The job failed.  JobManager tried to run a non-existent step (3) for job DbaMaintenance.
```

我现在可以明确地看到，任务失败是因为它试图运行一个不存在的作业步骤。这显然是作业设计中的一个错误。如果你回想一下代码清单 5-28，我们在那里创建了作业步骤，你可能记得我们为所有作业步骤将 `OnSuccessAction` 和 `OnFailureAction` 参数的公共属性设置为了 `GoToNextStep`。这包括了最后一个作业步骤，这意味着当最后一个作业步骤完成时，它会尝试执行“下一个作业步骤”，而这个步骤当然不存在。

我们可以通过将最后一个作业步骤更新为 `QuitWithSuccess` 或 `QuitWithFailure` 来解决这个问题，具体取决于该步骤的结果。这可以通过使用 `Set-DbaAgentJobStep` 命令实现，如代码清单 5-33 所示。这里，我们将作业名称传递给 `–Job` 参数，作业步骤名称传递给 `–StepName` 参数，以及我们想要更新的作业方面。在我们的场景中，将包含 `-OnSuccessAction` 和 `-OnFailAction` 参数。

```
$params = @{
    SqlInstance     = "localhost"
    Job             = "DbaMaintenance"
    StepName        = "RebuildIndexes"
    OnSuccessAction = "QuitWithSuccess"
    OnFailAction    = "QuitWithFailure"
}
Set-DbaAgentJobStep @params
```
代码清单 5-33
更新作业步骤

## 总结

DbaTools 提供了一个由社区维护的替代方案，用于替代 Microsoft 的 `SqlServer` PowerShell 模块。虽然它并非没有局限性，例如，无法为你已创建的角色分配权限，但它提供的功能覆盖范围远大于 Microsoft 的产品。该模块中的命令在用法上也提供了更高的一致性，这使得代码编写更快。

`Invoke-DbaQuery` 命令等同于 `SqlServer` 模块中的 `Invoke-SqlCmd` 命令。这两个命令的功能在很大程度上是一致的。两者之间最大的区别在于参数的处理方式。`Invoke-DbaQuery` 允许你通过哈希表传递参数，然后在你的 T-SQL 脚本中使用它们，就像在 T-SQL 中声明的一样。这有助于使代码更易于维护。

在撰写本文时，DbaTools 提供了超过 700 个命令和函数，我强烈建议你查看 dbatools.io/commands/ 上的完整命令列表。虽然在本章的有限篇幅内只能触及可用命令的皮毛，但你可以看到，这一系列功能提供了以一种非常快速、简单和灵活的方式管理对象（例如与安全和 SQL Server Agent 相关的对象）的能力，而无需依赖在 `Invoke-DbaQuery` 命令中编写 T-SQL 脚本。

# 6. 自动化安装

自动化实例构建不仅可以减少 DBA 的工作量和新数据层应用程序的上市时间，还可以降低运营开销，因为它为企业中所有新的数据层应用程序提供了一个一致的平台。

一个完全自动化的构建不仅仅是运行命令行的 `setup.exe`。我们还应该考虑如何优化 Windows 以及如何测试我们的自动化构建是否成功。

有多种方法可用于自动化 SQL Server 实例的构建。例如，我们可以使用 sysprep，或者在云中，我们可以使用已预装 SQL Server 的现成映像。然而，这些选项存在挑战，例如缺乏控制和灵活性。

因此，在本章中，在初步回顾如何使用 `setup.exe` 从命令行安装 SQL Server 之后，我们将重点介绍构建实例的两种最灵活的方法。具体来说，我们将研究一种脚本化方法，然后是使用 DSC 安装 SQL Server。这第二种技术建立在你在第 3 章学到的内容之上，我强烈建议你先阅读那一章。

## 安装实例

从命令行安装 SQL Server 涉及在 PowerShell 终端中运行 `setup.exe`。`setup.exe` 可以在 SQL Server 安装介质的根目录中找到。从 PowerShell 终端运行 `setup.exe` 时，你可以使用开关和参数来传递值，这些值将用于配置实例。

### 必需参数

虽然许多开关和参数是可选的，但有些必须始终包含。当你安装独立的数据库引擎实例时，表 6-1 中列出的参数始终是必需的。

表 6-1

必需参数

| 参数 | 用途 |
| --- | --- |
| `/IACCEPTSQLSERVERLICENSETERMS` | 确认你接受 SQL Server 许可条款 |
| `/SUPPRESSPRIVACYSTATEMENTNOTICE` | 防止显示隐私声明 |
| `/ACTION` | 指定你想要执行的操作，例如安装或升级 |
| `/FEATURES or /ROLE` | 指定你希望安装的功能 |
| `/INSTANCENAME` | 分配给实例的名称 |
| `/SQLSYSADMINACCOUNTS` | 将在数据库引擎实例中被授予管理权限的 Windows 安全上下文 |
| `/AGTSVCACCOUNT` | 用于运行 SQL Agent 服务的帐户 |
| `/AGTSVCPASSWORD` | 如果不是组管理服务帐户，则此参数用于指定 SQL Agent 服务帐户的密码 |
| `/SQLSVCACCOUNT` | 用于运行数据库引擎服务的帐户 |
| `/SQLSVCPASSWORD` | 如果不是 gMSA 或虚拟帐户，则此参数用于指定数据库引擎服务帐户的密码 |
| `/qs` | 执行无人参与安装。在 Windows Server Core 上是必需的，因为不支持安装向导 |

#### IACCEPTSQLSERVERLICENSETERMS 开关

因为 `/IACCEPTSQLSERVERLICENSETERMS` 是一个简单的开关，表示你接受许可条款，所以它不需要传递任何参数值。


### ACTION 参数

当执行独立实例的基本安装时，传递给`/ACTION`参数的值将是`install`；但是，`/ACTION`参数所有可能的值如表 6-2 所示。

**表 6-2**

`/ACTION` 参数接受的值

| 值 | 用途 |
| --- | --- |
| `Install` | 安装独立实例 |
| `PrepareImage` | 准备一个原始的独立镜像，不包含账户、计算机或网络的特定详细信息 |
| `CompleteImage` | 通过添加账户、计算机和网络详细信息来完成已准备的独立镜像的安装 |
| `Upgrade` | 从 SQL Server 2012、2014、2017 或 2019 升级实例 |
| `EditionUpgrade` | 将 SQL Server 2022 从较低版本（如 Developer Edition）升级到较高版本（如 Enterprise） |
| `Repair` | 修复损坏的实例 |
| `RebuildDatabase` | 重建损坏的系统数据库 |
| `Uninstall` | 卸载独立实例 |
| `InstallFailoverCluster` | 安装故障转移群集实例 |
| `PrepareFailoverCluster` | 准备一个原始的群集镜像，不包含账户、计算机或网络的特定详细信息 |
| `CompleteFailoverCluster` | 通过添加账户、计算机和网络详细信息来完成已准备的群集镜像的安装 |
| `AddNode` | 向故障转移群集添加节点 |
| `RemoveNode` | 从故障转移群集中移除节点 |

### FEATURES 参数

如表 6-3 所示，`/FEATURES` 参数用于指定一个逗号分隔的列表，该列表包含安装程序将要安装的功能，但并非所有功能都可在 Windows Server Core 上使用。

**表 6-3**

`/FEATURES` 参数的可接受值

| 参数值 | 在 Windows Core 上使用 | 描述 |
| --- | --- | --- |
| `SQL` | 否 | 完整的 SQL 引擎，包括全文搜索、复制和数据质量服务器 |
| `SQLEngine` | 是 | 数据库引擎 |
| `FullText` | 是 | 全文搜索 |
| `Replication` | 是 | 复制组件 |
| `DQ` | 否 | 数据质量服务器 |
| `PolyBase` | 是 | PolyBase 组件 |
| `AdvancedAnalytics` | 是 | 机器学习解决方案和数据库内 R 服务 |
| `AS` | 是 | Analysis Services |
| `DQC` | 否 | 数据质量客户端 |
| `IS` | 是 | Integration Services |
| `MDS` | 否 | 主数据服务 |
| `Tools` | 否 | 所有客户端工具 |
| `BC` | 否 | 向后兼容组件 |
| `Conn` | 是 | 连接组件 |
| `DREPLAY_CTLR` | 否 | 分布式重播控制器 |
| `DREPLAY_CLT` | 否 | 分布式重播客户端 |
| `SNAC_SDK` | 是 | 客户端连接 SDK |
| `SDK` | 否 | 客户端工具 SDK |
| `LocalDB` | 是 | SQL Server Express 的一种执行模式，供应用程序开发人员使用 |
| `IS_MASTER` | 是 | Integration Services Master，用于横向扩展的 SSIS 部署 |
| `IS_WORKER` | 是 | Integration Services Worker，用于横向扩展的 SSIS 部署 |
| `AZUREEXTENSION` | 是 | 将实例与 Azure ARC 集成以实现集中管理 |

> **注意**
> 如果你选择安装其他 SQL Server 功能，例如 Analysis Services 或 Integration Services，那么其他参数也会变为必需项。

### Role 参数

除了使用`/FEATURES`参数指定要安装的功能列表外，还可以使用`/ROLE`参数在预定义的角色中安装 SQL Server。`/ROLE`参数支持的角色详见表 6-4。

**表 6-4**

`/ROLE` 参数的可用值

| 参数值 | 描述 |
| --- | --- |
| `SPI_AS_ExistingFarm` | 在现有的 SharePoint 场中将 SSAS 安装为 PowerPivot 实例 |
| `SPI_AS_NewFarm` | 在新的、未配置的 SharePoint 场中安装数据库引擎和 SSAS 作为 PowerPivot 实例 |
| `AllFeatures_WithDefaults` | 安装 SQL Server 及其组件的所有功能。我建议不要使用此选项，除非是在最特殊的情况下，因为安装比实际需要更多的功能会增加 SQL Server 的安全足迹和资源利用足迹 |

### 基本安装

当你使用`setup.exe`的命令行参数时，应遵守表 6-5 中概述的语法规则。

**表 6-5**

命令行参数的语法规则

| 参数类型 | 语法 |
| --- | --- |
| 简单开关 | `/SWITCH` |
| 真/假 | `/PARAMETER=true/false` |
| 布尔值 | `/PARAMETER=0/1` |
| 文本 | `/PARAMETER="Value"` |
| 多值文本 | `/PARAMETER="Value1" "Value2"` |
| `/FEATURES` 参数 | `/FEATURES=Feature1,Feature2` |

> **提示**
> 对于文本参数，只有当值包含空格时才需要引号。但是，始终包含它们被认为是一种良好的做法。

假设你已经导航到安装介质的根目录，那么代码清单 6-1 提供了用于安装数据库引擎和复制的 PowerShell 语法。它为所有可选参数使用了默认值，但排序规则除外，我们将把它设置为 Windows 排序规则 `Latin1_General_CI_AS`。

```
.\SETUP.EXE /IACCEPTSQLSERVERLICENSETERMS /ACTION="Install" /FEATURES=SQLEngine,Replication /INSTANCENAME="SCRIPTING" /SQLSYSADMINACCOUNTS="Administrator" /SQLCOLLATION="Latin1_General_CI_AS" /qs /SUPPRESSPRIVACYSTATEMENTNOTICE
```

**代码清单 6-1**
从 PowerShell 安装 SQL Server

> **注意**
> 如果使用的是命令提示符而不是 PowerShell，则不需要前导的 `.\` 字符。

在此示例中，将安装一个名为 `SCRIPTING` 的 SQL Server 实例。由于未提供服务账户参数，数据库引擎和 SQL Agent 服务将在虚拟服务账户下运行。安装开始时，一个精简的、非交互式的安装向导版本将出现，以使你了解进度。


### 冒烟测试

在自动安装实例后，总是值得进行一些冒烟测试。这里的 `冒烟测试` 指的是快速、高层级的测试，旨在确保服务正在运行且实例可访问。

代码清单 6-2 中的脚本将使用 PowerShell 的 `Get-Service` cmdlet 来确认与 SCRIPTING 实例相关的服务是否存在，并检查其状态。此脚本使用星号作为通配符，返回所有包含我们实例名称的服务。这当然意味着，诸如 SQL Browser 之类的服务不会被返回。

```
Get-Service -displayname *SCRIPTING* | Select-Object name, displayname, status
代码清单 6-2
检查服务状态
```

你会注意到 SQL Server 和 SQL Agent 服务都已安装。你还可以看到 SQL Server 服务已启动，而 SQL Agent 服务已停止。这与我们的预期一致，因为我们没有为这两个服务使用启动模式参数。SQL Server 服务的默认启动模式是自动，而 SQL Agent 服务的默认启动模式是手动。

第二个推荐的冒烟测试是使用 `Invoke-Sqlcmd` 运行一个返回实例名称的 T-SQL 语句。要使用 `Invoke-Sqlcmd` cmdlet（或任何其他 SQL Server PowerShell cmdlet），需要安装 `sqlserver` PowerShell 模块。此模块取代了已弃用的 `SQLPS` 模块，并且包含更多 cmdlet。

一旦 `sqlserver` 模块安装完毕，就可以使用代码清单 6-3 中的脚本来返回实例名称。

> **注意**
>
> 这也是集群执行 `IsAlive` 测试时所使用的查询。它对系统影响很小，只是检查实例是否可访问。

```
Invoke-Sqlcmd –serverinstance "localhost\SCRIPTING" -query -TrustServerCertificate  "SELECT @@SERVERNAME"
代码清单 6-3
检查实例是否可访问
```

在此示例中，`-serverinstance` 开关用于指定你要连接到的实例名称，`-query` 开关指定了将要运行的查询。你应该能看到查询成功解析并返回了实例名称。

### 可选参数

有许多开关和参数可以可选地使用，以自定义你正在安装的实例的配置。可用于安装数据库引擎的可选开关和参数列于表 6-6 中。

> **提示**
>
> 如果使用的账户是 MSA/gMSA，则不应指定账户密码。这包括数据库引擎和 SQL Server Agent 服务的账户，否则这些账户是必填的。

表 6-6
可选参数



| 参数 | 用法 |
| --- | --- |
| `/AGTSVCSTARTUPTYPE` | 指定 SQL Server Agent 服务的启动模式。可设置为 `Automatic`、`Manual` 或 `Disabled` |
| `/BROWSERSVCSTARTUPTYPE` | 指定 SQL Server Browser 服务的启动模式。可设置为 `Automatic`、`Manual` 或 `Disabled` |
| `/CONFIGURATIONFILE` | 指定配置文件的路径，该文件包含一系列开关和参数，从而在运行安装程序时无需内联指定它们 |
| `/ENU` | 指定将使用英文版 SQL Server。如果您在具有本地化设置的服务器上安装英文版 SQL Server，并且介质包含英文和本地化操作系统的语言包，请使用此开关 |
| `/FILESTREAMLEVEL` | 用于启用 FILESTREAM 并设置所需的访问级别。可设置为 `0` 以禁用 FILESTREAM，`1` 以允许仅通过 SQL Server 连接，`2` 以允许 IO 流式处理，或 `3` 以允许远程流式处理。选项 `1` 到 `3` 是逐级构建的，因此指定级别 `3` 就隐式指定了级别 `1` 和 `2` |
| `/FILESTREAMSHARENAME` | 指定存储 FILESTREAM 数据的 Windows 文件共享的名称。当 `/FILESTREAMLEVEL` 设置为 `2` 或 `3` 时，此参数变为必需 |
| `/FTSVCACCOUNT` | 用于运行全文筛选器启动器服务的帐户 |
| `/FTSVCPASSWORD` | 用于运行全文筛选器启动器服务的帐户的密码 |
| `/HIDECONSOLE` | 指定应隐藏控制台 |
| `/INDICATEPROGRESS` | 使用此开关时，安装日志将在安装过程中输出到屏幕 |
| `/IACCEPTPYTHONLICENSETERMS` | 如果安装 Anaconda Python 包，并使用 `/q` 或 `/qs`，则必须指定此参数 |
| `/IACCEPTROPENLICENSETERMS` | 安装 Microsoft R 包，并使用 `/q` 或 `/qs` 时，必须指定此参数 |
| `/INSTANCEDIR` | 指定实例的文件夹位置 |
| `/INSTANCEID` | 指定实例的 ID。使用此参数被认为是不好的做法 |
| `/INSTALLSHAREDDIR` | 指定实例间共享的 64 位组件的文件夹位置 |
| `/INSTALLSHAREDWOWDIR` | 指定实例间共享的 32 位组件的文件夹位置。此位置不能与 64 位共享组件的位置相同 |
| `/INSTALLSQLDATADIR` | 指定实例数据的默认文件夹位置 |
| `/NPENABLED` | 指定是否应启用命名管道。可设置为 `0` 表示禁用，或 `1` 表示启用 |
| `/PID` | 指定 SQL Server 的 PID。除非介质已预置 PID，否则未指定此参数将导致安装评估版 |
| `/PBENGSVCACCOUNT` | 指定将用于运行 PolyBase 服务的帐户 |
| `/PBDMSSVCPASSWORD` | 指定将运行 PolyBase 服务的帐户的密码 |
| `/PBENGSVCSTARTUPTYPE` | 指定 PolyBase 的启动模式。可设置为 `Automatic`、`Manual` 或 `Disabled` |
| `/PBPORTRANGE` | 指定 PolyBase 服务要侦听的端口范围。必须包含至少六个端口 |
| `/PBSCALEOUT` | 指定数据库引擎是否是 PolyBase 横向扩展组的一部分 |
| `/SAPWD` | 指定 SA 帐户的密码。当使用 `/SECURITYMODE` 将实例配置为混合模式身份验证时，使用此参数。如果 `/SECURITYMODE` 设置为 `SQL`，则此参数变为必需 |
| `/SECURITYMODE` | 使用此参数（值为 `SQL`）来指定混合模式。如果不使用此参数，则将使用 Windows 身份验证 |
| `/SQLBACKUPDIR` | 指定 SQL Server 备份的默认位置 |
| `/SQLCOLLATION` | 指定实例将使用的排序规则 |
| `/SQLMAXMEMORY` | 指定数据库引擎（主要用于缓冲区缓存）可以使用的最大 RAM 量 |
| `/SQLMINMEMORY` | 指定数据库引擎（主要用于缓冲区缓存）可以使用的最小 RAM 量 |
| `/SQLSVCSTARTUPTYPE` | 指定数据库引擎服务的启动模式。可设置为 `Automatic`、`Manual` 或 `Disabled` |
| `/SQLTEMPDBDIR` | 指定 TempDB 数据文件的文件夹位置 |
| `/SQLTEMPDBLOGDIR` | 指定 TempDB 日志文件的文件夹位置 |
| `/SQLTEMPDBFILECOUNT` | 指定应创建的 TempDB 数据文件的数量 |
| `/SQLTEMPDBFILESIZE` | 指定每个 TempDB 数据文件的大小 |
| `/SQLTEMPDBFILEGROWTH` | 指定 TempDB 数据文件的增长增量 |
| `/SQLTEMPDBLOGFILESIZE` | 指定 TempDB 日志文件的初始大小 |
| `/SQLTEMPDBLOGFILEGROWTH` | 指定 TempDB 日志文件的增长增量 |
| `/SQLUSERDBDIR` | 指定用户数据库数据文件的默认位置 |
| `/SQLUSERDBLOGDIR` | 指定用户数据库日志文件的默认文件夹位置 |
| `/SQLSVCINSTANTFILEINIT` | 指定应授予数据库引擎服务帐户“执行卷维护任务”权限。可接受的值是 `true` 或 `false` |
| `/TCPENABLED` | 指定是否将启用 TCP。使用值 `0` 表示禁用，`1` 表示启用 |
| `/UPDATEENABLED` | 指定是否将使用产品更新功能。传递值 `0` 表示禁用，`1` 表示启用 |
| `/UPDATESOURCE` | 指定产品更新搜索更新的位置。值 `MU` 将搜索 Windows Update，但您也可以传递文件共享或 UNC 路径 |



### 产品更新

“产品更新”功能取代了 SQL Server 已弃用的流式安装功能，它使您能够在安装 SQL Server 基础二进制文件的同时，安装最新的 `CU`（累积更新）或 `GDR`（常规分发版本 - 针对安全问题的修复程序）。此功能可以节省数据库管理员在安装 SQL Server 实例后立即安装最新更新所需的时间和精力，也有助于确保新部署环境的补丁级别保持一致。

提示：从 SQL Server 2017 开始，不再发布服务包。所有更新均为 `CU` 或 `GDR`。

要使用此功能，必须在命令行安装时使用两个参数。第一个是 `/UPDATEENABLED` 参数。您应将其值指定为 `1` 或 `True`。第二个是 `/UPDATESOURCE` 参数。此参数将告诉安装程序在哪里查找产品更新。如果向此参数传递值 `MU`，则安装程序将检查 Microsoft Update 或 WSUS 服务；或者，您也可以提供一个文件夹的相对路径或网络共享的 `UNC`（统一命名约定）路径。

在下面的示例中，我们将研究如何使用此功能来安装包含 `CU1` 的 SQL Server 2022，该更新位于一个网络共享中。当您下载 `GDR` 或 `CU` 时，它们通常会被打包在一个自解压可执行文件中。这非常有用，因为即使您的环境中未使用 WSUS，一旦您批准了新的补丁级别，您只需简单地替换网络共享中的 `CU`；这样做之后，所有新部署都能接收最新的更新，而无需更改您用于构建新实例的 PowerShell 脚本。

清单 6-4 中的 PowerShell 命令将安装一个名为 `SCRIPTING2` 的 SQL Server 实例，同时安装位于文件服务器上的 `CU1`。

注意：您用于运行安装的帐户需要具有对文件共享的访问权限。

```powershell
.\SETUP.EXE /IACCEPTSQLSERVERLICENSETERMS /ACTION="Install" /FEATURES=SQLEngine,Replication /INSTANCENAME="SCRIPTING2" /SQLSVCACCOUNT="MyDomain\SQLServiceAccount1" /SQLSVCPASSWORD="Pa$$w0rd" /AGTSVCACCOUNT="MyDomain\SQLServiceAccount1" /AGTSVCPASSWORD="Pa$$w0rd" /SQLSYSADMINACCOUNTS="MyDomain\SQLDBA" /UPDATEENABLED=1 /UPDATESOURCE="\\192.168.183.1\SQL2022_CU1\" /qs
```
清单 6-4：在安装过程中安装 `CU`

清单 6-5 中的代码演示了如何检查两个实例之间的差异。该代码使用 `Invoke-sqlcmd` 连接到 `SCRIPTING2` 实例，并返回包含实例完整版本详细信息（包括内部版本号）的系统变量。实例名称也包含在内，以帮助我们轻松识别结果。

```powershell
$parameters = @{
ServerInstance = 'localhost\SCRIPTING2'
TrustServerCertificate = $true
Query               = "
SELECT
@@SERVERNAME
, @@VERSION
"
}
Invoke-sqlcmd @parameters | Format-List
```
清单 6-5：确定每个实例的内部版本

### 使用配置文件

清单 6-6 中的示例是一个配置文件的内容，其中已填充了安装名为 `EXPERTSCRIPTING3` 的实例所需的所有必需参数。它还包含启用命名管道和 `TCP/IP`、在可通过 `T-SQL` 访问的级别启用 `FILESTREAM`、将 `SQL Agent` 服务设置为自动启动以及将排序规则配置为 `Latin1_General_CI_AS` 的可选参数。在此 `.ini` 文件中，注释以行首的分号定义。

```ini
; SQL Server 2022 配置文件
[OPTIONS]
; 接受 SQL Server 许可协议
IACCEPTSQLSERVERLICENSETERMS
; 指定安装工作流，如 INSTALL、UNINSTALL 或 UPGRADE。
; 此为必需参数。
ACTION="Install"
; 安装程序将仅显示进度，无需任何用户交互。
QUIETSIMPLE="True"
; 指定要安装、卸载或升级的功能。
FEATURES=SQLENGINE,REPLICATION
; 指定默认实例或命名实例。MSSQLSERVER 是非 Express 版本的默认实例，
; SQLExpress 是 Express 版本的默认实例。安装 SQL Server 数据库引擎 (SQL)、
; 分析服务 (AS) 时，此参数为必需。
INSTANCENAME="EXPERTSCRIPTING3"
; 代理帐户名称
AGTSVCACCOUNT="MyDomain\SQLServiceAccount1"
; 代理帐户密码
AGTSVCPASSWORD="Pa$$w0rd"
; 安装后自动启动服务。
AGTSVCSTARTUPTYPE="Automatic"
; 要启用的 FILESTREAM 功能级别 (0, 1, 2 或 3)。
FILESTREAMLEVEL="1"
; 指定用于数据库引擎的 Windows 排序规则或 SQL 排序规则。
SQLCOLLATION="Latin1_General_CI_AS"
; SQL Server 服务的帐户：域\用户或系统帐户。
SQLSVCACCOUNT="MyDomain\SQLServiceAccount1"
; SQL Server 服务帐户的密码。
SQLSVCPASSWORD="Pa$$w0rd"
; 要配置为 SQL Server 系统管理员的 Windows 帐户。
SQLSYSADMINACCOUNTS="MyDomain\SQLDBA"
; 指定 0 为禁用，1 为启用 TCP/IP 协议。
TCPENABLED="1"
; 指定 0 为禁用，1 为启用命名管道协议。
NPENABLED="1"
```
清单 6-6：用于 `SCRIPTING3` 的配置文件

假设此配置文件已保存为 `c:\SQL2022\configuration1.ini`，则可以使用清单 6-7 中的代码从 PowerShell 运行 `setup.exe`。

```powershell
.\setup.exe /CONFIGURATIONFILE="c:\SQL2022\Configuration1.ini"
```
清单 6-7：使用配置文件安装 SQL Server

尽管这是配置文件的完全有效的用法，但您实际上可以更巧妙地利用此方法创建一个可重用的脚本，该脚本可在任何服务器上运行，以帮助您引入一致的构建过程。如果您的 Windows 运维团队尚未采用 `sysprep` 或使用其他方法来构建服务器，此方法特别有用。

在清单 6-8 中，您将看到另一个配置文件。然而，这次它只包含您期望在整个环境中保持一致的静态参数。会因每次安装而变化的参数（例如实例名称和帐户详细信息）已被省略。



## SQL Server 2022 安装配置

```
;SQL Server 2022 配置文件
[OPTIONS]
; 接受 SQL Server 许可协议
IACCEPTSQLSERVERLICENSETERMS
; 指定安装工作流，例如 INSTALL（安装）、UNINSTALL（卸载）或 UPGRADE（升级）。
; 此为必需参数。
ACTION="Install"
; 安装程序仅显示进度，不进行任何用户交互。
QUIETSIMPLE="True"
; 指定要安装、卸载或升级的功能。
FEATURES=SQLENGINE,REPLICATION
; 安装后自动启动服务。
AGTSVCSTARTUPTYPE="Automatic"
; 启用 FILESTREAM 功能的级别（0、1、2 或 3）。
FILESTREAMLEVEL="1"
; 指定数据库引擎使用的 Windows 排序规则或 SQL 排序规则。
SQLCOLLATION="Latin1_General_CI_AS"
; 要配置为 SQL Server 系统管理员的 Windows 帐户。
SQLSYSADMINACCOUNTS="MyDomain\SQLDBA"
; 指定 0 为禁用，1 为启用 TCP/IP 协议。
TCPENABLED="1"
; 指定 0 为禁用，1 为启用 Named Pipes 协议。
NPENABLED="1"
```
`Listing 6-8` `SCRIPTING4` 的配置文件

这意味着要成功安装该实例，你需要混合使用配置文件中的参数以及运行 `setup.exe` 命令时内联指定的参数，如`Listing 6-9`所示。此示例假设`Listing 6-8`中的配置已保存为 `C:\SQL2022\Configuration2.ini`，并将安装一个名为 `EXPERTSCRIPTING3` 的实例。

```
.\SETUP.EXE /INSTANCENAME="SCRIPTING4 /SQLSVCACCOUNT="MyDomain\SQLServiceAccount1" /SQLSVCPASSWORD="Pa$$w0rd" /AGTSVCACCOUNT="MyDomain\SQLServiceAccount1" /AGTSVCPASSWORD="Pa$$w0rd" /CONFIGURATIONFILE="C:\SQL2022\Configuration2.ini"
```
`Listing 6-9` 使用混合参数和配置文件安装 SQL Server

## 通过脚本进行自动化安装

这种方法的好处在于，我们拥有一个一致的配置文件，无需在每次构建新实例时都进行修改。然而，这个想法可以更进一步。如果我们把 PowerShell 命令保存为 PowerShell 脚本，那么就可以运行该脚本并传入参数，而不必每次都重写命令。这将为我们提供一个用于构建新实例的一致脚本，我们可以将其置于变更控制之下。`Listing 6-10`中的代码演示了如何构建一个参数化的 PowerShell 脚本，它将使用相同的配置文件。该脚本假设 `D:\` 是安装媒体的根文件夹。

```
param(
    [string]       $InstanceName,
    [PSCredential] $SQLServiceAccountCredential,
    [PSCredential] $AgentServiceAccountCredential
)
$params = @(
    '/INSTANCENAME="{0}"' -f $InstanceName
    '/SQLSVCACCOUNT="{0}"' -f $SQLServiceAccountCredential.Username
    '/SQLSVCPASSWORD="{0}"' -f $SQLServiceAccountCredential.GetNetworkCredential().Password
    '/AGTSVCACCOUNT="{0}"' -f $AgentServiceAccountCredential.Username
    '/AGTSVCPASSWORD="{0}"' -f$AgentServiceAccountCredential.GetNetworkCredential().Password
    '/CONFIGURATIONFILE="C:\SQL2022\Configuration2.ini"'
)
Start-Process -FilePath 'C:\SQLServerSetup\SETUP.EXE' -ArgumentList $params -Wait -NoNewWindow
```
`Listing 6-10` 用于自动安装的 PowerShell 脚本

假设此脚本保存为 `SQLAutoInstall1.ps1`，则可以使用`Listing 6-11`中的命令来构建一个名为 `SCRIPTING5` 的实例。该命令运行 PowerShell 脚本，传入参数，然后这些参数在 `setup.exe` 命令中被使用。

```
./SQLAutoInstall.ps1 -InstanceName 'SCRIPTING5' -SQLServiceAccount 'MyDomain\SQLServiceAccount1' -SQLServiceAccountPassword 'Pa$$w0rd' -AgentServiceAccount 'MyDomain\SQLServiceAccount1' -AgentServiceAccountPassword 'Pa$$w0rd'
```
`Listing 6-11` 运行 `SQLAutoInstall.ps1`

### 增强安装例程

你还可以进一步扩展 `SQLAutoInstall.ps1` 脚本，并用它来结合你在第 1 章学到的操作系统组件配置技术，以及你在本章前面学到的执行冒烟测试的技术。

安装实例后，修改后的脚本（如`Listing 6-12`所示，我们称之为 `SQLAutoInstall2.ps1`）会使用 `powercfg` 设置高性能电源计划，并使用 `Set-ItemProperty` 将后台服务优先级置于前台应用程序之上。然后运行冒烟测试以确保 SQL Server 和 SQL Agent 服务都在运行，并且实例可访问。

```
param(
    [string]       $InstanceName,
    [PSCredential] $SQLServiceAccountCredential,
    [PSCredential] $AgentServiceAccountCredential
)

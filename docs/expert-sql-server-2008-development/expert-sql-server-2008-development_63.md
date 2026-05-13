# 第五章 特权与授权

*图 5-1. 分层安全提供针对攻击的多级保护。*

## 使用模式组织数据

例如，通过在存储过程中包含日志记录代码来记录对敏感数据的每一次访问是轻而易举的。同样，存储过程可用于强制用户以细粒度的方式访问数据，方法是要求提供用作谓词来过滤数据的参数。如果不使用存储过程来封装数据访问逻辑，这些安全检查很难或不可能强制调用者执行。

SQL Server 2008 支持 ANSI 标准模式，它提供了一种将表和其他对象分段为逻辑组的方法。模式本质上是容器，任何数据库对象都可以放入其中，并且可以对模式中的所有项目统一批量应用某些操作或规则。这使得诸如管理授权之类的任务变得相当容易，因为通过将数据库划分为模式，你可以轻松地对相关对象进行分组并控制权限，而不必担心将来可能会向该集合中添加或移除哪些对象。随着新对象添加到模式中，现有权限会传播，从而允许你为给定模式设置一次访问权限，而不必在数据库发生变化时再次操作它们。

要创建模式，请使用 `CREATE SCHEMA` 命令。以下 T-SQL 创建了一个名为 `Sales` 的模式：

```
CREATE SCHEMA Sales;
GO
```

你也可以选择使用 `AUTHORIZATION` 子句指定模式所有者。如果未显式指定所有者，SQL Server 会将所有权分配给创建该模式的用户。

模式创建后，你就可以开始在模式内使用双部分命名法创建数据库对象，如下所示：

```
CREATE TABLE Sales.SalesData
(
SaleNumber int,
SaleDate datetime
);
GO
```

如果一个对象属于某个模式，那么必须使用其关联的模式名称来引用它；因此，要从 `SalesData` 表中进行选择，需要使用以下 SQL：

```
SELECT *
FROM Sales.SalesData;
GO
```

**注意** 在早期版本的 SQL Server 中，对表的引用以其所有者名称为前缀（例如，`Owner.SalesData`）。此语法已弃用，SQL Server 2008 中的双部分命名引用的是模式而非对象所有者。

当需要向模式中的对象应用权限时，模式的优越性就显而易见了。假设从权限的角度来看，每个对象都应被同等对待，那么只需一次授权即可授予用户访问模式内每个对象的权限。例如，运行以下 T-SQL 后，用户 `Alejandro` 将拥有从 `Sales` 模式中的每个表选择行的权限，即使以后添加了新表也是如此：

```
CREATE USER Alejandro
WITHOUT LOGIN;
GO

GRANT SELECT ON SCHEMA::Sales
TO Alejandro;
GO
```

需要注意的是，在最初创建时，模式中任何对象的所有者都将与模式本身的所有者相同。个别对象的所有者以后可以更改，但在大多数情况下，我建议你保持给定模式中的所有内容由同一用户拥有。这对于本章稍后将介绍的所有权链尤其重要。要显式设置对象的所有者，需要使用 `ALTER AUTHORIZATION` 命令，如以下 T-SQL 所示：

```
--创建一个用户
CREATE USER Javier
WITHOUT LOGIN;
GO

--创建一个表
CREATE TABLE JaviersData
(
SomeColumn int
);
GO

--将 Javier 设置为该表的所有者
ALTER AUTHORIZATION ON JaviersData
TO Javier;
GO
```

关于模式的最后一点说明是，还有一个命令可用于在模式之间移动对象。通过将 `ALTER SCHEMA` 与 `TRANSFER` 选项一起使用，你可以指定某个表应移动到另一个模式：

```
--创建一个新模式
CREATE SCHEMA Purchases;
GO

--将 SalesData 表移入新模式
ALTER SCHEMA Purchases
TRANSFER Sales.SalesData;
GO
```



--通过其新架构名称引用表

```sql
SELECT *
FROM Purchases.SalesData;
GO
```

架构是一项强大的功能，我建议你在处理彼此紧密相关的表集合时，考虑使用它们。那些使用多个数据库来在对象之间创建逻辑边界的旧式数据库应用程序也可能从架构中受益。可以将这些多个数据库整合为一个使用架构的单个数据库。好处在于，相同的逻辑边界依然存在，但由于对象位于同一数据库中，它们可以参与声明式引用完整性，并且可以一起进行备份。

## 使用 EXECUTE AS 进行基本模拟

在 SQL Server 中，切换到不同用户的执行上下文早已成为可能，可以使用 `SETUSER` 命令，如下面的代码清单所示：

```sql
SETUSER 'Alejandro';
GO
```

要恢复到之前的上下文，再次调用 `SETUSER` 但不指定用户名：

```sql
SETUSER;
GO
```

`SETUSER` 命令仅对 `sysadmin` 或 `db_owner` 角色的成员（分别在服务器和数据库级别）可用，因此对于设置最低权限场景并不实用。

此外，尽管 SQL Server 2008 仍然实现了该命令，但 Microsoft 联机丛书文档指出，未来版本的 SQL Server 可能不再支持 `SETUSER`，并建议改用 `EXECUTE AS` 命令。

`EXECUTE AS` 命令可供任何用户使用，并且模拟给定用户或服务器登录名的访问权限由权限设置控制，而非固定角色。与 `SETUSER` 相比的另一个好处是，`EXECUTE AS` 会在模块结束时自动恢复到原始上下文。而 `SETUSER` 则在控制权返回给调用者时，使模拟的上下文保持活动状态。这意味着无法使用 `SETUSER` 在存储过程内封装模拟操作，并保证调用者无法接管模拟的凭据。

## 第 5 章 „ 特权与授权

为了展示 `EXECUTE AS` 的效果，首先创建一个新用户和一个由该用户拥有的表：

```sql
CREATE USER Tom WITHOUT LOGIN;
GO

CREATE TABLE TomsData
(
    AColumn int
);
GO

ALTER AUTHORIZATION ON TomsData TO Tom;
GO
```

创建用户后，可以使用 `EXECUTE AS` 模拟它，并可以使用 `USER_NAME()` 函数验证模拟上下文：

```sql
EXECUTE AS USER = 'Tom';
GO

SELECT USER_NAME();
GO
```

„ `注意` 要使用 `EXECUTE AS` 语句模拟另一个用户或登录名，必须授予用户对指定目标的 `IMPERSONATE` 权限。

`SELECT` 语句返回值 `Tom`，表明这是当前被模拟的用户。

执行 `EXECUTE AS` 之后进行的任何操作都将使用 `Tom` 的凭据。例如，该用户可以修改 `TomsData` 表，因为 `Tom` 拥有该表。但是，尝试创建新表将会失败，因为 `Tom` 没有执行此操作的权限：

```sql
--此语句将成功
ALTER TABLE TomsData
ADD AnotherColumn datetime;
GO

--此语句将因“CREATE TABLE 权限被拒绝”而失败
CREATE TABLE MoreData
(
    YetAnotherColumn int
);
GO
```

在 `Tom` 权限上下文中完成数据库操作后，可以使用 `REVERT` 命令返回到外部上下文。如果在该上下文中模拟了另一个用户（即多次调用了 `EXECUTE AS`），则需要多次调用 `REVERT` 才能将上下文返回到你的登录名。可以随时检查 `USER_NAME()` 函数，以了解你正在谁的上下文中执行。

要查看嵌套模拟的效果，首先确保从 `Tom` 的上下文中恢复出来，然后按如下所示创建第二个用户。将使用 `GRANT IMPERSONATE` 授予该用户模拟 `Tom` 的权利：

```sql
CREATE USER Paul WITHOUT LOGIN;
GO

GRANT IMPERSONATE ON USER::Tom TO Paul;
GO
```



如果模拟了`Paul`，则会话将没有权限从`TomsData`表中选择行。为了获得这些权限，必须在`Paul`的上下文中模拟`Tom`：

```
EXECUTE AS USER='Paul';

GO

--Fails

SELECT *

FROM TomsData;

GO

EXECUTE AS USER='Tom';

GO

--Succeeds

SELECT *

FROM TomsData;

GO

REVERT;

GO

--Returns 'Paul' -- REVERT must be called again to fully revert

SELECT USER_NAME();

GO

REVERT;

GO
```

最重要的是要理解，当调用`EXECUTE AS`时，所有操作都将如同你以被模拟用户身份登录一样运行。除了获得被模拟用户拥有而外部用户所缺乏的权限外，你还将失去外部用户拥有而被模拟用户所缺乏的权限。

出于日志记录目的，有时记录实际登录的主体非常重要。由于`USER_NAME()`函数和`SUSER_NAME()`函数都将返回与被模拟用户关联的名称，因此必须使用`ORIGINAL_LOGIN()`函数来返回最外层服务器登录的名称。使用`ORIGINAL_LOGIN()`将允许你获取登录的服务器主体的名称，无论其模拟作用域嵌套多深。


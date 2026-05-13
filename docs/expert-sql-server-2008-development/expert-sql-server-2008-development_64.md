# 第五章：权限与授权

## 什么是模块？

接下来的每个权限提升示例都使用存储过程来演示特定的功能元素。但是，请注意，这些方法适用于`SQL Server`支持的任何类型的模块，而不仅仅是存储过程。模块被定义为可以在`SQL Server`内部创建的任何类型的代码容器：存储过程、视图、用户定义函数、触发器或 CLR 程序集。

## 所有权链

保护`SQL Server`资源的最常见方法是拒绝数据库用户对`SQL Server`资源的任何直接访问，仅通过存储过程或视图提供访问权限。如果数据库用户有权执行某个存储过程，并且该存储过程的所有者与存储过程中引用的资源的所有者是同一个数据库用户，那么执行该存储过程的用户将通过该存储过程获得对该资源的访问权限。这被称为所有权链。

为了说明，首先创建并切换到一个新数据库：

```
CREATE DATABASE OwnershipChain;

GO

USE OwnershipChain;

GO
```

现在创建两个数据库用户`Louis`和`Hugo`：

```
CREATE USER Louis

WITHOUT LOGIN;

GO

CREATE USER Hugo

WITHOUT LOGIN;

GO
```

> **注意**：对于本章此后续示例，你应该使用属于`sysadmin`服务器角色成员的登录名连接到`SQL Server`。

请注意，这两个用户都是使用`WITHOUT LOGIN`选项创建的，这意味着尽管这些用户存在于数据库中，但它们不与任何`SQL Server`登录绑定，因此无法通过登录服务器来以他们中的任何一个身份进行身份验证。此选项是创建前述代理用户的一种方法。

创建用户后，创建一个由`Louis`拥有的表：

```
CREATE TABLE SensitiveData

(

IntegerData int

);

GO

ALTER AUTHORIZATION ON SensitiveData TO Louis;

GO
```

此时，`Hugo`无法访问该表。为了在不授予对该表的直接权限的情况下创建访问路径，可以创建一个同样由`Louis`拥有的存储过程：

```
CREATE PROCEDURE SelectSensitiveData

AS

BEGIN

SET NOCOUNT ON;

SELECT *

FROM dbo.SensitiveData;

END;

GO

ALTER AUTHORIZATION ON SelectSensitiveData TO Louis;

GO
```

此时，`Hugo`对该表仍然没有任何权限；需要授予该用户执行存储过程的权限：

```
GRANT EXECUTE ON SelectSensitiveData TO Hugo;
```

此时，`Hugo`可以执行`SelectSensitiveData`存储过程，从而从`SensitiveData`表中进行选择。然而，这仅在满足以下条件时才有效：

1.  表和存储过程都由同一用户（本例中为`Louis`）拥有。
2.  表和存储过程位于同一数据库中。




# 游标基础

当应用程序执行一个查询时，SQL Server 会返回一个由行组成的数据集。通常，应用程序无法一次性处理多行数据；它们通过遍历 SQL Server 返回的结果集来一次处理一行。这一功能由 `cursor`（游标）提供，它是一种从多行结果集中一次处理一行的机制。

T-SQL 游标处理通常涉及以下步骤：

1.  声明游标，将其与一条 `SELECT` 语句关联，并定义游标的特征。
2.  打开游标以访问 `SELECT` 语句返回的结果集。
3.  从游标中提取一行。可选择通过游标修改该行。
4.  移动到结果集中的其他行。
5.  一旦结果集中的所有行都被处理完毕，关闭游标并释放分配给游标的资源。

你可以使用 T-SQL 语句或用于连接到 SQL Server 的数据访问层来创建游标。使用数据访问层创建的游标通常称为 `client`（客户端）游标。用 T-SQL 编写的游标称为 `server`（服务器端）游标。以下是一个服务器端游标处理表中查询结果的示例：

```
--将 SELECT 语句关联到游标并定义游标的特性
USE AdventureWorks2017;
GO
SET NOCOUNT ON
DECLARE MyCursor CURSOR /**/
FOR
SELECT adt.AddressTypeID,
adt.Name,
adt.ModifiedDate
FROM Person.AddressType AS adt;
--打开游标以访问 SELECT 语句返回的结果集
OPEN MyCursor;
--一次从 SELECT 语句返回的结果集中提取一行
DECLARE @AddressTypeId INT,
@Name VARCHAR(50),
@ModifiedDate DATETIME;
FETCH NEXT FROM MyCursor
INTO @AddressTypeId,
@Name,
@ModifiedDate;
WHILE @@FETCH_STATUS = 0
BEGIN
PRINT 'NAME =   ' + @Name;
--可选择通过游标修改该行
UPDATE Person.AddressType
SET Name = Name + 'z'
WHERE CURRENT OF MyCursor;
--在数据集中移动到其他行
FETCH NEXT FROM MyCursor
INTO @AddressTypeId,
@Name,
@ModifiedDate;
END
--关闭游标并释放分配给游标的所有资源
CLOSE MyCursor;
DEALLOCATE MyCursor;
```

游标的部分开销取决于其特性。SQL Server 和数据访问层提供的游标特性大致可分为三类。

*   `游标位置`：定义游标创建的位置。
*   `游标并发性`：定义游标与底层内容的隔离和同步程度。
*   `游标类型`：定义游标的特定特征。

在探讨游标成本之前，我将用几页的篇幅介绍游标的各项特性。你可以使用以下查询来撤销对 `Person.AddressType` 表的更改：

```
UPDATE Person.AddressType
SET Name = LEFT(Name, LEN(Name) - 1);
```

## 游标位置

根据其创建位置，游标可分为以下两类。

*   客户端游标
*   服务器端游标

T-SQL 游标总是在 SQL Server 上创建。但是，数据库 API 游标可以在客户端或服务器端创建。

##### 客户端游标

顾名思义，`客户端游标`是在运行应用程序的机器上创建的，无论该应用程序是服务、数据访问层还是用户前端。它具有以下特征：

*   在客户端机器上创建。
*   游标元数据保存在客户端机器上。
*   使用数据访问层创建。
*   适用于大多数数据访问层（OLEDB 提供程序和 ODBC 驱动程序）。
*   可以是仅向前游标或静态游标。

**注意**：游标类型，包括仅向前和静态类型，将在本章后面的“游标类型”部分进行描述。

#### 服务器端游标

`服务器端游标`是在 SQL Server 机器上创建的。它具有以下特征：

*   在服务器机器上创建。
*   游标元数据保存在服务器机器上。
*   使用数据访问层或 T-SQL 语句创建。
*   使用 T-SQL 语句创建的服务器端游标与 SQL Server 紧密集成。
*   可以是任何类型的游标。（游标类型将在本章后面解释。）

**注意**：客户端游标与服务器端游标的成本比较将在本章后面的“游标类型成本比较”部分介绍。

## 游标并发性

根据与底层内容所需的隔离和同步程度，游标可分为以下并发模型：

*   `只读`：不可更新的游标。
*   `乐观`：使用乐观并发模型的可更新游标（不对底层数据行保留锁）。
*   `滚动锁`：对任何要更新的数据行持有锁的可更新游标。

##### 只读

只读游标是不可更新的；不对基表持有锁。在提取游标行时，是否会对底层行获取（`S`）锁取决于连接的隔离级别以及用于游标 `SELECT` 语句中使用的任何锁提示。但是，一旦行被提取，默认情况下锁就会被释放。以下 T-SQL 语句创建一个只读 T-SQL 游标：

```
DECLARE MyCursor CURSOR READ_ONLY FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

尽可能使用最少的锁开销使得只读类型的游标更快、更安全。只需记住，你不能通过只读游标操作数据，这是为提升性能所做的牺牲。



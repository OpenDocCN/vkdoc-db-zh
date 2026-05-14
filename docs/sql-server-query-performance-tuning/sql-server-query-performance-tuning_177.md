# SQL Server 游标

```sql
FETCH NEXT FROM MyCursor INTO @AddressTypeId, @Name, @ModifiedDate;

WHILE @@FETCH_STATUS = 0

BEGIN

    PRINT 'NAME = ' + @Name;

    -- Optionally, modify the row through the cursor
    UPDATE Person.AddressType
    SET Name = Name + 'z'
    WHERE CURRENT OF MyCursor;

    FETCH NEXT FROM MyCursor
    INTO @AddressTypeId, @Name, @ModifiedDate;

END

-- Close the cursor and release all resources assigned to the cursor
CLOSE MyCursor;
DEALLOCATE MyCursor;
```

第 22 章 ■ 逐行处理

游标的开销部分取决于其特性。SQL Server 和数据访问层提供的游标特性可以大致分为三类。

*   *游标位置:* 定义游标创建的位置。
*   *游标并发性:* 定义游标与底层内容的隔离和同步程度。
*   *游标类型:* 定义游标的具体特性。

在查看游标的成本之前，我将用几页篇幅介绍游标的各项特性。

你可以用以下查询撤销对 `Person.AddressType` 表的更改：
```sql
UPDATE Person.AddressType
SET [Name] = LEFT([Name], LEN([Name]) - 1);
```

### 游标位置

根据其创建位置，游标可分为两类。

*   客户端游标
*   服务器端游标

T-SQL 游标总是在 SQL Server 上创建。而数据库 API 游标则可以在客户端或服务器端创建。

#### 客户端游标

顾名思义，*客户端游标*是在运行应用程序的机器上创建的，无论该应用程序是服务、数据访问层还是用户前端。它具有以下特点：

*   在客户端机器上创建。
*   游标元数据在客户端机器上维护。
*   使用数据访问层创建。
*   适用于大多数数据访问层（OLEDB 提供程序和 ODBC 驱动程序）。
*   可以是仅向前或静态游标。

■ **注意** 游标类型，包括仅向前和静态游标类型，将在本章后面的“游标类型”一节中描述。

#### 服务器端游标

*服务器端游标*是在 SQL Server 机器上创建的。它具有以下特点：

*   在服务器机器上创建。
*   游标元数据在服务器机器上维护。
*   使用数据访问层或 T-SQL 语句创建。
*   使用 T-SQL 语句创建的服务器端游标与 SQL Server 紧密集成。
*   可以是任何类型的游标。（游标类型将在本章后面解释。）

■ **注意** 客户端和服务器端游标之间的成本比较将在本章后面的“游标类型成本比较”一节中介绍。

### 游标并发性

根据与底层内容所需的隔离和同步程度，游标可分为以下几种并发模型：

*   *只读:* 不可更新的游标。
*   *乐观:* 使用乐观并发模型的可更新游标（不对底层数据行保留锁）。
*   *滚动锁:* 对任何要更新的数据行都持有锁的可更新游标。

#### 只读

只读游标是不可更新的；不对基表持有锁。在获取游标行时，是否在底层行上获取共享（S）锁取决于连接的隔离级别以及用于游标的 `SELECT` 语句中使用的任何锁提示。但是，一旦行被获取，默认情况下锁就会被释放。以下 T-SQL 语句创建一个只读 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR READ_ONLY
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

最低级别的锁定开销使得只读类型的游标更快、更安全。只需记住，你无法通过只读游标操作数据，这是你为性能所做的牺牲。

#### 乐观




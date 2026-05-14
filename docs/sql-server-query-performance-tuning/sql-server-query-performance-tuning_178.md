# 乐观并发模型

乐观并发模型允许游标更新。底层数据上不会持有锁。决定是否在底层行上获取（S）锁的因素与只读游标相同。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 第 22 章 ■ 逐行处理

乐观并发模型使用行版本控制来确定自该行被读入游标以来是否已被修改，而不是在将该行读入游标时锁定它。基于版本的乐观并发要求在创建游标的底层用户表中包含一个`ROWVERSION`列（以前是`TIMESTAMP`数据类型）。`ROWVERSION`数据类型是一个二进制数字，表示行的修改相对顺序。每次修改带有`ROWVERSION`列的行时，SQL Server 会将全局`ROWVERSION`值`@@DBTS`的当前值存储在`ROWVERSION`列中；然后递增`@@DBTS`值。

在通过乐观游标应用修改之前，SQL Server 会确定该行的当前`ROWVERSION`列值与它被读入游标时的`ROWVERSION`列值是否匹配。只有当`ROWVERSION`值匹配时，才会修改底层行，这表明该行在此期间未被其他用户修改。否则，将引发错误。如果发生错误，首先使用更新后的数据刷新游标。

如果底层表不包含`ROWVERSION`列，则游标默认使用基于值的乐观并发，这要求将行的当前值与读入游标时的值进行匹配。基于版本的并发控制比基于值的并发控制更高效，因为它需要更少的处理来确定底层行的修改。因此，为了获得具有乐观并发模型的游标的最佳性能，请确保底层表具有`ROWVERSION`列。

以下 T-SQL 语句创建一个乐观的 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR OPTIMISTIC
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

具有滚动锁并发的游标在底层行上持有（U）锁，直到获取另一个游标行或关闭游标。这可以防止其他用户在游标获取该行时修改底层行。滚动锁并发模型使游标可更新。

以下 T-SQL 语句创建一个具有滚动锁并发模型的 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR SCROLL_LOCKS
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

由于在引用的行上持有锁（直到获取另一个游标行或关闭游标），它会在此期间阻止所有其他试图修改该行的用户。这损害了数据库并发性。

### 游标类型

游标可分为以下四种类型：

*   前进式游标
*   静态游标
*   键集驱动游标
*   动态游标

我们将在接下来的章节中更详细地了解这四种类型。

[www.it-ebooks.info](http://www.it-ebooks.info/)

#### 前进式游标

前进式游标具有以下特征：

*   它们直接在基表上操作。
*   通常，直到使用游标`FETCH`操作获取游标行时，才会从底层表中检索行。但是，具有以下附加特征的数据库 API 前进式游标会首先从底层表中检索所有行：
    *   客户端游标位置
    *   服务器端游标位置和只读游标并发性。
*   它们仅支持通过游标向前滚动（`FETCH NEXT`）。
*   它们允许通过游标进行所有更改（`INSERT`、`UPDATE`和`DELETE`）。此外，这些游标反映了对底层表所做的所有更改。



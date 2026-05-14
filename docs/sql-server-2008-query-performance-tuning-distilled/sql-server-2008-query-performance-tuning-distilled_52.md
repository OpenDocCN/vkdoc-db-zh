# 乐观并发

基于值的乐观并发模型使游标可更新。不对底层数据持锁。决定是否在底层行上获取 (S) 锁的因素与只读游标相同。

乐观并发模型使用行版本控制来确定自该行被读入游标以来是否已被修改，而不是在读入游标时锁定该行。基于版本的乐观并发要求在创建游标的底层用户表中包含一个 `ROWVERSION` 列。`ROWVERSION` 数据类型是一个二进制数字，表示行上修改的相对顺序。每次修改包含 `ROWVERSION` 列的行时，SQL Server 会将全局 `ROWVERSION` 值 `@@DBTS` 的当前值存储在该行的 `ROWVERSION` 列中；然后递增 `@@DBTS` 值。

在通过乐观游标应用修改之前，SQL Server 会确定该行的当前 `ROWVERSION` 列值是否与读入游标时该行的 `ROWVERSION` 列值匹配。只有当 `ROWVERSION` 值匹配时，才修改底层行，这表明在此期间该行未被其他用户修改。否则，将引发错误。如果发生错误，请使用更新后的数据刷新游标。

如果底层表不包含 `ROWVERSION` 列，则游标默认使用基于值的乐观并发，这需要将行的当前值与读入游标时的值进行匹配。基于版本的并发控制比基于值的并发控制更高效，因为它确定底层行是否被修改所需的处理更少。因此，为了获得具有乐观并发模型的游标的最佳性能，请确保底层表具有 `ROWVERSION` 列。

以下 T-SQL 语句创建了一个乐观的 T-SQL 游标：
```sql
DECLARE MyCursor CURSOR OPTIMISTIC FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```


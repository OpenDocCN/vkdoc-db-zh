# 游标的前向特性与类型实现

数据库 API 游标和 T-SQL 游标对前向特性的实现方式不同。

数据访问层将前向游标特性实现为之前列出的四种游标类型之一。但 T-SQL 游标并非将前向游标特性作为一种游标类型来实现；相反，它将此特性实现为定义游标滚动行为的一种属性。因此，对于 T-SQL 游标，前向特性可用于定义其余三种游标类型的滚动行为。

#### FAST_FORWARD 游标

可以使用 `FAST_FORWARD` 语句创建具有只读属性的前向游标。T-SQL 语法提供了一个特定的游标类型选项 `FAST_FORWARD`，用于创建仅向前游标。`FAST_FORWARD` 游标的别名是“软管”，因为它是通过游标移动数据最快的方式，并且所有信息单向流动。但是，当“软管”仍然不如传统的基于集合的操作快时，请不要感到惊讶。以下 T-SQL 语句创建了一个仅向前 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR FAST_FORWARD
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

`FAST_FORWARD` 属性指定了一个启用了性能优化的、仅向前、只读的游标。

#### 静态游标

静态游标具有以下特征：

*   它们在打开游标时，在 `tempdb` 数据库中创建游标结果的快照。此后，静态游标在 `tempdb` 数据库中的快照上操作。
*   数据在打开游标时从基础表中检索。
*   静态游标支持所有滚动选项：`FETCH FIRST`、`FETCH NEXT`、`FETCH PRIOR`、`FETCH LAST`、`FETCH ABSOLUTE n` 和 `FETCH RELATIVE n`。
*   静态游标始终是只读的；不允许通过静态游标进行数据修改。此外，对基础表所做的更改（`INSERT`、`UPDATE` 和 `DELETE`）不会反映在游标中。

以下 T-SQL 语句创建了一个静态 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR STATIC
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

一些测试表明，静态游标的性能可以与仅向前游标相当，有时甚至更快。请务必在您自己的系统上测试此行为。

#### 键集驱动游标

键集驱动游标具有以下特征：

*   键集游标由一组称为键集的唯一标识符（或键）控制。键集由唯一标识结果集中行的一组列构建。
*   这些游标在打开游标时，在 `tempdb` 数据库中创建键集行。
*   游标中行的成员资格仅限于打开游标时在 `tempdb` 数据库中创建的键集行。
*   在获取游标行时，数据库引擎首先查看 `tempdb` 中的键集行，然后导航到基础表中的相应数据行以检索其余列。
*   它们支持所有滚动选项。
*   键集游标允许通过游标进行所有更改。在游标外部执行的 `INSERT` 不会反映在游标中，因为游标中行的成员资格仅限于打开游标时在 `tempdb` 中创建的键集行。通过游标执行的 `INSERT` 出现在游标末尾。对基础表执行的 `DELETE` 在游标导航到达已删除行时会引发错误。对基础表非键集列的 `UPDATE` 会反映在游标中。对键集列的 `UPDATE` 被视为对旧键值的 `DELETE` 和对新键值的 `INSERT`。如果更改使某行不符合成员资格或影响行的顺序，则该行不会消失或移动，除非关闭并重新打开游标。

以下 T-SQL 语句创建了一个键集驱动 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR KEYSET
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

#### 动态游标

动态游标具有以下特征：

*   动态游标直接在基础表上操作。
*   游标中行的成员资格不是固定的，因为它们直接在基础表上操作。
*   与仅向前游标一样，基础表中的行直到使用游标 `FETCH` 操作获取游标行时才会被检索。
*   动态游标支持除 `FETCH ABSOLUTE n` 之外的所有滚动选项，因为游标中行的成员资格不是固定的。
*   这些游标允许通过游标进行所有更改。此外，对基础表所做的所有更改都会反映在游标中。
*   动态游标不支持数据库 API 游标实现的所有属性和方法。诸如 `AbsolutePosition`、`Bookmark` 和 `RecordCount` 等属性，以及 `clone` 和 `Resync` 等方法，不受动态游标支持。相反，它们由键集驱动游标支持。

以下 T-SQL 语句创建了一个动态 T-SQL 游标：

```sql
DECLARE MyCursor CURSOR DYNAMIC
FOR
SELECT adt.Name
FROM Person.AddressType AS adt
WHERE adt.AddressTypeID = 1;
```

动态游标在所有情况下绝对是可能的最慢的游标。它获取更多的锁并保持更长时间，这极大地增加了其糟糕的性能。在设计系统时请考虑到这一点。


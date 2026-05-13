# 防御性编程原则与数据库最佳实践

防御性编程的核心原则之一，是识别出你的代码能正常运行的所有隐含假设。一旦这些假设被识别出来，就可以调整函数以消除对它们的依赖，或者明确地测试每个条件，并在条件不满足时做出预案。在某些情况下，“隐藏”的假设是由于代码不够明确而产生的。

为了说明这个概念，请看下面的代码清单，它创建并填充了 `Customers` 和 `Orders` 两个表：

```sql
CREATE TABLE Customers(
    CustID int,
    Name varchar(32),
    Address varchar(255));
INSERT INTO Customers(CustID, Name, Address) VALUES
    (1, 'Bob Smith', 'Flat 1, 27 Heigham Street'),
    (2, 'Tony James', '87 Long Road');
GO

CREATE TABLE Orders(
    OrderID INT,
    CustID INT,
    OrderDate DATE);
INSERT INTO Orders(OrderID, CustID, OrderDate) VALUES
    (1, 1, '2008-01-01'),
    (2, 1, '2008-03-04'),
    (3, 2, '2008-03-07');
GO
```

现在考虑以下查询，它从两个表中选取每个客户的订单列表：

```sql
SELECT
    Name,
    Address,
    OrderID
FROM
    Customers c
    JOIN Orders o ON c.CustID = o.CustID;
GO
```

查询成功执行，我们得到了预期的结果：

```
Bob Smith Flat 1, 27 Heigham Street 1
Bob Smith Flat 1, 27 Heigham Street 2
Tony James 87 Long Road 3
```

但隐藏的假设是什么？`SELECT` 查询中列出的列名没有用表名限定。如果将来表结构发生变化会怎样？假设向 `Orders` 表添加了一个 `Address` 列，以便为每个订单附加一个单独的送货地址，而不是依赖于 `Customers` 表中的地址：

```sql
ALTER TABLE Orders ADD Address varchar(255);
GO
```

现在，`SELECT` 查询中未限定的列名 `Address` 变得具有歧义。如果我们尝试再次运行原始查询，将会收到一个错误：

```
Msg 209, Level 16, State 1, Line 1
Ambiguous column name 'Address'.
```

由于没有识别并纠正原始代码中包含的隐藏假设，当向 `Orders` 表添加新列后，查询就中断了。一个可以防止此错误的简单做法是确保所有列名都带有适当的表名或别名前缀：

```sql
SELECT
    c.Name,
    c.Address,
    o.OrderID
FROM
    Customers c
    JOIN Orders o ON c.CustID = o.CustID;
GO
```

在上面的例子中，发现隐藏假设相当容易，因为 SQL Server 给出了描述性的错误信息，使任何开发人员都能快速定位并修复损坏的代码。然而，有时你可能就没那么幸运了，如下例所示。

假设你有一个 `MainData` 表，包含一些简单的值，如下面的代码清单所示：

```sql
CREATE TABLE MainData(
    ID int,
    Value char(3));
GO

INSERT INTO MainData(ID, Value) VALUES
    (1, 'abc'), (2, 'def'), (3, 'ghi'), (4, 'jkl');
GO
```

现在假设对 `MainData` 表的每一次更改都要记录在关联的 `ChangeLog` 表中。以下代码演示了这种结构，以及一个通过附加在 `MainData` 表上的 `UPDATE` 触发器自动填充 `ChangeLog` 表的机制：

```sql
CREATE TABLE ChangeLog(
    ChangeID int IDENTITY(1,1),
    RowID int,
    OldValue char(3),
    NewValue char(3),
    ChangeDate datetime);
GO

CREATE TRIGGER DataUpdate ON MainData
FOR UPDATE
AS
    DECLARE @ID int;
    SELECT @ID = ID FROM INSERTED;

    DECLARE @OldValue varchar(32);
    SELECT @OldValue = Value FROM DELETED;

    DECLARE @NewValue varchar(32);
    SELECT @NewValue = Value FROM INSERTED;

    INSERT INTO ChangeLog(RowID, OldValue, NewValue, ChangeDate)
        VALUES(@ID, @OldValue, @NewValue, GetDate());
GO
```

我们可以通过对 `MainData` 表运行一个简单的 `UPDATE` 查询来测试触发器：

```sql
UPDATE MainData SET Value = 'aaa' WHERE ID = 1;
GO
```



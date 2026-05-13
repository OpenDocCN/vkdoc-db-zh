# 索引

通常，SQL 标准不告诉 DBMS 如何完成其工作，并且，除其他外，它将内部存储和组织留给软件决定。

其后果之一是，数据可能不会以任何特别有用的顺序存储；它可能以对软件来说最快、最有效的方式存储。这就是为什么除非你使用`ORDER BY`子句强制排序，否则你不能确定`SELECT`语句的结果将以什么顺序出现。

这种安排的唯一缺点是，它使 DBMS 更难查找特定值，因为它可能在任何地方。

例如，如果你按姓氏搜索客户，它可能在任何地方，DBMS 可能很快找到它，也可能在几乎遍历完所有记录后才找到它。如果数据有序，查找会更快。

另一方面，如果数据按姓氏排序，那么在按电子邮件地址查找客户时就没有帮助。

解决方案是保留表本身，并维护一个单独的索引，这类似于书后的索引或图书馆目录。DBMS 不是搜索原始表，而是搜索索引。索引将包含数据的有序副本，以及对表中行位置的引用。

要创建索引，SQL 是：

```
CREATE INDEX name ON table(columns);
```

每个索引都有一个名称，并定义在特定的表上，使用一个或多个列。

在你的`SELECT`语句中，你不需要做任何不同的事情，所以你永远不需要担心它；如果存在索引并且 DBMS 认为值得使用，索引会被自动使用。如果表足够大，你应该会注意到搜索性能的提升。

索引确实需要数据库中额外的一点存储空间，并且确实需要一点额外的维护来跟上表的更改。因此，你不会索引所有内容：只索引那些你认为会经常搜索的列。

有两种类型的列你不需要索引。主键总是被索引，任何`UNIQUE`列也是如此。


## 向表中添加行

在这里，我们将操作其中一张表——`customers` 表。我们将添加和更改一些数据，并对表本身进行一些修改。

首先，我们将添加一个新客户。要添加行，请使用 `INSERT` 语句：

```
INSERT INTO table(columns)
VALUES (values);
```

例如：

```
INSERT INTO customers(givenname, familyname,
email, registered)
VALUES ('Norris', 'Lurker', 'norris.lurker@example.com',
current_timestamp);
```

`INSERT` 语句可用于添加新值（如此处所示），或从另一张表复制值。它通常包含一个列列表，但如果你承诺以正确的顺序填写所有列，则可以省略该列表。通常，包含列表更为可靠。

关于列列表，有一些明显的规则：

*   所有 `NOT NULL` 列*必须*被包含，除非它们有默认值。
*   值必须在数量（多少）和各自类型上与列匹配。
*   SQL 并不真的在乎你是否弄错了实际值，比如，假设，颠倒了 `givenname` 和 `familyname`，只要这些值是兼容的。

所有其他列将获得 `NULL` 值，但有一些例外。如果列定义包含 `DEFAULT`（例如 `spam` 列的 `false`），那么当你省略该列时，将使用该默认值。这包括任何带有默认值的 `NOT NULL` 列。这意味着你可以将它们排除在列列表之外。

如果你想强制使用 `DEFAULT` 列，也可以包含 `DEFAULT` 关键字作为虚拟值。然而，这在 SQLite 中不起作用。

即便如此：

*   当然，你也可以包含非 `NOT NULL` 的列。在这个例子中，我们没有这样做。
*   你也可以包含带有默认值的列，并为其指定一个替代值。

从技术上讲，主键 `id` 应该是 `NOT NULL` 的，因此原则上它应该被包含在列列表中。然而，在这个例子中，由于它是一个自动生成的数字，这起到了默认值的作用，因此你可以并且应该省略它。如果主键不是自动生成的，你则需要包含它。

现在再次运行 `INSERT` 语句，但更改电子邮件地址：

```
INSERT INTO customers(givenname, familyname,
email, registered)
VALUES ('Norris', 'Lurker', 'norris.lurker@example.net',
current_timestamp);
```

没错：你添加了同一个客户两次。你可以通过以下命令查看结果：

```
SELECT * FROM customers ORDER BY id DESC;
```

添加两个同名的客户在技术上并没有错，所以这可能不是错误。此时，SQL 无法帮助你决定你真的有两个同名的客户还是一个重复项，所以你必须通过其他方式来弄清楚；通常，这意味着亲自去追查他们。

请注意，即使是电子邮件地址也不能证明什么。尽管电子邮件地址不同（不允许重复的电子邮件地址），但这不会是第一个拥有多个电子邮件地址的用户。

## 从表中删除行

让我们假设第二个同名的客户确实是一个重复项，因此你需要删除它。为此，你使用：

```
DELETE
FROM table;
```

然而，你需要非常小心这个操作。它本身会删除*所有*行，这可能不是你的本意。如果你只想删除部分行，需要用 `WHERE` 子句来限定它：

```
DELETE
FROM table
WHERE …;
```

有时，你不确定是否针对了正确的行，当然也不想事后才发现。你可以通过将 `DELETE` 改为 `SELECT *` 来进行检查：

```
SELECT *
FROM table
WHERE …;
```

如果你确信那就是你的意图，你可以继续执行。如果你*仍然*不确定，你或许可以进行试运行。在一些数据库管理系统中，你可以*一次运行一条*以下语句：

```
--  PostgreSQL, MSSQL, MySQL / MariaDB:
BEGIN TRANSACTION;  --  MySQL / MariaDB: START TRANSACTION
DELETE FROM minipaintings;
SELECT * FROM minipaintings;
ROLLBACK;
SELECT * FROM minipaintings;
```

一个 `transaction`（事务）是一组允许你使用 `ROLLBACK`（回滚）撤销任何更改的语句。如果你想保留更改，则使用 `COMMIT`（提交）。本书中所有的数据库管理系统都支持事务^(¹¹)，但 SQLite 和 Oracle 不支持一次运行一条语句。

通常，单条语句会自动提交，这就是为什么删除（以及稍后将看到的更新）是如此危险的原因。

回到 `customers` 表，首先，检查你刚刚添加了哪些行：

```
SELECT * FROM customers ORDER BY id desc;
```

使用最后一个 id（在之前的逆序排列中是第一个）：

```
DELETE
FROM customers
WHERE id= … ;
```

现在，当你检查表时：

```
SELECT * FROM customers;
```

你应该只看到一个新的客户。

删除一个假定的重复项并非总是如此直接。有时，两行数据有不同的甚至冲突的数据，你需要决定保留什么、拒绝什么。

你应该在什么时候删除一行？简短的答案是永远不要。更详细的答案是，本该存在的数据应该保留。明显的例外是错误输入的数据（如本例）或测试数据。

另一个例外是，如果数据是试探性输入的，例如一笔可能无法成交的销售。如果你决定不想将其作为历史数据保留，之后可能会删除它。如果你真的有已经处理完毕的数据，那么你应该将其标记为过期（例如，使用另一个列）。

## 添加更多行

此时，你可以添加更多客户。对于大多数数据库管理系统，你可以使用单个 INSERT 语句添加多行：

```
--  Not Oracle:
INSERT INTO customers(givenname, familyname,
email, registered)
VALUES
('Sylvia','Nurke',
'sylvia.nurke@example.com', current_date),
('Murgatroyd','Murdoch',
'murgatroyd.murdoch@example.com', current_timestamp)
--  etc
;
```

然而，Oracle 不允许你这样做。对于 Oracle，有各种解决方法，但最简单的方法是使用传统的多个 `INSERT` 语句：

```
--  All DBMSs
INSERT INTO customers(givenname, familyname,
email, registered)
VALUES ('Sylvia','Nurke',
'sylvia.nurke@example.com', current_timestamp);
INSERT INTO customers(givenname, familyname,
email, registered)
VALUES ('Murgatroyd','Murdoch',
'murgatroyd.murdoch@example.com', current_timestamp);
--  etc
;
```

如果你有大量的行，多个单独的 `INSERT` 语句往往会变得非常慢，因此在 Oracle 中值得研究这些解决方法。示例数据库的脚本中充满了它们。

注意，`current_timestamp` 的值从技术上讲粒度太细，因为它还包括了时间。大多数（但并非全部）数据库管理系统有 `current_date`，这可能更合适。然而，数据库管理系统也会很乐意自动将 `current_timestamp` 转换以适应。

当你现在检查表时：

```
SELECT * FROM customers ORDER BY id DESC;
```

你会注意到重复客户所在位置有一个间隙。这里有两个原则共同作用：

1.  主键永远不应该被回收。这样，你就不会继承旧数据。
2.  自增列不会回绕，因此它们不会重复使用旧值。

可以覆盖这些原则，例如当你尝试修复数据时。


## 更新行

既然我们已经添加了更多客户，就可以开始补充一些缺失的细节了。比如，我们可以添加他们的电话号码。

要更改现有行中的值，请使用 `UPDATE` 语句：

```sql
UPDATE table
SET column=value    --  column=value, column=value, 等等
WHERE …
```

同样，你可以选择设置多个值。

例如，使用你新添加客户的某个 `id`：

```sql
SELECT id FROM customers ORDER BY id DESC;
```

你可以更新电话号码：

```sql
UPDATE customers
SET phone='0370101234'
WHERE id= … ;
```

由于 `id` 是主键，因此是唯一的，这将把影响限制在一行数据上。

`UPDATE` 语句可能非常危险。例如：

```sql
--  别这样做！
UPDATE customers
SET phone='0370101234';
```

这可能会毁了你的一天。没有 `WHERE` 子句，这将更改*所有*行的 `phone` 列。此时，电话号码无需是唯一的。

现在可能是反思 SQL 没有撤销功能的好时机。有时，犯错后你可以重建数据，有时数据库管理员可以重建。如果不能，那谁也无能为力。

你可以使用与 `DELETE` 语句相同的测试方法来测试你的 `UPDATE`。

与 `INSERT` 语句不同，你无法为每一行设置不同的值，除非该值可以被计算出来。

例如，你可以提高所有画作的价格：

```sql
--  除非你真想运行，否则别执行这个：
UPDATE paintings SET price=price*0.1;
```

这里，运行一次可以将价格提高 10%。唯一的问题是，如果你（不小心）再次运行它，价格将提高到 21%，然后是 33.1%，接着是 46.41%，嗯，你知道这样下去的结果。

电话号码的问题*可能*通过规定电话号码必须唯一来缓解。你可以使用 `CREATE UNIQUE INDEX` 语句修改现有表。例如：

```sql
--  非 MSSQL:
CREATE UNIQUE INDEX uq_customers_phone ON
customers(phone);
--  MSSQL:
CREATE UNIQUE INDEX uq_customers_phone ON
customers(phone)
WHERE phone IS NOT NULL;
```

Microsoft SQL 有个怪癖，会在检查唯一性时比较 `NULL` 值，因此你需要额外的子句来阻止它这样做。

请注意，不强制唯一电话号码*可能*有充分的理由，例如来自同一组织的多个客户。这类决策单凭数据库开发者无法做出——需要真正理解数据库在现实生活中将如何被使用。

## 修改表结构

一旦你有了一个表并开始填充数据，你可能会发现需要对表进行修改。此时删除并重建表为时已晚，但你可以使用 `ALTER` 语句在事后进行一些更改。

典型的更改包括添加或删除列、添加索引或更改列的类型。本章讨论的几乎所有表属性，只要在此过程中不违反现有的完整性规则，都可以追溯添加到现有表中。例如：

```sql
--  添加列
ALTER TABLE customers
ADD country VARCHAR(60);
--  删除列
ALTER TABLE customers
DROP COLUMN town, state, postcode;
--  添加外键
ALTER TABLE customers
ADD townid INT REFERENCES towns(id);
--  添加检查约束
ALTER TABLE saleitems
ADD CHECK (quantity>0);
--  删除约束
ALTER TABLE customers
DROP CONSTRAINT ck_customers_postcode;
--  添加 NOT NULL 约束
ALTER TABLE saleitems
ALTER COLUMN quantity SET NOT NULL;
--  添加默认值
ALTER TABLE saleitems
ALTER COLUMN quantity SET DEFAULT 1;
```

这里，我们将添加另一列。

虽然 `customers` 表包含地址，但不包含国家。如果你只想将销售限制在一个国家，这没问题；但如果你有更大的计划，你会想要添加国家：

```sql
--  所有 DBMS
ALTER TABLE customers
ADD country varchar(48);
--  MySQL / MariaDB
ALTER TABLE customers
ADD country varchar(48)
AFTER postcode;
```

新列将被放在最后。从美学上讲，它应该位于其他地址列之后。不幸的是，大多数 DBMS 不允许你指定新列出现的位置，除非使用一些风险稍高的技巧^(¹²)，所以我们只能接受它。正如你之前所见，MySQL/MariaDB 允许你指定位置。

现在，让我们假设你所有的客户都来自一个国家，但我们只对已有地址的客户做此假设。我们可以为这些客户设置国家，如下所示：

```sql
UPDATE customers
SET country='Australia'     --  或其他国家
WHERE state IS NOT NULL;
```

与所有 DDL 语句一样，你在现实中修改表的能力可能受到限制。数据库管理员可能不希望任何人随意改动数据库。

然而，对于示例数据库，我们假设你拥有完全控制权。

## 现实生活中的 DML

虽然你可能会花很多时间使用 `SELECT` 语句，但不太可能花太多时间使用 `INSERT`、`UPDATE` 或 `DELETE`。

`SELECT` 语句用于检索数据，除了安全和机密性问题外，它是一个相对无害的操作。一个简单的 `SELECT` 语句不会破坏任何东西。

另一方面，用其他语句搞乱你的数据库则*非常*容易。一旦你开始添加不应存在的数据，或者将某些内容更改为错误的值，或者删除了本应保留的数据，你的数据库就变得不可靠，因此也就毫无用处。

因此，数据库通常受到保护，以防止未经授权的篡改。

### 安全性

所有常规的 DBMS 都包含某种安全性。从这个意义上说，SQLite 不算常规，因为它的客户端内置于其他软件中，你想要实现的任何安全性都取决于宿主软件。如果你在使用 SQLite，可以忽略本节内容，或者礼貌地点点头，假装你感兴趣。

当你连接到数据库时，通常需要登录。有时登录是自动的，但通常需要你输入用户名和密码。

通常，安全性设置分为两层：

*   **用户**是拥有自己用户名和密码的个体。

*   **角色**是具有相似属性的用户集合。

一个用户可以被分配到多个角色。

通常，执行某项操作——任何操作——所需的权限被分配给一个角色。它们也可以分配给单个用户，但如果用户变动，这样维护起来会更困难。

这些权限包括（正如他们所说，但不限于）

*   可以使用 `SELECT` 语句读取哪些表

*   可以使用其他 DML 语句修改哪些表

*   是否可以更改表，或者是否可以创建或删除任何对象

根据 DBMS 的不同，这些权限可以非常细粒度。通常，它们由数据库管理员（DBA）控制。通常有一个具有此角色的特殊用户，称为 **root**。可以有多个 root 用户或 DBA。

如果你安装自己的 DBMS 软件，那么你就是 DBA，你可以做任何你想做的事情。



### 前端软件

你很少会直接使用`INSERT`、`UPDATE`或`DELETE`的另一个原因是，数据通常不是手动修改的。例如，示例数据库是为一个网络应用程序设计的，你通常会通过网页表单来进行所有的更改。在办公室环境中，使用类似的前端软件时，情况也是如此。

然而，无论数据实际如何输入，它最终都会以如前所述的 DML 语句的形式进入数据库。

例如，当你在网页表单上输入注册信息并提交时，数据会被网络服务器接收。接着，它会被服务器端脚本（通常用 PHP 等编程语言编写）处理，包装成 DML 语句，然后发送到数据库服务器进行进一步处理。

获取数据，例如查看目录，过程基本相同，但大多是反向的。提交搜索条件后，网络服务器脚本会生成一个合适的`SELECT`语句，用它从数据库服务器获取数据，适当地格式化结果，然后将其发送回用户。

另一方面，使用`SELECT`语句可以做的事情还有很多：

*   你可以像本书中一直做的那样，获取即席查询结果。
*   你可以获取数据，以便在统计或分析软件包中做进一步处理。

当然，即使你不直接使用其他 DML 语句，理解其背后的原理也非常重要。

## 总结

在本章中，我们探讨了如何创建数据库表以及如何在表中添加或更改数据。

表使用`CREATE TABLE`语句创建。该语句包括：

1.  列名
2.  数据类型
3.  其他表和列属性

表设计之后可以更改，例如添加触发器或索引。更严重的更改，如添加或删除列，可以使用`ALTER TABLE`语句来实现。

### 数据类型

数据主要有三种类型：

1.  数字
2.  字符串
3.  日期

上述类型有许多变体，这使得数据存储和处理更高效，并有助于验证数据值。

还有一些其他类型，如布尔型或二进制数据，在典型的数据库中不太常见。

### 约束

约束定义了哪些值被视为有效。标准约束包括：

*   `NOT NULL`
*   `UNIQUE`
*   `DEFAULT`
*   外键

你可以使用通用的`CHECK`约束来构建自己的附加约束。在这里，你可以添加一个类似于`WHERE`子句的条件，来定义你特定的验证规则。

### 外键

外键是对另一个表的引用，它也被视为一种约束，因为它将值限制为与另一个表中的值相匹配。

外键在子表中定义。

外键还会影响从父表中删除行的任何尝试。默认情况下，如果存在匹配的子行，则无法删除父行。但是，可以将其更改为将外键设置为`NULL`，或者将删除操作级联到所有子行。


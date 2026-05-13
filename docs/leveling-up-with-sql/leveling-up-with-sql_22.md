# 10. 更多技术：触发器、数据透视表和变量

在本书中，我们一直在探讨如何将 SQL 的知识和应用推向更深层次，并探索了多种技术，其中有些是新的，有些则不是。在审视某些技术时，特别是那些涉及聚合和公用表表达式的技术，我们也体会到了通过多层语句将 SQL 推向更深层次的感觉。在本章中，我们将稍微超越简单的 SQL，探索一些对 SQL 进行补充的技术。它们彼此之间没有直接关联，但都能让你在数据处理上做更多事情。

SQL 触发器是一些小的代码块，它们会在发生某些数据库事件时自动运行。我们将探讨其工作原理以及如何编写一个触发器。具体来说，我们将研究一个用于自动归档已删除数据的触发器。数据透视表是二维的聚合。它们允许你基于行和列数据构建汇总。我们将看一个准备待汇总数据的例子，以及如何生成数据透视表。变量是可以用来在语句之间保持值的临时数据片段。它们允许我们运行一组 SQL 语句，同时这些变量保存着从一个语句传递到另一个语句的中间值。在本章中，我们将学习在向多个表添加数据时，使用变量来保存临时值。

## 理解触发器

有时，一个简单的 SQL 查询并不足够。有时，你真正想要的是让一个查询触发一个或多个额外的查询。有时，你想要的就是一个 `触发器`。一个 `触发器` 是一小块代码，当数据库发生某些事情时，它会自动运行。触发器有多种类型，包括：

*   DML（数据操作语言）触发器在数据表发生某些更改时运行，例如执行 `INSERT`、`UPDATE` 或 `DELETE` 语句时。
*   DDL（数据定义语言）触发器在数据库结构发生更改时运行，例如执行 `CREATE`、`ALTER` 或 `DROP` 语句时。
*   登录触发器在用户登录时运行。

你可能使用 DDL 或登录触发器的一个原因是为了通过将活动存储在日志表中来跟踪活动。在这里，我们将更多地关注 DML 触发器。

触发器可用于弥补标准 DBMS 行为的一些不足。以下是一些可能需要触发器的情况：

*   你可能有一个活动表，希望每次进行更改时都更新一个日期列。你可以使用一个触发器来为每次插入或更新设置该列。
*   假设你有一个租赁表，其中输入了开始和结束日期。如果结束日期未输入，你希望其默认为开始日期。SQL 的默认功能没那么智能，但你可以设置一个触发器在插入新行时设置结束日期。
*   SQL 在通常意义上没有审计功能。你可以创建一个触发器，在每次添加、更新或删除行时，向日志表添加一些数据。

在这个例子中，我们将创建一个触发器，用于保存我们即将从 `sales` 表中删除的数据的副本。在之前的一些章节中，我们不得不面对这样一个事实：在 `sales` 表中，某些行的订单日期/时间是 `NULL`。可以推测，这些销售从未结账。到目前为止，我们一直相当宽容，不时地将它们过滤掉，但现在是时候处理它们了。我们可以按如下方式删除所有 `NULL` 销售：

```sql
--  还不是时候！
DELETE FROM sales WHERE ordered IS NULL;
```

请注意，`saleitems` 表到 `sales` 表有一个外键，如果有关联的任何项目，通常不允许删除销售记录。但是，如果你检查生成示例数据库的脚本，你会注意到 `ON DELETE CASCADE` 子句，它将自动删除孤立的销售项目。

你应该在什么时候删除数据？简短的答案是永不。更长的答案则更复杂。你会删除输入错误的数据，或者在完成测试后删除测试数据。在这个例子中，我们将删除 `ordered` 日期为 `NULL` 的销售记录；我们将假设该销售从未结账，并且客户永远不会回来完成它。然而，无论如何，我们会保留一份副本，以防万一。

大多数 DBMS 以非常相似的方式处理触发器，但也存在差异。我们将首先概述基本原理，然后介绍各个 DBMS 的具体细节。

### 触发器的一些基础知识

创建触发器的基本语法大致如下：

```sql
CREATE TRIGGER something
ON some_table
BEFORE DELETE
BEGIN
...
END
```

没有哪个 DBMS 的做法完全相同，但大致如此：

*   触发器当然有一个名称：`CREATE TRIGGER something`。
*   触发器附着在一个表上：`ON some_table`。
*   触发器附着在一个事件上。

该事件通常是 `BEFORE`、`AFTER` 或 `INSTEAD OF` 之一，后跟一个 DML 语句。在这个例子中，我们希望在数据被删除之前对旧数据做一些处理。对于示例触发器，我们将把旧数据复制到一个名为 `deleted_sales` 的表中。这意味着我们必须在数据消失之前获取到它。合适的事件是 `BEFORE DELETE`。

这会有点复杂，因为我们不仅想复制 `sales` 表的数据，还想复制 `saleitems` 表的数据。我们将通过把这些项目连接成一个字符串来实现。你真的不应该以这种方式保存多个项目，但对于归档来说已经足够好了，而且如果你需要，随时可以将其拆分。

归档表大致如下：

```sql
CREATE TABLE deleted_sales (
id INT PRIMARY KEY,     --  自增
saleid INT,
customerid INT,
items VARCHAR(255),
deleted_date TIMESTAMP  --  日期/时间
);
```

这个表已经创建好了。


### 准备待归档的数据

触发代码基本上会是一条 `INSERT` 语句，用于插入从 `sales` 和 `saleitems` 表中准备好的值。我们将在一个 `CTE` 中准备这些值：

```sql
--  PostgreSQL, MSSQL
WITH cte AS (
...
)
INSERT INTO deleted_sales(saleid, customerid, items, deleted_date)
SELECT saleid, customerid, items, current_timestamp
FROM cte;
--  MariaDB/MySQL, SQLite, Oracle
INSERT INTO deleted_sales(saleid, customerid, items, deleted_date)
WITH cte AS (
...
)
SELECT saleid, customerid, items, current_timestamp
FROM cte;
```

如你所见，对于某些 `DBMS`，你会以 `CTE` 开始，就像使用 `SELECT` 语句一样；而在其他系统中，你则以 `INSERT` 子句开始。

至于 `CTE` 本身，我们将从待删除的数据中派生它。

对于大多数 `DBMS`，每一行待删除的行都由一个名为 `old`（在 `Oracle` 中是 `:old`）的虚拟行表示。而 `MSSQL` 则有一个名为 `deleted` 的虚拟表。

如果我们只是从单个表归档，则不需要 `CTE`，我们可以简单地用以下方式复制行：

```sql
--  非 MSSQL: FOR EACH ROW
INSERT INTO deleted_sales
VALUES(old.saleid, old.customerid, '...', current_timestamp);
--  MSSQL: deleted 是一个虚拟表
INSERT INTO deleted_sales
SELECT saleid, customerid, '...', current_timestamp
FROM deleted;
```

然而，当涉及另一个表时，事情就不那么简单了。这里的计划是从另一个表读取书籍 ID 和数量，并根据 `DBMS` 使用 `string_agg`、`group_concat` 或 `listagg` 将它们组合起来。

为了生成数据，我们将使用连接并聚合结果：

```sql
WITH cte(saleid, customerid, items) AS (
SELECT
s.id, s.customerid,
string_agg(si.bookid || ':' || si.quantity, ';')
FROM sales AS s JOIN saleitems AS si ON s.id = si.saleid
WHERE s.id = old.id
GROUP BY s.id, s.customerid
)
```

前面的示例是针对 `PostgreSQL` 的，但其他系统几乎相同——只是 `string_agg()` 函数、连接符和表别名有所变化。

`items` 字符串将包含类似以下的内容：

```sql
123:3;456:1;789:2
```

也就是说，一个或多个 `bookid:quantity` 项用分号连接起来。

如果你确实需要拆分它，可以使用我们在第 9 章中用于分割字符串的相同技术。现在我们可以着手创建触发器了。

### 创建触发器

既然我们已经介绍了触发器工作原理的基础知识，我们现在可以编写实际的代码了。在很大程度上，它会与你之前看到的非常相似，但每个 `DBMS` 会有其变体。

一旦触发器创建好，我们可以用以下语句进行测试。计划是删除那些没有 `ordered` 日期的销售记录：

```sql
--  之前
SELECT * FROM sales ORDER BY id;
SELECT * FROM saleitems ORDER BY id;
SELECT * FROM deleted_sales ORDER BY id;
--  使用触发器删除
DELETE FROM sales WHERE ordered IS NULL;
--  之后
SELECT * FROM sales ORDER BY id;
SELECT * FROM saleitems ORDER BY id;
SELECT * FROM deleted_sales ORDER BY id;
```

我们现在将详细介绍各个 `DBMS` 的具体实现。

#### PostgreSQL 触发器

`PostgreSQL` 的触发器形式最不便捷，因为你首先需要准备一个 `function` 来包含触发器代码。函数是一个命名的代码块，之后可以在任何时间调用。

为了准备函数和触发器，我们可以从几个 `DROP` 语句开始：

```sql
DROP TRIGGER IF EXISTS archive_sales_trigger ON sales;
DROP FUNCTION IF EXISTS do_archive_sales;
```

该函数将基本包含前面描述的代码：

```sql
CREATE FUNCTION do_archive_sales() RETURNS TRIGGER
LANGUAGE plpgsql AS
$$BEGIN
WITH cte(saleid, customerid, items) AS (
SELECT
s.id, s.customerid,
string_agg(si.bookid || ':' || si.quantity, ';')
FROM sales AS s JOIN saleitems AS si
ON s.id = si.saleid
WHERE s.id = old.id
GROUP BY s.id, s.customerid
)
INSERT INTO deleted_sales(saleid, customerid, items, deleted_date)
SELECT saleid, customerid, items, current_timestamp
FROM cte;
RETURN old;
END$$;
```

如你所见，该函数包含了用于 `CTE` 和将数据复制到 `deleted_sales` 表的代码。关于函数本身，有几点需要注意：

*   函数有一个名称 (`do_archive_sales`) 并返回特定类型的结果，在本例中是 `TRIGGER`。
*   `PostgreSQL` 有多种替代编码语言可用于编写函数，但标准的是 `plpgsql`。
*   从技术上讲，函数定义是一个字符串。然而，使用单引号会干扰函数定义内部的单引号。`PostgreSQL` 允许使用替代的字符串定界符，在本例中是 `$$` 代码。这是编写 `PostgreSQL` 函数最神秘的部分。

一旦函数准备就绪，创建触发器就很简单了：

```sql
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
EXECUTE FUNCTION do_archive_sales();
```

你现在可以测试它了。

#### MySQL/MariaDB 触发器

对于 `MariaDB`/`MySQL`，触发器可以在单个代码块中编写。首先，我们编写删除触发器的代码：

```sql
DROP TRIGGER IF EXISTS archive_sales_trigger;
```

触发器代码的基本形式将是

```sql
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
BEGIN
...
END;
```

这里可能会有一个潜在的复杂情况。触发器代码包含一个 `BEGIN ... END` 块，以允许在一个块中执行多条语句。此时，`MariaDB`/`MySQL` 无法确定真正的结束在哪里，因此通常需要更改语句的结束分隔符：

```sql
DELIMITER $$
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
BEGIN
...
END; $$
DELIMITER ;
```

这里，分隔符被更改为 `$$`。不一定非得是这个，但它是一个你不太可能用于其他用途的组合。新的定界符用于标记代码的结束，之后再切换回分号。

之后，触发器代码如前所述：

```sql
DELIMITER $$
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
BEGIN
INSERT INTO deleted_sales(saleid, customerid, items, deleted_date)
WITH cte(saleid, customerid, items) AS (
SELECT
s.id, s.customerid,
group_concat(si.bookid || ':' || si.quantity SEPARATOR ';')
FROM sales AS s JOIN saleitems AS si ON s.id = si.saleid
WHERE s.id = old.id
GROUP BY s.id, s.customerid
)
SELECT saleid, customerid, items, current_timestamp
FROM cte;
END; $$
DELIMITER ;
```

你现在可以测试你的触发器了。



#### MSSQL 触发器

MSSQL 也提供了一种简单直接的创建触发器的方式。不过，这里存在一个复杂因素，我们需要设法规避。

但在那之前，我们先添加删除触发器的代码：

```
DROP TRIGGER IF EXISTS archive_sales_trigger;
```

对于其他数据库管理系统（DBMS），你会创建一个 `BEFORE DELETE` 触发器来在数据被删除前捕获它。但在 MSSQL 中，你没有这个选项：只有 `AFTER DELETE` 和 `INSTEAD OF DELETE`。在这两种情况下，都有一个名为 `deleted` 的虚拟表，其中包含了要被删除的行。

`AFTER DELETE` 的问题在于，即使 `deleted` 虚拟表拥有从 `sales` 表中删除的行，此时再去获取 `saleitems` 表中的相关行已经太晚了，因为它们也已经被删除了，并且没有对应的虚拟表。

为此，我们将采取不同的方法。我们将使用一个 `INSTEAD OF DELETE` 事件，这意味着 MSSQL 将运行触发器，而不是实际执行删除操作。技巧是在触发器末尾执行实际的删除操作来完成它：

```
CREATE TRIGGER archive_sales_trigger
ON sales
INSTEAD OF DELETE AS
BEGIN
...
DELETE FROM sales WHERE id IN(SELECT id FROM deleted);
END;
```

`deleted` 虚拟表仍然包含那些尚未被实际删除、但即将被删除的行（在触发器介入之前）。我们只需要从中获取 `id` 来识别最终应被删除的销售记录，以及其级联的销售明细项。

另一个复杂之处在于 MSSQL 不允许你直接拼接字符串和数字，因此你需要将数字转换为字符串：

```
cast(si.bookid AS varchar)+':'+cast(si.quantity AS varchar)
```

在 MSSQL 中，`varchar` 是 `varchar(30)` 的简写。对于整数来说这远远超出需要，但它会缩减到整数的实际大小，并且易于阅读。

完整的触发器代码如下：

```
CREATE TRIGGER archive_sales_trigger
ON sales
INSTEAD OF DELETE AS
BEGIN
WITH cte(saleid, customerid, items) AS (
SELECT
s.id, s.customerid,
string_agg(cast(si.bookid AS varchar)+':'
+cast(si.quantity AS varchar),';')
FROM
sales AS s
JOIN saleitems AS si ON s.id=si.saleid
JOIN deleted ON s.id=deleted.id
GROUP BY s.id, s.customerid
)
INSERT INTO deleted_sales(saleid, customerid, items,
deleted_date)
SELECT saleid, customerid, items, current_timestamp
FROM cte;
DELETE FROM sales
WHERE id IN(SELECT id FROM deleted);
END;
```

现在你可以删除你的销售记录了。

#### SQLite 触发器

在本书涉及的所有 DBMS 中，SQLite 编写触发器的方式是最简单、最直接的。

首先，我们可以编写删除触发器的代码：

```
DROP TRIGGER IF EXISTS archive_sales_trigger;
```

创建触发器的代码与之前的讨论几乎相同：

```
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
BEGIN
INSERT INTO deleted_sales(saleid, customerid, items,
deleted_date)
WITH cte(saleid, customerid, items) AS (
SELECT
s.id, s.customerid,
group_concat(si.bookid||':'||si.quantity,';')
FROM sales AS s JOIN saleitems AS si
ON s.id=si.saleid
WHERE s.id=old.id
GROUP BY s.id, s.customerid
)
SELECT saleid, customerid, items,current_timestamp
FROM cte;
END;
```

`FOR EACH ROW` 子句是可选的，因为在 SQLite 中目前没有其他替代选项。不过，包含它是为了明确指出该触发器应用于即将被删除的每一行。

现在你可以测试这个触发器了。

#### Oracle 触发器

在 Oracle 中编写触发器代码与之前概述的基本代码类似，但有一些复杂因素需要我们设法规避。

在此之前，我们可以编写删除触发器的代码：

```
--  DROP TRIGGER archive_sales_trigger;
```

代码被注释掉了，因为 Oracle 不支持 `IF EXISTS`。

第一个复杂因素是触发器代码由多个语句组成，因此很难分辨哪些语句属于同一个代码块。

Oracle 提供了一个替代的语句分隔符，用于当你需要组合多个语句时：

```
/
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
BEGIN
...
END;
/
```

代码前后的斜杠 (`/`) 定义了代码块。斜杠之间的所有内容，包括以分号结尾的语句，都将被视为一个代码块。

第二个复杂因素是 Oracle 不喜欢修改触发操作的表本身。解决方案是告诉 Oracle 该代码属于一个独立的事务：

```
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
DECLARE
PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
...
COMMIT;
END;
```

事务是一组可以撤销（“回滚”）的更改（如果出现问题）。然而，要保留更改，你需要使用 `COMMIT` 语句。

其余代码与之前讨论的非常相似：

```
/
CREATE TRIGGER archive_sales_trigger
BEFORE DELETE ON sales
FOR EACH ROW
DECLARE
PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
INSERT INTO deleted_sales(saleid, customerid, items, deleted_date)
WITH cte(saleid,customerid,items) AS (
SELECT
s.id, s.customerid,
listagg(si.bookid||':'||si.quantity,';')
FROM sales s JOIN saleitems si ON s.id=si.saleid
WHERE s.id=:old.id
GROUP BY s.id, s.customerid
)
SELECT saleid, customerid, items, current_timestamp
FROM cte;
COMMIT;
END;
/
```

现在你可以测试这个触发器了。

### 触发器的优缺点

触发器的主要作用是为数据库添加原本不存在的行为。例如，DBMS 已经支持默认值和列约束，因此，虽然你可以使用触发器来实现类似的功能，但应该首先检查内置特性是否能满足需求。

触发器曾经派上用场的一个例子是早期版本的 Oracle。其他 DBMS 很早就实现了自增主键，但 Oracle 直到较新的版本才支持。在这种情况下，你会使用一个触发器来维护和利用一个单独的序列，为下一行插入的数据填充主键。

我们之前已经提到过一些情况，例如你可能希望为列提供一个比内置默认功能更复杂的默认值，或者你可能希望自动更新某个列。在这种情况下，触发器或许能提供这种额外功能。

然而，触发器也可能被滥用。如果一个表上有 DML 触发器，那么每次你对表数据进行任何更改时，总会有一点额外的工作，这可能会增加额外的负担。

另一个问题是触发器可能会给数据库增加一些神秘感，特别是对数据库的其他用户而言。每次你做某件事时，总会引发其他事情的发生。这可能会使故障排查变得棘手一些，也让检查数据是否正确变得困难一些。



## 数据透视

优秀数据库设计的一个重要原则是每列承担不同的职责。在此基础上，各列之间保持独立。这就是为什么我们在第 2 章中花费大量精力将城镇详情从`customers`表中分离出来的原因之一。

然而，在某些情况下，这种设计并不适合分析。以典型的账目表为例：

| **日期** | **描述** | **餐饮** | **交通** | **住宿** | **杂项** |
| --- | --- | --- | --- | --- | --- |
| . . . | . . . | . . . | . . . | . . . | . . . |
| . . . | . . . | . . . | . . . | . . . | . . . |

这种布局非常易于理解和分析。若想获取特定类别的总计，只需纵向相加该列。若想获取特定项目的总计，只需横向相加。在电子表格发明让计算机承担所有繁重工作之前，这类事情通常是手工完成的。

你可能会在遇到的数据库表中见到这类设计。然而，对于 SQL 表来说，这并非好的设计方案：

*   将一个值放入某列就排除了放入另一列的可能：各列之间存在深度依赖。
*   最终会出现大量空白。
*   新增类别意味着需要在表设计中添加新列。最终可能导致列数过多。
*   数据分析变得更困难，因为现在需要跨*列*进行计算：SQL 聚合函数是设计用于跨*行*聚合的。

更好的设计是：

| **日期** | **描述** | **类别** | **金额** |
| --- | --- | --- | --- |
| . . . | . . . | . . . | . . . |
| . . . | . . . | . . . | . . . |

通常规则是：类别应置于行中，而非列中。

尽管如此，能够将第二种形式的数据转换为第一种形式仍然很有用。你可以在演示文稿中使用它，并可能将其转化为出色的图表。

### 透视数据

从第二种形式生成第一种结果被称为**数据透视**。其核心思想是让类别围绕轴心旋转（摆动），从纵向变为横向。

如果在电子表格程序中读取数据，你可以简单地完成透视。不过，你也可以直接在数据库中生成透视数据，尽管过程不那么简单。

在电子表格中生成透视表有以下优势：

*   交互性更强，可以轻松更改透视和汇总的内容。
*   电子表格会更自动生成类别；正如你将看到的，从数据库内部操作则不那么方便。
*   离将其转化为图表仅一步之遥。

另一方面，使用数据库有以下优势：

*   数据在单一环境中生成。
*   你可以创建视图，随时重新生成透视表。

在 SQL 中生成透视表主要有两种方式：

*   手动：使用`GROUP BY`子句，可以在多列上聚合数据。
*   MSSQL 和 Oracle 内置了透视表功能来完成此任务。PostgreSQL 也有类似功能，但非内置，需要安装。

由于数据透视的目的是创建汇总，你通常需要使用分组字段或自行对值进行分组。例如：

*   可以使用现有的`state`列（它是地址的一个分组）。
*   可以使用日期函数，如`month()`按月份对日期进行分组。
*   可以使用字符串函数提取字符串的公共部分。

透视表大致如下所示：

| **行分组** | **列分组** | **列分组** | **列分组** |
| --- | --- | --- | --- |
| 组 1 | . . . | . . . | . . . |
| 组 2 | . . . | . . . | . . . |
| 组 3 | . . . | . . . | . . . |

我们将看到如何对`sales`和`customers`表中的数据进行透视，以获取按州和 VIP 类别划分的销售总额。结果大致如下：

| **州** | **金卡** | **银卡** | **铜卡** |
| --- | --- | --- | --- |
| . . . | . . . | . . . | . . . |
| . . . | . . . | . . . | . . . |
| . . . | . . . | . . . | . . . |

原则上，你可以转置表格，让 VIP 组纵向排列，州横向排列。不过，当前这种版本看起来会更整洁。



### 手动进行数据透视

正如我们之前所见，通常在聚合数据之前需要先进行准备。这个特定的数据汇总将需要来自四张表的数据：`customers`（客户）、`towns`（城镇）、`vip`（贵宾）和`sales`（销售）。幸运的是，`customerdetails`（客户详情）视图已经合并了`customers`和`towns`表，因此我们可以将需要处理的表数量减少到三张。

所有的准备工作将在多个公用表表达式（CTE）中完成：

*   `vip`表包含一个状态编号。`statuses` CTE 将是一个字面量表，它将名称分配给编号。
*   `customerinfo` CTE 将连接各个表并选择我们需要汇总的列。
*   `salesdata`将是一个聚合查询，这将是我们数据透视表汇总的第一步。

```
WITH
statuses AS (
...
),
customerinfo AS (
...
),
salesdata AS (
)
...
```

有了这些 CTE，我们将运行另一个聚合查询，从而生成我们的数据透视表。

`status` CTE 很简单。我们只需要将状态编号与名称进行匹配：

```
WITH
statuses(status, statusname) As (
--  PostgreSQL, SQLite, MariaDB (Not MySQL):
VALUES (1,'Gold'), (2,'Silver'), (3,'Bronze')
--  MySQL:
VALUES row(1,'Gold'), row(2,'Silver'),
row(3,'Bronze')
--  MSSQL:
SELECT * FROM (VALUES (1,'Gold'),(2,'Silver'),
(3,'Bronze'))
--  Oracle:
SELECT 1,'Gold' FROM dual
UNION ALL SELECT 2,'Silver' FROM dual
UNION ALL SELECT 3,'Bronze' FROM dual
)
```

`customerinfo` CTE 将把这个状态表与`customerdetails`视图和`vip`表连接起来，以获取客户的 ID、州（state）和状态名称：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
SELECT customerdetails.id, state, statuses.statusname
FROM
customerdetails
LEFT JOIN vip ON customerdetails.id=vip.id
LEFT JOIN statuses ON vip.status=statuses.status
)
SELECT *
FROM customerinfo;
```

如果现在进行测试，你会得到类似这样的结果：

| **Id** | **state** | **statusname** |
| --- | --- | --- |
| 407 | NSW | Bronze |
| 299 | QLD | Gold |
| 21 | [NULL] | Gold |
| 597 | TAS | [NULL] |
| 106 | NSW | Gold |
| 26 | VIC | Gold |
| ~ 303 行 ~ |

此时，你可以按`state`或`statusname`进行分组，查看每种状态的数量，但我们更感兴趣的是销售总额。

为此，我们需要在另一个 CTE 中，将前面的结果与`sales`表连接起来：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
...
),
salesdata(state, statusname, total) AS (
SELECT state, statusname, total
FROM customerinfo JOIN sales
ON customerinfo.id=sales.customerid
)
SELECT *
FROM salesdata;
```

再次测试我们目前得到的结果，会得到：

| **State** | **statusname** | **total** |
| --- | --- | --- |
| NSW | [NULL] | 56 |
| NSW | Silver | 43.5 |
| VIC | [NULL] | 70 |
| QLD | [NULL] | 28 |
| VIC | Gold | 24.5 |
| VIC | [NULL] | 133 |
| ~ 5294 行 ~ |

所有这些只是为了准备好数据。我们现在要做的是生成分组行。

显然，你需要一个聚合查询，按`state`分组。通常，它看起来会是这样：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
...
),
salesdata(state, statusname, total) AS (
...
)
SELECT state, sum(total)
FROM salesdata
GROUP BY state;
```

这样会得到：

| **State** | **sum** |
| --- | --- |
| WA | 20274 |
| ACT | 6781.5 |
| TAS | 28193 |
| VIC | 79199.5 |
| NSW | 101889 |
| NT | 6151 |
| QLD | 53331.5 |
| SA | 30977.5 |

然而，为了获得那种分类账表格的外观，我们将使用聚合过滤器来生成三个独立的总计：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
...
),
salesdata(state, statusname, total) AS (
...
)
SELECT
state,
sum(CASE WHEN statusname='Gold' THEN total END) AS gold,
sum(CASE WHEN statusname='Silver' THEN total END)
AS silver,
sum(CASE WHEN statusname='Bronze' THEN total END)
AS bronze
FROM salesdata;
```

这最终会给你以下结果：

| **State** | **gold** | **silver** | **bronze** |
| --- | --- | --- | --- |
| WA | 213 | 1655 | [NULL] |
| ACT | 1272.5 | [NULL] | [NULL] |
| TAS | 4182.5 | 2203 | 2764.5 |
| VIC | 8190 | 5875 | 5752.5 |
| NSW | 11068.5 | 9319 | 10760.5 |
| NT | [NULL] | [NULL] | 339.5 |
| QLD | 5094 | 3522.5 | 10480 |
| SA | 644 | 1390.5 | 3362 |

你可能会忍不住问，是否有更简单的方法。答案是，其实没有。困难的部分始终在于为数据透视准备数据。

不过，对于少数几种数据库管理系统（DBMS），最后一步可以通过内置功能来实现。

### 使用透视功能（MSSQL, Oracle）

MSSQL 和 Oracle 都提供了一个非标准的`pivot`功能，可以简化数据透视表的生成。它采用以下形式：

*   *聚合*是你想要应用的聚合函数。在这里，是`sum(total)`。
*   *列*是其值将横跨表格显示的列。在这里，是`statusname`。
*   *列名列表*是一组值，这些值将成为数据透视表中的列标题。在这里，是`Gold, Silver, Bronze`。
*   别名是你想指定的任何别名。这里虽然用不到，但它是必需的。毕竟，数据透视表是一个虚拟表。

```
SELECT ...
FROM ...
PIVOT (aggregate FOR column IN(columnnames)) AS alias
```

在我们的例子中，数据透视表将如下所示：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
...
),
salesdata(state, statusname, total) AS (
...
)
SELECT *
FROM salesdata
--  MSSQL:
PIVOT (sum(total) FOR statusname IN (Gold, Silver, Bronze))
AS whatever
--  Oracle:
PIVOT (sum(total) FOR statusname IN ('Gold' AS Gold, 'Silver'
AS Silver, 'Bronze' AS Bronze))
;
```

这比我们之前使用的过滤聚合稍微简单一些。但是请注意，这种技术有一些需要注意的地方。

MSSQL 和 Oracle 的语法并不完全相同：

*   在 MSSQL 中，列名列表是一个简单的名称列表。另外，请注意 `PIVOT` 子句需要一个别名。
*   在 Oracle 中，列名列表是一个字符串列表；但是，它们被设置了别名，以防止名称中出现单引号。`PIVOT` 子句本身不需要别名。

你会注意到 `state` 列没有出现在 `PIVOT` 子句中；只有 `statusname` 和 `total`。*任何未在* `PIVOT` *子句中提及的列都将作为分组行出现。* 如果有不止一个这样的列，你可以制作更复杂的数据透视表，但你需要确保要进行透视的（虚拟）表没有任何多余的、不需要的列。

你还会注意到 `IN` 表达式不是普通的 `IN` 表达式。首先，它不是一个值列表，而是一个列名列表。

除此之外，你不能使用子查询来获取列名列表。你必须事先知道列名会是什么，并且必须自己手动输入它们。

使用 `pivot` 功能并不像它可能做到的那样方便，但如果可用，它仍然比过滤聚合简单。然而，你仍然需要先在数据准备上投入一些精力。


### 反透视功能

MSSQL 和 Oracle 都可以使用 `UNPIVOT` 来逆转该过程。注意到透视表是反规范化数据，而 `UNPIVOT` 子句可以给你一个规范化后的结果。其核心思想是，分散在表中各个类别列中的汇总信息，将会以行的形式出现在表中。

示例数据库中没有以透视表形式存储的表，这是合理的。不过，在之前的工作中，我们已经得到了一个现成的透视表。我们可以用它来演示反透视功能。

首先，我们需要将最终的 `SELECT` 语句包装到另一个 CTE（公共表表达式）中：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
...
),
salesdata(state, statusname, total) AS (
...
),  --  额外的逗号
pivottable AS (
SELECT *
FROM salesdata
PIVOT ...
)
SELECT *
FROM pivottable
;
```

如果你运行这个，你会得到和之前一样的结果；我们只是把结果放进了 `pivottable` 这个 CTE 中。

下一步是在 `SELECT` 语句的末尾添加 `UNPIVOT` 子句：

```
WITH
statuses(status, statusname) AS (
...
),
customerinfo(id, state, statusname) AS (
...
),
salesdata(state, statusname, total) AS (
...
),
pivottable AS (
...
)
SELECT *
FROM pivottable
-- MSSQL:
UNPIVOT (
total FOR statuses IN (Gold,Silver,Bronze)
) AS w
--  Oracle:
UNPIVOT (
total FOR statuses IN (Gold,Silver,Bronze)
)
```

你应该会看到类似这样的结果：

| 州 (state) | 总额 (total) | 状态 (statuses) |
| --- | --- | --- |
| QLD | 5532.5 | Gold |
| QLD | 3557.5 | Silver |
| QLD | 10937 | Bronze |
| VIC | 8352 | Gold |
| VIC | 6381.5 | Silver |
| VIC | 6023 | Bronze |
| NSW | 11526 | Gold |
| NSW | 9567 | Silver |
| NSW | 11941.5 | Bronze |
| NT | 349.5 | Bronze |
| ACT | 1387 | Gold |
| TAS | 4574 | Gold |
| TAS | 2459.5 | Silver |
| TAS | 2873.5 | Bronze |
| SA | 826.5 | Gold |
| SA | 1634.5 | Silver |
| SA | 3709.5 | Bronze |
| WA | 213 | Gold |
| WA | 1655 | Silver |

`UNPIVOT` 子句比 `PIVOT` 子句更神秘一些。它唯一明确提及的列是 `statuses` 列，并且你同样需要列出其可能的值。从那里，DBMS（数据库管理系统）会神奇地推断出有一个 `state` 列，而剩下的数据则会出现在另一列中，我们称之为 `total`。

## 使用 SQL 变量

SQL 不是一门编程语言。使用编程语言时，你通过一系列步骤来编写如何完成工作的代码。SQL 是一种声明式语言，你编写你想要完成的任务，而由 DBMS 决定如何去完成它。

然而，有时你需要分多个步骤完成一个工作，这时能够分步编写代码就很有用了。我们在第 3 章中看到了这一点，其中添加一笔新销售就涉及多个步骤。

第 3 章中的流程缺失的是存储中间值的能力。这正是我们将在本节中探讨的内容。

许多 DBMS 通过特殊函数或系统/全局变量的形式提供有关当前数据库环境的信息。有时，可以使用 `SET` 命令将这些系统变量设置为新值。但这并不是我们在本节中要讨论的内容。在本节中，我们探讨的是你为自己创建和设置的变量。

**变量** 是一个临时的数据片段。通常，你会在使用前**声明**它并定义其数据类型。你可以在声明时设置它，或者更常见的是，在后续步骤中设置它。

通常，一个变量与一个称为**函数**或**过程**（取决于 DBMS 和你试图在代码中完成的任务）的存储代码块相关联。在本节中，我们将在不存储代码的情况下进行操作。

不同的 DBMS 在使用变量的过程上略有不同：

*   **PostgreSQL** 历史上将变量限制在存储代码块（函数）内。然而，在版本 11 中，他们引入了匿名（`DO`）代码块，允许你编写代码而无需存储该代码块。PostgreSQL 变量必须用数据类型声明。
*   **MariaDB/MySQL** 对变量最宽松，你无需在使用前声明变量。变量名以 `@` 符号为前缀。
*   **MSSQL** 变量必须用数据类型声明。它们以 `@` 符号为前缀。
*   **Oracle** 变量用数据类型声明。

    SQLite 不在此列，因为它不支持变量。SQLite 通常嵌入在宿主应用程序中。假设你正在为宿主应用程序编写编程代码，你可以在那里拥有所有你想要的额外变量和功能。

### 代码块

如果你使用的客户端便于一次运行一条语句，你可能会发现在处理包含多条语句的代码块时会有点混乱。使用分隔符将你的代码块括起来会更容易处理。

对于不同的 DBMS，分隔符如下所示：

```
--  PostgreSQL
DO $$
...
END $$;
--  MariaDB/MySQL
DELIMITER $$
...
$$
DELIMITER ;
--  MSSQL
GO
...
GO
--  Oracle
/
...
/
```

最终，你可能只是高亮显示所有代码行并一起运行它们。这就是我们在尝试以下代码时推荐的做法。不要尝试一次只运行一行。

在下面的代码中，我们将执行第 3 章中添加新销售的操作。然后，我们特意记录了新的销售 ID，以便在后续语句中使用。然而，这次我们将使用变量来存储中间值，以便我们可以在一个批处理中运行代码。

代码将大致遵循以下步骤：

1.  设置要使用的数据。
2.  插入销售记录。
3.  将新的销售 ID 存入一个变量。
4.  使用该销售 ID，插入销售项目。
5.  使用该销售 ID，用它们的价格更新销售项目。
6.  当然，还是使用该销售 ID，用总额更新新的销售记录。

同时，我们还将设置几个其他变量：

*   一个存储客户 ID 的变量
*   一个存储订购日期/时间的变量

如果能有另一个变量来存储销售项目就更好了。然而，大多数 DBMS 不擅长在不增加定义自定义数据类型来完成这项工作的大量额外麻烦的情况下定义多值变量。在这里，我们试图保持简单。

接下来将是四个类似的版本，展示如何编写该代码块。

### 更新后用于添加销售的代码

以下代码块的大纲基本相同：

*   定义变量
*   运行 `INSERT` 和 `UPDATE` 语句
*   测试结果

我们将讨论针对主要 DBMS 的代码。

### 在 PostgreSQL 中使用变量

最初，PostgreSQL 不允许你在存储代码块之外进行任何此类操作。从版本 11 开始，你可以使用**匿名块**。如果你使用的是旧版 PostgreSQL，那么很遗憾，此功能不可用。

匿名块定义在 `DO ... END` 之间：

```
DO $$
...
END $$ ;
```

`$$` 符号用于将多个语句视为单个块。这样，分号就不会提前终止块。

变量在 `DECLARE` 部分声明：

```
DO $$
DECLARE
cid INT := 42;
od TIMESTAMP := current_timestamp;
sid INT;
END $$ ;
```

变量名可以随意取，但你可能会面临与后续代码中列名冲突的风险。一些开发者会给变量名加上下划线前缀（例如 `_cid`）。

`sid` 变量是一个整数，稍后将被赋值。`cid` 和 `od` 变量分别用于存储客户 ID 和订购日期/时间。它们通过特殊运算符 `:=` 从一开始就进行了赋值。

实际的代码位于 `BEGIN ... END` 块内。它将是你在第 3 章中使用过的所有代码，但连贯地运行在一起。重要的是，变量 `sid` 用于管理新的销售 ID：

```
DO $$
DECLARE
cid INT := 42;
od TIMESTAMP := current_timestamp;
sid INT;
BEGIN
INSERT INTO sales(customerid, ordered)
VALUES(cid, current_timestamp)
RETURNING id INTO sid;
INSERT INTO saleitems(saleid, bookid, quantity)
VALUES
(sid,123,3),
(sid,456,1),
(sid,789,2);
UPDATE saleitems AS si
SET price=(SELECT price FROM books AS b
WHERE b.id=si.bookid)
WHERE saleid=sid;
UPDATE sales
SET total=(SELECT sum(price*quantity)
FROM saleitems WHERE saleid=sid)
WHERE id=sid;
END $$;
```

`sid` 变量的值来自第一个 `INSERT` 语句中的 `RETURNING` 子句。从那时起，它就在后续的语句中被使用。

你可以使用以下语句测试结果：

```
SELECT * FROM sales ORDER BY id DESC;
SELECT * FROM saleitems ORDER BY id DESC;
```

你应该能在顶部看到新的销售记录和销售项目。

### 在 MariaDB/MySQL 中使用变量

MariaDB/MySQL 使用变量的方法最简单。在存储函数或过程之外，你无需声明变量或其类型：直接使用即可。

另外，它并没有严格的匿名代码块概念，因此定义一个更多是出于组织结构的考虑。我们还是会这样做，尽管它实际上没什么区别：

```
DELIMITER $$
BEGIN
END; $$
DELIMITER ;
```

首先，我们给几个变量赋值：

```
DELIMITER $$
BEGIN
SET @cid = 42;
SET @od = current_timestamp;
SET @sid = NULL;
END; $$
DELIMITER ;
```

变量以 `@` 字符为前缀。这使它们更明显，并避免了可能与列名的冲突。

语句 `SET @sid = NULL;` 是不必要的。由于你不需要声明变量，我们加入这个语句只是为了明确表示我们稍后将使用 `@sid` 变量。

完整的代码如下所示：

```
DELIMITER $$
BEGIN
SET @cid = 42;
SET @od = current_timestamp;
SET @sid = NULL;    -- 非必需；只是为了明确
INSERT INTO sales(customerid, ordered)
VALUES(@cid, @od);
SET @sid = last_insert_id();
INSERT INTO saleitems(saleid,bookid,quantity)
VALUES
(@sid,123,3),
(@sid,456,1),
(@sid,789,2);
UPDATE saleitems
SET price=(SELECT price FROM books
WHERE books.id=saleitems.bookid)
WHERE saleid=@sid;
UPDATE sales
SET total=(SELECT sum(price*quantity)
FROM saleitems WHERE saleid=@sid)
WHERE id=@sid;
END;
$$
DELIMITER ;
```

注意这个语句：

```
SET @sid = last_insert_id();
```

当你添加一个具有自动生成主键的新行时，你需要获取新值以便稍后使用。`last_insert_id()` 函数获取当前会话中最近一次自动生成的值。你会注意到它没有指定是哪个表：这就是为什么你需要在 `INSERT` 语句后立即调用它。

如你所见，其余代码与第 3 章中的基本相同，只是使用 `@sid` 变量来管理新的销售 ID。

你可以使用以下语句测试结果：

```
SELECT * FROM sales ORDER BY id DESC;
SELECT * FROM saleitems ORDER BY id DESC;
```

你应该能在顶部看到新的销售记录和销售项目。

### 在 MSSQL 中使用变量

MSSQL 代码可以写在由 `GO` 关键字分隔的块内：

```
GO
...
GO
```

`GO` 关键字实际上不是 Microsoft SQL 语言的一部分（任何其他 SQL 语言也不是）。它实际上是给客户端软件的一个指令，将内部内容视为单个批处理并执行。有些客户端允许你缩进关键字，有些允许在同一行添加分号和注释，但最安全的做法是不缩进它，也不在同一行添加任何其他内容。

Microsoft 没有用于声明变量的块，但它有一个语句。要声明三个变量，你可以使用三个语句：

```
GO
DECLARE @cid INT = 42;
DECLARE @od datetime2 = current_timestamp;
DECLARE @sid INT;
GO
```

或者你可以使用一个语句，用逗号分隔变量：

```
GO
DECLARE
@cid INT = 42,
@od datetime2 = current_timestamp,
@sid INT;
GO
```

变量以 `@` 字符为前缀，这使它们易于识别并易于与列名区分开。

`@sid` 是一个整数，稍后将被赋值。

其余的代码与我们在第 3 章中所做的类似，但新的销售 ID 将在 `@sid` 变量中管理：

```
GO
DECLARE @cid INT = 42;
DECLARE @od datetime2 = current_timestamp;
DECLARE @sid INT;
INSERT INTO sales(customerid,ordered)
VALUES(@cid, @od);
SET @sid = scope_identity();
INSERT INTO saleitems(saleid,bookid,quantity)
VALUES
(@sid,123,3),
(@sid,456,1),
(@sid,789,2);
UPDATE saleitems
SET price=(SELECT price FROM books
WHERE books.id=saleitems.bookid)
WHERE saleid=@sid;
UPDATE sales
SET total=(SELECT sum(price*quantity)
FROM saleitems WHERE saleid=@sid)
WHERE id=@sid;
GO
```

`@sid` 变量的值来自 `scope_identity()` 函数。你会注意到它没有指定是哪个表：这就是为什么你需要在 `INSERT` 语句后立即调用它。从那时起，它就在后续的语句中被使用。

你可以使用以下语句测试结果：

```
SELECT * FROM sales ORDER BY id DESC;
SELECT * FROM saleitems ORDER BY id DESC;
```

你应该能在顶部看到新的销售记录和销售项目。

### 在 Oracle 中使用变量

Oracle 代码块可以用正斜杠分隔：

```
/
/
```

当时机成熟，整个代码块将作为一个批次运行。

变量在 `DECLARE` 部分声明：

```
/
DECLARE
cid INT := 42;
od TIMESTAMP := current_timestamp;
sid INT;
/
```

变量名可以任意取，但你可能会面临与后续代码中的列名冲突的风险。一些开发者会在名称前加下划线（例如 `_cid`）。

`sid` 是一个整数变量，将在稍后赋值。`cid` 和 `od` 变量分别代表客户 ID 和订单日期/时间。它们通过特殊运算符 `:=` 从一开始就赋值。

实际代码位于 `BEGIN ... END` 块内。它将是你在第 3 章中使用过的所有代码，但会连在一起运行。关键部分是变量 `sid` 用于管理新的销售 ID：

```
/
DECLARE
cid INT := 42;
od TIMESTAMP := current_timestamp;
sid INT;
BEGIN
INSERT INTO sales(customerid,ordered)
VALUES(cid, od)
RETURNING id INTO sid;
INSERT INTO saleitems(saleid,bookid,quantity)
VALUES (sid,123,3);
INSERT INTO saleitems(saleid,bookid,quantity)
VALUES (sid,456,1);
INSERT INTO saleitems(saleid,bookid,quantity)
VALUES (sid,789,2);
UPDATE saleitems
SET price=(SELECT price FROM books
WHERE b.id=saleitems.bookid)
WHERE saleid=sid;
UPDATE sales
SET total=(SELECT sum(price*quantity) FROM saleitems
WHERE saleid=sid)
WHERE id=sid;
END;
/
```

`sid` 变量从第一个 `INSERT` 语句中的 `RETURNING` 子句获取其值。此后，它在剩余的语句中被使用。

你可以使用以下语句测试结果：

```
SELECT * FROM sales ORDER BY id DESC;
SELECT * FROM saleitems ORDER BY id DESC;
```

你应该能在顶部看到新的销售记录和销售项。

## 回顾

在本章中，我们探讨了一些可以用来更充分利用数据库的额外技术。

### 触发器

触发器是响应数据库中发生的某些事件而运行的代码脚本。通常，这些事件包括 `INSERT`、`UPDATE` 和 `DELETE`。使用触发器，你可以拦截事件并对受影响的表或其他表进行额外的更改。一些触发器可以更进一步，与 DBMS 或操作系统更紧密地协作。

我们通过创建一个响应从 `sales` 表中删除操作的触发器来探索这个概念。在这个例子中，我们将销售记录和匹配的销售项的数据复制到一个 `archive` 表中。

不同的 DBMS 在细节上有所不同，但通常遵循相同的原则：

*   为表上的事件定义触发器。
*   触发器代码可以访问即将受影响的数据（例如 `INSERTED` 和 `DELETED` 虚拟表）。
*   利用这些数据，触发器代码可以继续执行额外的 SQL 操作。

### 数据透视表

数据透视表是一种虚拟表，它按行和列汇总数据。它是一种二维聚合。

大多数情况下，原始表数据并不适合直接这样汇总。你需要付出一些努力，以正确的形式准备数据，并使其在一个或多个 CTE 中可用。

你可以结合使用两种技术手动创建数据透视表：

*   聚合查询生成纵向分组和要汇总的数据。
*   带有聚合过滤器的 `SELECT` 语句为每个横向类别生成汇总。

`MSSQL` 和 `Oracle` 都有一个非标准的 `PIVOT` 子句，它将在一定程度上自动化前面提到的第二个过程。然而，它仍然需要 SQL 开发人员的一些输入来完成工作。

### SQL 变量

在本章中，我们使用变量来简化代码，这些代码最初在第 3 章介绍，通过插入多个表并更新它们来添加一笔销售。

我们使用的大多数 SQL 都涉及单个语句。其中一些语句通过使用 CTE 生成中间数据，实际上是多部分语句。

在需要更复杂的代码在多个语句中运行的情况下，你可能需要存储中间值。这些值保存在变量中，变量是临时的数据片段。

在本章中，我们将变量用于两个目的：

*   保存要在代码中使用的固定值
*   存储由部分代码生成的中间值

在大多数 DBMS 中，变量在代码块内声明和使用。在大多数情况下，变量及其值在代码块运行后就会消失。然而，`MariaDB/MySQL` 会在运行后保留变量。

`SQLite` 不支持变量。预计宿主应用程序将处理本应由变量管理的临时数据。


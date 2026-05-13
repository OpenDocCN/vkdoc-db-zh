# 17. 更新或删除表

### 从表中删除数据

有时你可能想要从表中删除数据。也许是在你学习 SQL 时插入数据出了错，或者是为了移除不再需要的数据。在 SQL 中，有办法做到这一点，它涉及使用 `DELETE` 语句。

`DELETE` 语句允许你从表中删除数据。它的工作方式类似于 `UPDATE` 语句，你需要指定表名和要删除记录的条件。`DELETE` 语句看起来像这样：

```
DELETE FROM 表名
[WHERE 条件];
```

该语句以 `DELETE FROM` 开始，然后你提及要从中删除数据的表名。接着你可以指定一个 `WHERE` 子句。这个 `WHERE` 子句用于确定你想要删除哪些记录。

就像 `UPDATE` 语句一样，`DELETE` 语句上的 `WHERE` 子句是可选的。这意味着，如果你不小心，很容易就会删除表中的所有记录！在运行 `DELETE` 语句之前，请确保你有 `WHERE` 子句，除非你真的想删除表中的所有记录。

注意：`DELETE` 语句上的 `WHERE` 子句是可选的。如果省略它，表中的所有记录都将被删除。

#### 删除记录

让我们从 office 表中删除一条记录。office 表目前如下所示。

| ID | 地址 |
| --- | --- |
| 1 | 123 Smith Street |
| 2 | 45 Main Street |
| 3 | 10 Collins Road |

我们想从这个表中删除 “10 Collins Road” 的记录。为此，你可以编写如下语句：

```
DELETE FROM office
WHERE address = '10 Collins Road';
```

这将找到 office 表中地址为 “10 Collins Road” 的所有记录，并将它们从表中删除。

如果你在 SQL Developer 中运行此语句，屏幕底部的输出面板将显示 “1 row deleted.”。就像 `INSERT` 和 `UPDATE` 语句一样，`DELETE` 语句不会向你显示哪些记录被删除了。

你可以在表上运行 `SELECT` 语句，以查看执行 `DELETE` 语句后表的样子：

| ID | 地址 |
| --- | --- |
| 1 | 123 Smith Street |
| 2 | 45 Main Street |

```
SELECT *
FROM office;
```

你可以看到 “10 Collins Road” 的记录已不在表中。

当你用 `INSERT` 语句添加数据或用 `UPDATE` 语句更新数据时，在运行 `COMMIT` 语句保存更改或运行 `ROLLBACK` 语句撤销更改之前，数据并没有永久保存在数据库中。同样的功能也适用于 `DELETE` 语句：在运行 `COMMIT` 语句或使用 `ROLLBACK` 撤销删除操作之前，数据并没有被永久删除。

我们想提交这个语句，所以运行一个 `COMMIT` 语句或点击 `COMMIT` 按钮来保存你的更改。

#### 检查哪些记录将被删除

`DELETE` 语句在运行前不会告诉你哪些记录将被删除。它不会弹出一个 “你确定吗？” 的对话框让你确认这些记录。

不过，你可以对你的 `DELETE` 语句做一个小小的改动，以便在运行前查看哪些记录会受到影响。你可以简单地将 `DELETE` 改为 `SELECT *`。

例如，你的语句可能如下所示：

```
DELETE FROM office
WHERE address = '10 Collins Road';
```

要在运行 `DELETE` 语句之前查看哪些记录将被删除，请将 `DELETE` 一词替换为 `SELECT *`。

```
SELECT *
FROM office
WHERE address = '10 Collins Road';
```

你可以运行这个 `SELECT` 查询来查看符合 `WHERE` 子句条件的记录。再将这个语句改回 `DELETE` 就会删除那些记录。这对于像这样简单的查询，甚至对于 `WHERE` 子句更复杂的查询都很有帮助。

#### 删除表中的所有记录

如果你想删除表中的所有记录，也可以使用 `DELETE` 语句。这对于重置表或删除其所有数据（以便重新插入）很有用。

要删除表中的所有记录，请编写一个没有 `WHERE` 子句的 `DELETE` 语句。

```
DELETE FROM office;
```

这将删除表中的所有记录。你也可以运行一个 `WHERE` 子句满足所有条件的 `DELETE` 语句来达到相同的效果。在我们的例子中，这些查询将删除所有数据：

```
DELETE FROM office
WHERE id >= 1;
DELETE FROM office
WHERE address IS NOT NULL;
```

你可以运行这些查询，所有记录都将从你的表中移除。如果你想再次插入记录，可以运行前面章节中的 `INSERT` 语句。

### 总结

SQL 中的 `UPDATE` 语句允许你更新表中已存在的数据。你不需要删除并重新插入数据来修正它。你可以使用 `WHERE` 子句（其工作方式与 `SELECT` 语句中的相同）来更新表中符合特定条件的一列或多列。

你可以使用 `DELETE` 语句从表中删除记录。该语句有一个可选的 `WHERE` 子句，让你指定要删除哪些记录，如果你省略它，所有记录都将被删除。

### 为什么要更新表的结构？

在 SQL 中，你可以更新表的结构。这意味着你可以添加或删除列、重命名列、重命名表、更改列的数据类型等等。你为什么要这样做呢？有几个原因。

在软件开发的世界里，一旦某样东西被开发出来，就不能保证它会永远保持不变。这适用于整个应用程序，一直到它所使用的代码和表。

你可能需要修改一个表来添加一个新列，以捕获更多信息。这在你最初创建表时是未知的，因为它是一个新的需求。

另一种情况是，如果一个列被创建了，但出于某种原因需要重命名，例如为了与另一个表匹配，或者为了反映它的真实用途。

为什么不直接删除表然后重新创建呢？有时那样更容易，但大多数情况下，修改表更容易。这是因为你并不总是有创建表的 SQL 语句，而且如果你删除了表，你将丢失其中的所有数据。

我们将通过一些修改表的示例来演示如何操作。

### ALTER TABLE 语句

修改表结构的方法是使用一个名为 `ALTER TABLE` 的语句。它将改变表的结构，这与 `UPDATE` 语句（它更改表内的数据）不同。

`ALTER TABLE` 语句可以做相当多的事情：

*   向表中添加一个或多个列
*   更改列的数据类型
*   向列添加约束（如主键或外键）
*   从表中删除列
*   重命名列
*   重命名表

`ALTER TABLE` 语句如下所示：

```
ALTER TABLE 表名 修改表子句;
```

它包括几个部分：

*   以 ALTER TABLE 开头。
*   然后指定你想要修改的表名。
*   最后，你指定一个 `alter_table_clause`。它描述了你要做的更改。

让我们看一些修改表的例子。

在最近几章中，你已经学习了如何操作表和数据。具体来说，你已经学习了如何：

*   创建表
*   向表中添加数据
*   更新表中的数据
*   从表中删除数据

还有两个与表相关的操作你将学习到：如何更新表的结构，以及如何删除表。

### 示例：添加列

你可以使用 `ALTER TABLE` 语句向现有表中添加新列。这类语句看起来如下所示：
```
ALTER TABLE table_name
ADD column_name column_definition;
```

它包含了表名、新列的名称以及你想要添加的新列的详细信息，例如数据类型。

假设你想为 `employee` 表添加一个新列来存储员工职位。这是一个文本值，100 个字符应该足够容纳它。你的 `ALTER TABLE` 语句看起来会是这样：
```
ALTER TABLE employee
ADD job_title VARCHAR2(100);
```

这个语句会在 `employee` 表中添加一个名为 `job_title` 的新列，数据类型为 `VARCHAR2`，最大长度为 100 字节。列名的长度限制为 128 字节（截至 12.2 版本）。虽然较长的列名可能更具描述性，但需要在列名足够描述性与不过长之间取得平衡。

要运行此语句，只需像处理其他语句一样，将其输入到 SQL Developer 窗口中，然后点击“运行”按钮。你会在输出面板中收到一条消息，内容为“Table EMPLOYEE altered.”。

如果你向一个已包含数据的表中添加一个新列，那么这个列中的值会是什么呢？让我们来一探究竟。你可以通过运行一条 `SELECT` 语句来查看表中的内容，包括你的新列：
```
SELECT *
FROM employee;
```

你的表应该大致如下所示，具体取决于你是否运行了前面章节中的 `UPDATE` 语句：

| ID | LAST_NAME | SALARY | JOB_TITLE |
| --- | --- | --- | --- |
| 1 | JONES | 30000 | (null) |
| 2 | SMITH | 35000 | (null) |
| 3 | KING | 45000 | (null) |
| 4 | SIMPSON | 52000 | (null) |
| 5 | ANDERSON | 31000 | (null) |
| 6 | COOPER | (null) | (null) |
| 7 | ADAMS | (null) | (null) |
| 8 | SMITH | 62000 | (null) |
| 9 | PATRICK | 40000 | (null) |

新的 `job_title` 列中每条记录的值都是 `NULL`。这是因为当我们添加列时，不可能知道每条记录的对应值应该是什么，因此它被设置为 `NULL`。你可以运行一条 `UPDATE` 语句来为现有记录更新 `job_title` 的值。

### 示例：更改数据类型

`ALTER TABLE` 语句的另一项用途是更改列的数据类型。这在以下等情况中非常有用：
*   你需要扩大列的容量，例如增加 `NUMBER` 或 `VARCHAR2` 字段的大小。
*   你要将一个列从文本值更改为数字值，因为它将要引用另一个表中的主键。
*   你需要捕获日期时间值的额外部分。

更改一个已包含数据的列的数据类型可能会导致问题，所以需要谨慎操作。根据你所做的更改，数据可能会被保留，或者你可能会收到错误。如果你确实想要更改列的数据类型，你的语句看起来会是这样：
```
ALTER TABLE table_name
MODIFY column_name data_type;
```

举个例子，如果你想将 `job_title` 列从 `VARCHAR2(100)` 更改为 `NVARCHAR2(250)`，你的语句看起来会是这样：
```
ALTER TABLE employee
MODIFY job_title NVARCHAR2(250);
```

你只需要指定新的数据类型，而不需要指定旧的数据类型。如果你在 SQL Developer 中的数据库上运行此语句，你会收到一条“Table EMPLOYEE altered”的消息。

### 示例：添加主键

在之前的章节中，我们解释了主键的概念（表中用于唯一标识一行的一个或多个列）。你在创建 `sales_meeting` 表时为其添加了主键。你也可以使用 `ALTER TABLE` 语句向已存在的表中添加主键。

用于添加主键的 `ALTER TABLE` 语句看起来如下所示：
```
ALTER TABLE table_name
ADD CONSTRAINT constraint_name constraint_type;
```

这个语句使用了 “`constraint`” 这个词。约束是数据库中的一种对象，用于对表中的列应用规则。主键就是一种约束。适用于主键的规则有：
*   它必须是唯一的。
*   它不能包含任何 `NULL` 值。
*   每个表只能有一个主键。
*   它可以应用于一个或多个列。

我们的 `employee` 表目前没有主键，但你现在可以添加一个。用于此操作的语句将是：
```
ALTER TABLE employee
ADD CONSTRAINT pk_emp_id PRIMARY KEY (id);
```

这个语句包含了：
*   `ALTER TABLE employee` 来标识正在被修改的表。
*   `ADD CONSTRAINT` 来指定要在表上添加什么。
*   约束名称 “`pk_emp_id`”，这是新约束的名称。
*   `PRIMARY KEY`，指定这是一个主键约束。
*   列名 `id`，这意味着 `employee` 表上的 `id` 列是主键。

为什么我们要给主键起一个名字？主键是存储在数据库中的东西，因此它需要一个名字。当你创建一个表时，你可以在想要设为主键的列后面直接添加 `PRIMARY KEY` 字样，而无需指定主键约束的名称。然而，Oracle 在创建时会自动为主键设置一个名称，那将是一个由字母和数字组成的长字符串。使用你自己的命名可以使日后更容易识别。

我在这个例子中使用的主键名称是 `pk_emp_id`。在为约束（如主键）命名时，我喜欢遵循一个标准的命名格式。这样你只需通过名称就能轻易知道它是什么。这个命名格式包括：
*   前两个字母表示约束类型（“`pk`”），然后加一个下划线。
*   表名的缩写（“`emp`” 是 `employee` 的缩写），然后加一个下划线。
*   列的名称，如果是长名称可以缩写（这里是 `id` 列）。
*   我使用的名字是 `pk_emp_id`，这意味着它是 `employee` 表上使用 `id` 列的主键。

如果你还没有运行，你应该在数据库上运行这条 `ALTER TABLE` 语句。下一个例子需要用到它。


### 示例：添加外键

外键是一个或多个列，用于引用另一张表中的主键。外键是良好数据库设计的一部分。Oracle 允许你添加外键约束，这些约束会强制执行外键规则，确保被引用的表中存在对应的记录。

例如，我们的 `sales_meeting` 表有一个 `employee_id` 列，如下所示：

```
ID   EMPLOYEE_ID   COMPANY            MEETING_DATE
1    5             ABC Construction   10/AUG/2018
2    2             BW Signage         21/AUG/2018
```

`employee_id` 列是外键，它引用了员工表的 id 列。我们可以使用 `ALTER TABLE` 命令为这张表添加一个外键约束：

```
ALTER TABLE sales_meeting
ADD CONSTRAINT fk_sm_empid FOREIGN KEY (employee_id) REFERENCES employee(id);
```

这条语句包含：

*   `ALTER TABLE sales_meeting` 用于指明我们要修改哪张表。
*   `ADD CONSTRAINT fk_sm_emp_id`，为新的约束指定一个名称。
*   `FOREIGN KEY (employee_id)` 表明这是一个外键约束，`employee_id` 是此约束应用到的当前表的列。
*   `REFERENCES employee (id)` 表示当前表中的这个列引用了 `employee` 表的 `id` 列。

如果你运行这条语句，会收到消息“Table SALES_MEETING altered”。这表示外键已应用到这张表。它确保：

`sales_meeting` 表中的 `employee_id` 与员工 `id` 列中的某条记录匹配。如果你为 `employee_id` 15 添加一条 `sales_meeting` 记录，但不存在 `id` 为 15 的员工记录，你会得到一个错误。

拥有相关 `sales_meeting` 记录的员工记录不能被删除。可以通过设置更改这一行为，但默认情况下是不允许的。你不能删除 `id` 为 5 的员工，因为他们有一条 `sales_meeting` 记录。

如果你运行此语句，而被引用的表（本例中是 `employee` 表）在 `id` 列上没有主键，你会得到如下错误信息：

```
Error starting at line : 21 in command -
ALTER TABLE sales_meeting
ADD CONSTRAINT fk_sm_empid FOREIGN KEY (employee_id) REFERENCES employee(id)
Error report -
ORA-02270: no matching unique or primary key for this column-list
02270. 00000 -  "no matching unique or primary key for this column-list"
*Cause:    A REFERENCES clause in a CREATE/ALTER TABLE statement
gives a column-list for which there is no matching unique or primary key constraint in the referenced table.
*Action:   Find the correct column names using the ALL_CONS_COLUMNS
catalog view
```

这意味着 `employee` 表在 `id` 列上没有主键。

### 示例：重命名列

要重命名表中的列，可以使用 `ALTER TABLE` 语句。语句如下：

```
ALTER TABLE table_name
RENAME COLUMN column_name TO new_column_name;
```

假设你想将新列从 `job_title` 重命名为 `position_title`。执行此操作的语句是：

```
ALTER TABLE employee
RENAME COLUMN job_title TO position_title;
```

如果你运行这条语句，你会像之前的示例一样得到“Table EMPLOYEE altered”的消息。你也可以通过运行 `employee` 表上的 `SELECT` 语句来查看变更已生效：

```
SELECT *
FROM employee;
ID   LAST_NAME   SALARY   POSITION_TITLE
1    JONES       30000    (null)
2    SMITH       35000    (null)
3    KING        45000    (null)
4    SIMPSON     52000    (null)
5    ANDERSON    31000    (null)
6    COOPER      (null)   (null)
7    ADAMS       (null)   (null)
8    SMITH       62000    (null)
9    PATRICK     40000    (null)
```

此语句并未更改该列中的任何值。

### 示例：删除列

要从表中删除列，你也可以使用 `ALTER TABLE` 语句：

```
ALTER TABLE table_name
DROP [COLUMN] column_name
[CASCADE CONSTRAINTS];
```

关于这条语句有几点需要注意：

*   它使用了 `DROP` 关键字，这是从数据库中删除对象时使用的词。
*   `COLUMN` 这个词是可选的。如果省略 `COLUMN`，列名必须用括号括起来。
*   `CASCADE CONSTRAINTS` 这些词是可选的，我们稍后会解释其含义。

让我们删除 `employee` 表上的新列，该列名为 `position_title`。为此，我们的语句将是：

```
ALTER TABLE employee
DROP COLUMN position_title;
```

或者，这条语句同样有效：

```
ALTER TABLE employee
DROP (position_title);
```

如果你运行这两条语句中的任意一条，该列将从表中移除。你可以通过运行 `SELECT` 语句来确认：

```
SELECT *
FROM employee;
ID  LAST_NAME   SALARY
1   JONES       30000
2   SMITH       35000
3   KING        45000
4   SIMPSON     52000
5   ANDERSON    31000
6   COOPER      (null)
7   ADAMS       (null)
8   SMITH       62000
9   PATRICK     40000
```

`CASCADE CONSTRAINTS` 是什么意思？如果你尝试从一个有约束应用的表中删除列，比如 `sales_meeting` 表中的 `employee_id` 列，你会得到一个错误，提示无法删除具有约束的列。如果你想删除该列及其相关约束以避免此错误，可以指定 `CASCADE CONSTRAINTS`。这意味着该列及其所有约束都将被删除。

### 示例：重命名表

我们将要查看的 `ALTER TABLE` 语句的最后一个操作是重命名表。如果表有更合适的名称，或者你需要对软件进行更改而要求表名变更，你可能希望重命名表。

为此，`ALTER TABLE` 语句如下所示：

```
ALTER TABLE table_name
RENAME TO new_table_name;
```

假设我们为其创建数据库的公司认定，与客户的会议应该不仅仅代表销售。这意味着我们应该将 `sales_meeting` 表重命名为更具描述性的名称，例如 `customer_meeting`。

你的 `ALTER TABLE` 语句将是：

```
ALTER TABLE sales_meeting
RENAME TO customer_meeting;
```

当你运行这条语句时，表名将在数据库中更新。如果你尝试在旧表名上运行 `SELECT` 语句，会得到一个错误，提示表不存在。如果你在新表名上运行 `SELECT` 语句，你将看到表中的结果。

### 使用 DROP TABLE 删除表

到目前为止，本章你已经学习了如何修改表及其列。还有一件事我们尚未涉及，那就是从数据库中删除表。

如果你不再需要某个表，你可能希望删除它。删除不使用的表比将其留在数据库中更好，因为删除它们可以节省数据库空间，并且减少其他需要对数据库编写查询的人的困惑。

从数据库中删除表被称为“删除”（dropping）表。`drop` 一词用于描述从数据库中删除对象的过程，而表是一种对象。

`DROP TABLE` 语句用于删除表，其形式如下：

```
DROP TABLE table_name;
```

这是一个简短的语句，你唯一需要做的就是添加表名。例如，假设你需要删除 `office` 表。为此，你的语句将是：

```
DROP TABLE office;
```

如果你运行这条语句，该表将从数据库中删除。如果你想重建该表，可以使用前面章节提到的 `CREATE TABLE` 语句。

#### 注意

`ALTER TABLE` 和 `DROP TABLE` 都会自动执行 `COMMIT`，这意味着你无法通过运行 `ROLLBACK` 来撤销该语句。



### 摘要

许多对现有表的调整可以通过 `ALTER TABLE` 语句完成，例如添加、删除、丢弃列，以及重命名表。诸如主键和外键之类的约束也可以使用 `ALTER TABLE` 语句添加到现有表中。使用 `DROP TABLE` 语句可以从数据库中移除表。


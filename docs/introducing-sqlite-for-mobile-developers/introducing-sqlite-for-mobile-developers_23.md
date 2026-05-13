# 第 4 章：使用关系模型和 SQLite

你已经看到了如何创建表、向其添加行、向其中插入数据，然后检索数据。诚然，到目前为止涉及的数据量不大，但这些是数据库管理的基础。在此过程中，你已经看到 SQLite（像许多其他库和数据管理器一样）通常嵌入到框架、语言、类或数据库管理系统（`DBMS`）中。如前所述，如今，大多数 `DBMS` 都是关系数据库管理系统（`RDBMS`）。它们基于 E. F. Codd 于 1970 年首次提出的关系模型，并且通常使用 SQL 或其衍生物（如 SQLite！）来创建、管理和查询关系数据库。

你已经看到了如何使用众多 SQLite 图形化编辑器之一（你可以在网上搜索“SQLite editor”，可能还会包含你的操作系统来找到它们）。你也看到了如何直接使用基本的 SQLite 语法来创建表和处理其数据。本章增加了另一个工具：`sqlite3`，这是一个作为 SQLite 一部分分发的命令行工具。

> **注意：** 如今大多数数据库都使用关系模型，因此 `DBMS` 和 `RDBMS` 在某种程度上是可互换的术语。本章让你看到什么是*不可互换*的：它仅适用于关系数据库（如 SQLite）。关系数据库是两个表可以*相关*的数据库。是*表*彼此关联，并且这种关系是基于相关表中的*数据*创建的。因此，尽管本书大部分地方使用了 `DBMS`，但在本章中，因为主题是关系数据，我们使用术语 `RDBMS`。关系连接两个表，因此为了讨论关系，我们需要从两个表开始。

随着关系的构建，它们会变得非常复杂（也非常强大）。因此，有些人会回避它们，但没有理由担心。

本章概述了关系如何工作、为什么应该使用它们以及创建它们是多么容易。关系通常涉及两个或多个表，因此本章首先向你展示如何创建两个逻辑上相关的表。

然后你将看到如何使用 SQLite 来实际关联它们。（关系实际上可以从单个表关联回自身。这称为*自连接*。）

## 构建 Users 表

考虑在多人游戏中记录分数的情况。你可以基于第 3 章的基础知识来创建一个类似 `SimpleTable` 的表。在这种情况下，表是用于用户的。从数据管理的角度来看，一个包含名字和出生地的表与一个包含名字和电子邮件地址的表差别不大。（主要区别在于第二个表在跟踪游戏分数方面更有用，而这正是本例的重点。）

当你查看表 4-1 所示的表并将其与 `SimpleTable`（第 3 章中的表 3-1）进行比较时，会看到另一个区别。显式主键（列 `PK`）缺失了。此表使用 SQLite 内置的机制来为每一行创建一个具有唯一值的主键。请记住这一点，因为我们稍后会回到这一点。

**表 4-1.** 用户表

| 姓名 | 电子邮箱 |
| :--- | :--- |
| Rex | rex@champlainarts.com |
| Anni | anni@champlainarts.com |
| Toby | toby@champlainarts.com |

> **注意：** 如前所述，在 SQLite 中，除了括号内，大小写无关紧要。按照惯例，SQL 关键字大写。在本书中，表名和列名在文本中通常大写；在示例代码中，它们可能大写也可能不大写，以强化“这对 SQLite 来说无关紧要”这一事实。在实践中，确定一个要遵循的惯例是一个非常好的主意。这是因为当今许多语言确实能识别大写和小写字母的区别。因此，当你在编写 SQLite 代码（其中 `UseR`、`user` 和 `USER` 是相同的）和 Swift、Objective-C、Python、C 等其他语言（其中大小写重要）之间切换时，不遵守标准可能会无意中为自己或他人引入错误。（此外，命名和大小写的标准规则使你的代码在未来更易于阅读和维护。）

这是一个良好的开端，并且你已经在第 3 章中看到了创建此类表的代码。根据修改后的列名和数据，以下是创建和填充该表的方法。

```
sqlite> create table users (
...> Name char (128) not null,
...> email char (128)
...> );
sqlite> insert into users (Name, email) VALUES ("Rex", "rex@champlainarts.com");
sqlite> insert into users (Name, email) VALUES ("Anni", "anni@champlainarts.com");
sqlite> insert into users (Name, email) VALUES ("Toby", "toby@champlainarts.com");
sqlite> select * from users;
Rex|rex@champlainarts.com
Anni|anni@champlainarts.com
Toby|toby@champlainarts.com
sqlite>
```

现在你需要相关的表来跟踪分数。

## 构建 Scores 表

构建 `Scores` 表基本上是相同的代码。为了简单起见，你可以让每个用户从零分开始。在这个假设下，代码如下。

```
sqlite> create table scores (
...> Name char(128),
...> score integer
...> );
sqlite> insert into scores (name, score) VALUES ("Rex", 0);
sqlite> insert into scores (name, score) VALUES ("Anni", 0);
sqlite> insert into scores (name, score) VALUES ("Toby", 0);
sqlite> select * from scores;
Rex|0
Anni|0
Toby|0
sqlite>
```

表 4-2 显示了此时 `Scores` 表中的数据。



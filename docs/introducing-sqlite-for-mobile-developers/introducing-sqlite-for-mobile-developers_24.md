# 第四章：使用关系模型与 SQLite

你可以使用之前用过的语法来检索数据。若要从 `Users` 表中获取 `name` 和 `email` 地址，请使用以下代码：

***表 4-2.** 分数表*

**姓名**

**分数**

**Rex**

**0**

**Anni**

**0**

**Toby**

**0**

`sqlite> select name, email from users where name = "Anni";`
`Anni|anni@champlainarts.com`
`sqlite>`

你可以用类似的方式从 `Scores` 表中获取 `score`：

`sqlite> select name, score from scores where name = "Anni";`
`Anni|0`
`sqlite>`

## 关联表格

你需要一种方法在单个 `SELECT` 语句中引用两个表（这就是 *别名* 发挥作用的地方）。你还需要实际实现这种关联，而这又涉及到主键。本节将涵盖这两个主题。

### 使用别名在 SELECT 语句中标识多个表

挑战在于将表格和别名结合起来使用。你需要为 Anni 选择数据，但这里有一个问题：`Scores` 和 `Users` 表都有名为 `Name` 的列。如果你使用 `where name = "Anni"` 这样的条件，SQLite 将无法知道你指的是哪个 `Name` 列。你可以在 `SELECT` 语句中通过标识列来区分这两个列。具体如何标识并不重要，因此你可以这样写：

```
select a.name, email, score
from users a, scores b
where a.name = "Anni" and b.name = "Anni";
```

你为每个表使用了一个 *别名*。在 *FROM* 子句中，别名跟在表名后面；在 *SELECT* 和 *WHERE* 子句中，别名位于表名之前，并用句点与表名分隔。因此，在 `SELECT` 命令和 `WHERE` 子句中，你得到 `select a.name` 和 `where a.name = "Anni"`，而在 *FROM* 子句中，你得到 `FROM users a`。

虽然你可以使用任何想要的别名，但如果使用实际名称或其缩写，可以使代码更易于阅读，如下所示：

```
select users.name, email, score
from users, scores
where users.name = "Anni" and scores.name = "Anni";
```

含义明确（只在一个表中出现）的名称不需要别名。

如果你对数据和数据库有较多工作经验，就会认识到这种结构非常脆弱。一切都依赖于两个表中的名称相同。如果任何地方出现拼写错误，数据就无法匹配。即使没有拼写错误，如果名称更改（由于各种现实原因，这确实会发生）会怎样呢？你可能会得到一些使用早期名称的 `Scores` 记录和另一些使用后期名称的记录。

在继续探讨这个问题之前，值得看看 SQLite 已经在幕后做了什么：它将帮助你解决因名称更改而出现的问题。

### 使用 `rowid` 主键

在第 [3 章](http://dx.doi.org/10.1007/978-1-4842-1766-5_3)中，讨论了用于唯一标识表中每一行的 **主键**。

它们在检索数据时非常有用，尤其是在数据本身可能不正确或由于修订或其他情况而与你认为的情况不同时。如果你没有指定主键，SQLite 已经为每个表中的每一行创建了一个唯一的键。它被称为 *rowid*，你可以在任何你没有提供主键的表中看到它。对于本节所示的 `Users` 表，你可以使用以下代码访问主键：

```
sqlite> select name, rowid from users;
Rex|1
Anni|2
Toby|3
sqlite>
```

`Scores` 表的代码显示了其主键。

```
sqlite> select name, rowid from scores;
Rex|1
Anni|2
Toby|3
sqlite>
```

## 更改一个表中的名称

为了更改存储在表一行的数据，你需要使用 `UPDATE` 命令。因此，如果你想将数据库中的 Toby 改为 George，你可以使用如下语法：

```
sqlite> UPDATE scores
...> SET name = "George"
...> WHERE name = 'Toby';
```


你开始了解 SQL 是如何工作的了。像 `WHERE` 和 `FROM` 这样的基本子句，其语法是反复使用的。这段语法中唯一的新东西是 `UPDATE` 命令，它接受一个表名。这里的其他内容你都已经见过了。

如果你尝试重新运行上一节中的查询，你会看到 `Users` 和 `Scores` 表中的数据。

```
sqlite> select * from users;
Rex|rex@champlainarts.com
Anni|anni@champlainarts.com
Toby|toby@champlainarts.com

sqlite> select * from scores;
Rex|0
Anni|0
George|0

sqlite>
```

如果实际上 George 和 Toby 是两个不同的人，那么数据对不上是正常的。但如果情况是 Toby 决定现在使用他的中间名 George，那么 George 和 Toby 实际上是同一个人。如何将 Toby 的所有数据转移到 George 名下呢？

重新输入或者使用 `UPDATE` 是一种处理方法，但更好的方法是利用 SQLite 内置的主键作为*外键*。

### 使用外键

如果你跟着操作了，就可以使用以下语法看到两个表中的 `rowid` 和 `name` 值。对于 `Users` 表，语法如下。表 4-3 显示了数据。

```
sqlite> select rowid, name from users;
1|Rex
2|Anni
3|Toby
```

#### 表 4-3. Users 表

| rowid | 名称 | 电子邮箱 |
| :--- | :--- | :--- |
| 1 | Rex | rex@champlainarts.com |
| 2 | Anni | anni@champlainarts.com |
| 3 | Toby | toby@champlainarts.com |

对于 `Scores` 表，语法如下：

```
sqlite> select rowid, name from scores;
1|Rex
2|Anni
3|George
```

表 4-4 显示了分数。

#### 表 4-4. Scores 表

| rowid | 名称 | 分数 |
| :--- | :--- | :--- |
| 1 | Rex | 0 |
| 2 | Anni | 0 |
| 3 | George | 0 |

解决方案是使用一个*外键*。外键是一个表中的值，用于标识另一个表中的行。在这个案例中，你希望 `Users` 表中的 `rowid` 3 (Toby) 用于匹配 `Scores` 表中的 `rowid` 3 (George)。名称虽然不同（George 和 Toby），但你希望这种关联能够生效，以反映他们是同一个人的两个不同值这一事实。

> **注意**
> `rowid` 3 是你想要在两个文件之间建立关联的值，这只是基于这些文件构建顺序的偶然结果。本章后面，你会看到这些数字可以有不同的值，但目前，你看到的可能就是这里显示的相同值。

解决方案是创建一个新列——在数据库领域中通常被称为 `FK` 或 `Foreign Key`（外键）。有时数据库设计师会根据其来源给它命名，例如 `scorekey` 或 `scoreid`，以及 `userkey` 或 `userid`。因此，如果你查看 `Scores` 表，可能会看到主键或 `rowid`，以及一个独立的 `userkey`。如果你把 `Users` 表中的 `rowid` 值作为起点（因为那很可能是第一个被创建的表），那么你希望在 `Scores` 表中有一个匹配的 `userid` 值。

要给现有表添加一列，你需要使用 `ALTER TABLE` 命令。你需要指定要修改的表名、要添加的列名以及该列的类型。以下是用于创建一个名为 `userid` 的新列的语法：

```sql
ALTER TABLE Scores ADD COLUMN userid integer;
```

要查看此命令的结果，请在 `sqlite3` 中使用 `.schema` 命令。（因为它是一个 `sqlite3` 命令，所以以句点开头，并且末尾不需要分号。该命令接受一个参数，即你想要查看的表名。）如果你现在运行它，结果如下。你会看到 `userid` 已经被添加进去了。

```
sqlite> .schema scores
CREATE TABLE scores (Name char(128),
score integer
, userid integer);
sqlite>
```

现在，你需要为你刚刚添加到 `Scores` 表中的 `userid` 列输入数据。`Scores` 表中的 `userid` 值应该是 `Users` 表中对应记录的值。目前它们碰巧是相同的，但你很快就会改变它们。

以下是为 `Scores` 表中的 `rowid` 输入数据的语法。基本上，这与你将名称从 Toby 更改为 George 的方式相同。



sqlite> `update scores set userid = 1 where name = 'Rex';`

sqlite> `update scores set userid = 2 where name = 'Anni';`

sqlite> `update scores set userid = 3 where name = 'George';`

目前，只需验证 `Scores` 表中的数据是否如这里所示（无论是在 SQLite 中，还是在表 4-5 中）。

sqlite> `select rowid, userid, name, score from Scores;`

1|1|Rex|0
2|2|Anni|0
3|3|George|0

sqlite>

**表 4-5.** 添加了用户 ID 的 Scores 表

| 行 ID | 用户 ID | 姓名 | 分数 |
|-------|--------|------|-------|
| Rex   | Anni   | George |       |

## 第 4 章 ■ 处理关系模型与 SQLite

此表中的姓名不再具有关联意义，因为 `Scores` 表和 `Users` 表之间的关系是基于 `Users` 表的 `rowid` 与 `Scores` 表的 `userid` 相匹配而建立的。在 SQLite 中，`ALTER TABLE` 命令不允许你删除列：推荐的解决方法是删除旧表并创建一个新表（在许多情况下，更合适的方法是导出数据，导入到新表，然后删除旧表）。因此，目前你可以忽略 `Scores` 表中的 `Name` 列。从你的角度看，`Scores` 表可以如表 4-6 所示（尽管在实际表中仍存在一个不再需要的列）。

**表 4-6.** 添加了用户 ID 并忽略姓名的 Scores 表

| 行 ID | 用户 ID | 分数 |
|-------|--------|-------|
|       |        |       |

## 连接表

获取所需数据最简单的方法是使用 `WHERE` 子句来指定你需要 `users.rowid` 与 `scores.userid` 相匹配的数据。（请记住，当你没有提供主键时，SQLite 会自动创建 `users.rowid`，但你必须将适当的 `users.rowid` 值输入到 `scores.userid` 列中，才能使关系生效。）

如果你已经这样做了（并且如果你遵循了本章的步骤，那么你已经做到了），那么下面的代码将给出你想要的结果：

sqlite> `select users.name, scores.score from users, scores where users.rowid = scores.userid;`

Rex|0
Anni|0
Toby|0

sqlite>

这里有几点需要注意：

- 姓名来自 `Users` 表；`Scores` 表中的 `Name` 字段不再使用。因此，即使你可能在 `Scores` 表中仍然有 George 这个名字，`userid` 为 3 的记录（最初对应的是 Toby）也会显示为 George（假设这是同一个人，只是名字被正确更改了）。
- 只显示匹配的记录。向 `Users` 或 `Scores` 表添加一些新记录：除非 `Users` 表中的 `rowid` 与 `Scores` 表中的 `userid` 匹配，否则它们不会显示。
- 如果 `Scores` 表中的某条记录没有 `userid`，它将不会显示。

## 第 4 章 ■ 处理关系模型与 SQLite

### 小结

你可以在此通用结构的基础上，通过添加更多数据来扩展。尝试组合使用新记录，以及用户和分数的匹配与非匹配 ID。

为一个现有用户添加一条 `Scores` 记录，使得一个人有两条记录，但姓名来自 `Names` 表是正确的。

在第 [5](http://dx.doi.org/10.1007/978-1-4842-1766-5_5) 章中，你将探索更多增强 `SELECT` 语句的方法。

## 第 5 章 使用 SQLite 功能——SELECT 语句能做什么

`SELECT` 语句的功能远不止获取数据。除了连接子句（在第 [4](http://dx.doi.org/10.1007/978-1-4842-1766-5_4) 章中简要讨论过），你还可以对数据进行排序和分组，并提供中间表来进行类似于子查询的操作。如果你习惯于*过程式编程*（描述过程的每个步骤……做这个……做那个……如果这个为真则做另一件事否则做那件事……），你可能会注意到 SQL 和关系数据库鼓励你用*声明式编程*的思维方式来思考（获取所有是 X 的东西，对所有 X 做某事……）。

当你在声明式编程的世界里工作时，你能放入声明中的东西越多，你就越有利。用英语来说，这意味着以下所有内容：

-   与其编写代码来测试有效数据，不如尝试在数据库或框架中强制执行验证规则。（Xcode Core Data 模型编辑器将此推向了图形化的极限。）



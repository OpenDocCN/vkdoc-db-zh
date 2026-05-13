# 使用 SQLite 的 SELECT 语句功能

## 概述：采用声明式思维

与其循环遍历数据集中的每个项目并测试值以决定采用哪个过程分支，不如获取所有的 X 并对它们执行操作。（实际上，你是将 `if` 语句移入了 `SELECT` 语句中。）

本章探讨了许多可以改进你代码的 SQL 子句。请记住，实际上你是在编写一些函数式代码，但以声明式的方式进行，从而达到声明式的目标：对组内（或 `SELECT` 结果中）的每个项目执行相同的操作。

你还将学习如何在 SQLite 语句中使用变量。这可以显著提升你的 SQLite 性能。这个概念在 PHP、Android 和 Core Data 中都有实现，更不用说在许多常见数据库中了。

## 查看测试数据

本章的示例中使用了一些测试数据。这些数据实际上是在测试第 12 章的 Score 应用程序时生成的。所使用的表是 `Score`，用到的值包括 `rowid`（主键）、`userid`（用户编号，外键）和 `score`（具体分数）。一个给定的用户可以有多个分数，但每个 `user` 和 `rowid` 是唯一的。

以下是使用的基本数据：

```
sqlite> select rowid, userid, score from Score;
1|1|10
2|2|20
3|3|30
4|1|99
5|1|99
6|1|9
7|1|99
8|1|99
9|1|17
10|1|31
11|1|23
12|1|50
24|2|16
25|3|8
26|2|99
27|1|3
sqlite>
```

## 排序使数据更易用

行号是有序的，但要以有意义的方式查看数据，按如下方式排序是合理的：

```
sqlite> select rowid, userid, score from Score order by userid;
1|1|10
4|1|99
5|1|99
6|1|9
7|1|99
8|1|99
9|1|17
10|1|31
11|1|23
12|1|50
27|1|3
2|2|20
24|2|16
26|2|99
3|3|30
25|3|8
sqlite>
```

## 分组可整合数据

当你对数据进行分组时，会将每个组值的数据合并为单个项。

如果你想查找 `score` 的最大值，可以使用以下代码：

```
sqlite> select max(score) from Score;
```

> **注意** 在本例和其他示例中，请随时参考基本数据列表以理解结果。

如果对数据进行分组，你可以获得每个 `userid` 的最大值，如下所示：

```
sqlite> select userid, max(score) from Score group by userid;
1|99
2|99
3|30
sqlite>
```

生活不仅仅关乎最大值和最小值：SQLite 还能帮你找到中间值（平均值）。如果这些分数代表团队而不是用户，那么根据平均分获胜的将是第 1 队：

```
sqlite> select userid, avg(score) from Score group by userid;
1|49.0
2|45.0
3|19.0
sqlite>
```

下次当你想写循环来计算平均值或查找最大值、最小值时，请记住 `group by`。事实上，如果你在使用像 SQLite 这样支持 SQL（即使是比 SQLite 更基础、功能更弱的 SQL）的关系数据库，下次当你开始写循环来处理查询结果时，试着问问自己：“我能否在查询内部完成这个操作？”

## 在查询中使用变量

在第 6 章中，你将看到如何在 PHP 数据对象（PDO）中使用 SQLite。它在你的 PHP 代码中是一个单例对象，但它绝对是一个拥有自身方法的对象。正如你将在下一章看到的，查询数据库的过程包括使用四个方法。

- `new PDO` 创建 PDO 并连接到你的 SQLite 数据库。
- `prepare` 以你的 SQLite 查询作为其参数。
- `execute` 执行查询。
- `$query->fetchAll` 从查询中检索数据。

还有一个简单的 `query()` 调用，它一步完成查询的创建和执行。

许多数据库支持这种流程——创建可重复使用的查询，以及创建一次性运行的查询。当你意识到可以在查询内部创建变量时，逐步创建、准备和执行查询的过程的优势就变得清晰了。

因此，可用于从用户表中获取所有名字的基本查询可以是：
`$query = $sqlite->prepare("select * from users;");`

检索具有特定 `name` 值的记录的查询可以是：
`$query = $sqlite->prepare("select * from users where Name = 'Rex';");`
此时，使用一步完成“准备”和“执行”的调用似乎更容易：
`$query = $sqlite->query("select * from users where Name = 'Rex'");`

然而，如果你使用变量，独立语句的威力和效率就会变得清晰。这里有一个例子。从 `new PDO` 和 `prepare` 开始，但在查询中使用一个变量，如 `:name`。

```
$sqlite = new PDO('sqlite:sqlitephp.sqlite');
$query = $sqlite->prepare("select * from users where name = :name;");
```

你不必立即使用 `$query`，但当你决定使用它时，可以用如下代码为 `:name` 填充值：
`$query->bindValue(':name', "Rex");`

或者，如果你想使用一个变量：
```
$theName = "Rex";
$query->bindValue(':name', $theName);
```

然后执行查询：
`$query->execute()`

你可以在稍后更改绑定的值并重新执行查询：
```
$query->bindValue(':name', "Anni");
$query->execute()
```

这个过程节省了一些内存，并且效率高得多。查询可以在缺失数据尚未存在时进行准备，这样当你回来执行它时，需要做的工作就少得多。

## 总结

在本章中，你看到了一些可以在 SQLite 的 `SELECT` 语句中使用的额外技巧。任何可以移入查询中的操作，都意味着你需要编写和调试的代码会少一些。

有了这些背景知识，现在是时候学习在 PHP、Android 和 OpenDoc 中使用 SQLite 了。它们完成相同的事情，但方法略有不同。

保持不变的是 SQLite。


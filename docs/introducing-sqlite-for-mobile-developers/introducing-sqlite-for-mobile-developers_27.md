# 在 PHP 中使用 SQLite

SQLite 本身非常轻量级：在许多情况下，它对用户及其设备造成的负担极小。这种简洁结构与轻量级实现的结合，使得 SQLite 成为网站中使用的流行工具。由于它是单用户实现，常用于存储单个用户的数据，但正如之前提到的，它也可以用来为使用任何更强大 SQL 实现构建的大型系统进行原型设计。

无论你正在进行单用户项目还是为更大规模的系统制作原型，SQLite 都是一个不错的选择，并且它与 PHP 配合得很好。本章将简要概述（或回顾）PHP；然后展示如何使用 PHP 在网页上实现[第 3 章](http://dx.doi.org/10.1007/978-1-4842-1766-5_3)和[第 4 章](http://dx.doi.org/10.1007/978-1-4842-1766-5_4)中所示的基本表。

> **注意** 本章重点介绍 `php`/SQLite 集成的基础知识。关于如何在一系列网页中使用的示例，请参见[第 11 章](http://dx.doi.org/10.1007/978-1-4842-1766-5_11)。为了测试本章的代码，你需要一个基本的文本编辑器——最好是像 BBEdit 这样的编辑器，它能为你高亮显示语法，以便轻松发现小错误。你还需要一个服务器来上传你的 `php` 文件以及一个 SQLite 数据库。如果你正在维护一个网站，你会拥有访问这样一个服务器的权限来发布你的 `html` 页面。服务器上可能安装了也可能没有安装 `php`。大多数服务器都有，但出于安全原因，有些会限制访问。如果你遇到任何困难，请咨询你的服务器管理员。

### 为 PHP 定义“单用户”

SQLite 是为单用户设计的。它常与 `php` 一起用于实现网站。这是否意味着你的 `php`/SQLite 网站是为单用户设计的？

答案是“是的”，但细节可能会让你感到惊讶。当你在网页中使用 `php` 时，存在一个单一用户——那就是 Web 服务器。根据你的应用程序不同，SQLite 可能是一个好选择，也可能不是（但对于原型设计和测试来说，它几乎总是一个好选择）。静态数据库在大多数情况下如果数据量不大，都能很好地工作。

如果你的应用需要更新数据库，并且使用 SQLite，当用户数量稍多时（这也取决于用户更新数据的频率），你可能会很快遇到麻烦。

## 将 PHP 与 SQLite 结合：基础知识

`PHP` 可能是使用最广泛的服务器端编程语言。严格来说，它是一种脚本语言，这意味着它在运行时（在实际应用中，通常指网页加载时）被解释执行。在这种场景下，`PHP` 脚本的输出通常是一个网页。当 `PHP` 脚本运行时，它可以执行各种各样的操作，包括与其他进程和设备通信，在 SQLite 的上下文中，它可以与驻留在 Web 服务器上的 SQLite 数据库进行通信。

## 验证你的环境中的 PHP

首先验证你的服务器上是否安装了 `PHP`，并且你可以访问它。（如果你已经使用 `PHP` 处理网站或其他文件，可以跳过此步骤。）

下面是一个简单的基于 `PHP` 的网页代码。它只是请求在服务器上显示当前的 `PHP` 状态信息。如果它不起作用，请咨询你的管理员，因为如果缺少 `PHP` 部分，你将无法试验 SQLite 和 `PHP`。除了 `PHP`，你可能还需要验证服务器上是否可用 SQLite，但在许多（可能是大多数）情况下都是可用的。

> **警告** 与 SQLite 不同，`php` 是大小写敏感的，所以请注意你的大小写。这也是一个很好的理由，在你的 SQLite 代码中标准化使用区分大小写的列名，尽管这对 SQLite 来说无关紧要。当你开始用 `php` 操作数据时，大小写就会很重要了。

这是用于测试 `PHP` 的示例代码。粗体部分是 HTML 代码中的 `PHP` 代码——它由 `<?PHP` 和 `?>` 界定。它只是调用了 `phpinfo()` 函数。将此代码放在你服务器上的一个文件中。这个文件应该具有 `php` 扩展名（通常是小写，但在许多服务器上大小写无关紧要）。如果你将其命名为 `test.php` 并上传到你网站的根目录（大多数情况下与 `index.html` 或 `index.php` 在一起），你可以通过访问 [`mywebsite/test.php`](http://mywebsite/test.php) 来访问它——包括运行其中嵌入的 `PHP` 代码。

```html
<!DOCTYPE html>
<html>
<head>
<title>Chapter6</title>
<meta name="generator" content="BBEdit 11.1" />
</head>
<body>
<?PHP
phpinfo();
?>
</body>
</html>
```

### 准备 SQLite 数据库

你可以使用几乎任何 SQLite 命令配合 `PHP`，但在实践中，最常用的命令是那些管理已存在表中数据的命令。在某些环境中，它可能是一个空表，当用户访问你的网站时，由你的网站填充数据。

以下是准备[第 4 章](http://dx.doi.org/10.1007/978-1-4842-1766-5_4)中使用的一对数据库的代码。它们将在本章中再次使用。有几部分值得特别注意。它们以粗体显示。

*   在此示例中，使用名为 `sqlitephp.sqlite` 的数据库启动 `sqlite3`。如果它不存在，`sqlite3` 会创建它。通常，使用 `.sqlite` 或 `.db` 作为扩展名。（如果你使用自己的扩展名，在上传到网站时可能会遇到问题。）
*   创建了 `Users` 表。注意 `Name` 列不能为空——你必须有一个名字。[第 4 章](http://dx.doi.org/10.1007/978-1-4842-1766-5_4)中使用过的数据被填入。（这里显示为表 6-1。）
*   创建表并插入数据后，使用 `SELECT` 语句显示数据。注意明确请求了 `rowid`：典型的 `SELECT *` 会显示除像 `rowid` 这样的隐藏列之外的所有列。
*   你需要看到 `rowid` 值以便将它们插入 `Scores` 表中。`insert` 语句将 `rowid` 值插入 `Scores` 表，以便可以使用这种关系。表 6-2 显示了 `Scores` 表。
*   最后，一个 `SELECT` 语句使用来自 `Users` 和 `Scores` 表的数据，通过 `rowid`（用户表）与 `userid`（分数表）相关联，显示了姓名和分数。表 6-3 显示了数据。

完整的 `sqlite3` 代码如下所示。

**表 6-1.** 用户表

| **rowid** | **Name** | **E-mail** |
| :--- | :--- | :--- |
| 1 | Rex | rex@champlainarts.com |
| 2 | Anni | anni@champlainarts.com |
| 3 | Toby | toby@champlainarts.com |

**表 6-2.** 分数表

| **userid** | **Score** |
| :--- | :--- |
| 1 | 10 |
| 2 | 20 |
| 3 | 30 |

**表 6-3.** 关联的用户与分数表

| **Name** | **Score** |
| :--- | :--- |
| Rex | 10 |
| Anni | 20 |
| Toby | 30 |

```sql
Jesses-Mac-Pro:~ jessefeiler$ sqlite3 sqlitephp.sqlite
SQLite version 3.8.10.2 2015-05-20 18:17:19
Enter ".help" for usage hints.
sqlite> create table users (Name char (128) not null, email char (128));
sqlite> insert into users (Name, email) VALUES ("Rex", "rex@champlainarts.com");
sqlite> insert into users (Name, email) VALUES ("Anni", "anni@champlainarts.com");
sqlite> insert into users (Name, email) VALUES ("Toby", "toby@champlainarts.com");
sqlite> select rowid, name from users;
1|Rex
2|Anni
3|Toby
sqlite> create table scores (userid integer, score integer);
sqlite> insert into scores (userid, score) VALUES (1, 10);
sqlite> insert into scores (userid, score) VALUES (2, 20);
sqlite> insert into scores (userid, score) VALUES (3, 30);
sqlite> select name, score from users, scores where userid = users.rowid;
Rex|10
Anni|20
Toby|30
sqlite>
```

将你的数据库上传到你的 Web 服务器。它应该在你的桌面 Finder（OS X）或“开始”菜单（Windows）中可见。在此示例中，它被上传到你网站的根目录。

### PDO 与旧版 PHP/SQLite 数据库连接


## 连接到你的 SQLite 数据库

连接 SQLite 到你的数据库主要有两种方式。较旧的方式是过程式风格，而较新的方式（自那遥远的 2005 年起）是基于对象的。PDO 扩展（PHP 数据对象）在大多数情况下都更稳健，是更好的选择，除非你需要修改那些以旧式风格访问 SQLite 数据库的旧 PHP 代码。

如果你对使用面向对象编程有所顾虑，请放心，`PDO` 确实是面向对象的，但它与标准 PHP 代码配合得非常好。你不需要将所有代码都转换为对象和类：你可以将 `PDO` 用于你的 SQLite 数据库，并且，除非你特意寻找，否则你甚至可能不会意识到你正在使用一个对象。

`PDO` 是 SQLite 的一个扩展，但这个扩展已内置在当前版本中；事实上，你需要刻意努力才能让它不可用。本节将对其进行描述，虽然你可能会在网上的各种地方找到示例代码，但请记住 `PDO` 至今已有十多年历史，并且是当前版本。你可能会在使用旧式代码打开较旧的 SQLite 数据库时遇到困难。（如果你在查阅的代码中发现 `sqlite_open`，那就是较旧的——非 `PDO`——风格。）

这个过程涉及五个步骤，我们稍后将详细讨论。

1.  **创建一个新的 PDO 实例**。它包含你想要连接的数据库的名称。
2.  **创建一个查询**。通常将其放在一个名为 `$query` 的变量中，但你可以使用任何名称。通过调用你的 `PDO` 实例上的 `prepare` 方法来创建查询。如果成功，它会返回一个 `PDOStatement` 对象。
3.  **执行查询**。你通过在你刚刚创建的查询（`PDOStatement`）上调用 `execute` 方法来完成，如 `$query->execute()`。它会返回 `TRUE` 或 `FALSE`，但（目前）不会返回任何值给你。
4.  **从执行后的查询中获取结果**。（现在你拥有数据了。）数据通常存储在一个名为 `$results` 的数组变量中。你可以使用 `$query->fetch()` 来获取下一行结果。你可以使用 `$query->fetchAll()` 来获取所有结果的一个数组。选择哪种方法取决于你以及你的具体需求（以及你预期检索多少数据）。默认情况下，获取的结果是一个关联数组，元素由 SQL 列名标识。
5.  **使用你准备、执行并获取的查询结果**。

以下是每个步骤的细节。

你从一个 PHP 文件开始，你将使用此处显示的五个步骤通过该文件访问你的 SQLite。如果你将此文件命名为 `phpsql.php` 并将其上传到你的网站，你可以通过 `http://mywebsite/phpsql.php` 访问它。完整的文件将在本章后面展示。

以下是相关组件的详细说明。

## 1. 创建一个新的 PDO 对象

创建一个新的 `PDO` 对象并将其放入一个局部变量中。通常，该变量名为 `$sqlite`，但你可以使用任何名称。创建新的 `PDO` 对象时，必须传入 SQLite 文件的名称，包括相对于服务器上你的 PHP 文件位置的任何路径。引号内的名称以数据源名称（DSN）`sqlite:` 开头。这是包含文件位置的引号字符串的一部分。

以下是一个例子：
```php
$sqlite = new PDO('sqlite:sqlitephp.sqlite');
```

你还可以使用另一种风格。对你（以及将来修改你代码的人）来说，这种风格可能更清晰，也可能不会。这两种风格是可以互换的。
```php
$db = 'sqlite:sqlitephp.sqlphpproject';
$sqlite = new PDO($db);
```

> **注意** 还有其他 DSN 字符串，包括 `mysql`、`pgsql`（postgres）、`msql`、`sybase` 或 `dblib`（SQLServer 或 Sybase）、`odbc`、`ibm`（IBM DB2）、`informix`、`firebird`（Firebird 或 Interbase）。

#### 正确处理错误

为了你的用户（以及你在调试时），优雅地处理错误是一个好习惯。`or die` 子句在非常基础的层面上做到了这一点。你提供一个将被显示的字符串。在 PHP 代码生成网页的情况下，引号内的字符串可以包含 HTML 格式，如下例所示：
```php
$sqlite = new PDO('sqlite:sqlitephp.sqlite')
or die ("<h1>无法打开数据库</h1>");
```

出于调试目的，当 `or die` 没有失败时，你可能仍想提供一条诊断信息。你总可以在投入生产之前删除这些信息，但它们将帮助你跟踪每一步，查看其是否成功。
```php
echo "<p>PDO 成功</p>";
```

最终，最好使用 `try/catch` 块，但 `or die` 更简洁，并在本书中用于简化代码。

### 2. 创建并准备查询

你使用的查询与你在任何其他环境中（如 `sqlite3` 交互式编辑器）直接使用的查询相同。与 `PDO` 的唯一区别在于，你在使用查询之前先创建它。

以下是一个创建查询、将其存储在变量中并捕获失败以及记录成功（后者仅用于调试）的例子。注意，你必须在你创建的 `PDO` 实例上调用 `prepare` 方法——本例中是 `$sqlite`。
```php
$query = $sqlite->prepare ("select Name from users;")
or die ("<h1>无法准备</h1>");

echo "<p>准备成功</p>";
```

### 3. 执行查询

现在你已经通过 `PDO` 实例的 `prepare` 方法创建了一个查询，你可以在该查询上调用 `execute` 方法来实际运行它。像往常一样，捕获错误，并在调试时打印出成功信息。以下是代码。请注意，在此示例中，`or die` 子句没有使用任何特定的 HTML 格式。这只是为了向你展示两种方式都可以实现。
```php
$query->execute() or die ("无法执行");

echo "<p>执行成功</p>";
```

### 4. 获取结果

`execute` 方法返回 `TRUE` 或 `FALSE`，但在你获取它之前数据不可用。（如果你以前使用过其他数据库，你会认出这种模式。）你可以使用 `fetchAll` 一次性获取所有行，或者使用 `fetch` 单独获取每一行。

`fetch` 版本一次获取一行。默认情况下，它将结果集的下一行作为一个数组返回，该数组既按列名索引，也按一个反映你的 SQLite `SELECT` 语句中项目的、从零开始的列号索引。还有一个可选的 `$cursor_offset` 参数，允许你在结果集中跳转。

`fetchAll` 版本一次性获取所有数据作为一个数组。你将在下一步中看到 `fetchAll` 和 `fetch`。

### 5. 使用结果

你在代码中循环遍历返回的数组元素。无论哪种情况，你都在 PHP 代码中 `echo` 结果值，将返回的值与你想要使用的任何 HTML 结合起来。例如，在 `fetchAll` 场景中，你循环遍历返回的每一行，将每一行放入一个局部变量中（通常称为 `$row`，但你可以使用任何名称）。

以下是相关的获取和循环代码。（它引用了先前在表 6-1 中显示的 `Users` 表。）
```php
$result = $query->fetchAll() or die ("无法 fetchAll");

echo "fetchAll 成功 "; // 用于调试

foreach ($result as $row) {
    echo "<p>姓名: {$row['Name']}</p>";
}
```

将本章所有步骤组合在一起，你可以得到一个如下所示的 PHP 文件。请记得为你的数据库名称和你想要显示的列名进行定制。需要定制的代码以粗体显示。

> **注意** 为你想要创建的页面定制 HTML，但也请参阅第 11 章，其中添加了更多与 SQLite 配合良好的常用 HTML 代码。
```php
<html>
<head>
</head>
<body>
<?php
$sqlite = new PDO('sqlite:**sqlitephp.sqlite**')
or die ("<h1>无法打开数据库</h1>");

echo "<p>PDO 成功</p>";

$query = $sqlite->prepare ("select Name from users;")
```


```
否则终止 ("<h1>无法准备</h1>");

echo "<p>准备成功</p>";

$query->execute() or die ("无法执行");

echo "<p>执行成功</p>";

$result = $query->fetchAll() or die ("无法获取所有结果");

echo "成功获取所有结果 ";

foreach ($result as $row) {

echo "<p>".$row['Name']."</p>";

}

?>

</body>

</html>
```


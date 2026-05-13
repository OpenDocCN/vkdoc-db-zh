# 第三章：使用 SQLite 基础：存储与检索数据

它允许你直接试验 SQLite 代码，在本书中它被用作 SQLite 语法的试金石。（当你使用第三方工具（如图形编辑器）时，可能会遇到语法的细微变化，例如语句末尾是否需要分号——在 SQLite 中是需要的。）你可以通过 `.quit` 或 `.exit` 结束 `sqlite3` 会话。注意开头的句点，因为两者都是 `sqlite3` 命令，但末尾没有分号，因为它们不是 SQLite 语法（后者才需要分号）。

本书中的大部分 `sqlite3` 代码都在每行开头显示了提示符，这样你就能看出哪些是多行命令。请记住，命令末尾始终需要一个分号——如果你之前忘记输入，可以单独把它放在最后一行。

`sqlite3` 会与它为你创建的临时数据库一起工作，或者你也可以管理自己的数据库。这里展示这些命令是因为它们是 `sqlite3` 命令，而不是 SQLite 语法。

在以下小节中，我们将回顾你最需要使用的基本 `sqlite3` 命令。你可以在 [www.sqlite.org/cli.html.](http://www.sqlite.org/cli.html) 找到更多关于 `sqlite3` 命令的信息。

> **注意：** `sqlite3` 内部的命令以句点开头。

## 运行 sqlite3 并让它创建一个新数据库

只需输入 `sqlite3`。完成后，输入 `.exit` 或 `.quit`。这是一个简单的示例会话。

```
Jesses-Mac-Pro:~ jessefeiler$ sqlite3

SQLite version 3.8.10.2 2015-05-20 18:17:19
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .exit
```

## 创建并命名一个新的 sqlite3 数据库

命令 `sqlite3 mydatabase` 将创建一个名为 `mydatabase` 的新数据库。

```
Jesses-Mac-Pro:~ jessefeiler$ sqlite3 mydatabase

SQLite version 3.8.10.2 2015-05-20 18:17:19
Enter ".help" for usage hints.
sqlite> .quit
```

### 删除数据库

默认情况下，数据库创建在用户的根目录级别（只要那是你在命令行编辑器中设置的目录）。因此，如果你在 OS X 上，可以从 Finder 中删除它。只需转到你的根目录级别（即，与 Desktop、Documents、Downloads、Library、Movies、Music、Pictures、Public 和 Sites 等文件夹同级），然后删除你刚刚创建的数据库即可。

## 运行 sqlite3 并打开一个现有数据库

要打开一个现有数据库，请使用 `.open` 命令。如果你想在终端（或无论你的命令行编辑器是什么）中更改目录，请使用你的标准命令（通常是 `cd`）。在 OS X 的终端中，输入 `cd` 后，只需将你想要放置文件的文件夹拖入终端：它会自动获取路径并将其插入到你的代码中。

```
Jesses-Mac-Pro:~ jessefeiler$ sqlite3

SQLite version 3.8.10.2 2015-05-20 18:17:19
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open testdb
sqlite> .quit
```

## 试验 SQLite 语法

SQLPro for SQLite 编辑器对于演示 SQLite 语法很有用，因为像许多其他 SQLite 编辑器一样，你可以使用更高级别的界面（如图形用户界面 GUI）来操作数据库，同时也可以看到生成的底层 SQLite 代码。在本书的后续部分，你将要处理的正是这些 SQLite 代码。

本节让你探索 SQLite 语法，但请记住，你将创建的大部分 SQLite 代码都将嵌入到你编写的应用程序中（或包含在你的应用程序中的框架、DBMS 或库内）。

网上有许多轻量级的 SQL 编辑器可用（只需搜索 “SQLite editor”）。由于 `sqlite` 文件是跨平台的，你可以使用这些编辑器来处理你拥有的任何 `sqlite` 文件（当然，要遵守操作系统实施的安全限制）。本章使用 SQLPro for SQLite 作为构建在 SQLite 之上的简单 GUI 的示例。它可在 [www.sqlitepro.com.](http://www.sqlitepro.com/) 获取。

这将向你展示从用户的角度来看，表 3-1（先前在第 1 章展示，此处重复）是如何创建的。

**表 3-1.** SimpleTable

| PK | 名称 | 来源 |
|----|------|--------|
|    | Cecelia | 澳大利亚 |
|    | Leif | 冰岛 |
|    | Charlotte | 美国 |

因为图形编辑器为你提供了对数据库所做操作的视图，它可能比命令行界面更容易使用，后者除了命令行提示符外没有其他指导。如果你更喜欢从命令行开始，请放心，在本章中，你不仅会在创建时看到表格的图形表示，还会看到在每一步生成的 SQLite 语法。

> **注意：** DB Browser 是另一个用于 SQLite 的编辑工具。它可用于 Windows、Mac OS X 和 Linux，网址是 [`sqlitebrowser.org`](http://sqlitebrowser.org/)。

SQLite 使用了 SQL 的一个子集（一个非常大的子集）。此外，对标准 SQL 语法也有一些细微的修改（坦白说，几乎每个 DBMS 都会做这样或那样的一些细微修改）。请放心，本章中展示的 SQL 在几乎每个环境中都适用。如果你从网上下载一个 SQL 编辑器（即标准 SQL 编辑器，而不是 SQLite 编辑器），你使用的语法可能与 SQLite 有非常细微的差别。

特别是，非常流行的 MySQL DBMS 有几个流行的编辑器——其中许多是免费的——但你可能会发现一些微小的差异。大多数人（包括我）都可以很好地工作，而无需担心这些区别：如果它们真的出现，解决起来也很简单。可能最重要的要点是，SQLite 中实现的 SQLite 语法可在 `sqlite.org` 获取。SQL 本身并不像 HTML 那样是一个标准，这就是为什么你可能会遇到这些差异。

## 关于主键

SQLite 表中的每一行都有一个 `rowid`——一个唯一的标识符，让你可以访问特定的行。通常，会为表自动创建一个唯一的 `rowid`。它甚至可能对用户不可见。理想情况下，主键不仅是唯一的，而且是无意义的。

事实上，无意义的主键通常比有意义的更有用。如果主键是一个人的名字、生日或地址，它可能会改变。只有一个完全无意义、不依赖于任何其他事物的值才能成功地作为主键。

这就是数据库开发人员经常隐藏其主键的原因之一——或者，在 SQLite 的情况下，他们让数据库在幕后处理它。如果你不希望 SQLite 为你做这件事，SQLPro for SQLite 中的复选框允许你使用幕后机制。在这个例子中，为了在第四章中演示关系，它被暴露了出来。

## 使用图形化 SQLite 编辑器探索你的 sqlite3 数据库

当你第一次打开图形化 SQLite 编辑器时，你可能会看到一个空的数据库（或者根本没有数据库）。和任何现代应用程序一样，系统可能会给你提供重新打开上次打开的数据库或导航到现有数据库的选项。

> **提示：** 请记住，SQLite 数据库位于它自己的文件中，通常带有 `sqlite` 扩展名。


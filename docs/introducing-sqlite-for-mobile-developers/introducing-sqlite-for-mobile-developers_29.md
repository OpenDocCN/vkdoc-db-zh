# 第 7 章 ■ 在 Android/Java 中使用 SQLite

部分原因是其轻量级且属于公共领域，SQLite 是许多对空间、功耗和成本有约束的项目的明显选择。由于这些及其他原因，它被内置到各种操作系统中，并在许多设备上使用。

本章向你展示在 Android 中使用 SQLite 的一些关键要素。

它被内置在 `android.database.sqlite` 中，因此你可以直接使用。使用 SQLite 的基础知识是相同的，无论你是使用 PHP 构建网站（参见第[6](http://dx.doi.org/10.1007/978-1-4842-1766-5_6)章和第[10](http://dx.doi.org/10.1007/978-1-4842-1766-5_10)章），还是在 iOS 或 OS X 上使用 Core Data（参见第[8](http://dx.doi.org/10.1007/978-1-4842-1766-5_8)章和第[12](http://dx.doi.org/10.1007/978-1-4842-1766-5_12)章）。

## 将 SQLite 与任何操作系统、框架或语言集成

无论你使用什么开发平台，将 SQLite 集成到你的应用程序中总是相同的（实际上，这适用于任何数据库）。

1.  设计你的数据库。这不是 SQLite 特有的问题。请参阅旁注，“关键的数据库设计步骤”。
2.  在运行时将你的应用程序连接到数据库。
3.  使用连接来添加、删除、插入或更新数据。
4.  在你的应用程序中使用操作结果。这可能包括通过添加或删除项目来刷新界面，或者可能包括处理新项目中的数据，或你和用户所需的任何其他内容。这不是 SQLite 特有的问题。

## 第 7 章 ■ 在 Android/Java 中使用 SQLite

### 关键的数据库设计步骤

第一步几乎与你的开发环境无关。识别你的数据、弄清楚关系并定义验证规则，只需要思考、讨论和对数据库进行一些草图绘制。此步骤的一个关键部分是为组件命名（是“Client”、“User”还是“Customer”），以便最终用户以及开发和维护代码的人员能够理解发生了什么。对于任何依赖数据的应用程序（无论是关系数据库、键值存储还是平面文件）的开发来说，这都是一个关键且经常被忽视的方面。本书专注于 SQLite，但如果你不熟悉数据管理的概念和可用的设计工具，请确保至少快速达到基本水平。

在将 SQLite 与 PHP、Android/Java 以及用于 iOS 和 OS X 的 Core Data 集成方面，存在某种递进关系。使用 PHP 时，你通常处理非常可见的 SQLite 代码，正如你将看到的，它大部分通常是过程式代码。当你使用 Core Data 时，你将使用面向对象的代码，框架及其类在幕后使用 SQLite 代码：你很少直接编写 SQLite 代码，但你一直在使用它。Android 和 Java 则取了一个中间地带，因此 SQLite 代码比在 Core Data 中更可见，但不如使用 PHP 时那么可见。

尽管有面向对象的 PHP 代码编写方式，但大体上你是在过程式环境中工作。然而，在当今世界，将 PHP 与数据库集成的推荐最佳实践是使用 PHP 数据对象（`PDO`）扩展。这是一个面向对象的类，它封装了你需要用来处理数据库的代码。它有针对 SQLite 和其他主要数据库的版本，因此你的 `PDO` 代码相对容易移植。

第[6](http://dx.doi.org/10.1007/978-1-4842-1766-5_6)章向你展示了在 PHP 中使用 SQLite 的编码基础。以下是步骤总结。这只是一个总结。在实践中，你将充实 `foreach` 循环，并且可能会用 `try`/`catch` 块替换 `or die`。

```
$sqlite = new PDO('sqlite:sqlitephp.sqlite');

$query = $sqlite->prepare (...SQLite 查询...);

$query->execute() or die ("无法执行");

$result = $query->fetchAll() or die ("无法获取所有结果");

foreach ($result as $row) {

...处理每一行结果

}
```

## 使用 Android 和 SQLite

Android 的 NotePad 示例是展示 SQLite 与 Android 集成的绝佳例子。你可以从 [`android.googlesource.com/platform/development/+/05523fb0b48280a5364908b00768ec71edb847a2/samples/NotePad/src/com/example/android/notepad/NotePadProvider.java`](https://android.googlesource.com/platform/development/+/05523fb0b48280a5364908b00768ec71edb847a2/samples/NotePad/src/com/example/android/notepad/NotePadProvider.java) 下载它。

该示例中的 `NotePadProvider` 代码扩展了 `ContentProvider`，后者封装了你需要的基本 SQLite 功能。这是你自己的应用程序中需要的那种代码。你可以阅读代码，但这里有一些需要注意的关键点。你需要在自己的应用程序中用你自己的值和字符串（比如你的列名、表名和数据库名）来实现它们。

### 使用静态值

如果你是这个环境的新手，你可能需要做一点搜索来找到使用的众多静态变量。这里有一个快速指南，告诉你它们可能在哪里。

#### 静态值可能在 API 中

`android.provider.BaseColumns` API 被 Android 系统中的内容提供者广泛使用。结构化数据在整个 Android 和许多其他操作系统中被使用（Cocoa 和 Cocoa Touch 的表类提供了有些类似的功能）。在示例代码中，来自 `BaseColumns` 的两个静态值被频繁使用。两者都是字符串。`_COUNT` 是目录中的行数，`_ID` 是行的唯一 ID。

#### 静态值可能在导入的文件中。

`NotePad.java` 包含 `NotePad` 类。该类本身包含 `NoteColumns` 类。

它包含以下静态值（以及其他）：

```
public static final String TITLE = "title";
public static final String NOTE = "note";
public static final String CREATED_DATE = "created";
public static final String MODIFIED_DATE = "modified";
```

#### 静态值可能在主文件中


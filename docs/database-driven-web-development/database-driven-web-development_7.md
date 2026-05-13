# 11. 数据库驱动的菜单系统

我们都见过优秀网站上的下拉菜单。它们是一个永恒的功能——已经被使用了几十年，并且无疑将继续使用很多年。在本章中，我们将运用之前关于 JavaScript 技巧的章节中介绍的知识，探讨 HTML、CSS、DOM 和 JavaScript 如何协同工作以提供一些相当惊人的功能。结合 Perl 脚本语言和 MySQL 数据库，你就拥有了一个非常强大的网页功能的基础。

## 下拉菜单是如何工作的？

下拉菜单理解起来很简单，只需一些提示。你的下拉菜单使用 `GET` 方法来加载你将使用的链接。值通过锚元素内声明的参数传递给脚本。

### create.pl

与大多数 Perl 脚本一样，在调用具有工作功能的脚本之前，总是需要创建一些表。这些脚本通常用于账户信息，如联系信息或个人详细信息——例如姓名和地址。你能做的限制取决于你的想象力。

`create.pl` 是一个简单的脚本，开始了我们对下拉菜单系统的探索。我用它来创建我们将在本章后面部分使用的基础表。在这个简单的脚本中，我们不填充任何值。我们只是创建基础表，值将通过其他脚本存储其中。请查看清单 11-1。

```perl
#!/usr/bin/perl
use DBI;
use CGI;
use CGI::Carp qw(fatalsToBrowser);
$cgi = new CGI;
#### 连接
$dbh = DBI->connect('DBI:mysql:host=localhost;
database=menu_system', 'user_name', 'password',
{'RaiseError' => 1}) or die "Cannot Connect to Database";
$query = qq{CREATE TABLE pages (
id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
id1 VARCHAR (250) NOT NULL,
id2 VARCHAR (250) NOT NULL,
hits VARCHAR (250) NOT NULL
)};
$sth = $dbh->prepare($query);
$sth->execute();
$sth->finish();
$dbh->disconnect();
#### 打印内容
print qq{Content-type: text/html\n\n};
print qq{

创建菜单表

数据库表已创建

};
1;
```
清单 11-1
检查 `create.pl`

如你所见，我们连接到数据库并创建了一个名为 "pages" 的表。目前这是一个空表，但下一个工作脚本将向数据库中填入值，你很快就会看到。这是你第一次看到 `id`、`id1`、`id2` 和 `hits` 列。我几乎在接手的每个项目中都使用它们。它们是数据库模型的起点。随着项目的增长，数据库系统的复杂性也会增加。

这个小程序中的第一步仅仅是数据库连接代码。它有四个项目，用于说明数据库的位置 (`localhost`)、正在使用哪个数据库（数据库名）、用户名和密码。

数据库位置声明为 `localhost`。当 Perl 脚本和 MySQL 数据库在同一服务器上运行时使用此值。如果你使用多服务器设置，可以将 URL 或 IP 地址声明为该值。

数据库名就是你给数据库起的名字。它可以是任何字母或数字，但尽量不要在数据库名中加入标点符号。一些常用的标点符号可能会干扰脚本。左单引号在这方面是出了名的。根据脚本的位置和操作，可能会抛出警告消息或致命错误。用户名和密码也应遵循此约定。

数据库连接代码之后是创建表。注意 `id` 列是 `auto_increment`。这意味着 `id` 列的值将从零开始。每当将新的表行插入数据库时，这个数字将增加一。这是一种方便的方法来指定表数据在将用于生成完整文档的变量和数组中的组织顺序。

我创建的表中的其他列是 `id1`、`id2` 和 `hits`。`id1` 和 `id2` 可以是任何你希望的数据类型。我在这里包含它们是为了让你更好地理解如何组织数据库表、行和列。

完成表创建后，我向浏览器的目标（即当前浏览器窗口）发送一个虽小但完整的 HTML 文档。现在该表已准备就绪。

### populate.pl

`populate.pl` 脚本用于向数据库插入值。正如我之前所说，你希望你的数据库表直观明了。我们将使用的表有四列：一个 `auto_increment` 的 `id`，以及 `id1`、`id2` 和 `hits`。我们将使用这个脚本做的是通过变量和数组将菜单项插入数据库。

我们最终要构建的菜单系统由两个下拉菜单组成。第一个简称为 `heading1`，包含两个链接。第二个下拉菜单名为 `heading2`，有四个列表项。`heading2` 有趣的地方在于列表项分为两列。如果你有异常长的菜单项列表，这是一个方便的功能。将一个长列表分成两个较短的列表将使菜单更易于阅读，并且成为一个更明显、功能更强的页面项目（清单 11-2）。

```perl
#!/usr/bin/perl
use DBI;
use CGI;
use CGI::Carp qw(fatalsToBrowser);
$cgi = new CGI;
#### 创建 heading1 和 heading2 变量及数组
@link1 = ('1', '2');
@link2 = ('1', '2', '3', '4');
$id1 = "1";
$id2 = "2";
### 连接，见鬼！！！
$dbh = DBI->connect('DBI:mysql:host=localhost;database=databasename', 'username', 'password', {'RaiseError' => 1}) or die "Cannot Connect to Database";
$count = "0";
foreach (@link1) {
$query = qq{INSERT INTO pages (id1, id2)
VALUES
('$id1', '$link1[$count]')};
$sth = $dbh->prepare($query);
$sth->execute;
++$count;
};
$count = "0";
foreach (@link2) {
$query = qq{INSERT INTO pages (id1, id2)
VALUES
('$id2', '$link2[$count]')};
$sth = $dbh->prepare($query);
$sth->execute;
++$count;
};
$sth->finish();
$dbh->disconnect();
#### 打印内容
print qq{Content-type: text/html\n\n};
print qq{

实验

数据库条目已插入
```


};
1;

#### 清单 11-2
#### 检查 populate.pl

```perl
```

如你所见，`populate.pl` 是**构建出色菜单系统**的一个良好开端示例。两个数组被遍历，并与两个变量 `$id1` 和 `$id2` 一起存入数据库。在完成这些简单的数据库插入后，一些 HTML 标记被发送到浏览器，显示一条消息，表明一切顺利。

### page.cgi

与所有 Perl 脚本一样，我们从 shebang 行开始。我总是把要使用的模块声明在脚本顶部。这样可以一目了然地看到脚本使用了哪些模块。有两个参数传递给脚本：`id1` 和 `id2`。它们包含在直观命名的 `$id1` 和 `$id2` 变量中。`page.cgi` 展示了将值传递给脚本是多么容易。它还展示了在哪里插入你的数据库调用以及生成 HTML 内容。

查看清单 11-3，这是一个完整的脚本，它将 `id1` 和 `id2` 参数存储在名为 `pages` 的表中。另请注意，有一个“hits”列。每次运行脚本时，它都会通过一个非常简单的语句递增该列。

```perl
#!/usr/bin/perl
use DBI;
use CGI;
use CGI::Carp qw(fatalsToBrowser);
$cgi = new CGI;
$id1 = $cgi->param('id1');
$id2 = $cgi->param('id2');
### connect, damnit !!!
$dbh = DBI->connect('DBI:mysql:host=localhost;
database=menu_system', 'user_name', 'password', {'RaiseError' => 1})
or die "Cannot Connect to Database";
$query = qq{INSERT INTO pages (id1, id2) VALUES ('$id1', '$id2') WHERE id2 = $id2};
$sth = $dbh->prepare($query);
$sth->execute;
$query = qq{UPDATE pages SET Hits = Hits + 1 WHERE id2 = $id2};
$sth = $dbh->prepare($query);
$sth->execute;
$query = qq{SELECT id1, id2, hits FROM pages WHERE id2 = $id2};
$sth = $dbh->prepare($query);
$sth->execute();
while (@thisid = $sth->fetchrow_array()) {
$id1 = $thisid[0];
$id2 = $thisid[1];
$hits = $thisid[2];
}
$sth->finish();
$dbh->disconnect();
### print da shtuff
print qq{Content-type: text/html\n\n};
print qq{

Experiment

Heading $id1, Page $id2, Visited $hits times

};
```

#### 清单 11-3
#### 检查 page.cgi

我们以插入操作开始数据库调用。包含在 `id1` 和 `id2` 参数中的值被插入到 `pages` 表中。

### menu.html

现在轮到呈现我们将要处理的文档的代码了。我将 HTML 标记保持得尽可能简单，以帮助学习。你们大多数人应该能够理解这个简单但完整的网页，它将作为下一个讨论主题。

#### 代码块一

这个代码块代表了一整套 CSS 规则。一如既往，我将其保持在最低限度以帮助学习。注意，我们从页面顶部的简单 `<html>` 元素开始。`!DOCTYPE` 元素不再必需。

此列表中的 CSS 规则展示了七种样式。请习惯在文档头部使用 CSS 规则。请注意，CSS 规则包含在大括号内，并整体封装在开头的 `<STYLE>` 和结尾的 `</STYLE>` 元素之间。

```css
--- Menu System ---

body{font-family:arial;}
table{font-size:10pt;
background:#336699;
}
a{
color:white;
text-decoration:none;
font:bold;
}
a:hover{
color:white;
text-decoration: underline;
}
td.menu{
background:#336699;
}
table.menu {
font-size:10pt;
position:absolute;
visibility:hidden;
}
.white {
font-size:10pt;
color: white;
font-weight: bold;
}
```

如你所见，有几种不同的语法组合允许 CSS 执行一些相当强大的操作。`td.menu` 适用于所有 `name` 属性为 "menu" 的 `TD` 元素。`table.menu` 也是如此。

#### 开始代码块二

这个简单但至关重要的代码块，是我们开始深入探讨我们将要探索的数据库驱动下拉菜单系统功能的地方。一如既往，JavaScript 代码写在开头的 `<script>` 和结尾的 `</script>` 元素之间。你可以在页面上的任何位置声明 JavaScript 代码，但通常的约定是将它们声明在文档头部。

#### getElementById()语句的语法

```javascript
function showmenu(elmnt) {
document.getElementById(elmnt).style.visibility="visible";
}
function hidemenu(elmnt) {
document.getElementById(elmnt).style.visibility="hidden";
}

代码清单 11-4
getElementById() 语句的语法
```

这段代码，即代码清单 11-4，将 `visibility()` 切换为 `visible` 或 `hidden`。我们在此这样做的原因是，我们希望整个文档都能选择使用下拉菜单。每次都将你的 JavaScript 代码插入到文档的头部始终是一个好主意。

#### 代码块三

现在进入文档的 BODY 部分。我们正在探讨的 HTML 标记简单而直观，即使你对编写文档内容所涉及的内容只有模糊的了解。我们从一个简单的表格声明开始，该声明显示了表格的基本维度，然后是进一步的表格数据，由 `TR` 和 `TD` 元素使用。请注意，最底部的代码块显示了两列，并排排列。当 `visible()` 方法被切换时，这将转化为多列菜单。

```html
&nbsp;&nbsp;标题 01

列表项

列表项

标题 02&nbsp;&nbsp;

列表项

列表项

列表项

列表项
```

请注意菜单数据是如何放置在文档中的。你可以在一个页面上放置尽可能多的下拉菜单窗口。特别重要的是 `id` 和 `name` 属性。它们在多个位置声明，以确保菜单数据显示在正确的菜单中。

#### 开始代码块四

代码块四包含一个封装了 `iFrame` 的简单表格的声明。正如你可能在之前的代码示例中注意到的，所有的锚点元素都将 `target` 属性设置为 `output`。`iFrame` 通过 `name` 属性被指定为 `output`。因此，脚本的输出将被写入 `iFrame`。整个页面不会刷新，只有 `iFrame` 的内容会刷新。本章中的所有脚本都以这种方式组合在一起，因此所有输出都被写入 `iFrame`。

正如你所看到的，名为 `empty.html` 的文件首先被加载到 `iFrame` 窗口中。这原本也可以是一个功能完整的 Perl 脚本，该脚本能够执行数据库交互，然后将生成的内容写入 `iFrame`。

## 总结

在本章中，我们探讨了一个简单但功能齐全的数据库驱动的下拉菜单系统的创建。下拉菜单系统将长期存在。现在你知道了如何创建一个基本的下拉菜单，但你也可以尝试使用它并添加功能。


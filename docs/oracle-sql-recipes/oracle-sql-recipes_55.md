# 第 18 章 ■ 对象管理

##### 18-12. 为对象创建别名

### 问题

你已被授予访问其他用户表的权限。你想知道是否有一种方法可以为这些表创建一个指针，这样每次访问表时就不必在表名前加上模式名。

### 解决方案

使用 `CREATE SYNONYM` 命令为其他数据库对象创建别名。以下示例为属于 `INV_MGMT` 用户的名为 `INV` 的表创建了一个同义词：

```sql
create or replace synonym inv for inv_mgmt.inv;
```

一旦创建了 `INV` 同义词，你就可以直接操作 `INV_MGMT.INV` 表。如果已授予对 `INV_MGMT.INV` 表的 `select` 访问权限，你现在就可以通过引用同义词 `INV` 进行选择。例如：

```sql
select * from inv;
```

创建同义词并不会创建访问对象的权限。此类权限必须单独授予，通常在创建同义词之前完成。

### 工作原理

创建一个指向其他对象的同义词，消除了指定对象所属模式或名称的需要。这让你能在对象和用户之间创建一个抽象层，通常称为对象透明性。同义词允许你将对象与访问对象的用户分开进行透明管理。

例如，你可以使用同义词在一个数据库内设置多个应用环境。每个环境都有自己的同义词，指向不同用户的对象，允许你在同一个数据库内对几个完全不同的模式运行完全相同的代码。你可能需要这样做，因为你无法负担为开发、测试、质量保证、生产等分别构建独立的服务器或数据库。

你可以为以下类型的数据库对象创建同义词：
- 表
- 视图或对象视图
- 其他同义词
- 通过数据库链接的远程对象
- PL/SQL 包、过程和函数
- 物化视图
- 序列
- Java 类模式对象
- 用户定义的对象类型

你还可以将同义词定义为 `public`，这意味着数据库中的任何用户都可以访问该同义词。有时，懒惰（或缺乏经验的）数据库管理员会这样做：

```sql
grant all on books to public;

create public synonym books for inv_mgmt.books;
```

现在，任何能连接到数据库的用户都可以对存在于 `INV_MGMT` 模式中的 `BOOKS` 表执行任何 `INSERT`、`UPDATE`、`DELETE` 或 `SELECT` 操作。DBA 可能会倾向于这样做，这样他们就不必为每个需要访问的模式单独设置授权和同义词。

这几乎总是一个坏主意。使用 `public` 同义词存在几个问题：
- 如果你没有意识到全局定义（`public`）的同义词，故障排除可能会很困难。
- 共享一个数据库的应用程序，如果多个应用程序使用在数据库内不唯一的 `public` 同义词，可能会发生对象名冲突。
- 安全性应按需管理，而不是批发式地授予。

有时，为需要私有同义词的模式动态生成所有表或视图的同义词很有用。以下脚本使用 SQL*Plus 命令来格式化并捕获一个 SQL 脚本的输出，该脚本为模式内的所有表生成同义词：

```sql
CONNECT &&master_user/&&master_pwd.@&&tns_alias
--
SET LINESIZE 132 PAGESIZE 0 ECHO OFF FEEDBACK OFF VERIFY OFF HEAD OFF TERM OFF TRIMSPOOL ON
--
SPO gen_syns_dyn.sql
--
select 'create synonym ' || table_name || ' for ' || '&&master_user..' ||
table_name || ';'
from user_tables;
--
SPO OFF;
--
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

### 只要用户对视图拥有 `select` 访问权限，他们就可以执行以下 SQL 语句来显示该视图所基于的 SQL：

```sql
select text from all_views where view_name like upper('&view_name');
```

如果你从 SQL*Plus 运行此 SQL，请使用 `SET LONG` 命令以显示整个视图。例如：

```
SQL> set long 5000
```




# 第 18 章 ■ 对象管理

检查约束在处理查找表时表现良好，特别是当您有一个包含少量相对静态值的短列表时，例如某列只能取`Y`或`N`值。在这种情况下，值列表很可能不会改变，并且除了`Y`或`N`之外没有其他信息需要存储，因此检查约束是合适的解决方案。

如果您有一个需要定期更新的长值列表，那么使用表和外键约束会是更好的解决方案（详见方案 18-14）。如果您有必须验证的复杂业务逻辑，那么应用程序代码会更合适。对于必须始终强制执行且可以用简单 SQL 表达式编写的业务规则，检查约束很有效。

[www.it-ebooks.info](http://www.it-ebooks.info/)

例如，如果您有一个业务规则要确保金额始终大于零，您可以使用检查约束：

```sql
alter table sales add constraint "sales_amt must be > 0" check(sales_amt > 0);
```

检查约束必须对插入或更新的行计算为真或未知（空值）结果。您不能在检查约束中使用子查询或序列。同时，也不能引用`UID`、`USER`、`SYSDATE`或`USERENV`等 SQL 函数，或者`LEVEL`、`ROWNUM`等伪列。

另一个常见的检查条件是判断列是否为空，为此应使用`NOT NULL`约束。`NOT NULL`约束可以通过多种方式定义。最简单的技术如下所示：

```sql
create table emp(
    emp_id number,
    emp_name varchar2(30) not null
);
```

一个稍好的方法是给`NOT NULL`约束起一个您认为有意义的名字：

```sql
create table emp(
    emp_id number,
    emp_name varchar2(30) constraint emp_name_nn not null
);
```

如果您需要修改现有表的列，可以使用`ALTER TABLE`命令。要使以下命令生效，被定义为`NOT NULL`的列中必须不能有任何`NULL`值。

```sql
alter table emp modify(emp_name not null);
```

如果当前被定义为`NOT NULL`的列中存在`NULL`值，那么要么先将该列更新为非空值，要么为该列指定一个默认值：

```sql
alter table emp modify(emp_name default 'not available');
```

## 18-16. 在数据库之间建立连接

### 问题

您希望在连接到本地数据库的同时，对远程数据库中的表运行 SQL 语句。

### 解决方案

使用数据库链接在两个数据库之间建立连接。您必须使用一种命名方法来解析远程数据库的位置。两种常用于解析远程数据库位置的方法是：

-   简单连接命名
-   本地命名

`简单连接命名`方法将远程数据库的位置直接嵌入到数据库链接中。在此示例中，远程主机是`OSS.EAST`，远程监听器的端口号是`1521`，远程实例名称是`BRDSTN`：

```sql
create database link mss connect to e123 identified by e123
using 'oss.east:1521/BRDSTN';
```

`CONNECT TO`和`IDENTIFIED BY`子句分别标识远程用户和密码。远程用户账户不必是远程对象的所有者，它可以是一个具有访问远程对象权限的普通账户。

`本地命名`方法通过`tnsnames.ora`文件中的条目解析远程服务。该文件需要位于 Oracle 客户端库在尝试连接到远程数据库时会查找的目录中，例如由操作系统变量`TNS_ADMIN`指定的目录。在此示例中，TNS（透明网络底层，Oracle 的网络架构）信息首先被放置在`tnsnames.ora`文件中：

```
mss =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL=TCP) (HOST=oss.east)(PORT=1521))
    (CONNECT_DATA=(SERVICE_NAME=BRDSTN)(SERVER=DEDICATED)))
```


#### 第 18 章：对象管理

## 创建数据库链接

现在可以通过引用 `tnsnames.ora` 文件中的 TNS 信息来创建数据库链接：

```sql
create database link mss connect to E123 identified by E123 using 'MSS';
```

要删除数据库链接，请使用 `DROP DATABASE LINK` 命令：

```sql
drop database link mss;
```

> **注意：** 还有另外两种远程数据库名称解析方法——`目录命名`（LDAP）和`外部命名`。有关如何实施此操作的详细信息，请参阅 *Oracle 数据库网络服务管理员指南*，该文档可在 `http://www.oracle.com/technology` 获取。

## 工作原理

数据库链接只不过是到另一个数据库的命名连接。它使您能够访问存在于不同数据库中的对象。换句话说，数据库链接允许您对存在于远程数据库中的表和视图执行 `插入`、`更新`、`删除` 和 `选择` 语句。另一个数据库不必是 Oracle 数据库。但是，您必须使用 Oracle 异构服务来访问非 Oracle 数据库。

创建数据库链接后，您可以使用语法 `<远程对象>@<数据库链接>` 来引用远程表和视图。此示例从远程视图 `INVENTORY_V` 中选择计数：

```sql
select count(*) from inventory_v@mss;
```

您还可以在本地视图、同义词或物化视图中引用数据库链接，以在远程对象和本地对象之间提供一个透明层。此示例创建一个指向远程 `INVENTORY_V` 视图的本地同义词：

```sql
create synonym inventory_v for inventory_v@mss;
```

以下查询将显示有关用户数据库链接的信息：

```sql
select db_link,username,password,host from user_db_links;
```

要查看数据库中所有数据库链接的信息，请查询 `DBA_DB_LINKS` 视图。使用 `ALL_DB_LINKS` 查看您的账户有权访问的所有数据库链接。查询 `USER_DB_LINKS` 以显示由您的账户拥有的数据库链接。

您还可以创建一个可供数据库中所有用户访问的公共数据库链接。要创建公共链接，请使用 `PUBLIC` 子句：

```sql
create public database link mss connect to e123 identified by e123 using 'oss.east:1521/BRDSTN';
```

##### 18-17. 创建自增值

### 问题

您希望能够自动递增一个值，用于填充表中的主键。

### 解决方案

创建一个序列，然后在 `插入` 语句中引用它。以下示例使用一个序列来填充父表的主键值，然后使用同一个序列在子表中填充相应的外键值：

```sql
create sequence inv_seq;
```

现在在 `插入` 语句中使用该序列。首次访问序列时，请使用 `NEXTVAL` 伪列：

```sql
insert into inv(inv_id, inv_desc) values (inv_seq.nextval, 'Book');
```

如果想重用同一个序列值，可以通过 `CURRVAL` 伪列引用它。

接下来，向子表中插入一条记录，该记录的外键列使用与其父主键值相同的值：

```sql
insert into inv_lines(inv_line_id,inv_id,inv_item_desc) values (1, inv_seq.currval, 'Tome1');
insert into inv_lines(inv_line_id,inv_id,inv_item_desc) values (2, inv_seq.currval, 'Tome2');
```

### 工作原理

`序列` 是一种数据库对象，用户可以访问它来选择唯一的整数。序列通常用于生成整数以填充主键和外键列。通过 `选择`、`插入` 或 `更新` 语句访问序列时，序列会递增。Oracle 保证选择的序列号将是唯一的。没有两个会话可以选择相同的序列号。

如果未为序列指定起始编号和最大编号，默认情况下起始编号为 1，增量为 1，最大值为 `10²⁷`。此示例指定起始值为 1000，最大值为 1000000：



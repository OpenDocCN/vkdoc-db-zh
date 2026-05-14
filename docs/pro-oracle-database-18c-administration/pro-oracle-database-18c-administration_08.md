# 9. 视图、同义词和序列

本章重点介绍视图、同义词和序列。视图广泛应用于报表应用程序中，也用于向用户展示数据的子集。同义词提供了一种透明的方式，允许用户显示和使用其他用户的对象。序列通常用于生成唯一的整数，以填充主键和外键值。

**注意**
尽管视图、同义词和序列看起来不如表和索引重要，但事实是，理解它们几乎同样重要。任何具有一定复杂度的应用程序都会涉及本章所讨论的内容。

## 实现视图

从某种意义上说，你可以将视图视为存储在数据库中的 SQL 语句。从概念上讲，当你从视图中选择时，Oracle 会在数据字典中查找视图定义，执行该视图所基于的查询，并返回结果。

除了从视图中选择数据，在某些场景下，还可以对视图执行 `INSERT`、`UPDATE` 和 `DELETE` 语句，这会导致对基础表数据的修改。因此，在这个意义上，与其简单地将视图描述为存储的 SQL 语句，不如将其概念化为建立在其他表或视图（或两者兼有）之上的逻辑表，这样更为准确。

话虽如此，以下是视图的常见用途：

*   创建一种高效存储 SQL 查询以供重用的方法。
*   在应用程序和物理表之间提供一个接口层。
*   向应用程序隐藏 SQL 查询的复杂性。
*   仅向用户报告列或行的子集，或两者兼有。

考虑到这一切，下一步是创建一个视图并观察其一些特性。

### 创建视图

你可以在表、物化视图或其他视图上创建视图。要创建视图，你的用户帐户必须具有 `CREATE VIEW` 系统权限。如果你想在另一个用户的模式中创建视图，那么你必须具有 `CREATE ANY VIEW` 权限。

作为参考，本节中的视图创建示例依赖于以下基表：

```
SQL> create table sales(
sales_id number primary key
,amnt    number
,state   varchar2(2)
,sales_person_id number);
```

同时假设表初始插入了以下数据：

```
SQL> insert into sales values(1, 222, 'CO', 8773);
SQL> insert into sales values(20, 827, 'FL', 9222);
```

然后，使用 `CREATE VIEW` 语句来创建视图。以下代码创建了一个视图（如果视图已存在则替换它），该视图从 `SALES` 表中选择列和行的子集：

```
SQL> create or replace view sales_rockies as
select sales_id, amnt, state
from sales
where state in ('CO','UT','WY','ID','AZ');
```

**注意**
如果你不想意外替换现有的视图定义，请使用 `CREATE VIEW view`，而不是 `CREATE OR REPLACE VIEW view`。如果视图已存在，`CREATE VIEW <view>` 语句将抛出 `ORA-00955` 错误，而 `CREATE OR REPLACE VIEW view` 会覆盖现有定义。

现在，当你从 `SALES_ROCKIES` 中选择时，它会执行视图查询并从 `SALES` 表返回相应的数据：

```
SQL> select * from sales_rockies;
```

根据视图查询，直观可见输出仅显示以下列和一行：

```
SALES_ID       AMNT ST
---------- ---------- --
        1        222 CO
```

不那么明显的是，你还可以对视图发出 `UPDATE`、`INSERT` 和 `DELETE` 语句，这会导致基础表数据的修改。例如，以下针对视图的插入语句会导致在 `SALES` 表中插入一条记录：

```
SQL> insert into sales_rockies(
sales_id, amnt, state)
values
(2,100,'CO');
```

此外，作为表和视图的所有者（或作为 DBA），你可以将 DML 权限授予视图的其他用户。例如，你可以将视图的 `SELECT`、`INSERT`、`UPDATE` 和 `DELETE` 权限授予另一个用户，这将允许该用户通过视图选择和修改数据。但是，拥有视图上的权限并不会给予用户直接 SQL 访问底层表的权限。因此，任何被授予视图权限的用户都将能够通过视图操作数据，但不能对视图所基于的对象发出 SQL。

请注意，你可以向视图插入一个值，导致基础表中出现一行该视图无法选择的数据：


## SQL 视图操作与限制

### 基本查询与插入

```
SQL> insert into sales_rockies(
sales_id, amnt, state)
values (3,123,'CA');
SQL> select * from sales_rockies;
SALES_ID       AMNT ST
---------- ---------- --
1        222 CO
2        100 CO
```

对比之下，对基表的查询显示存在一些视图无法返回的行：

```
SQL> select * from sales;
SALES_ID       AMNT ST SALES_PERSON_ID
---------- ---------- -- ---------------
1        222 CO            8773
20        827 FL            9222
2        100 CO
3        123 CA
```

如果你希望视图只允许那些最终能被该视图查询到的数据修改的插入和更新语句，那么可以使用`WITH CHECK OPTION`（参见下一节“检查更新”）。

### 检查更新

你可以指定，仅当底层表中的数据能被视图查询到时，才允许通过该视图修改这些数据。此行为通过`WITH CHECK OPTION`启用：

```
SQL> create or replace view sales_rockies as
select sales_id, amnt, state
from sales
where state in ('CO','UT','WY','ID','AZ')
with check option;
```

使用`WITH CHECK OPTION`意味着你只能插入或更新那些能被视图查询返回的行。例如，以下`UPDATE`语句能成功执行，因为它没有以导致该行无法被视图查询返回的方式修改底层数据：

```
SQL> update sales_rockies set state='ID' where sales_id=1;
```

然而，下一个更新语句会失败，因为它试图将`STATE`列更新为一个无法被视图查询选中的值：

```
SQL> update sales_rockies set state='CA' where sales_id=1;
```

在此例中，会抛出以下错误：

```
ORA-01402: view WITH CHECK OPTION where-clause violation
```

我很少看到`WITH CHECK OPTION`被使用。话虽如此，如果你的业务需求要求可更新视图只能更新能被其查询选中的数据，那么请务必使用此特性。

### 创建只读视图

如果你不希望用户能对视图执行`INSERT`、`UPDATE`或`DELETE`操作，那么不要将该视图的相应对象权限授予该用户。此外，对于任何你不希望底层表被修改的视图，你还应该使用`WITH READ ONLY`子句来创建它。默认行为是视图是可更新的（假设对象权限存在）。

此示例创建一个带有`WITH READ ONLY`子句的视图：

```
SQL> create or replace view sales_rockies as
select sales_id, amnt, state
from sales
where state in ('CO','UT','WY','ID','AZ')
with read only;
```

即使用户（包括所有者）拥有删除、插入或更新该视图的权限，如果尝试进行此类操作，也会抛出以下错误：

```
ORA-42399: cannot perform a DML operation on a read-only view
```

如果你使用视图进行报表展示，并且从未打算将视图用作修改底层表数据的机制，那么你应该总是使用`WITH READ ONLY`子句创建视图。这样做可以防止通过本不打算用于修改数据的视图意外修改底层表。

### 可更新连接视图

如果视图基于的 SQL 查询的`FROM`子句中定义了多个表，仍然有可能更新底层的表。这被称为可更新连接视图。

作为参考，以下是本节示例中使用的两个表的`CREATE TABLE`语句：

```
SQL> create table emp(
emp_id number primary key
,emp_name varchar2(15)
,dept_id number);
--
SQL> create table dept(
dept_id number primary key
,dept_name varchar2(15),
constraint emp_dept_fk
foreign key(dept_id) references dept(dept_id));
```

以下是为这两个表准备的一些种子数据：

```
SQL> insert into dept values(1,'HR');
SQL> insert into dept values(2,'IT');
SQL> insert into dept values(3,'SALES');
SQL> insert into emp values(10,'John',2);
SQL> insert into emp values(20,'Bob',1);
SQL> insert into emp values(30,'Craig',2);
SQL> insert into emp values(40,'Joe',3);
SQL> insert into emp values(50,'Jane',1);
SQL> insert into emp values(60,'Mark',2);
```

这是一个基于上述基表的可更新连接视图示例：

```
SQL> create or replace view emp_dept_v
as
select a.emp_id, a.emp_name, b.dept_name, b.dept_id
from emp a, dept b
where a.dept_id = b.dept_id;
```

关于允许执行 DML 操作的列，存在一些限制。例如，仅当满足以下条件时，才能更新底层表中的列：

*   DML 语句必须只修改一个底层表。
*   创建视图时未使用`READ ONLY`子句。
*   被更新的列属于连接视图中的键保留表（一个连接视图中只有一个键保留表）。

如果表的主键也可用于唯一标识视图返回的行，则视图中的该底层表是键保留的。数据示例将有助于说明底层表是否是键保留的。在此场景中，`EMP`表的主键是`EMP_ID`列；`DEPT`表的主键是`DEPT_ID`列。以下是查询本节前面列出的视图所返回的一些示例数据：

```
EMP_ID EMP_NAME        DEPT_NAME          DEPT_ID
---------- --------------- --------------- ----------
10 John            IT                       2
20 Bob             HR                       1
30 Craig           IT                       2
40 Joe             SALES                    3
50 Jane            HR                       1
60 Mark            IT                       2
```

从视图的输出可以看出，`EMP_ID`列始终是唯一的。因此，`EMP`表是键保留的（并且其列可以被更新）。相比之下，视图的输出显示`DEPT_ID`列可能不唯一。因此，`DEPT`表不是键保留的（并且其列不能被更新）。

当你更新视图时，任何导致映射到底层`EMP`表的列的修改都应被允许，因为`EMP`表在此视图中是键保留的。例如，此`UPDATE`语句会成功：

```
SQL> update emp_dept_v set emp_name = 'Jon' where emp_name = 'John';
```

但是，导致更新`DEPT`表列的语句则不被允许。下一条语句尝试更新视图中映射到`DEPT`表的一列：

```
SQL> update emp_dept_v set dept_name = 'HR West' where dept_name = 'HR';
```

以下是抛出的错误信息：

```
ORA-01779: cannot modify a column which maps to a non key-preserved table
```

总结一下，一个可更新连接视图可以从多个表中选择数据，但连接视图中只有一个表是键保留的。查询中表的主键和外键关系决定了哪个表是键保留的。



## 创建 INSTEAD OF 触发器
对于非只读的视图，当你对视图发出 DML 语句时，Oracle 会尝试修改该视图所基于的表中的数据。也可以指示 Oracle 忽略 DML 语句，转而执行一个 PL/SQL 代码块。此功能被称为 `INSTEAD OF` 触发器。它允许你以常规连接视图无法实现的方式修改底层基础表。

我并不是 `INSTEAD OF` 触发器的忠实拥护者。在我看来，如果你正在考虑使用它们，你应该重新思考一下如何发出 DML 语句来修改基础表。也许你应该允许应用程序直接对基础表执行 `INSERT`、`UPDATE` 和 `DELETE` 语句，而不是试图在视图上构建 PL/SQL `INSTEAD OF` 触发器。

请思考一下你将如何维护和排查 `INSTEAD OF` 触发器引发的问题。下一位数据库管理员弄清楚基础表是如何被修改的，是否会很困难？下一位数据库管理员或开发人员修改 `INSTEAD OF` 触发器是否容易？当 `INSTEAD OF` 触发器抛出错误时，能否明显看出是哪段代码抛出的错误以及如何解决该问题？

话虽如此，如果你确定需要在视图上使用 `INSTEAD OF` 触发器，请使用 `INSTEAD OF` 子句来创建它，并将所需的 PL/SQL 嵌入其中。此示例在 `EMP_DEPT_V` 视图上创建了一个 `INSTEAD OF` 触发器：

```sql
SQL> create or replace trigger emp_dept_v_updt
instead of update on emp_dept_v
for each row
begin
update emp set emp_name=UPPER(:new.emp_name)
where emp_id=:old.emp_id;
end;
/
```

现在，当对 `EMP_DEPT_V` 执行更新时，DML 语句不会被执行，Oracle 会拦截该语句并运行 `INSTEAD OF` PL/SQL 代码；例如，

```sql
SQL> update emp_dept_v set emp_name='Jonathan' where emp_id = 10;
1 row updated.
```

然后，你可以通过查询数据来验证触发器是否正确地更新了表：

```sql
SQL> select * from emp_dept_v;
EMP_ID EMP_NAME        DEPT_NAME          DEPT_ID
---------- --------------- --------------- ----------
10 JONATHAN        IT                       2
20 Bob             HR                       1
30 Craig           IT                       2
40 Joe             SALES                    3
50 Jane            HR                       1
60 Mark            IT                       2
```

这段代码是一个简单的例子，但它说明了你可以让 PL/SQL 代码代替在视图上运行的 DML 来执行。再次强调，使用 `INSTEAD OF` 触发器时要谨慎；务必确保你有信心能够高效地诊断和解决可能出现的任何相关问题。

## 实现不可见列
从 Oracle Database 12c 开始，你可以创建或修改表或视图中的列为不可见列（有关向表添加不可见列的详细信息，请参见第 7 章）。不可见列的一个优点是确保向表或视图添加列不会破坏任何现有的应用程序代码。如果应用程序代码没有显式访问不可见列，那么对于应用程序来说，就好像该列不存在一样。

一个小例子将展示不可见列的用处。假设你有一个表已创建并填充了一些数据，如下所示：

```sql
SQL> create table sales(
sales_id number primary key
,amnt    number
,state   varchar2(2)
,sales_person_id number);
--
SQL> insert into sales values(1, 222, 'CO', 8773);
SQL> insert into sales values(20, 827, 'FL', 9222);
```

此外，你还有一个基于前面表的视图，创建方式如下：

```sql
SQL> create or replace view sales_co as
select sales_id, amnt, state
from sales where state = 'CO';
```

为此示例的目的，假设你还有一个这样的报表表：

```sql
SQL> create table rep_co(
sales_id number
,amnt    number
,state   varchar2(2));
```

并且，它使用 `SELECT *` 通过这个插入语句进行填充：

```sql
SQL> insert into rep_co select * from sales_co;
```

一段时间后，视图中添加了一个新列：

```sql
SQL> create or replace view sales_co as
select sales_id, amnt, state, sales_person_id
from sales where state = 'CO';
```

现在，考虑一下插入到 `REP_CO` 中的语句会发生什么。因为它使用了 `SELECT *`，所以会中断，因为 `REP_CO` 表中没有添加相应的列：

```sql
SQL> insert into rep_co select * from sales_co;
ORA-00913: too many values
```

之前的插入语句不再能够填充 `REP_CO` 表，因为该语句没有考虑到已添加到视图中的额外列。

现在，考虑相同的场景，但使用不可见列将列添加到 `SALES_CO` 视图中：

```sql
SQL> create or replace view sales_co
(sales_id, amnt, state, sales_person_id invisible)
as
select
sales_id, amnt, state, sales_person_id
from sales
where state = 'CO';
```

当视图列被定义为不可见时，这意味着在描述视图或在 `SELECT *` 的输出中不会显示该列。这确保了基于 `SELECT *` 的插入语句将继续有效。

有人可以成功地辩称，你永远不应该创建基于 `SELECT *` 的插入语句，因此你永远不会遇到此问题。或者，有人可能会争辩说，此示例中的 `REP_CO` 表也应该添加一个列以避免此问题。然而，在处理第三方应用程序时，你通常无法控制编写不佳的代码。在这种情况下，你可以向视图添加不可见列，而不必担心破坏任何现有代码。

话虽如此，不可见列并非完全不可见。如果你知道不可见列的名称，你可以直接从中选择；例如，

```sql
SQL> select sales_id, amnt, state, sales_person_id from sales_co;
```

从这个意义上说，不可见列只是对编写不佳的应用程序代码或不知道该列存在的用户不可见。

## 修改视图定义
如果你需要修改视图所基于的 SQL 查询，那么可以删除并重新创建视图，或者使用 `CREATE OR REPLACE` 语法，如前面的示例所示。例如，假设你向 `SALES` 表添加了一个 `REGION` 列：

```sql
SQL> alter table sales add (region varchar2(30));
```

现在，要将 `REGION` 列添加到 `SALES_ROCKIES` 视图中，请运行以下命令来替换现有的视图定义：

```sql
SQL> create or replace view sales_rockies as
select sales_id, amnt, state, region
from sales
where state in ('CO','UT','WY','ID','AZ')
with read only;
```

使用 `CREATE OR REPLACE` 方法的优势在于，你不必为先前已授予权限的用户重新建立对视图的访问权限。`CREATE OR REPLACE` 的替代方法是删除并使用新定义重新创建视图。如果你删除并重新创建视图，则必须向先前被授予对该已删除并重新创建的对象访问权限的任何用户或角色重新授予权限。因此，在更改视图结构时，我几乎从不使用删除并重新创建的方法。

如果你从表中删除一个列，并且有一个相应的视图引用了被删除的列，会发生什么？例如，

```sql
SQL> alter table sales drop (region);
```

如果你尝试从视图中选择数据，将会收到 `ORA-04063` 错误。在修改基础表时，你可以通过编译视图来检查视图是否受到表更改的影响；例如，

```sql
SQL> alter view sales_rockies compile;
Warning: View altered with compilation errors.
```

通过这种方式，你可以主动确定表更改是否影响依赖的视图。在这种情况下，你应该重新创建不包含被删除表列的视图。



## 显示用于创建视图的 SQL

有时，在排查视图返回信息的问题时，你需要查看视图所基于的 SQL 查询。视图定义存储在 `DBA/USER/ALL_VIEWS` 视图的 `TEXT` 列中。请注意，`DBA/USER/ALL_VIEWS` 视图的 `TEXT` 列是 `LONG` 数据类型，默认情况下，SQL*Plus 只显示此类型的 80 个字符。你可以按如下方式将其设置得更长：

```
SQL> set long 5000
```

现在，使用以下脚本显示特定用户视图的关联文本：

```
SQL> select view_name, text
from dba_views
where owner = upper('&owner')
and view_name like upper('&view_name');
```

你也可以查询 `ALL_VIEWS` 来获取你有权访问的任何视图的文本：

```
SQL> select text
from all_views
where owner='MV_MAINT'
and view_name='SALES_ROCKIES';
```

如果你想显示当前模式中存在的视图文本，请使用 `USER_VIEWS`：

```
SQL> select text
from user_views
where view_name=upper('&view_name');
```

注意
`DBA/ALL/USER_VIEWS` 的 `TEXT` 列不会隐藏被定义为不可见列的相关信息。

你也可以使用 `DBMS_METADATA` 包的 `GET_DDL` 函数来显示视图的代码。从 `GET_DDL` 返回的数据类型是 `CLOB`；因此，如果你从 SQL*Plus 运行它，请确保先将 `LONG` 变量设置为足够大的尺寸以显示所有文本。例如，以下是将 `LONG` 设置为 5,000 个字符的方法：

```
SQL> set long 5000
```

现在，你可以通过使用 `SELECT` 语句调用 `DBMS_METADATA.GET_DDL` 来显示视图定义，如下所示：

```
SQL> select dbms_metadata.get_ddl('VIEW','SALES_ROCKIES') from dual;
```

如果你想显示当前连接用户所有视图的 DDL，请运行此 SQL：

```
SQL> select dbms_metadata.get_ddl('VIEW', view_name) from user_views;
```

## 重命名视图

重命名视图有几个充分的理由。你可能希望更改名称以使其更好地符合某个标准，或者你可能希望在删除视图前对其重命名，以便更好地确定它是否正在被使用。使用 `RENAME` 语句来更改视图的名称。此示例重命名了一个视图：

```
SQL> rename sales_rockies to sales_rockies_old;
```

你应该看到此消息：

```
Table renamed.
```

如果上述消息显示为 “View renamed” 会更合理；请注意，在此情况下，消息与操作并不完全匹配。

## 删除视图

在删除视图之前，请考虑先对其重命名。如果你确定一个视图不再被使用，那么保持你的模式尽可能整洁并删除任何未使用的对象是有意义的。使用 `DROP VIEW` 语句来删除视图：

```
SQL> drop view sales_rockies_old;
```

请记住，当你删除一个视图时，任何依赖于它的视图、物化视图和同义词都会变为无效。此外，与被删除视图相关的任何授权也会被移除。

## 管理同义词

同义词提供了一种为对象创建替代名称或别名的机制。例如，假设 `USER1` 是当前连接的用户，并且 `USER1` 对 `USER2` 的 `EMP` 表拥有 select 访问权限。如果没有同义词，`USER1` 必须像下面这样从 `USER2` 的 `EMP` 表中进行选择：

```
SQL> select * from user2.emp;
```

假设 `USER1` 拥有 `CREATE SYNONYM` 系统权限，它可以执行以下操作：

```
SQL> create synonym emp for user2.emp;
```

现在，`USER1` 可以透明地从 `USER2` 的 `EMP` 表中进行选择：

```
SQL> select * from emp;
```

你可以为以下类型的数据库对象创建同义词：

*   表
*   视图、对象视图
*   其他同义词
*   通过数据库链接的远程对象
*   PL/SQL 包、过程和函数
*   物化视图
*   序列
*   Java 类模式对象
*   用户定义的对象类型

创建一个指向另一个对象的同义词消除了指定模式所有者的需要，并且还允许你指定一个与对象名称不匹配的同义词名称。这让你可以在对象和用户之间创建一个抽象层，通常称为对象透明性。同义词允许你透明地管理对象，并与访问对象的用户分开管理。你还可以将对象无缝地重新定位到不同的模式甚至不同的数据库。引用这些序列的应用程序代码不需要更改——只需更改同义词的定义。现在，借助仅模式账户，同义词将允许引用对象，因为没有账户登录到这些模式中。

提示
你可以使用同义词在一个数据库内设置多个应用程序环境。每个环境都有自己的同义词，这些同义词指向不同用户的对象，允许你针对一个数据库内的几个不同模式运行相同的代码。你可能会这样做，因为你无法负担为开发、测试、质量保证、生产等构建单独的服务器或数据库。

### 创建同义词

用户在创建同义词之前必须被授予 `CREATE SYNONYM` 系统权限。一旦授予该权限，就可以使用 `CREATE SYNONYM` 命令为另一个数据库对象创建别名。如果你想让该语句在同义词不存在时创建同义词，或者在同义词已存在时替换同义词定义，可以指定 `CREATE OR REPLACE SYNONYM`。这通常是可接受的行为。

在此示例中，将创建一个提供对视图访问权限的同义词。首先，视图的所有者必须授予对该视图的 select 访问权限。这里，视图的所有者是 `MV_MAINT`：

```
SQL> show user;
USER is "MV_MAINT"
SQL> grant select on sales_rockies to app_user;
```

接下来，以将要创建同义词的用户身份连接到数据库。

```
SQL> conn app_user/foo
```

以 `APP_USER` 身份连接后，创建一个指向 `MV_MAINT` 拥有的名为 `SALES_ROCKIES` 的视图的同义词：

```
SQL> create or replace synonym sales_rockies for mv_maint.sales_rockies;
```

现在，`APP_USER` 可以直接引用 `SALES_ROCKIES` 视图：

```
SQL> select * from sales_rockies;
```

使用 `CREATE SYNONYM` 命令时，如果你没有指定 `OR REPLACE`（如示例所示），并且同义词已经存在，则会抛出 `ORA-00955` 错误。如果覆盖任何先前存在的同义词定义是可以接受的，那么请指定 `OR REPLACE` 子句。

同义词的创建并不会同时创建访问对象的权限。此类权限必须单独授予，通常在你创建同义词之前授予（如示例所示）。

默认情况下，当你创建同义词时，它是一个私有同义词。这意味着它由创建该同义词的用户拥有，并且其他用户无法访问它，除非他们被授予了适当的对象权限。

### 创建公共同义词

你也可以将同义词定义为公共的（有关私有同义词的讨论，请参阅前面的“创建同义词”部分），这意味着数据库中的任何用户都可以访问该同义词。有时，缺乏经验的 DBA 会执行以下操作：

```
SQL> grant all on sales to public;
SQL> create public synonym sales for mv_maint.sales;
```

现在，任何能够连接到数据库的用户都可以对存在于 `MV_MAINT` 模式中的 `SALES` 表执行任何 `INSERT`、`UPDATE`、`DELETE` 或 `SELECT` 操作。你可能倾向于这样做，这样你就不必为每个需要访问的模式单独设置授权和同义词。这几乎总是一个坏主意。使用公共同义词存在一些问题：

*   如果你没有意识到全局定义的（公共）同义词，故障排除可能会很麻烦；DBA 往往会忘记或不知道已经创建了公共同义词。




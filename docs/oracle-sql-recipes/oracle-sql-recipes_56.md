# 第 18 章 ■ 对象管理

##### 18-13. 在表中强制唯一行

### 问题

您需要确保输入数据库的数据符合特定的业务规则。例如，您需要确保不会在相同部门编号下误添加一个新部门。您希望使用 `PRIMARY KEY`（主键）约束来强制实现唯一性。

### 解决方案

在实施数据库时，您创建的大多数表都需要一个 `PRIMARY KEY` 约束，以保证表中的每条记录都能被唯一标识。有多种技术可以为表添加主键约束。第一个示例在定义列的同时内联创建主键：

```sql
create table dept(
    dept_id number primary key,
    dept_desc varchar2(30)
);
```

如果您从 `USER_CONSTRAINTS` 中查询 `CONSTRAINT_NAME`，您会注意到 Oracle 为该约束生成了一个晦涩的名称（类似 `SYS_C003682`）。使用以下语法可以显式地为主键约束命名：

```sql
create table dept(
    dept_id number constraint dept_pk primary key using index tablespace users,
    dept_desc varchar2(30)
);
```

**注意** 当您创建主键约束时，Oracle 还会创建一个与约束同名的唯一索引。

您也可以在列定义之后指定主键约束定义。这样做的好处是您可以在多个列上定义约束。下一个示例在创建表时创建主键，但不是内联于列定义：

```sql
create table dept(
    dept_id number,
    dept_desc varchar2(30),
    constraint dept_pk primary key (dept_id)
    using index tablespace prod_index
);
```

如果表已经创建，而您想添加主键约束，请使用 `ALTER TABLE` 语句。此示例在 `DEPT` 表的 `DEPT_ID` 列上放置一个主键约束：

```sql
alter table dept add constraint dept_pk primary key (dept_id)
using index tablespace users;
```

当启用主键约束时，Oracle 会自动创建一个与主键约束关联的唯一索引。一些 DBA 更喜欢先在主键列上创建一个非唯一索引，然后再定义主键约束：

```sql
create index dept_pk on dept(dept_id);

alter table dept add constraint dept_pk primary key (dept_id);
```

这种方法的优势在于，您可以独立于索引删除或禁用主键约束。在处理大型数据集时，您可能需要这种灵活性。如果您在创建主键约束之前没有创建索引，那么每当您删除或禁用主键约束时，索引也会被自动删除。

对于使用哪种方法创建主键感到困惑？所有方法都是有效的，各有优点。下表总结了主键和唯一键约束的创建方法。

```text
**表 18-1. 主键和唯一键约束创建方法**

**约束创建方法** | **优点** | **缺点**
--- | --- | ---
内联，无名称 | 非常简单 | Oracle 生成的名称使故障排查更难；对存储属性的控制较少；仅应用于单列
内联，有名称 | 简单；用户定义的名称使故障排查更容易 | 比内联无名称需要更多考虑
内联，有名称和表空间 | 完全控制约束和索引的命名及存储 | 语法更复杂
```

注意 `SELECT` 语句中带有两个点的 `&&MASTER_USER` 变量。在 & 符号变量末尾的单个点指示 `SQL*Plus` 将单个点后的任何内容连接到该 & 符号变量。因此，当您将两个点放在一起时，就告诉 `SQL*Plus` 将一个单点连接到 & 符号变量包含的字符串。

[www.it-ebooks.info](http://www.it-ebooks.info/)



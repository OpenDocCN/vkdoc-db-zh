# 第 18 章 ■ 对象管理

## 约束定义方法比较

- 用户定义的名称和表空间；较不简单；定义使故障排除更容易。
- 在列定义之后（行外）：可以对多个列进行操作。
- `ALTER TABLE 仅添加约束`：索引和约束分离；更复杂。
- 在单独的语句（和文件）中管理约束与表创建脚本；可以对多个列进行操作。
- `CREATE INDEX`，`ALTER TABLE add constraint`：最复杂，维护更多，组件更多；因此您可以删除/禁用约束而不影响索引；可以对多个列进行操作。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 工作原理

主键约束用于确保表中的每条记录都可以被唯一标识。一种常见的表设计技术是创建一个无意义的数字列来包含唯一生成的系统序列号（也称为代理键）。在代理键列上放置主键约束以确保其唯一性。

除了主键约束，我们还建议您在业务用户用于唯一标识记录的列（也称为逻辑键）上放置唯一键约束。同时使用代理键和逻辑键：
- 允许您在单个数字列上高效地连接父子表；
- 允许更新逻辑键列而无需更改代理键。

唯一键保证表中所定义列的唯一性。主键和唯一约束之间存在一些细微差别。每个表只能定义一个主键，但可以有多个唯一键。主键不允许其任何列包含空值，而唯一键允许空值。

与主键约束一样，创建唯一列约束有几种方法。此方法将 `UNIQUE` 关键字与列行内定义一起使用：
```sql
create table dept(dept_id number, dept_desc varchar2(30) unique);
```

如果想显式命名约束，请使用 `CONSTRAINT` 关键字：
```sql
create table dept(dept_id number,
    dept_desc varchar2(30) constraint dept_desc_uk1 unique);
```

您还可以在行内添加表空间信息，用于为唯一约束自动创建的关联唯一索引：
```sql
create table dept(dept_id number,
    dept_desc varchar2(30) constraint dept_desc_uk1
    unique using index tablespace prod_index);
```

**注意：**当您创建唯一键约束时，Oracle 还会创建一个与该约束同名的唯一索引。

与主键一样，唯一约束可以在列定义之外指定：
```sql
create table dept(dept_id number,
    dept_desc varchar2(30),
    constraint dept_desc_uk1 unique (dept_desc) using index tablespace prod_index);
```

您也可以通过修改表来添加唯一约束：
```sql
alter table dept add constraint dept_desc_uk1 unique (dept_desc);
```

并且您可以在定义唯一键约束之前，在感兴趣的列上创建索引：
```sql
create index dept_desc_uk1 on dept(dept_desc);

alter table dept add constraint dept_desc_uk1 unique(dept_desc);
```

当处理大型数据集并且您希望能够在不删除关联索引的情况下禁用或删除唯一约束时，这可能很有帮助。

[www.it-ebooks.info](http://www.it-ebooks.info/)

##### 18-14. 确保查找值存在

### 问题

您有一条业务规则，规定一列的值必须始终在有效的值列表中。例如，在 `EMP` 表中输入的任何 `DEPT_ID` 都必须始终存在于主表 `DEPT` 中。

### 解决方案

创建一个外键约束，以确保输入到外键列的值作为唯一键或主键值存在于父表中。创建外键约束有几种方法。以下在 `EMP` 表的 `DEPT_ID` 列上创建了一个外键约束：
```sql
create table emp(
    emp_id number,
    name varchar2(30),
    dept_id constraint emp_dept_fk references dept(dept_id));
```

请注意，`DEPT_ID` 的数据类型没有显式定义。它从被引用的 `DEPT` 表的 `DEPT_ID` 列派生数据类型。在定义列时，您也可以显式指定数据类型（无论是否定义外键）：
```sql
create table emp(
    emp_id number,
    name varchar2(30),
    dept_id number constraint emp_dept_fk references dept(dept_id));
```

外键定义也可以在 `CREATE TABLE` 语句中的列定义之外指定：
```sql
create table emp(
    emp_id number,
    name varchar2(30),
    dept_id number,
    constraint emp_dept_fk foreign key (dept_id) references dept(dept_id)
);
```

您还可以通过修改现有表来添加外键约束：
```sql
alter table emp add constraint emp_dept_fk foreign key (dept_id)
    references dept(dept_id);
```

**注意：**与主键和唯一键约束不同，Oracle 不会自动为外键列添加索引。您必须在外键列上显式创建索引。

### 工作原理

外键约束用于确保列值包含在定义的值列表中。使用数据库约束是强制执行数据在插入或更新被允许之前必须是预定义值的一种有效方式。约束的优点在于，一旦启用，就无法输入违反给定约束的数据。

外键约束是一种将有效值列表存储在表中的技术。此技术适用于以下场景：
- 值列表中的条目很多
- 需要存储有关查找值的其他信息
- 容易通过 SQL 选择、插入、更新或删除查找值

例如，您可能有一个 `STATES` 查找表，其中包含 `STATE_CODE`、`STATE_NAME`、`STATE_DESC` 和 `REGION` 等列。您的业务规则可能是您有一个 `ADDRESSES` 表，其中包含一个 `STATE_CODE` 列，在此列中输入的任何值都必须存在于 `STATES` 表中。对于此场景，使用带有外键约束的单独表非常有效。

如果您要检查的条件是一个不常更改的小列表，请考虑使用检查约束而不是外键约束（有关详细信息，请参阅配方 18-15）。例如，如果您有一个列将始终定义为仅包含 `0` 或 `1`，则检查约束是一个高效的解决方案。

[www.it-ebooks.info](http://www.it-ebooks.info/)

## 18-15. 检查数据是否满足条件

### 问题

您有一条业务规则，规定一列必须包含 `0` 或 `1` 的值。您希望确保该业务规则不会被违反。

### 解决方案

使用检查完整性约束为列定义 SQL 检查。创建检查约束有几种方法。例如，您可以在创建表时定义检查约束。以下定义了 `ST_FLG` 列必须为 `0` 或 `1`：
```sql
create table emp(
    emp_id number,
    emp_name varchar2(30),
    st_flg number(1) CHECK (st_flg in (0,1))
);
```

一个稍好一点的方法是给检查约束一个名称：
```sql
create table emp(
    emp_id number,
    emp_name varchar2(30),
    st_flg number(1) constraint st_flg_chk CHECK (st_flg in (0,1))
);
```

一个更具描述性的命名约束的方法是在约束名称中嵌入实际描述被违反条件的信息。例如：
```sql
CREATE table emp(
    emp_id number,
    emp_name varchar2(30),
    st_flg number(1) constraint "st_flg must be 0 or 1" check (st_flg in (0,1))
);
```

您还可以修改现有列以包含约束。该列不得包含任何违反正在启用的约束的值：
```sql
alter table emp add constraint "st_flg must be 0 or 1" check (st_flg in (0,1));
```

### 工作原理



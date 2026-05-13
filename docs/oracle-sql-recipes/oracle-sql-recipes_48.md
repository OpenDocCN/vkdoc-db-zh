# 第 18 章 对象管理

本章重点介绍用于管理数据库对象的 SQL 技术，包括表、索引、视图和同义词等对象类型。用于创建和修改对象的 SQL 语句通常被称为数据定义语言（DDL）语句。管理 DDL 是数据库管理员日常工作的一部分。DBA 经常生成和修改 DDL 脚本来构建开发、测试、预生产和生产数据库。因此，DBA 需要特别擅长编写 DDL。

有些人会争辩说，只有数据库管理员才应该维护和管理 DDL，开发人员不应该插手模式管理。然而，现实情况是，开发人员常常发现自己身处需要编写和维护 DDL 的环境中。例如，你可能在一个没有 DBA 的团队工作，或者你已经创建了自己的个人数据库用于开发和测试。在这些情况下，开发人员需要执行数据库管理任务，因此必须熟悉编写 DDL 语句。

本章不会涵盖所有可用的对象操作语句。相反，它重点介绍最常遇到的用户对象及其管理方法。我们将首先看基本的表创建和修改语句，然后逐步介绍其他典型的对象维护任务。

##### 18-1. 创建表

### 问题

你想执行创建表的基本任务。

### 解决方案

毫无疑问，你使用`CREATE TABLE`命令来完成此任务。以下是一个简单的例子：
```
create table d_sources(
    d_source_id number not null,
    source_type varchar2(32),
    create_dtt date default sysdate not null,
    update_dtt timestamp(5)
);
```

这段代码只创建了基本列定义的表，如果你赶时间，这通常就足够了。如果不指定任何表空间信息，表将在分配给你用户账户的默认表空间中创建。

查询`USER_TABLES`以查看当前连接账户内的表的元数据。从`ALL_TABLES`中选择以显示你的账户已被授予访问权限的表的信息。使用`DBA_TABLES`视图显示数据库中所有表的信息。

> **注意** 为什么使用`VARCHAR2`数据类型而不是`VARCHAR`数据类型？Oracle 是这么说的。尽管`VARCHAR`目前与`VARCHAR2`同义，但文档指出未来`VARCHAR`将被重新定义为单独的数据类型。

### 工作原理

Oracle 表是一个灵活且功能丰富的数据存储容器。如果你在 Oracle 文档中查找`CREATE TABLE`命令，你会发现大约七十页的细节。当你真正需要的只是一些好的例子时，翻阅冗长的文档可能令人生畏。

以下代码展示了创建表时最常用的功能。例如，这个 DDL 定义了主键、外键和表空间信息：
```
create table computer_systems(
    computer_system_id number(38, 0) not null,
    agent_uuid varchar2(256),
    operating_system_id number(19, 0) not null,
    hardware_model varchar2(50),
    create_dtt date default sysdate not null,
    update_dtt date,
    constraint computer_systems_pk primary key (computer_system_id)
        using index tablespace inv_mgmt_index
) tablespace inv_mgmt_data;

--

comment on column computer_systems.computer_system_id is
    'Surrogate key generated via an Oracle sequence.';

--

create unique index computer_system_uk1 on computer_systems(agent_uuid)
    tablespace inv_mgmt_index;

--

alter table computer_systems add constraint computer_systems_fk1
    foreign key (operating_system_id)
    references operating_systems(operating_system_id);
```

在表和索引创建语句中，请注意我们没有指定任何表物理空间属性。我们只为表和索引指定了表空间。如果没有指定表级空间属性，表将从表空间继承其空间属性。这简化了表的管理和维护。

让表从表空间继承其物理空间属性是一个好主意。如果你有需要不同物理空间属性的表，你可以创建单独的表空间来容纳具有不同需求的表。例如，你可以创建一个`DATA_LARGE`表空间，其扩展大小为 16M，以及一个`DATA_SMALL`表空间，其扩展大小为 128K。（关于扩展和其他逻辑结构的讨论见第 10 章，创建表空间的示例见配方 17-4。）

##### 18-2. 临时存储数据

### 问题

你想临时存储查询结果，以便会话稍后使用。

### 解决方案

使用`CREATE GLOBAL TEMPORARY TABLE`语句创建一个仅临时存储数据的表。

你可以指定临时表是保留数据直至会话结束还是直至事务提交。使用`ON COMMIT PRESERVE ROWS`指定数据应在用户会话结束时删除。在此例中，行将被保留，直到用户显式删除数据或用户终止会话：
```
create global temporary table today_regs
    on commit preserve rows
    as select * from f_registrations
    where create_dtt > sysdate - 1;
```

指定`ON COMMIT DELETE ROWS`表示数据应在事务结束时删除。

以下示例创建了一个名为`TEMP_OUTPUT`的临时表，并指定记录应在每个提交的事务结束时删除：
```
create global temporary table temp_output(temp_row varchar2(30))
    on commit delete rows;
```

> **注意** 如果你没有为全局临时表指定提交方法，则默认是`ON COMMIT DELETE ROWS`。

你可以创建一个临时表并授予其他用户访问它的权限。但是，一个会话只能查看它自己插入到表中的数据。换句话说，如果两个会话使用同一个临时表，一个会话无法选择另一个会话插入到临时表中的任何数据。

### 工作原理

全局临时表对于需要在表结构中短暂存储数据的应用程序非常有用。



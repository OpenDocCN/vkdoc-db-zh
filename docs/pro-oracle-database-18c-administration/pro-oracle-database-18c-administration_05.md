# 7. 表与约束

本书前面的章节涵盖了为你准备创建数据库对象的下一个逻辑步骤的主题。例如，在开始创建表之前，你需要安装 Oracle 二进制文件并创建数据库、表空间和用户。通常，为应用程序创建的第一个对象是表、约束和索引。本章重点介绍表和约束的管理。索引的管理在第 8 章中介绍。

表是数据库中数据的基本存储容器。你通过 DDL 语句（如 `CREATE TABLE` 和 `ALTER TABLE`）创建和修改表结构。
你通过 DML 语句（`INSERT`、`UPDATE`、`DELETE`、`MERGE`、`SELECT`）访问和操作表数据。

> 提示
> DDL 和 DML 语句之间的一个重要区别是，对于 DML 语句，你必须显式发出 `COMMIT` 或 `ROLLBACK` 来结束事务。




#### 约束

约束是一种强制数据遵守业务规则的机制。例如，你可能有一个业务要求：客户表中的所有客户 ID 必须唯一。在这种情况下，你可以使用主键约束来保证插入或更新到`CUSTOMER`表中的所有客户 ID 都是唯一的。约束会在数据被插入、更新和删除时进行检查，以确保没有违反任何业务规则。

本章介绍创建和维护表及约束的常用技术。通常，当你创建一个表时，该表需要定义一个或多个约束；因此，将约束管理与表一起介绍是合理的。本章第一部分重点关注常见的表创建和维护任务，后半部分则详细讨论约束管理。

### 理解表类型

Oracle 数据库支持多种强大且稳健的表类型。这些不同的类型在表 7-1 中描述。

表 7-1 Oracle 表类型描述

| 表类型 | 描述 | 典型用途 |
| :--- | :--- | :--- |
| 堆组织表 | 默认表类型，也是最常用的 | 除非有特定原因，否则应使用的表类型 |
| 临时表 | 会话私有数据，在会话或事务期间存储；空间在临时段中分配 | 程序需要临时表结构来存储和排序数据；程序结束后不再需要该表 |
| 索引组织表 | 数据存储在按主键排序的 B 树（平衡树）索引结构中 | 主要按主键列查询的表；提供快速的随机访问 |
| 分区表 | 由独立物理段组成的逻辑表 | 用于拥有数百万行数据的大表 |
| 外部表 | 使用存储在数据库外部操作系统文件中的数据的表 | 用于高效访问数据库外部文件（如 CSV 文件）中的数据 |
| 内存外部表 | 不需要加载到 Oracle 存储中，作为大数据集扫描一部分使用的数据 | 可以同时用于 RDBMS 和 Hadoop 内存扫描的数据 |
| 聚簇表 | 共享相同数据块的一组表 | 用于减少经常在同一列上连接的表的 I/O 操作 |
| 散列聚簇表 | 使用散列函数存储和检索数据的表 | 用于基本静态（初始加载后不再增长）的表以减少 I/O |
| 嵌套表 | 包含另一表作为数据类型的列的表 | 很少使用 |
| 对象表 | 包含对象类型作为数据类型的列的表 | 很少使用 |

本章重点介绍最常用的表类型：特别是堆组织表、索引组织表和临时表。分区表在数据仓库环境中广泛使用，将在第 12 章单独介绍。外部表在第 14 章介绍。有关本书未涵盖的表类型的详细信息，请参阅《SQL 语言参考指南》，该指南可从 Oracle 技术网络网站（`http://otn.oracle.com`）下载。

### 理解数据类型

创建表时，必须指定列名和相应的数据类型。作为 DBA，你应该理解每种数据类型的适当用法。我见过许多因选择了错误数据类型而导致的应用程序问题（性能和数据准确性）。例如，当应该使用日期数据类型时却使用了字符串，这会导致在进行日期计算和报告时出现不必要的转换和麻烦。更糟糕的是，一旦错误的数据类型在生产环境中实现，修改数据类型可能非常困难，因为这会引入可能破坏现有代码的更改。一旦走错路，再回头选择正确的道路将极其困难。更有可能的情况是，你会在试图迫使选择不当的数据类型去完成它本不该做的工作时，不得不进行一次又一次的修补。

说到这里，Oracle 支持以下几组数据类型：

*   字符型
*   数值型
*   日期/时间型
*   `RAW`
*   `ROWID`
*   LOB
*   JSON

以下各节将提供简要描述和使用建议。

> 注意：本书不涵盖专业数据类型、任意类型、空间类型、媒体类型和用户定义类型。有关这些数据类型的更多详细信息，请参阅可从 Oracle 技术网络网站（`http://otn.oracle.com`）获取的《SQL 语言参考指南》。JSON 将作为 Oracle 18c 的新特性和增强功能简要介绍。

#### 字符型

使用字符数据类型来存储字符和字符串数据。Oracle 中可用的字符数据类型如下：

*   `VARCHAR2`
*   `CHAR`
*   `NVARCHAR2`和`NCHAR`

##### VARCHAR2

`VARCHAR2`数据类型是在大多数情况下用于保存字符/字符串数据的类型。`VARCHAR2`仅根据字符串中的字符数分配空间。如果你将一个单字符字符串插入到定义为`VARCHAR2(30)`的列中，Oracle 只会消耗一个字符的空间。以下示例验证了此行为：

```sql
SQL> create table d(d varchar2(30));
insert into d values ('a');
select dump(d) from d;
```

以下是输出片段，验证了只分配了 1 字节（1B）：

```
DUMP(D)

Typ=1 Len=1
```

> 注意：Oracle 确实有另一个数据类型`VARCHAR`（没有“2”）。我提到这一点是因为你肯定会在 Oracle DBA 职业生涯的某个时刻遇到这种数据类型。Oracle 目前将`VARCHAR`定义为`VARCHAR2`的同义词。Oracle 强烈建议你使用`VARCHAR2`（而不是`VARCHAR`），因为 Oracle 的文档指出`VARCHAR`将来可能会有不同的用途。

当你定义`VARCHAR2`列时，必须指定长度。有两种方式：`BYTE`和`CHAR`。`BYTE`指定以字节为单位的最大字符串长度，而`CHAR`指定最大字符数。例如，要指定一个最多包含 30 字节的字符串，定义如下：

```sql
varchar2(30 byte)
```

要指定一个最多可以包含 30 个字符的字符串，定义如下：

```sql
varchar2(30 char)
```

许多 DBA 没有意识到，如果你既不指定`BYTE`也不指定`CHAR`，那么默认长度是以字节计算的。换句话说，`VARCHAR2(30)`等同于`VARCHAR2(30 byte)`。在几乎所有情况下，使用`CHAR`来指定长度更安全。使用多字节字符集时，如果你指定长度为`VARCHAR2(30 byte)`，可能无法得到可预测的结果，因为某些字符需要多于 1 字节的存储空间。相比之下，如果你指定`VARCHAR2(30 char)`，无论某些字符是否需要多于 1 字节，你总是可以在字符串中存储 30 个字符。

##### CHAR

（此处原文结束，后续内容未提供）


#### VARCHAR2 vs CHAR

在几乎所有场景中，`VARCHAR2`都比`CHAR`更可取。`VARCHAR2`数据类型比`CHAR`更灵活且空间效率更高。这是因为`CHAR`是固定长度的字符字段。如果你定义一个`CHAR(30)`并插入一个仅包含一个字符的字符串，Oracle 将分配 30 字节的空间。这可能是一种低效的空间使用方式。如果确实使用`CHAR`，那么只应在值的大小不会改变且绝对静态的情况下使用。我通常只在长度小于 8 且大小绝对固定时才使用`CHAR`。以下示例验证了这一行为：

```sql
SQL> create table d(d char(30));
insert into d values ('a');
select dump(d) from d;
```

以下是输出片段，验证了已消耗 30 字节：

```sql
DUMP(D)

Typ=96 Len=30
```

#### NVARCHAR2 and NCHAR

如果你的数据库最初是用单字节、固定宽度字符集创建的，但后来需要在同一数据库中存储多字节字符集数据，那么`NVARCHAR2`和`NCHAR`数据类型就很有用。你可以使用`NVARCHAR2`和`NCHAR`数据类型来满足此需求。更简单的做法是标准化使用`VARCHAR2`并提供足够的长度来处理多字节字符，或使用以字符为单位的长度，而不是使用`NVARCHAR2`和`NCHAR`。

> 注意
> 对于 Oracle Database 11g 及更低版本，`VARCHAR2`或`NVARCHAR2`数据类型允许的最大大小为 4,000。在 Oracle Database 12c 及更高版本中，你可以在`VARCHAR2`或`NVARCHAR2`数据类型中指定最多 32,767 个字符。
> 在 12c 之前，如果你想存储大于 4,000 个字符的字符数据，逻辑上的选择是`CLOB`（更多详情请参见本章后面的“LOB”部分）。

#### Numeric

使用数值数据类型来存储可能需要使用数学函数（如`SUM`、`AVG`、`MAX`和`MIN`）的数据。切勿在字符数据类型中存储数值信息。当你使用`VARCHAR2`来存储本质上是数字的数据时，你就是在给系统引入未来的故障。最终，你会希望报告或对数字数据运行计算，而如果它们不是数值数据类型，你将得到不可预测且通常是错误的结果。

Oracle 支持三种数值数据类型：

*   `NUMBER`
*   `BINARY_DOUBLE`
*   `BINARY_FLOAT`

对于大多数情况，你会使用`NUMBER`数据类型来处理任何类型的数字数据。其语法为`NUMBER(scale, precision)`，其中`scale`是总位数，`precision`是小数点右侧的位数。因此，对于定义为`NUMBER(5, 2)`的数字，你可以存储值+/-999.99。总共五位数字，其中两位用于小数点右侧的精度。如果定义为`NUMBER(5)`，则值可以位于小数点右侧或左侧，总位数为五位，2.4563 可以存入，55,555 同样可以。

> 提示
> Oracle 允许`NUMBER`数据类型最多 38 位数字。这对于几乎任何类型的数值应用都足够了。

有时让 DBA 困惑的是，你可以创建一个表，其列定义为`INT`、`INTEGER`、`REAL`、`DECIMAL`等。这些数据类型在 Oracle 中都是通过`NUMBER`数据类型实现的。例如，指定为`INTEGER`的列被实现为`NUMBER(38)`。`BINARY_DOUBLE`和`BINARY_FLOAT`数据类型用于科学计算。它们映射到 Java 数据类型`DOUBLE`和`FLOAT`。除非你的应用程序正在进行火箭科学计算，否则请使用`NUMBER`数据类型来满足所有数值需求。

#### Date/Time

在捕获和报告与日期相关的信息时，你应始终使用`DATE`或`TIMESTAMP`数据类型（而不是`VARCHAR2`）。使用正确的与日期相关的数据类型允许你执行准确的 Oracle 日期计算和聚合，以及用于报告的可靠排序。如果你对包含日期信息的字段使用`VARCHAR2`，你就是在保证未来的报告不一致和不必要的转换函数（如`TO_DATE`和`TO_CHAR`）。

Oracle 支持三种与日期相关的数据类型：

*   `DATE`
*   `TIMESTAMP`
*   `INTERVAL`

`DATE`数据类型包含一个日期成分和一个粒度到秒的时间成分。默认情况下，如果在插入数据时未指定时间成分，则时间值默认为午夜（0 时 0 秒）。如果你需要跟踪比秒更细粒度的时间，则使用`TIMESTAMP`；否则，请随意使用`DATE`。

`TIMESTAMP`数据类型包含一个日期成分和一个粒度到秒小数的时间成分。定义`TIMESTAMP`时，可以指定小数秒精度部分。例如，如果你想要小数点右侧五位的分数精度，可以指定为：

```sql
TIMESTAMP(5)
```

最大分数精度为 9；默认值为 6。如果你指定 0 分数精度，则等同于`DATE`数据类型。

`TIMESTAMP`数据类型还有两种额外变体：`TIMESTAMP WITH TIME ZONE`和`TIMESTAMP WITH LOCAL TIME ZONE`。这些是时区感知的数据类型，意味着当用户选择数据时，时间值会调整到用户会话的时区。

Oracle 还提供了一种`INTERVAL`数据类型。这旨在存储一段持续时间或时间间隔。有两种类型：`INTERVAL YEAR TO MONTH`和`INTERVAL DAY TO SECOND`。当需要年份和月份的精度时使用前者。当需要存储粒度到天和秒的间隔数据时使用后者。

> 选择你的间隔类型
> 在选择间隔类型时，让你的选择由你希望结果具有的粒度级别驱动。例如，你可以使用`INTERVAL DAY TO SECOND`来存储长达数年的间隔——只是你会用天数来表示这样的间隔，也许是数百天。如果你只记录年份和月份数，那么你永远无法得到正确的天数，因为一年或一个月所代表的天数取决于所讨论的具体年份和月份。
> 同样，如果你需要以月为单位的粒度，你无法根据天数反推正确月数。因此，请选择与你应用所需粒度相匹配的类型。

#### RAW

`RAW`数据类型允许你在列中存储二进制数据。此类数据有时用于存储全局唯一标识符或少量加密数据。

> 注意
> 在 Oracle Database 12c 之前，`RAW`列的最大大小为 2,000 字节。从 Oracle Database 12c 开始，你可以声明`RAW`的最大大小为 32,767 字节。如果你有大量二进制数据要存储，请使用`BLOB`。

如果你从`RAW`列中选择数据，SQL*Plus 会隐式地对检索到的数据应用内置的`RAWTOHEX`函数。数据以十六进制格式显示，使用字符 0-9 和 A-F。当向`RAW`列插入数据时，会隐式应用内置的`HEXTORAW`。

这很重要，因为如果你在`RAW`列上创建索引，优化器可能会忽略该索引，因为 SQL*Plus 在 SQL 引用`RAW`列的位置隐式地应用了函数。普通索引可能无用，而使用`RAWTOHEX`的基于函数的索引可能会带来显著的性能改进。


## ROWID
当 DBA 听到 **ROWID** 这个词时，他们通常会想到为每一行提供的、包含该行在磁盘上物理位置的伪列；这是正确的。然而，许多 DBA 并未意识到 Oracle 支持一种实际的 `ROWID` 数据类型，这意味着你可以创建一个表，其列被定义为 `ROWID` 类型。

`ROWID` 数据类型有一些实际用途。一个有效的应用场景是，当你在尝试启用引用完整性约束遇到问题时，希望捕获违反约束的行的 `ROWID`。在这种情况下，你可以创建一个包含 `ROWID` 类型列的表，并在其中存储违规记录的 `ROWID`。这为你提供了一种捕获和解决违规数据问题的有效方法（有关详细信息，请参阅本章后面的“启用约束”一节）。

**提示**
永远不要试图在表中使用 `ROWID` 数据类型以及行的 `ROWID` 作为主键值。这是因为表中行的 `ROWID` 可能会改变。例如，一个 `ALTER TABLE...MOVE` 命令可能会改变表中的每一个 `ROWID`。通常，表中行的主键值绝不应该改变。因此，不要使用 `ROWID` 作为主键值，而应使用序列生成的非有意义数字（有关进一步讨论，请参阅本章后面的“创建带有自增（标识）列的表”一节）。

## LOB
Oracle 支持通过 LOB 数据类型在列中存储大量数据。Oracle 支持以下类型的 LOB：

*   `CLOB`
*   `NCLOB`
*   `BLOB`
*   `BFILE`

#### 提示
`LONG` 和 `LONG RAW` 数据类型已弃用，不应再使用。

如果你的文本数据无法容纳在 `VARCHAR2` 的限制内，那么你应该使用 `CLOB` 来存储这些数据。`CLOB` 对于存储大量字符数据（例如日志文件）非常有用。`NCLOB` 与 `CLOB` 类似，但允许存储以数据库国家字符集编码的信息。
`BLOB` 是大量通常不适合人类阅读的二进制数据。典型的 `BLOB` 数据包括图像、音频和视频文件。
`CLOB`、`NCLOB` 和 `BLOB` 被称为内部 LOB。这是因为它们存储在 Oracle 数据库内部。这些数据类型驻留在与数据库关联的数据文件中。
`BFILE` 被称为外部 LOB。`BFILE` 列存储指向数据库外部操作系统文件的指针。当无法将大型二进制文件存储在数据库中时，请使用 `BFILE`。`BFILE` 不参与数据库事务，也不受 Oracle 安全或备份与恢复的保护。如果你需要这些功能，请使用 `BLOB` 而不是 `BFILE`。

**提示**
有关 LOB 的完整讨论，请参见第 11 章。

## JSON
以前的 Oracle 版本有过程可以将表数据转换为 JSON 或将 JSON 读入数据库。现在，JSON 可以通过 JSON 列放入数据库表中。不需要知道 JSON 数据的模式或任何其他详细信息，它可以与其他数据一起存储在表中，并使用 SQL 进行查询。

以下是一个创建带有 JSON 列的表示例：

```sql
SQL> CREATE TABLE dept
(deptno     NUMBER(10)
,dname      VARCHAR2(14 CHAR)
,dprojects        VARCHAR2(32767)
CONSTRAINT ensure_json CHECK (dprojects is JSON));
```

可以使用 SQL `INSERT` 语句将 JSON 与其他列一起插入到列中，也可以使用 SQL 查询 JSON 数据。

```sql
SELECT dept.deptno, dept.dprojects.projectID, dept.dprojects.projectName from dept;
```

这将从 JSON 数据中提取每个项目的 `projectID` 和 `projectName`。使用 JSON 当然还有更多内容，并且有程序包可以简化数据库中数据的处理。它允许提取和放入数据库的 JSON 数据 API。将 JSON 数据存储在列中，可以针对数据库运行简单查询以处理其他数据列。

## 创建表
Oracle 的每个新版本都扩展了表的特性。试想一下：Oracle 12c 版本的 *Oracle SQL 语言参考指南* 提供了超过 80 页与 `CREATE TABLE` 语句相关的语法。此外，`ALTER TABLE` 语句又占用了 90 多页与表维护相关的详细信息。在大多数情况下，你通常只需要使用可用表选项的一小部分。

以下是创建表时应考虑的一般因素：

*   表类型（堆表、临时表、索引组织表、分区表等）
*   命名约定
*   列数据类型和长度
*   约束（主键、外键等）
*   索引要求（详见第 8 章）
*   初始存储要求
*   特殊功能（虚拟列、只读、并行、压缩、无日志、不可见列等）
*   增长需求
*   表及其索引的表空间

在运行 `CREATE TABLE` 语句之前，你需要仔细考虑上一个列表中的每个项目。为此，DBA 通常使用数据建模工具来帮助管理用于创建数据库对象的 DDL 脚本。数据建模工具允许你以可视化方式定义表、关系以及底层的数据库特性。

### 创建堆表
你使用 `CREATE TABLE` 语句来创建表，并指定列的名称、数据类型和长度。Oracle 默认的表类型是堆表。术语“堆”意味着数据在表中不是按特定顺序存储的（相反，它们是一堆数据）。以下是一个创建堆表的简单示例：

```sql
SQL> CREATE TABLE dept
(deptno     NUMBER(10)
,dname      VARCHAR2(14 CHAR)
,loc        VARCHAR2(14 CHAR));
```

如果未指定表空间，则表将在创建该表的用户的默认永久表空间中创建。对于少数小型测试表，允许表在默认永久表空间中创建是可以的。对于更复杂的情况，你应该显式指定希望表创建在哪个表空间中。作为参考（在后续示例中），以下是两个示例表空间 `HR_DATA` 和 `HR_INDEX` 的创建脚本：

```sql
SQL> CREATE TABLESPACE hr_data
DATAFILE '/u01/dbfile/O18C/hr_data01.dbf' SIZE 1000m
EXTENT MANAGEMENT LOCAL
UNIFORM SIZE 512k SEGMENT SPACE MANAGEMENT AUTO;
--
SQL> CREATE TABLESPACE hr_index
DATAFILE '/u01/dbfile/O18C/hr_index01.dbf' SIZE 100m
EXTENT MANAGEMENT LOCAL
UNIFORM SIZE 512k SEGMENT SPACE MANAGEMENT AUTO;
```

通常，在创建表时，你还应指定约束，例如主键。以下代码显示了创建表时最常用的功能。此 DDL 定义了主键、外键、表空间信息和注释：



## 数据库表设计指南与最佳实践

### 建表 SQL 示例

```sql
SQL> CREATE TABLE dept
(deptno     NUMBER(10)
,dname      VARCHAR2(14 CHAR)
,loc        VARCHAR2(14 CHAR)
,CONSTRAINT dept_pk PRIMARY KEY (deptno)
USING INDEX TABLESPACE hr_index
) TABLESPACE hr_data;
--
SQL> COMMENT ON TABLE dept IS 'Department table';
--
SQL> CREATE UNIQUE INDEX dept_uk1 ON dept(dname)
TABLESPACE hr_index;
--
SQL> CREATE TABLE emp
(empno      NUMBER(10)
,ename      VARCHAR2(10 CHAR)
,job        VARCHAR2(9 CHAR)
,mgr        NUMBER(4)
,hiredate   DATE
,sal        NUMBER(7,2)
,comm       NUMBER(7,2)
,deptno     NUMBER(10)
,CONSTRAINT emp_pk PRIMARY KEY (empno)
USING INDEX TABLESPACE hr_index
) TABLESPACE hr_data;
--
SQL> COMMENT ON TABLE emp IS 'Employee table';
--
SQL> ALTER TABLE emp ADD CONSTRAINT emp_fk1
FOREIGN KEY (deptno)
REFERENCES dept(deptno);
--
SQL> CREATE INDEX emp_fk1 ON emp(deptno)
TABLESPACE hr_index;
```

### 设计考虑

创建表时，我通常不指定表级的物理空间属性。表会从创建它的表空间继承其空间属性。这简化了管理和维护。如果你有需要不同物理空间属性的表，那么可以创建独立的表空间来容纳具有不同需求的表。例如，你可以创建一个具有 16MB 区大小的`HR_DATA_LARGE`表空间和一个具有 128KB 区大小的`HR_DATA_SMALL`表空间，并根据表的存储需求选择在何处创建表。有关创建表空间的详细信息，请参见第 4 章。

表 7-2 列出了一些指导原则，它们并非硬性规定；请根据你的环境需要调整。其中一些建议可能看起来显而易见。然而，在多年来继承了许多数据库之后，我见过每一条建议都以某种方式被违反，这使得数据库维护变得困难和笨拙。

**表 7-2 创建表时需考虑的指导原则**

| 建议 | 理由 |
| :--- | :--- |
| 在命名表、列、约束、触发器、索引等时使用标准。 | 有助于记录应用程序并简化维护。 |
| 如果列总是包含数字数据，请将其设为数字数据类型。 | 强制执行业务规则，并在使用 Oracle SQL 数学函数（对于字符“01”和数字“1”行为可能不同）时提供最大的灵活性、性能和一致性。 |
| 如果你有定义数字字段长度和精度的业务规则，请强制执行；例如，`NUMBER(7,2)`。如果你没有业务规则，请使用`NUMBER(38)`。 | 强制执行业务规则并保持数据整洁。 |
| 对于可变长度的字符数据，使用`VARCHAR2`（而不是`VARCHAR`）。 | 遵循 Oracle 的建议，对字符数据使用`VARCHAR2`（而不是`VARCHAR`）。Oracle 文档指出，未来`VARCHAR`将被重新定义为一个独立的数据类型。 |
| 对于字符数据，在`CHAR`中指定大小；例如，`VARCHAR2(30 CHAR)`。 | 在处理多字节数据时，你将获得更可预测的结果，因为多字节字符通常存储在超过 1 个字节中。 |
| 如果你有指定列最大长度的业务规则，请使用该长度，而不是将所有列都设为`VARCHAR2(4000)`。 | 强制执行业务规则并保持数据整洁。 |
| 适当地使用`DATE`和`TIMESTAMP`数据类型。 | 强制执行业务规则，确保数据格式适当，并在使用 SQL 日期函数时提供最大的灵活性。 |
| 为表和索引指定独立的表空间。让表和索引从表空间继承存储属性。 | 简化管理和维护。 |
| 大多数表应创建主键。 | 强制执行业务规则，并允许你唯一标识每一行。 |
| 创建一个数字代理键作为每个表的主键。使用代理键的标识列或序列来填充。 | 使连接更简单、更高效。 |
| 脱线创建主键约束。 | 在创建主键时提供更大的灵活性，尤其是在主键由多列组成的情况下。 |
| 为逻辑用户（可识别的列组合，使行独一无二）创建唯一键。 | 强制执行业务规则并保持数据整洁。 |
| 为表和列创建注释。 | 有助于记录应用程序并简化维护。 |
| 尽可能避免使用 LOB 数据类型。 | 防止与 LOB 列相关的维护问题，例如意外增长和复制时的性能问题。 |
| 如果一个列应该总是有值，请使用`NOT NULL`约束来强制执行。 | 强制执行业务规则并保持数据整洁。 |
| 创建审计类列，例如`CREATE_DTT`和`UPDATE_DTT`，这些列通过默认值或触发器或两者自动填充。 | 有助于维护和确定数据何时被插入或更新，或两者。其他类型的审计列包括插入和更新行的用户。 |
| 在适当的地方使用检查约束。 | 强制执行业务规则并保持数据整洁。 |
| 在适当的地方定义外键。 | 强制执行业务规则并保持数据整洁。 |

### 实现虚拟列

从 Oracle 数据库 11g 及更高版本开始，你可以在表定义中创建虚拟列。虚拟列基于同一表中的一个或多个现有列，或常量、SQL 函数和用户定义的 PL/SQL 函数的组合，或两者。虚拟列不存储在磁盘上；它们在运行时 SQL 查询执行时进行评估。虚拟列可以被索引并存储统计信息。

在 Oracle 数据库 11g 之前，你可以通过`SELECT`语句或视图定义来模拟虚拟列。例如，以下 SQL `SELECT`语句在查询执行时生成一个虚拟值：

```sql
SQL> select inv_id, inv_count,
case when inv_count < 100 then 'GETTING LOW'
when inv_count = 100 then 'OKAY'
end inv_status
from inv;
```

为什么要使用虚拟列？这样做有以下优点：
*   你可以在虚拟列上创建索引；Oracle 在内部会创建一个基于函数的索引。
*   你可以在虚拟列中存储统计信息，这些信息可被基于成本的优化器（CBO）使用。
*   虚拟列可以在`WHERE`子句中被引用。
*   虚拟列在数据库中永久定义；这样的列只有一个中心定义。

以下是一个创建包含虚拟列的表的示例：

```sql
SQL> create table inv(
inv_id number
,inv_count number
,inv_status generated always as (
case when inv_count < 100 then 'GETTING LOW'
when inv_count = 100 then 'OKAY'
end)
);
```

在前面的代码清单中，指定`GENERATED ALWAYS`是可选的。例如，以下清单与前一个等效：

```sql
SQL> create table inv(
inv_id number
,inv_count number
,inv_status as (
case when inv_count < 100 then 'GETTING LOW'
when inv_count = 100 then 'OKAY'
end)
);
```

我更喜欢添加`GENERATED ALWAYS`，因为它在我心中强化了该列始终是虚拟的这一概念。`GENERATED ALWAYS`有助于内联记录你所做的操作。这对你之后长期维护的其他 DBA 有帮助。

要查看虚拟列生成的值，首先向表中插入一些数据：

```sql
SQL> insert into inv (inv_id, inv_count) values (1,99);
```

接下来，从表中选择以查看生成的值：

```sql
SQL> select * from inv;
```

以下是一些示例输出：

```
    INV_ID  INV_COUNT INV_STATUS
---------- ---------- --------------------
         1         99 GETTING LOW
```

> **注意**
> 如果你向表中插入数据，在设置为`GENERATED ALWAYS AS`的列中不会存储任何内容。虚拟值在你从表中选择时生成。

你也可以修改表以包含一个虚拟列：



## Oracle 数据库高级表特性

### 虚拟列

```
SQL> alter table inv add(
inv_comm generated always as(inv_count * 0.1) virtual
);
```

您可以修改现有虚拟列的定义：

```
SQL> alter table inv modify inv_status generated always as(
case when inv_count > 50 and inv_count < 200 then 'OKAY'
end);
```

虚拟列可以在 SQL 查询（DML 或 DDL）中访问。例如，假设您想根据虚拟列的值更新一个永久列：

```
SQL> update inv set inv_count=100 where inv_status='OKAY';
```

虚拟列本身无法通过 `UPDATE` 语句的 `SET` 子句进行更新。但是，您可以在 `UPDATE` 或 `DELETE` 语句的 `WHERE` 子句中引用虚拟列。

您可以选择性地指定虚拟列的数据类型。如果省略数据类型，Oracle 会从定义虚拟列的表达式中推导出数据类型。

虚拟列有几个注意事项：

*   只能在常规的堆组织表上定义虚拟列。无法在索引组织表、外部表、临时表、对象表或集群表上定义虚拟列。
*   虚拟列不能引用其他虚拟列。
*   虚拟列只能引用其定义所在表中的列。
*   虚拟列的输出必须是标量值（即单个值，而非值集合）。

要查看虚拟列的定义，请使用 `DBMS_METADATA` 包来获取与表关联的 DDL。如果您在 SQL*Plus 中进行选择，需要将 `LONG` 变量设置为足够大的值以显示所有返回的数据：

```
SQL> set long 10000;
SQL> select dbms_metadata.get_ddl('TABLE','INV') from dual;
```

以下是输出片段：

```
SQL> CREATE TABLE "INV_MGMT"."INV"
(    "INV_ID" NUMBER,
"INV_COUNT" NUMBER,
"INV_STATUS" VARCHAR2(11) GENERATED ALWAYS AS (CASE  WHEN "INV_COUNT">50 AND "INV_COUNT"<200 THEN 'OKAY' END) VIRTUAL ...
```

## 实现不可见列

从 Oracle Database 12c 开始，您可以创建不可见列。当列不可见时，无法通过以下方式查看：

*   `DESCRIBE` 命令
*   `SELECT *` （用于访问表的所有列）
*   PL/SQL 中的 `%ROWTYPE`
*   Oracle 调用接口 (OCI) 内的描述

但是，如果在 `SELECT` 子句中显式指定或在 DML 语句（`INSERT`、`UPDATE`、`DELETE` 或 `MERGE`）中直接引用，仍然可以访问该列。不可见列也可以像可见列一样被索引。

不可见列的主要用途是确保向表中添加列不会破坏任何现有的应用程序代码。如果应用程序代码没有显式访问不可见列，那么对于应用程序来说，就好像该列不存在一样。

可以创建包含不可见列的表，也可以添加或更改列使其不可见。定义为不可见的列也可以更改为可见。以下是创建包含不可见列的表的示例：

```
SQL> create table inv
(inv_id     number
,inv_desc   varchar2(30 char)
,inv_profit number invisible);
```

现在，当描述该表时，请注意不可见列不会显示：

```
SQL> desc inv
Name                               Null?    Type
---------------------------------- -------- ----------------------------
INV_ID                                      NUMBER
INV_DESC                                    VARCHAR2(30 CHAR)
```

定义为不可见的列，如果在 `SELECT` 语句或任何 DML 操作中直接指定，仍然可以访问。例如，从表中选择时，可以通过在 `SELECT` 子句中指定它来查看不可见列：

```
SQL> select inv_id, inv_desc, inv_profit from inv;
```

注意：创建包含不可见列的表时，至少必须有一个可见列。

### 创建只读表

您可以将单个表置于只读模式。这样做可以防止任何 `INSERT`、`UPDATE` 或 `DELETE` 语句在表上运行。另一种方法是将表空间设为只读，并将此表用于静态的只读表。

在表级别需要只读功能有几个原因：

*   表中的数据是历史数据，在正常情况下永远不应更新。
*   您正在对表执行某些维护，并希望确保在更新过程中表不会发生变化。
*   您想要删除该表，但在删除之前，希望更好地确定是否有任何用户试图更新该表。

使用 `ALTER TABLE` 语句将表置于只读模式：

```
SQL> alter table inv read only;
```

您可以通过执行以下查询来验证只读表的状态：

```
SQL> select table_name, read_only from user_tables where read_only='YES';
```

要将只读表修改为读写模式，请执行以下 SQL：

```
SQL> alter table inv read write ;
```

注意：只读表功能要求数据库初始化参数 `COMPATIBLE` 设置为 11.1.0 或更高。

### 理解延迟段创建

从 Oracle Database 11g Release 2 开始，当您创建表时，关联段的创建会延迟到向表中插入第一行时进行。这个特性有一些有趣的影响。例如，如果您最初为应用程序创建数千个对象（例如首次安装时），在向应用程序表插入数据之前，任何表（或关联索引）都不会消耗空间。这意味着创建表时初始 DDL 运行速度更快，但第一条 `INSERT` 语句运行速度稍慢。

为了说明延迟段的概念，首先创建一个表：

```
SQL> create table inv(inv_id number, inv_desc varchar2(30 CHAR));
```

您可以通过检查 `USER_TABLES` 来验证表是否已创建：

```
SQL> select
table_name
,segment_created
from user_tables
where table_name='INV';
```

以下是一些示例输出：

```
TABLE_NAME                     SEG
------------------------------ ---
INV                            NO
```

接下来，查询 `USER_SEGMENTS` 以验证是否尚未为表分配段：

```
SQL> select
segment_name
,segment_type
,bytes
from user_segments
where segment_name='INV'
and segment_type='TABLE';
```

此示例的相应输出为：

```
no rows selected
```

现在，向表中插入一行：

```
SQL> insert into inv values(1,'BOOK');
```

重新运行查询，从 `USER_SEGMENTS` 中选择，并注意已创建段：

```
SEGMENT_NAME    SEGMENT_TYPE            BYTES
--------------- ------------------ ----------
INV             TABLE                   65536
```

如果您习惯于使用旧版本的 Oracle，延迟段创建特性可能会导致混淆。例如，如果您有查询 `DBA_SEGMENTS` 或 `DBA_EXTENTS` 的与空间相关的监控报告，请注意，在向表插入第一行之前，这些视图不会填充表或与表关联的任何索引的信息。

注意：您可以通过将数据库初始化参数 `DEFERRED_SEGMENT_CREATION` 设置为 `FALSE` 来禁用延迟段创建功能。此参数的默认值为 `TRUE`。

### 创建具有自增（Identity）列的表

从 Oracle Database 12c 开始，您可以定义一个在插入数据时自动填充并递增的列。此功能非常适合自动填充主键列。



## Oracle 数据库中的自增列（Identity Columns）

### 介绍
在 Oracle Database 12c 之前，您必须手动创建序列（sequence），然后在插入表数据时引用该序列。有时，数据库管理员（DBA）会在表上创建触发器（trigger），以模拟基于序列的自增列（详见第 9 章）。

### 使用 `GENERATED AS IDENTITY` 子句
您可以使用 `GENERATED AS IDENTITY` 子句来定义一个自增（identity）列。此示例创建了一个表，其主键列将自动填充并递增：

```
SQL> create table inv(
inv_id number generated as identity
,inv_desc varchar2(30 char));
--
SQL> alter table inv add constraint inv_pk primary key (inv_id);
```

现在，您无需指定主键值即可填充表：

```
SQL> insert into inv (inv_desc) values ('Book');
SQL> insert into inv (inv_desc) values ('Table');
```

从表中查询显示，`INV_ID` 列已被自动填充：

```
SQL> select * from inv;
```

以下是此示例的样本输出：

```
INV_ID INV_DESC
---------- ------------------------------
1 Book
2 Table
```

### 底层机制
当您创建一个身份列时，Oracle 会自动创建一个序列（sequence）并将该序列与列关联。您可以在 `USER_SEQUENCES` 中查看序列信息：

```
SQL> select sequence_name, min_value, increment_by from user_sequences;
```

以下是此示例的样本输出：

```
SEQUENCE_NAME         MIN_VALUE INCREMENT_BY
-------------------- ---------- ------------
ISEQ$$_43216                  1            1
```

您可以通过以下查询来识别身份列：

```
SQL> select table_name, identity_column
from user_tab_columns
where identity_column='YES';
```

### 插入行为与注意事项
当您创建一个带有身份列的表时（如前面的示例），您不能直接为身份列指定值。例如，如果您尝试：

```
SQL> insert into inv values(3,'Chair');
```

您将收到错误：

```
ORA-32795: cannot insert into a generated always identity column
```

如果出于某种原因，您需要偶尔向身份列插入值，那么在创建表时请使用以下语法：

```
SQL> create table inv(
inv_id number generated by default on null as identity
,inv_desc varchar2(30 char));
```

### 自定义序列
由于填充身份列的底层机制是一个序列，因此您可以控制序列的创建方式（就像手动创建序列一样）。例如，您可以指定序列的起始数字以及每次递增的量。此示例指定底层序列从数字 50 开始，每次递增 2：

```
SQL> create table inv(
inv_id number generated as identity (start with 50 increment by 2)
,inv_desc varchar2(30 char));
```

### 使用自增列的注意事项
使用自增（identity）列时，需要注意以下几点：

*   每个表只允许一个自增列。
*   它们必须是数值类型。
*   它们不能有默认值。
*   `NOT NULL` 和 `NOT DEFERRABLE` 约束会被隐式应用。
*   `CREATE TABLE ... AS SELECT` 不会继承身份列属性。

同时请记住，在向自增列插入数据后，如果您执行回滚（rollback），事务会被回滚，但来自序列的自增值不会回滚。这是序列的预期行为。您可以回滚这样的插入操作，但序列值一旦被使用就会消耗掉。

> **提示**
> 有关如何管理序列的详细信息，请参阅第 9 章。

## 允许默认并行 SQL 执行
如果您处理大型表，可以考虑将表创建为 `PARALLEL`。这指示 Oracle 为查询以及后续的 `INSERT`、`UPDATE`、`DELETE`、`MERGE` 和查询语句设置并行度。此示例创建了一个 `PARALLEL` 子句为 `2` 的表：

```
SQL> create table inv
(inv_id     number,
inv_desc   varchar2(30 char),
create_dtt date default sysdate)
parallel 2;
```



您可以指定 `PARALLEL`、`NOPARALLEL` 或 `PARALLEL N`。如果您未指定 `N`，Oracle 将根据 `PARALLEL_THREADS_PER_CPU` 初始化参数设置并行度。您可以通过此查询验证并行度：

```
SQL> select table_name, degree from user_tables;
```

这里需要注意的主要问题是，如果表创建时使用了默认的并行度，任何后续的查询都将使用并行线程执行。您可能会疑惑为什么某个查询或 DML 语句会在未显式调用并行操作的情况下并行执行。

### 这些 P_0 进程是从哪里来的？

有一次，我接到一个生产支持人员的电话，他报告说由于 `ORA-00020 maximum number of processes` 错误，没有人能连接到数据库。我登录到服务器，注意到有数百个 `ora_p` 并行查询进程正在运行。

我不得不手动终止一些进程，以便能够连接到数据库。经过进一步检查，我将这些并行查询会话追踪到一条 SQL 语句和一个表。在这个案例中，该表创建时设置了默认并行度为 64（别问我为什么），这反过来在查询该表时产生了数百个进程和会话。这达到了数据库允许的最大连接数，并导致了问题。可以通过将并行设置为 `NOPARALLEL` 或 1 来解决此问题。此外，资源管理器也会限制允许的连接数。

您也可以修改表来更改其默认并行度：

```
SQL> alter table inv parallel 1;
```

从 Oracle 18c 开始，资源参数 `PQ_TIMEOUT_ACTION` 可用于超时不活动的并行查询。这将允许高优先级的并行查询获得执行所需的资源。还有一种更简单的方法来取消失控的 SQL，而无需使用 `ALTER SYSTEM CANCEL SQL` 语句手动终止进程。

**提示**：请记住，`PARALLEL_THREADS_PER_CPU` 是平台相关的，并且在开发环境和生产环境之间可能有所不同。因此，如果您未指定并行度，并行操作的行为可能会因环境而异。

## 压缩表数据

随着数据库的增长，您可能需要考虑表级压缩。压缩数据的好处是使用更少的磁盘空间和内存，并减少 I/O。读取压缩数据的查询可能运行得更快，因为需要处理的块更少。但是，在写入和读取发生时，由于数据被压缩和解压缩，CPU 使用率会增加，因此需要权衡取舍。

从 Oracle Database 12c 开始，有四种可用的压缩类型：

*   基本压缩
*   高级行压缩（以前称为 OLTP 压缩）
*   数据仓库压缩（混合列压缩）
*   归档压缩（混合列压缩）

基本压缩通过 `COMPRESS` 或 `COMPRESS BASIC` 子句启用（它们是同义的）。此示例创建一个使用基本压缩的表：

```
SQL> create table inv
(inv_id     number,
inv_desc   varchar2(300 char),
create_dtt timestamp)
compress basic;
```

基本压缩在数据直接路径插入表时提供压缩。默认情况下，使用 `COMPRESS BASIC` 创建的表的 `PCTFREE` 设置为 0。您可以在创建表时指定 `PCTFREE` 来覆盖此设置。

**注意**：基本压缩需要 Oracle 企业版，但不需要额外许可。其他类型的压缩是数据库的额外许可选项。与数据库选项一样，需要评估存储成本和压缩比，以提供此选项的正确成本分析。

高级行压缩通过 `ROW STORE COMPRESS ADVANCED` 子句启用：

```
SQL> create table inv
(inv_id     number,
inv_desc   varchar2(300 char),
create_dtt timestamp)
row store compress advanced;
```

高级行压缩在最初将数据插入表中以及任何后续 DML 操作时提供压缩。您可以通过以下 `SELECT` 语句验证表的压缩情况：

```
SQL> select table_name, compression, compress_for
from user_tables
where table_name='INV';
```

以下是一些示例输出：

```
TABLE_NAME           COMPRESS COMPRESS_FOR
-------------------- -------- ------------------------------
INV                  ENABLED  ADVANCED
```

**注意**：OLTP 表压缩是 Oracle 高级压缩选项的一项功能。此选项需要从 Oracle 获得额外许可，并且仅在 Oracle 企业版中可用。

您也可以使用压缩子句创建表空间。在该表空间中创建的任何表都将继承表空间的压缩设置。例如，以下是如何为表空间设置默认压缩级别：

```
SQL> CREATE TABLESPACE hr_data
DEFAULT ROW STORE COMPRESS ADVANCED
DATAFILE '/u01/dbfile/O12C/hr_data01.dbf' SIZE 100m
EXTENT MANAGEMENT LOCAL
UNIFORM SIZE 512k SEGMENT SPACE MANAGEMENT AUTO;
```

如果有一个已存在的表，您可以修改它以允许压缩（基本或高级）：

```
SQL> alter table inv row store compress advanced;
```

**注意**：Oracle 不支持对超过 255 列的表进行压缩。

修改为允许压缩并不会压缩表中的现有数据。您需要使用 Data Pump 重建表或移动表，以压缩启用压缩之前存在于其中的数据：

```
SQL> alter table inv move;
```

**注意**：如果您移动了表，那么您还需要重建任何相关的索引。

您可以通过 `NOCOMPRESS` 子句禁用压缩。这不会影响表中现有的数据。相反，它会影响未来的插入（基本和高级行压缩）和未来的 DML（高级行压缩）；例如：

```
SQL> alter table inv nocompress;
```

Oracle 还具有仓库和归档混合列压缩功能，当使用某些类型的存储（如 Exadata）时可用。这种类型的压缩通过 `COLUMN STORE COMPRESS FOR QUERY LOW|HIGH` 或 `COLUMN STORE COMPRESS FOR ARCHIVE LOW|HIGH` 子句启用。有关此类压缩的更多详细信息，请参阅 Oracle 技术网络网站 ([`http://otn.oracle.com`](http://otn.oracle.com))。

## 避免重做日志生成

在创建表时，您可以选择指定 `NOLOGGING` 子句。`NOLOGGING` 功能可以显著减少某些类型操作的重做日志生成量。有时，当您处理大量数据时，出于性能原因，在最初创建表并将数据插入表中时减少重做日志生成是可取的。

消除重做日志生成的缺点是，在数据加载后（并且在您能够备份表之前）发生故障的情况下，您无法通过 `NOLOGGING` 恢复创建的数据。如果您能容忍一定的数据丢失风险，那么可以使用 `NOLOGGING`，但在数据加载后应尽快备份表。如果您的数据至关重要，请不要使用 `NOLOGGING`。如果您的数据可以轻松重新创建，那么 `NOLOGGING` 在尝试提高大型数据加载的性能时是可取的。

一种看法是 `NOLOGGING` 会消除表上所有 DML 操作的重做日志生成。这是不正确的。`NOLOGGING` 功能从不影响常规 DML 语句（普通的 `INSERT`、`UPDATE` 和 `DELETE`）的重做日志生成。

`NOLOGGING` 功能可以显著减少以下类型操作的重做日志生成：

*   SQL*Loader 直接路径加载
*   直接路径 `INSERT /*+ append */`
*   `CREATE TABLE AS SELECT`
*   `ALTER TABLE MOVE`
*   创建或重建索引


### NOLOGGING 注意事项与使用级别

使用 `NOLOGGING` 时需要注意一些特性（或称为“功能”）。如果你的数据库处于 `FORCE LOGGING` 模式，那么所有操作都会生成重做日志，无论你是否指定了 `NOLOGGING`。同样地，当你加载一个表时，如果该表定义了引用外键约束，那么无论是否指定 `NOLOGGING`，都会生成重做日志。

你可以在以下级别之一指定 `NOLOGGING`：

*   语句级
*   `CREATE TABLE` 或 `ALTER TABLE`
*   `CREATE TABLESPACE` 或 `ALTER TABLESPACE`

我倾向于在语句或表级别指定 `NOLOGGING` 子句。在这些场景下，执行语句或 DDL 的 DBA 很明显知道使用了 `NOLOGGING`。如果你在表空间级别指定 `NOLOGGING`，那么在该表空间内创建对象的每个 DBA 都必须了解这个表空间级别的设置。在有多个 DBA 的团队中，很容易出现一个 DBA 不知道另一个 DBA 已创建了一个带有 `NOLOGGING` 的表空间。

### 在创建表时使用 NOLOGGING

以下示例首先创建一个带有 `NOLOGGING` 选项的表：
```
SQL> create table inv(inv_id number)
tablespace users
nologging;
```

接下来，使用一些测试数据进行直接路径插入，并提交数据：
```
SQL> insert /*+ append */ into inv select level from dual
connect by level <= 10000; commit;
```

#### NOLOGGING 模式下的恢复场景

如果你在以 `NOLOGGING` 模式填充表之后（并且在备份该表之前）发生介质故障，会发生什么？在恢复和还原操作之后，表看起来可能已被还原：
```
SQL> desc inv
Name                               Null?    Type
---------------------------------- -------- ----------------------------
INV_ID                                      NUMBER
```

但是，尝试执行一个扫描表中所有数据块的查询：
```
SQL> select * from inv;
```

这里会抛出一个错误，指示数据文件中存在逻辑损坏：
```
ORA-01578: ORACLE data block corrupted (file # 5, block # 203)
ORA-01110: data file 5: '/u01/dbfile/O18C/users01.dbf'
ORA-26040: Data block was loaded using the NOLOGGING option
```

换句话说，数据是不可恢复的，因为没有重做日志来还原它们。再次强调，`NOLOGGING` 选项适用于那些可以轻松重新生成的大批量数据加载场景，尤其是在 `NOLOGGING` 操作之后、备份数据库之前发生故障的情况。

#### NOLOGGING 设置的优先级

如果你在语句级别指定了日志子句，它会覆盖任何表或表空间设置。如果你在表级别指定了日志子句，它会为任何未指定日志子句的语句设置默认模式，并覆盖表空间的日志设置。如果你在表空间级别指定了日志子句，它会为任何未指定日志子句的 `CREATE TABLE` 语句设置默认日志记录方式。

#### 验证日志记录模式

你可以通过以下方式验证数据库的日志模式：
```
SQL> select name, log_mode, force_logging from v$database;
```

下一个语句验证表空间的日志模式：
```
SQL> select tablespace_name, logging from dba_tablespaces;
```

并且，此示例验证表的日志模式：
```
SQL> select owner, table_name, logging from dba_tables where logging = 'NO';
```

#### 查看 NOLOGGING 的效果

你可以通过几种不同的方式查看 `NOLOGGING` 的效果。一种是启用带有统计信息的自动跟踪，并查看重做日志大小：
```
SQL> set autotrace trace statistics;
```

然后，运行一个直接路径 `INSERT` 语句，并查看重做大小统计信息：
```
insert /*+ append */ into inv select level from dual
connect by level <= 10000;
```

这是输出片段：
```
Statistics
----------------------------------------------------------
          0  recursive calls
          0  db block gets
        109  consistent gets
          0  physical reads
      13772  redo size
...
```

禁用日志记录后，对于直接路径操作，你应该会看到比常规 `INSERT` 语句小得多的重做大小数字，例如：
```
SQL> insert into inv select level from dual
connect by level <= 10000;
```

这是部分输出，表明重做大小要大得多：
```
Statistics
----------------------------------------------------------
          0  recursive calls
        108  db block gets
     28479  consistent gets
          0  physical reads
     159152  redo size
...
```

确定 `NOLOGGING` 效果的另一种方法是，测量启用日志记录和在 `NOLOGGING` 模式下操作所生成的重做日志量。如果你有一个可以进行测试的开发环境，你可以监控在操作进行时重做日志切换的频率。另一个简单的测试是计时操作在有日志记录和没有日志记录的情况下分别需要多长时间。在 `NOLOGGING` 模式下执行的操作应该更快（因为只生成了最少量的重做日志）。

## 基于查询创建表 (CTAS)

有时，基于现有表的定义创建表是很方便的。例如，假设你想在修改表的结构或数据之前快速创建一个表的备份。使用 `CREATE TABLE AS SELECT` 语句（CTAS）来实现这一点；例如：
```
create table inv_backup
as select * from inv;
```

前面的语句创建了一个完全相同的表，包含所有数据。如果你不想要包含数据——你只想复制表的结构——那么提供一个永远评估为假的 `WHERE` 子句（在这个例子中，1 永远不等于 2）：
```
SQL> create table inv_empty
as select * from inv
where 1=2;
```

你也可以在创建 CTAS 表时指定不记录重做日志。对于大型数据集，这可以减少创建表所需的时间：
```
SQL> create table inv_backup
nologging
as select * from inv;
```

请注意，使用带有 `NOLOGGING` 子句的 CTAS 技术会将表创建为 `NOLOGGING`，并且不会生成作为 `SELECT` 语句结果填充表的数据恢复所需的重做日志。此外，如果创建 CTAS 表的表空间被定义为 `NOLOGGING`，则不会生成重做日志。在这些场景中，如果你在能够备份表之前发生故障，你将无法恢复和还原你的表。如果你的数据至关重要，请不要使用 `NOLOGGING` 子句。

你也可以指定并行度和存储参数。根据 CPU 的数量，你可能会看到一些性能提升：
```
SQL> create table inv_backup
nologging
tablespace hr_data
parallel 2
as select * from inv;
```

> **注意**
> CTAS 技术不会创建任何索引、约束或触发器。如果你需要原始表中的这些对象，必须单独创建索引和触发器。

## 启用 DDL 日志记录

Oracle 允许你将 DDL 语句的日志记录到一个日志文件中。这种类型的日志记录通过 `ENABLE_DDL_LOGGING` 参数（默认为 `FALSE`）开启。你可以在会话或系统级别设置此参数。此功能为你提供了关于已发出哪些 DDL 语句以及它们何时运行的审计跟踪。以下是在系统级别设置此参数的示例：
```
SQL> alter system set enable_ddl_logging=true scope=both;
```

将此参数设置为 `TRUE` 后，DDL 语句将被记录到日志文件中。Oracle 不会记录每种类型的 DDL 语句，只将最常见的类型记录到日志文件中。DDL 日志文件的确切位置和文件数量因数据库版本而异。可以通过此查询确定此文件的位置（目录路径）：
```
SQL> select value from v$diag_info where name='Diag Alert';
VALUE
--------------------------------------------------------------------------------
/ora01/app/oracle/diag/rdbms/o18c/O18C/alert
```

根据审计的类型，有多个文件捕获 DDL 日志记录。要找到这些文件，首先确定你的诊断主目录位置：
```
SQL> select value from v$diag_info where name='ADR Home';
VALUE
--------------------------------------------------------------------------------
/ora01/app/oracle/diag/rdbms/o18c/O18C
```

现在，将你当前的工作目录更改为前面的目录和 `log` 子目录；例如：
```
$ cd /ora01/app/oracle/diag/rdbms/o18c/O18C/log
```


## 修改表

在此目录中，将有一个格式为 `ddl_<SID>.log` 的文件。
该文件记录了启用 DDL 日志后所发出的 DDL 语句。您也可以在 `log.xml` 文件中查看 DDL 日志。该文件位于之前提到的 `log` 目录下的 `ddl` 子目录中；例如，

```
$ cd /ora01/app/oracle/diag/rdbms/o18c/O18C/log/ddl
```

导航到前述目录后，您可以使用 `vi` 等操作系统实用程序查看 `log.xml` 文件。

## 修改表

修改表是一项常见的任务。新的需求通常意味着您需要重命名、添加、删除或更改列的数据类型。在开发环境中，更改表可能是一项微不足道的任务：您通常不需要处理大量数据或数百个用户同时访问一个表。然而，对于活跃的生产系统，您需要理解尝试更改当前正在访问或已填充数据的表所带来的影响。

### 获取所需锁

当您修改表时，必须对该表持有排他锁。一个问题是，如果某个 DML 事务已持有该表的锁，您就无法更改它。在这种情况下，您会收到此错误：

```
ORA-00054: resource busy and acquire with NOWAIT specified or timeout expired
```

前面的错误消息有些令人困惑，因为它让您认为可以通过使用 `NOWAIT` 获取锁来解决此问题。然而，这是一个通用消息，在您发出的 DDL 无法获得表的排他锁时生成。在这种情况下，您有几个选项：

*   发出 DDL 命令并收到 `ORA-00054` 错误后，快速反复按斜杠键（`/`），希望能在事务间隙修改表。
*   等待维护窗口来安排更改表，以免锁住用户。关闭数据库并以受限模式启动，修改表，然后打开数据库供正常使用。
*   设置 `DDL_LOCK_TIMEOUT` 参数。

前面列表中的最后一项指示 Oracle 反复尝试运行 DDL 语句，直到获得表上的所需锁。您可以在系统或会话级别设置 `DDL_LOCK_TIMEOUT` 参数。下面的示例指示 Oracle 反复尝试获取锁 100 秒：

```
SQL> alter session set ddl_lock_timeout=100;
```

系统级别 `DDL_LOCK_TIMEOUT` 初始化参数的默认值为 `0`。如果您想修改系统中每个会话的默认行为，请发出 `ALTER SYSTEM SET` 语句。以下命令将系统默认超时值设置为 10 秒：

```
SQL> alter system set ddl_lock_timeout=10 scope=both;
```

> **注意**
> 使用 `DBMS_REDEFINITION` 包进行在线表操作，允许更改列类型、名称、大小等，以及重命名表。使用此包还可以进行在线操作，而无需中断数据库用户来实施其他选项。这也是在进行切换前验证新过程和表更改的好方法。

### 重命名表

重命名表有几个原因：

*   使表符合标准。
*   在删除表之前更好地确定它是否正在被使用。

此示例将表从 `INV` 重命名为 `INV_OLD`：

```
SQL> rename inv to inv_old;
```

如果成功，您应该会看到此消息：

```
Table renamed.
```

### 添加列

使用 `ALTER TABLE ... ADD` 语句向表中添加列。此示例向 `INV` 表添加一列：

```
SQL> alter table inv add(inv_count number);
```

如果成功，您应该会看到此消息：

```
Table altered.
```

### 修改列

有时，您需要修改列以调整其大小或更改其数据类型。使用 `ALTER TABLE ... MODIFY` 语句调整列的大小。此示例将列的大小更改为 256 个字符：

```
SQL> alter table inv modify inv_desc varchar2(256 char);
```

如果减小列的大小，请首先确保不存在大于减小后大小的值：

```
SQL> select max(length(<column_name>)) from <table_name>;
```

当您将列更改为 `NOT NULL` 时，每列都必须有有效值。首先，验证没有 `NULL` 值：

```
SQL> select <column_name> from <table_name> where <column_name> is null;
```

如果您要修改为 `NOT NULL` 的列的任何行具有 `NULL` 值，则必须首先更新该列以包含一个值。以下是修改列为 `NOT NULL` 的示例：

```
SQL> alter table inv modify(inv_desc not null);
```

您还可以修改列以具有默认值。每当向表插入记录但未为列提供值时，都会使用默认值：

```
SQL> alter table inv modify(inv_desc default 'No Desc');
```

如果想删除列的默认值，请将其设置为 `NULL`：

```
SQL> alter table inv modify(inv_desc default NULL);
```

有时，您需要更改表的数据类型；例如，原本错误地定义为 `VARCHAR2` 的列需要更改为 `NUMBER`。在更改列的数据类型之前，首先验证现有列的所有值是否为有效的数字值。以下是一个简单的 PL/SQL 脚本：

```
SQL> create or replace function isnum(v_in varchar2)
return varchar is
val_err exception;
pragma exception_init(val_err, -6502); -- char to num conv. error
scrub_num number;
begin
scrub_num := to_number(v_in);
return 'Y';
exception when val_err then
return 'N';
end;
/
```

您可以使用 `ISNUM` 函数来检测列中的数据是否为数字。该函数定义了一个 PL/SQL 编译指示异常，用于处理 `ORA-06502` 字符到数字的转换错误。当遇到此错误时，异常处理器会捕获它并返回 `N`。如果传递给 `ISNUM` 函数的值是数字，则返回 `Y`。如果该值无法转换为数字，则返回 `N`。以下是一个演示前述概念的简单示例：

```
SQL> create table stage(hold_col varchar2(30));
SQL> insert into stage values(1);
SQL> insert into stage values('x');
SQL> select hold_col from stage where isnum(hold_col)='N';
HOLD_COL
------------------------------
x
```

同样，当您将字符列修改为 `DATE` 或 `TIMESTAMP` 数据类型时，最好先检查数据是否可以成功转换。以下是一个执行此操作的函数：

```
SQL> create or replace function isdate(p_in varchar2, f_in varchar2)
return varchar is
scrub_dt date;
begin
scrub_dt := to_date(p_in, f_in);
return 'Y';
exception when others then
return 'N';
end;
/
```

当您调用 `ISDATE` 函数时，需要传递一个有效的日期格式掩码，例如 `YYYYMMDD`。以下是一个演示前述概念的简单示例：

```
SQL> create table stage2 (hold_col varchar2(30));
SQL> insert into stage2 values('20130103');
SQL> insert into stage2 values('03-JAN-13');
SQL> select hold_col from stage2 where isdate(hold_col,'YYYYMMDD')='N';
HOLD_COL
------------------------------
03-JAN-13
```

### 重命名列

重命名列有几个原因：

*   有时需求会变化，您可能希望修改列名以更好地反映该列的用途。
*   如果您计划删除列，先重命名该列有助于更好地确定是否有用户或应用程序正在访问它。

使用 `ALTER TABLE ... RENAME` 语句重命名列：

```
SQL> alter table inv rename column inv_count to inv_amt;
```



## 丢弃列
表有时会包含从未使用过的列。这可能是由于最初的需求发生了变化或不够准确。如果表中包含未使用的列，应考虑将其丢弃。如果保留未使用的列，可能会导致未来的数据库管理员不清楚该列的用途，并且该列可能会不必要地消耗空间。

在丢弃列之前，建议先对其进行重命名。这样可以有机会确定是否有任何用户或应用程序正在使用该列。当确信该列未被使用后，首先使用数据泵导出工具备份表，然后再丢弃该列。这些策略为您提供了补救选项，以防在丢弃列后又发现需要它。

要丢弃列，请使用`ALTER TABLE ... DROP`语句：
```sql
SQL> alter table inv drop (inv_name);
```

请注意，如果要从中移除列的表包含大量数据，`DROP`操作可能需要一些时间。这种延迟可能导致在表被修改期间事务被延迟（因为`ALTER TABLE`语句会锁定表）。在这种情况下，您可能希望先将列标记为未使用，然后在维护窗口时再丢弃它：
```sql
SQL> alter table inv set unused (inv_name);
```
当列被标记为未使用后，它将不再出现在表描述中。`SET UNUSED`子句不会产生与丢弃列相关的开销。此技术允许您快速阻止 SQL 查询或应用程序看到或使用该列。任何试图访问未使用列的查询都会收到以下错误：
```sql
ORA-00904: ... invalid identifier
```
之后，您可以在为应用程序安排了一些停机时间时丢弃任何未使用的列。使用`DROP UNUSED`子句来移除所有标记为`UNUSED`的列：
```sql
SQL> alter table inv drop unused columns;
```

### 显示表 DDL
有时，数据库管理员在记录创建或修改表时所使用的 DDL 方面做得不够好。通常，应将数据库 DDL 代码维护在源代码控制仓库或某种建模工具中。如果您的团队没有 DDL 源代码，有几种方法可以手动重现 DDL：
*   查询数据字典。
*   使用数据泵。
*   使用`DBMS_METADATA`包。

在过去，比如版本 7 及更早的时候，数据库管理员经常编写查询数据字典的 SQL，试图提取重新创建对象所需的 DDL。虽然这种方法比没有要好，但常常容易出错，因为 SQL 没有考虑到所有的对象创建特性。

数据泵实用程序是生成用于创建数据库对象的 DDL 的绝佳方法。使用数据泵生成 DDL 的方法在第 13 章有详细说明。

`DBMS_METADATA`包的`GET_DDL`函数通常是显示创建对象所需 DDL 的最快方法。此示例展示了如何为名为`INV`的表生成 DDL：
```sql
SQL> set long 10000
SQL> select dbms_metadata.get_ddl('TABLE','INV') from dual;
```
以下是一些示例输出：
```
DBMS_METADATA.GET_DDL('TABLE','INV')

SQL>  CREATE TABLE "MV_MAINT"."INV"
(    "INV_ID" NUMBER,
"INV_DESC" VARCHAR2(30 CHAR),
"INV_COUNT" NUMBER
) SEGMENT CREATION DEFERRED
PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255
NOCOMPRESS LOGGING
TABLESPACE "USERS";
```
以下 SQL 语句显示了模式中所有表的 DDL：
```sql
SQL> select
dbms_metadata.get_ddl('TABLE',table_name)
from user_tables;
```
如果要显示其他用户拥有的表的 DDL，请向`GET_DDL`过程添加`SCHEMA`参数：
```sql
SQL> select
dbms_metadata.get_ddl(object_type=>'TABLE', name=>'INV', schema=>'INV_APP')
from dual;
```
> **注意**
> 您可以显示几乎任何数据库对象类型的 DDL，例如`INDEX`、`FUNCTION`、`ROLE`、`PACKAGE`、`MATERIALIZED VIEW`、`PROFILE`、`CONSTRAINT`、`SEQUENCE`和`SYNONYM`。

### 丢弃表
如果要从用户中移除对象（如表），请使用`DROP TABLE`语句。此示例丢弃名为`INV`的表：
```sql
SQL> drop table inv;
```
您应该会看到以下确认信息：
```
Table dropped.
```
如果您尝试丢弃父表（其主键或唯一键在子表中被引用为外键），您会看到类似以下错误：
```sql
ORA-02449: unique/primary keys in table referenced by foreign keys
```
您需要先丢弃引用的外键约束，或者在丢弃父表时使用`CASCADE CONSTRAINTS`选项：
```sql
SQL> drop table inv cascade constraints;
```
必须是表的所有者或具有`DROP ANY TABLE`系统权限才能丢弃表。如果您拥有`DROP ANY TABLE`权限，则可以通过在表名前加上模式名来丢弃其他模式中的表：
```sql
SQL> drop table inv_mgmt.inv;
```
如果不在表名前加上用户名，Oracle 会假定您要丢弃当前模式中的表。

> **提示**
> 如果启用了闪回查询或闪回数据库，请记住，对于意外丢弃的表，您可以将其闪回到丢弃前的状态。

### 恢复表
假设您意外丢弃了一个表，并希望恢复它。首先，验证要恢复的表是否在回收站中：
```sql
SQL> show recyclebin;
```
以下是一些示例输出：
```
ORIGINAL NAME  RECYCLEBIN NAME                OBJECT TYPE         DROP TIME
-------------  ---------------                -----------  ----------------
INV            BIN$0F27WtJGbXngQ4TQTwq5Hw==$0 TABLE      2012-12-08:12:56:45
```
接下来，使用`FLASHBACK TABLE...TO BEFORE DROP`语句来恢复被丢弃的表：
```sql
SQL> flashback table inv to before drop;
```
> **注意**
> 对于在`SYSTEM`表空间中创建的表，不能使用`FLASHBACK TABLE...TO BEFORE DROP`语句。

当您执行`DROP TABLE`语句（不带`PURGE`）时，表实际上被重命名（以`BIN$`开头）并放入回收站。回收站是一种机制，允许您查看与被丢弃对象相关的一些元数据。您可以通过查询`DBA_SEGMENTS`来查看重命名对象的完整元数据：
```sql
SQL> select
owner
,segment_name
,segment_type
,tablespace_name
from dba_segments
where segment_name like 'BIN$%';
```
`FLASHBACK TABLE`语句只是将表重命名为其原始名称。默认情况下，`RECYCLEBIN`功能是启用的。您可以通过将`RECYCLEBIN`初始化参数设置为`OFF`来更改默认值。

我建议不要禁用`RECYCLEBIN`功能。更安全的做法是保持此功能启用，并清除`RECYCLEBIN`以永久删除您想要删除的对象。这意味着与被丢弃表关联的空间直到您清除`RECYCLEBIN`才会被释放。如果要清除当前连接用户的回收站中的所有内容，请使用`PURGE RECYCLEBIN`语句：
```sql
SQL> purge recyclebin;
```
如果要清除数据库中所有用户的回收站，请以具有 DBA 权限的用户执行以下操作：
```sql
SQL> purge dba_recyclebin;
```
如果要绕过`RECYCLEBIN`功能并永久丢弃表，请使用`DROP TABLE`语句的`PURGE`选项：
```sql
SQL> drop table inv purge;
```
您不能使用`FLASHBACK TABLE`语句来检索使用`PURGE`选项丢弃的表。表使用的所有空间都会被释放，任何相关的索引和触发器也会被丢弃。

### 从表中移除数据



## DELETE 与 TRUNCATE 语句对比

您可以使用 `DELETE` 语句或 `TRUNCATE` 语句从表中删除记录。您需要了解这两种方法之间的一些重要区别。表 7-3 总结了 `DELETE` 和 `TRUNCATE` 语句的属性。

### 表 7-3 `DELETE` 与 `TRUNCATE` 的特性

| 特性 | DELETE | TRUNCATE |
| :--- | :--- | :--- |
| 可选择 `COMMIT` 或 `ROLLBACK` | YES | NO |
| 生成 undo | YES | NO |
| 将高水位线重置为 0 | NO | YES |
| 受外键约束影响 | NO | YES |
| 处理大量数据时性能良好 | NO | YES |

### 使用 DELETE

一个很大的区别是 `DELETE` 语句可以被提交或回滚。提交 `DELETE` 语句会使更改永久生效：

```
SQL> delete from inv;
SQL> commit;
```

如果您发出 `ROLLBACK` 语句而不是 `COMMIT`，则表中的数据将恢复到执行 `DELETE` 之前的状态。

### 使用 TRUNCATE

`TRUNCATE` 是一个 DDL 语句。这意味着 Oracle 在运行后会自动提交该语句（以及当前事务），因此无法回滚 `TRUNCATE` 语句。如果您在删除数据时需要选择回滚（而不是提交）的选项，则应使用 `DELETE` 语句。然而，`DELETE` 语句的缺点是会生成大量的 undo 和 redo 信息。因此，对于大表，`TRUNCATE` 语句通常是删除数据最有效的方法。

> 注意：可能可以使用 CTAS 语句运行闪回查询，以查询 truncate 之前的数据，并将这些数据恢复到另一个表中。您可以 `CREATE TABLE tableflashback as SELECT * from table as of TIMESTAMP….`。这取决于 UNDO 的大小和保留时间，但在从备份恢复之前是值得研究的方法。

此示例使用 `TRUNCATE` 语句从 `COMPUTER_SYSTEMS` 表中删除所有数据：

```
SQL> truncate table computer_systems;
```

默认情况下，Oracle 会释放表使用的所有空间，除了由 `MINEXTENTS` 表存储参数定义的空间。如果您不希望 `TRUNCATE` 语句释放区，请使用 `REUSE STORAGE` 参数：

```
SQL> truncate table computer_systems reuse storage;
```

`TRUNCATE` 语句将表的高水位线重置为 0。当您使用 `DELETE` 语句从表中删除数据时，高水位线不会改变。使用 `TRUNCATE` 语句并重置高水位线的一个优点是，全表扫描只会搜索高水位线以下的块中的行。这可能会对性能产生显著影响。

您不能截断一个已定义主键且该主键被子表中一个启用的外键约束引用的表——即使子表包含零行。Oracle 会阻止您这样做，因为在多用户系统中，在您截断子表和随后截断父表之间的时间段内，另一个会话有可能向子表中填充数据。在这种情况下，您必须暂时禁用引用的外键约束，发出 `TRUNCATE` 语句，然后重新启用约束。

因为 `TRUNCATE` 语句是 DDL，您不能将两个独立的表的截断作为一个事务来处理。将此行为与 `DELETE` 进行比较。Oracle 允许您在约束启用（这些约束引用了一个子表）的情况下，使用 `DELETE` 语句从父表中删除行。这是因为 `DELETE` 生成 undo，是读一致的，并且可以被回滚。

> 注意：另一种从表中删除数据的方法是删除并重新创建表。然而，这意味着您还必须重新创建属于该表的任何索引、约束、授权和触发器。此外，当您删除表时，在重新创建它并重新发出任何所需的授权之前，该表是不可用的。通常，删除并重新创建表仅在开发或测试环境中是可接受的。

## 查看与调整高水位线

Oracle 将表的高水位线定义为段中已使用空间和未使用空间之间的边界。当您创建一个表时，Oracle 会为该表分配一定数量的区，由 `MINEXTENTS` 表存储参数定义。每个区包含一定数量的块。在数据插入表之前，没有任何块被使用，高水位线为 0。

随着数据被插入表中，以及区的分配，高水位线边界会上升。`DELETE` 语句不会重置高水位线。

您需要了解关于高水位线的几个性能相关问题：

*   SQL 查询全表扫描
*   直接路径加载的空间使用

Oracle 在执行查询时有时需要扫描表的每个块（在高水位线以下）。这称为全表扫描。如果从表中删除了大量数据，即使对于零行的表，全表扫描也可能需要很长时间才能完成。

此外，在进行直接路径加载时，Oracle 在高水位线之上插入数据。您最终可能会在一个经常被删除数据并通过直接路径机制加载的表中，留下大量未使用的空间。

有几种方法可以检测高水位线以下的空间：

*   Autotrace 工具
*   `DBMS_SPACE` 包
*   从数据字典 `extents` 视图中查询

Autotrace 工具提供了一种检测高水位线问题的简单方法。Autotrace 的优势在于它易于使用，并且输出易于解释。

您可以使用 `DBMS_SPACE` 包来确定在使用自动段空间管理的表空间中创建的对象的高水位线。`DBMS_SPACE` 包允许您以编程方式检查高水位线问题。这种方法的缺点是输出有些晦涩，难以得出具体的答案。

从 `DBA/ALL/USER_EXTENTS` 中查询为您提供了诸如消耗的区数和字节数等信息。这是检测高水位线问题的一种快速简便的方法。

### 通过跟踪检测高水位线以下的空间

您可以运行这个简单的测试来检测是否存在高水位线以下未使用空间的问题：

1.  `SQL> set autotrace trace statistics`。
2.  执行进行全表扫描的查询。
3.  比较处理的行数与逻辑 I/O 数（内存和磁盘访问）。

如果处理的行数很少，但逻辑 I/O 数很高，则可能存在高水位线以下的空闲块问题。下面是一个说明此技术的简单示例：

```
SQL> set autotrace trace statistics
```

下一个查询在 `INV` 表上生成全表扫描：

```
SQL> select * from inv;
```

这是 `AUTOTRACE` 输出的一个片段：

```
no rows selected
Statistics
----------------------------------------------------------
          4  recursive calls
          0  db block gets
       7371  consistent gets
       2311  physical reads
```

返回的行数为零，但有 7,371 次一致读取（内存访问）和 2,311 次物理磁盘读取，表明高水位线以下有空闲空间。

接下来，截断该表，并再次运行查询：

```
SQL> truncate table inv;
SQL> select * from inv;
```

这是 `AUTOTRACE` 输出的部分列表：

```
no rows selected
Statistics
----------------------------------------------------------
          6  recursive calls
          0  db block gets
         12  consistent gets
          0  physical reads
```

请注意，内存访问和物理读取的次数现在非常小。



#### 使用 DBMS_SPACE 检测高水位标记之下的空间

你可以使用 `DBMS_SPACE` 包来检测高水位标记之下的空闲块。以下是一个可以从 SQL*Plus 调用的 PL/SQL 匿名块：

```
SQL> set serverout on size 1000000
SQL> declare
p_fs1_bytes number;
p_fs2_bytes number;
p_fs3_bytes number;
p_fs4_bytes number;
p_fs1_blocks number;
p_fs2_blocks number;
p_fs3_blocks number;
p_fs4_blocks number;
p_full_bytes number;
p_full_blocks number;
p_unformatted_bytes number;
p_unformatted_blocks number;
begin
dbms_space.space_usage(
segment_owner      => user,
segment_name       => 'INV',
segment_type       => 'TABLE',
fs1_bytes          => p_fs1_bytes,
fs1_blocks         => p_fs1_blocks,
fs2_bytes          => p_fs2_bytes,
fs2_blocks         => p_fs2_blocks,
fs3_bytes          => p_fs3_bytes,
fs3_blocks         => p_fs3_blocks,
fs4_bytes          => p_fs4_bytes,
fs4_blocks         => p_fs4_blocks,
full_bytes         => p_full_bytes,
full_blocks        => p_full_blocks,
unformatted_blocks => p_unformatted_blocks,
unformatted_bytes  => p_unformatted_bytes
);
dbms_output.put_line('FS1: blocks = '||p_fs1_blocks);
dbms_output.put_line('FS2: blocks = '||p_fs2_blocks);
dbms_output.put_line('FS3: blocks = '||p_fs3_blocks);
dbms_output.put_line('FS4: blocks = '||p_fs4_blocks);
dbms_output.put_line('Full blocks = '||p_full_blocks);
end;
/
```

在这个场景中，你想要检查 `INV` 表在高水位标记之下的空闲空间。以下是上述 PL/SQL 的输出结果：

```
FS1: blocks = 0
FS2: blocks = 0
FS3: blocks = 0
FS4: blocks = 3646
Full blocks = 0
```

在上面的输出中，`FS1` 参数显示有 0 个块具有 0 到 25% 的空闲空间。`FS2` 参数显示有 0 个块具有 25 到 50% 的空闲空间。`FS3` 参数显示有 0 个块具有 50 到 75% 的空闲空间。`FS4` 参数显示有 3，646 个块具有 75 到 100% 的空闲空间。最后，有 0 个满块。因为没有满块，并且大量块几乎是空的，你可以看到高水位标记之下确实存在空闲空间。

### 从数据字典 Extents 视图中查询

你也可以通过查询 `DBA/ALL/USER_EXTENTS` 视图来检测存在高水位标记问题的表。如果一个表分配了大量区间（extent），但行数为零，这表明有大量数据已从该表中删除；例如：

```
SQL> select count(*) from user_extents where segment_name='INV';
COUNT(*)
```

现在，检查表中的行数：

```
SQL> select count(*) from inv;
COUNT(*)
```

上面的表很可能曾经插入过数据，从而导致了区间的分配。随后，数据被删除，但这些区间仍然保留着。

### 降低高水位标记

如何降低表的高水位标记？你可以使用几种技术将高水位标记重置为 0：

*   `TRUNCATE` 语句
*   `ALTER TABLE ... SHRINK SPACE`
*   `ALTER TABLE ... MOVE`

使用 `TRUNCATE` 语句已在本章前面讨论过（参见“使用 TRUNCATE”一节）。缩小表和移动表将在以下章节中讨论。

#### 缩小表

要重新调整高水位标记，你必须为该表启用行移动（row movement），然后使用 `ALTER TABLE...SHRINK SPACE` 语句。创建表的表空间必须是在创建时启用了自动段空间管理（automatic segment space management）。你可以通过此查询来确定表空间的段空间管理类型：

```
SQL> select tablespace_name, segment_space_management from dba_tablespaces;
```

表所属表空间的 `SEGMENT_SPACE_MANAGEMENT` 值必须是 `AUTO`。接下来，你需要为要缩小的表启用行移动。此示例为 `INV` 表启用行移动：

```
SQL> alter table inv enable row movement;
```

现在，你可以缩小该表使用的空间：

```
SQL> alter table inv shrink space;
```

你还可以通过 `CASCADE` 子句来缩小与任何索引段相关的空间：

```
SQL> alter table inv shrink space cascade;
```

#### 移动表

移动表意味着在当前表空间中重建表，或者在另一个表空间中构建它。你可能想要移动表，因为其当前表空间存在磁盘空间存储问题，或者因为你想降低该表的高水位标记。

使用 `ALTER TABLE ... MOVE` 语句将表从一个表空间移动到另一个表空间。此示例将 `INV` 表移动到 `USERS` 表空间：

```
SQL> alter table inv move tablespace users;
```

你可以通过查询 `USER_TABLES` 来验证表是否已被移动：

```
SQL> select table_name, tablespace_name from user_tables where table_name='INV';
TABLE_NAME           TABLESPACE_NAME
-------------------- ------------------------------
INV                  USERS
```

注意：`ALTER TABLE ... MOVE` 语句在执行期间不允许 DML 操作。虽然存在一些限制，但有一个 `ALTER TABLE ... ONLINE MOVE` 命令不会限制对表的访问，或者你可以使用 `DBMS_REDEFINITION` 包。

你也可以在移动表时指定 `NOLOGGING`：

```
SQL> alter table inv move tablespace users nologging;
```

使用 `NOLOGGING` 移动表会消除表重定位时通常会生成的大部分重做日志。使用 `NOLOGGING` 的缺点是，如果在表移动后立即发生故障（并且你没有表移动后的备份），那么你将无法恢复表的内容。如果表中的数据至关重要，则在移动时不要使用 `NOLOGGING`。

当你移动一个表时，它的所有索引都会变得不可用。这是因为表的索引在结构中包含 `ROWID`。表 `ROWID` 包含物理位置信息。鉴于表从一个表空间移动到另一个表空间时，其 `ROWID` 会发生变化（因为表行现在物理上位于不同的数据文件中），表上的任何索引都包含不正确信息。要重建索引，请使用 `ALTER INDEX ... REBUILD` 命令。

## Oracle ROWID

每个表中的每一行都有一个地址。行的地址由以下组合确定：

```
数据文件号
块号
行在块内的位置
对象号
```

你可以通过查询 `ROWID` 伪列来显示表中某一行的地址；例如：

```
SQL> select rowid, emp_id from emp;
```

以下是一些示例输出：

```
ROWID                  EMP_ID
------------------ ----------
AAAFJAAAFAAAAJfAAA          1
```

`ROWID` 伪列值并未物理存储在数据库中。Oracle 在你查询时计算其值。`ROWID` 的内容显示为 Base64 值，可能包含字符 A–Z、a–z、0–9、+ 和 /。你可以使用 `DBMS_ROWID` 包将 `ROWID` 值转换为有意义的信息。例如，要显示行存储的相对文件号，请执行此语句：

```
SQL> select dbms_rowid.rowid_relative_fno(rowid), emp_id from emp;
```

以下是一些示例输出：

```
DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID)     EMP_ID
------------------------------------ ----------
5                                        1
```

你可以在 SQL 语句的 `SELECT` 和 `WHERE` 子句中使用 `ROWID` 值。在大多数情况下，`ROWID` 唯一地标识一行。然而，不同表中的行可能存储在同一个簇中，因此可能包含具有相同 `ROWID` 的行。

## 创建临时表


## 使用临时表

使用 `CREATE GLOBAL TEMPORARY TABLE` 语句创建一个仅临时存储数据的表。您可以指定该临时表是保留数据直至会话结束，还是直至事务提交。使用 `ON COMMIT PRESERVE ROWS` 来指定数据在用户会话结束时删除。在此示例中，行将被保留，直到用户显式删除数据或终止会话：

```sql
SQL> create global temporary table today_regs
on commit preserve rows
as select * from f_registrations
where create_dtt > sysdate - 1;
```

指定 `ON COMMIT DELETE ROWS` 以指示数据应在事务结束时删除。以下示例创建了一个名为 `TEMP_OUTPUT` 的临时表，并指定记录应在每个提交的事务结束时被删除：

```sql
create global temporary table temp_output(
temp_row varchar2(30))
on commit delete rows;
```

> **注意**
> 如果您未为全局临时表指定提交方法，则默认为 `ON COMMIT DELETE ROWS`。

您可以创建临时表并授予其他用户访问权限。但是，一个会话只能查看其自身插入到表中的数据。换句话说，如果两个会话正在使用同一个临时表，一个会话无法查询由另一个会话插入到临时表中的任何数据。

全局临时表对于需要在表结构中短暂存储数据的应用程序非常有用。创建临时表后，它将一直存在，直到您将其删除。换句话说，临时表的定义是“永久的”——短暂的是数据（在这个意义上，“临时表”这个术语可能会产生误导）。

您可以通过查询 `DBA/ALL/USER_TABLES` 的 `TEMPORARY` 列来查看表是否是临时的：

```sql
SQL> select table_name, temporary from user_tables;
```

临时表在 `TEMPORARY` 列中被标记为 `Y`。常规表在 `TEMPORARY` 列中包含 `N`。

当您在临时表中创建记录时，空间会在您的默认临时表空间中分配。您可以通过运行以下 SQL 进行验证：

```sql
SQL> select username, contents, segtype from v$sort_usage;
```

如果您处理大量行并且需要更好的选择性检索行的性能，可以考虑在临时表的相应列上创建索引：

```sql
SQL> create index temp_index on temp_output(temp_row);
```

使用 `DROP TABLE` 命令来删除临时表：

```sql
SQL> drop table temp_output;
```

## 临时表与重做日志

对全局临时表的数据块所做的更改不会生成重做数据。但是，针对临时表的事务会生成回滚（撤销）数据。由于回滚数据会生成重做，因此与临时表事务相关联的会有一些重做数据。您可以通过打开统计信息跟踪并查看向临时表插入记录时的重做大小来验证这一点：

```sql
SQL> set autotrace on
```

接下来，向临时表中插入几条记录：

```sql
SQL> insert into temp_output values(1);
```

以下是输出的一个片段（仅显示重做大小）：

```
140 redo size
```

临时表的重做负载小于常规表，因为生成的重做仅与临时表事务的回滚（撤销）数据相关联。此外，从 Oracle Database 12c 开始，临时对象的撤销存储在临时表空间中，而不是撤销表空间中。

## 创建索引组织表

索引组织表（IOT）在表数据通常通过主键查询访问时是高效的对象。使用 `ORGANIZATION INDEX` 子句来创建 IOT：

```sql
SQL> create table prod_sku
(prod_sku_id number,
sku varchar2(256),
create_dtt timestamp(5),
constraint prod_sku_pk primary key(prod_sku_id)
)
organization index
including sku
pctthreshold 30
tablespace inv_data
overflow
tablespace inv_data;
```

IOT 将表行的全部内容存储在 B-tree 索引结构中。IOT 为具有主键上的精确匹配或范围搜索（或两者）的查询提供快速访问。所有指定的列，直到并包括 `INCLUDING` 子句中指定的列，都存储在与 `PROD_SKU_ID` 主键列相同的块中。换句话说，`INCLUDING` 子句指定了在表段中保留的最后一个列。在 `INCLUDING` 子句中指定的列之后的列存储在溢出数据段中。在上面的示例中，`CREATE_DTT` 列存储在溢出段中。`PCTTHRESHOLD` 指定了在索引块中为 IOT 行保留的空间百分比。该值可以是 1 到 50，如果未指定值，则默认为 50。索引块中必须有足够的空间来存储主键。

`OVERFLOW` 子句详细说明了哪个表空间应用于存储溢出数据段。请注意，`DBA/ALL/USER_TABLES` 包含创建 IOT 时使用的表名的条目。此外，`DBA/ALL/USER_INDEXES` 包含指定的主键约束名称的记录。对于 IOT，`INDEX_TYPE` 列包含值 `IOT - TOP`：

```sql
SQL> select index_name,table_name,index_type from user_indexes;
```

## 管理约束

接下来的几节将讨论约束。约束提供了一种确保数据符合特定业务规则的机制。您必须了解可用的约束类型以及何时适合使用它们。Oracle 提供了几种类型的约束：

*   主键
*   唯一键
*   外键
*   检查约束
*   `NOT NULL`

以下部分将讨论如何实现和管理这些约束。

### 创建主键约束

在实现数据库时，您创建的大多数表都需要一个主键约束，以保证表中的每条记录都可以被唯一标识。有多种技术可以向表添加主键约束。第一个示例在列定义中内联创建主键：

```sql
SQL> create table dept(
dept_id number primary key
,dept_desc varchar2(30));
```

如果您从 `USER_CONSTRAINTS` 中选择 `CONSTRAINT_NAME`，请注意 Oracle 会为约束生成一个隐晦的名称（例如 `SYS_C003682`）。使用以下语法显式地为主键约束命名：

```sql
SQL> create table dept(
dept_id number constraint dept_pk primary key using index tablespace users,
dept_desc varchar2(30));
```

> **注意**
> 创建主键约束时，Oracle 还会创建一个与约束同名的唯一索引。您可以通过 `USING INDEX TABLESPACE` 子句控制唯一索引放置在哪个表空间中。

您也可以在定义列之后再指定主键约束定义。这样做的好处是可以跨多列定义约束。下一个示例在创建表时创建主键，但不内联于列定义：

```sql
SQL> create table dept(
dept_id number,
dept_desc varchar2(30),
constraint dept_pk primary key (dept_id)
using index tablespace users);
```

如果表已经创建，并且您想添加主键约束，请使用 `ALTER TABLE` 语句。此示例在 `DEPT` 表的 `DEPT_ID` 列上放置一个主键约束：

```sql
SQL> alter table dept
add constraint dept_pk primary key (dept_id)
using index tablespace users;
```

启用主键约束时，Oracle 会自动创建一个与主键约束关联的唯一索引。一些 DBA 更喜欢先在主键列上创建一个非唯一索引，然后再定义主键约束：

```sql
SQL> create index dept_pk on dept(dept_id) tablespace users;
SQL> alter table dept add constraint dept_pk primary key (dept_id);
```


### 优化主键约束方法

这种方法的优势在于，你可以独立于索引禁用或删除主键约束。当你处理大型数据集时，这种灵活性可能非常有用。如果你在创建主键约束之前没有创建索引，那么每当禁用或删除主键约束时，关联的索引也会被自动删除。

对于使用哪种方法创建主键感到困惑吗？所有方法都是有效的，并各有其优点。表 7-4 总结了主键和唯一键约束的创建方法。我曾在不同场景下使用所有这些方法来创建主键约束。通常，我会使用`ALTER TABLE`语句，它在表创建之后添加约束。

#### 表 7-4 主键和唯一键约束创建方法

| 约束创建方法 | 优点 | 缺点 |
| :--- | :--- | :--- |
| 内联，无名称 | 非常简单 | Oracle 生成的名称使故障排除更难；对存储属性控制较少；仅应用于单个列 |
| 内联，带名称 | 简单；用户定义的名称使故障排除更容易 | 比内联无名称需要更多思考 |
| 内联，带名称和表空间定义 | 用户定义的名称和表空间；使故障排除更容易 | 不那么简单 |
| 在列定义之后（行外） | 用户定义的名称和表空间；可对多个列操作 | 不那么简单 |
| `ALTER TABLE add just constraint` | 允许你在独立于表创建脚本的语句（和文件）中管理约束；可对多个列操作 | 更复杂 |
| `CREATE INDEX`，然后`ALTER TABLE add constraint` | 分离索引和约束，因此你可以在不影响索引的情况下删除/禁用约束；可对多个列操作 | 最复杂：需要维护的部分更多，活动组件更多 |

### 强制唯一键值

除了创建主键约束外，你还应该在那些在表内应始终保持唯一的列组合上创建唯一约束。例如，对于一个表的主键，常见做法是使用一个通过序列填充的数字键（有时称为代理键）。除了代理主键，有时用户还会有一个（或多个）业务上用于唯一标识记录的列（也称为逻辑键）。同时使用代理键和逻辑键可以做到：

*   允许你通过单个数字列高效地连接父表和子表
*   允许更新逻辑键列而不改变代理键

唯一键保证了在定义列（或列组合）上的唯一性。主键和唯一键约束之间存在一些细微差别。例如，每个表只能定义一个主键，但可以有多个唯一键。此外，主键不允许其任何列包含`NULL`值，而唯一键允许`NULL`值。

与主键约束类似，你可以使用多种方法来创建唯一列约束。此方法使用`UNIQUE`关键字与列内联定义：

```sql
SQL> create table dept(
dept_id number
,dept_desc varchar2(30) unique);
```

如果你想显式命名约束，可以使用`CONSTRAINT`关键字：

```sql
SQL> create table dept(
dept_id number
,dept_desc varchar2(30) constraint dept_desc_uk1 unique);
```

与主键类似，Oracle 会自动创建一个与唯一键约束关联的索引。你可以内联指定用于关联唯一索引的表空间信息：

```sql
SQL> create table dept(
dept_id number
,dept_desc varchar2(30) constraint dept_desc_uk1
unique using index tablespace users);
```

你也可以修改表以添加唯一约束：

```sql
SQL> alter table dept
add constraint dept_desc_uk1 unique (dept_desc)
using index tablespace users;
```

你还可以在定义唯一键约束之前，先对目标列创建索引：

```sql
SQL> create index dept_desc_uk1 on dept(dept_desc) tablespace users;
SQL> alter table dept add constraint dept_desc_uk1 unique(dept_desc);
```

当你处理大型数据集，并且希望在删除或禁用唯一约束时不删除关联的索引时，这种方法会很有帮助。

> 提示：你也可以通过唯一索引来强制执行唯一键约束。有关使用唯一索引强制执行唯一约束的详细信息，请参见第 8 章。

### 创建外键约束

外键约束用于确保某个列值包含在一组预定义的值列表中。使用外键约束是在插入或更新操作允许之前，强制数据为预定义值的有效方法。此技术适用于以下场景：

*   值列表包含大量条目。
*   需要存储关于查找值的其他信息。
*   通过 SQL 选择、插入、更新或删除值很容易。

例如，假设创建`EMP`表时包含一个`DEPT_ID`列。为确保每个员工都被分配到一个有效的部门，你可以创建一个外键约束，强制要求`EMP`表中的每个`DEPT_ID`都必须存在于`DEPT`表中。

> 提示：如果你要检查的条件是一个变化不频繁的小列表，请考虑使用检查约束而不是外键约束。例如，如果你有一个列将始终定义为只包含 0 或 1，那么检查约束是一个高效的解决方案。

作为参考，以下是这些示例中父表`DEPT`的创建方式：

```sql
SQL> create table dept(
dept_id number primary key,
dept_desc varchar2(30));
```

外键必须引用父表中已定义了主键或唯一键的列。`DEPT`是父表，其`DEPT_ID`上定义了主键。

你可以使用几种方法来创建外键约束。以下示例在`EMP`表的`DEPT_ID`列上创建了一个外键约束：

```sql
SQL> create table emp(
emp_id number,
name varchar2(30),
dept_id constraint emp_dept_fk references dept(dept_id));
```

注意，`DEPT_ID`数据类型未显式定义。外键约束从被引用的`DEPT`表的`DEPT_ID`列派生数据类型。你也可以在定义列时显式指定数据类型（无论是否有外键定义）：

```sql
SQL> create table emp(
emp_id number,
name varchar2(30),
dept_id number constraint emp_dept_fk references dept(dept_id));
```

你还可以在`CREATE TABLE`语句中，将外键定义与列定义分开（行外）指定：

```sql
SQL> create table emp(
emp_id number,
name varchar2(30),
dept_id number,
constraint emp_dept_fk foreign key (dept_id) references dept(dept_id)
);
```

并且，你可以修改现有表来添加外键约束：

```sql
SQL> alter table emp
add constraint emp_dept_fk foreign key (dept_id)
references dept(dept_id);
```

> 注意：与主键和唯一键约束不同，Oracle 不会自动为外键列添加索引；你必须显式地在它们上创建索引。有关为什么在创建外键列索引很重要以及如何检测没有关联索引的外键列的讨论，请参见第 8 章。



## 检查特定的数据条件

当你有一个简短、相对静态的值列表时（例如，一个列的值只能是 `Y` 或 `N`），使用 `CHECK` 约束进行查找非常合适。在这种情况下，值列表很可能不会改变，并且除了 `Y` 或 `N` 之外不需要存储任何其他信息，因此 `CHECK` 约束是合适的解决方案。如果你有一个需要定期更新的长值列表，那么使用表和外键约束是更好的解决方案。

此外，对于必须始终强制执行且可以用简单 SQL 表达式编写的业务规则，`CHECK` 约束也很有效。如果你有需要验证的复杂业务逻辑，那么应用程序代码更合适。

你可以在创建表时定义 `CHECK` 约束。以下语句强制 `ST_FLG` 列只能包含 `0` 或 `1`：

```sql
SQL> create table emp(
emp_id number,
emp_name varchar2(30),
st_flg number(1) CHECK (st_flg in (0,1))
);
```

一个稍好的方法是给 `CHECK` 约束起一个名字：

```sql
SQL> create table emp(
emp_id number,
emp_name varchar2(30),
st_flg number(1) constraint st_flg_chk CHECK (st_flg in (0,1))
);
```

一种更具描述性的约束命名方式是在约束名中嵌入描述违反条件的信息；例如，

```sql
SQL> create table emp(
emp_id number,
emp_name varchar2(30),
st_flg number(1) constraint "st_flg must be 0 or 1" check (st_flg in (0,1))
);
```

你也可以通过 `ALTER` 修改现有列来添加约束。该列不能包含任何违反即将启用的约束的值：

```sql
SQL> alter table emp add constraint
"st_flg must be 0 or 1" check (st_flg in (0,1));
```

## 注意

在插入或更新的行中，`CHECK` 约束的求值结果必须为 `TRUE` 或未知（`NULL`）值。你不能在 `CHECK` 约束中使用子查询或序列。另外，你不能引用 SQL 函数 `UID`、`USER`、`SYSDATE` 或 `USERENV`，也不能引用伪列 `LEVEL` 或 `ROWNUM`。

## 强制非空条件

另一个需要检查的常见条件是列是否为 `null`；你可以使用 `NOT NULL` 约束来实现。`NOT NULL` 约束可以通过多种方式定义。这里展示的是最简单的技术：

```sql
SQL> create table emp(
emp_id number,
emp_name varchar2(30) not null);
```

一个稍好的方法是给 `NOT NULL` 约束起一个对你有意义的名字。为约束命名可以让你看清约束的用途，而不是使用系统生成的约束名称，后者可能会与主键或外键约束混淆：

```sql
SQL> create table emp(
emp_id number,
emp_name varchar2(30) constraint emp_name_nn not null);
```

如果你需要修改现有表的列，请使用 `ALTER TABLE` 命令。要使以下命令生效，被定义为 `NOT NULL` 的列中不能存在任何 `NULL` 值：

```sql
SQL> alter table emp modify(emp_name not null);
```

## 注意

如果当前正在定义为 `NOT NULL` 的列中存在 `NULL` 值，你必须首先更新表，使该列的每一行都有值。

## 禁用约束

Oracle 的一个很好的特性是，你可以在不删除和重新创建约束的情况下 `禁用` 和 `启用` 约束。这意味着你不必了解重新创建已删除约束所需的 DDL 语句。

有时，你需要禁用约束。例如，你可能正尝试 `TRUNCATE` 一个表，但收到以下错误消息：

```
ORA-02266: unique/primary keys in table referenced by enabled foreign keys
```

Oracle 不允许对父表执行 `TRUNCATE` 操作，如果该父表的主键被子表中的一个启用外键所引用。如果你需要 `TRUNCATE` 一个父表，你必须首先 `禁用` 所有引用该父表主键的已启用外键约束。运行此查询以确定需要禁用的约束的名称：

```sql
SQL> col primary_key_table form a18
SQL> col primary_key_constraint form a18
SQL> col fk_child_table form a18
SQL> col fk_child_table_constraint form a18
--
SQL> select
b.table_name primary_key_table
,b.constraint_name primary_key_constraint
,a.table_name fk_child_table
,a.constraint_name fk_child_table_constraint
from dba_constraints a
,dba_constraints b
where a.r_constraint_name = b.constraint_name
and a.r_owner = b.owner
and a.constraint_type = 'R'
and b.owner = upper('&table_owner')
and b.table_name = upper('&pk_table_name');
```

对于此示例，只有一个外键依赖项：

```
PRIMARY_KEY_TAB PRIMARY_KEY_CON FK_CHILD_TABLE  FK_CHILD_TABLE_
--------------- --------------- --------------- ---------------
DEPT            DEPT_PK         EMP             EMP_DEPT_FK
```

使用 `ALTER TABLE` 语句来 `禁用` 表上的约束。在本例中，只有一个外键需要禁用：

```sql
SQL> alter table emp disable constraint emp_dept_fk;
```

你现在可以 `TRUNCATE` 父表了：

```sql
SQL> truncate table dept;
```

不要忘记在 `TRUNCATE` 操作完成后 `重新启用` 外键约束，如下所示：

```sql
SQL> alter table emp enable constraint emp_dept_fk;
```

你可以使用 `DISABLE` 子句的 `CASCADE` 选项来 `禁用` 主键及其所有相关的外键约束。例如，下一行代码禁用了与主键约束相关的所有外键约束：

```sql
SQL> alter table dept disable constraint dept_pk cascade;
```

此语句不会级联通过所有依赖层级；它仅禁用直接依赖于 `DEPT_PK` 的外键约束。还要记住，没有 `ENABLE...CASCADE` 语句。要重新启用约束，你必须查询数据字典以确定哪些约束已被禁用，然后逐一重新启用它们。

有时，在加载数据时，你会遇到这样的情况：在加载数据之前禁用所有外键很方便。在这些情况下，`imp` 实用程序按字母顺序导入表，并不确保子表在父表之前导入。你可能还希望并行运行多个导入作业以利用并行硬件。在这种情况下，你可以 `禁用` 外键，执行导入，然后重新启用外键。

以下是一个使用 SQL 生成 SQL 脚本来为用户 `禁用` 所有外键约束的脚本：

```sql
SQL> set lines 132 trimsp on head off feed off verify off echo off pagesize 0
SQL> spo dis_dyn.sql
SQL> select 'alter table ' || a.table_name
|| ' disable constraint ' || a.constraint_name || ';'
from dba_constraints a
,dba_constraints b
where a.r_constraint_name = b.constraint_name
and a.r_owner = b.owner
and a.constraint_type = 'R'
and b.owner = upper('&table_owner');
SQL> spo off;
```

此脚本生成一个名为 `dis_dyn.sql` 的文件，其中包含为用户禁用所有外键约束的 SQL 语句。

## 启用约束

本节包含一些脚本，可帮助你 `启用` 已禁用的约束。下面列出的是一个脚本，它创建一个文件，其中包含为指定用户的所有表重新启用任何外键约束所需的 SQL 语句：

```sql
SQL> set lines 132 trimsp on head off feed off verify off echo off pagesize 0
SQL> spo enable_dyn.sql
SQL> select 'alter table ' || a.table_name
|| ' enable constraint ' || a.constraint_name || ';'
from dba_constraints a
,dba_constraints b
where a.r_constraint_name = b.constraint_name
and a.r_owner = b.owner
and a.constraint_type = 'R'
and b.owner = upper('&table_owner');
SQL> spo off;
```


## 约束管理

启用约束时，默认情况下 Oracle 会检查数据是否违反约束定义。如果您相当确定数据完整性良好，并且不想因重新验证约束而带来性能开销，可以在重新启用约束时使用 `NOVALIDATE` 子句。示例如下：

```sql
SQL> select 'alter table ' || a.table_name
|| ' modify constraint ' || a.constraint_name || ' enable novalidate;'
from dba_constraints a
,dba_constraints b
where a.r_constraint_name = b.constraint_name
and a.r_owner = b.owner
and a.constraint_type = 'R'
and b.owner = upper('&table_owner');
```

`NOVALIDATE` 子句指示 Oracle 不对正在启用的约束进行验证，但它确实会强制执行任何新的 DML 活动都遵守约束定义。

在多用户系统中，存在另一种可能性：在外键约束被禁用期间，另一个会话已向子表插入了数据。如果发生这种情况，当您尝试重新启用外键时，会看到以下错误：

```sql
ORA-02298: cannot validate (.) - parent keys not found
```

在这种情况下，您可以使用 `ENABLE NOVALIDATE` 子句：

```sql
SQL> alter table emp enable novalidate constraint emp_dept_fk;
```

要清理违反约束的行，首先请确保您在当前连接的模式中创建了一个 `EXCEPTIONS` 表。如果您没有 `EXCEPTIONS` 表，请使用此脚本创建一个：

```sql
SQL> @?/rdbms/admin/utlexcpt.sql
```

接下来，使用 `EXCEPTIONS INTO` 子句将违反约束的行填充到 `EXCEPTIONS` 表中：

```sql
SQL> alter table emp modify constraint emp_dept_fk validate
exceptions into exceptions;
```

只要存在违反约束的行，此语句仍会抛出 `ORA-02298` 错误。该语句还会将任何有问题的行的记录插入到 `EXCEPTIONS` 表中。现在，您可以使用 `EXCEPTIONS` 表的 `ROW_ID` 列来删除任何违反约束的记录。

此处，您可以看到需要从 `EMP` 表中删除一行：

```sql
SQL> select * from exceptions;
```

以下是一些示例输出：

```sql
ROW_ID             OWNER    TABLE_NAME CONSTRAINT
------------------ -------- ---------- --------------------
AAAFKQAABAAAK8JAAB MV_MAINT EMP        EMP_DEPT_FK
```

要删除违规记录，请发出 `DELETE` 语句：

```sql
SQL> delete from emp where rowid = 'AAAFKQAABAAAK8JAAB';
```

如果 `EXCEPTIONS` 表包含许多记录，您可以运行如下查询，按 `OWNER` 和 `TABLE_NAME` 删除：

```sql
SQL> delete from emp where rowid in
(select row_id
from exceptions
where owner=upper('&owner') and table_name = upper('&table_name'));
```

您也可能会遇到需要禁用主键或唯一键约束（或两者）的情况。例如，您可能希望执行大型数据加载，并且出于性能原因希望禁用主键和唯一键约束。您不希望因为每一行插入时都需要检查而产生开销。

用于禁用外键的相同通用技术也适用于禁用主键和唯一键。运行此查询以显示用户的主键和唯一键约束：

```sql
SQL> select
a.table_name
,a.constraint_name
,a.constraint_type
from dba_constraints a
where a.owner = upper('&table_owner')
and a.constraint_type in ('P','U')
order by a.table_name;
```

当表名和约束名确定后，使用 `ALTER TABLE` 语句禁用约束：

```sql
SQL> alter table dept disable constraint dept_pk;
```

> **注意**
> Oracle 不允许您禁用在已启用的外键约束中引用的主键或唯一键约束。您必须先禁用外键约束。

## 总结

本章重点介绍了与创建和维护表相关的基本活动。表是在数据库中存储数据的容器。关键的表管理任务包括修改、移动、删除、收缩和删除表。您还必须熟悉如何实现和使用特殊表类型，例如临时表、索引组织表（IOT）和只读表。

Oracle 还提供了各种约束来帮助您管理表中的数据。约束是数据完整性的基石。在大多数情况下，每个表都应包含一个主键约束，以确保每一行都是唯一可识别的。此外，任何父子关系都应通过外键约束来强制执行。您可以使用唯一约束来实现要求列或列组合具有唯一性的业务规则。检查约束和 `NOT NULL` 约束可确保列包含业务指定的数据要求。

可以使用 `DBMS_REDEFINITION` 包在线修改表，并且可以生成脚本来准备表更改。这需要通过开发环境进行测试，以便将所有部分（如索引和启用约束）作为一个脚本一起运行，这样就不会忘记启用或创建键和索引。

创建表后，下一个合乎逻辑的活动是在适当的地方创建索引。索引是可选的数据库对象，有助于提高性能。索引的创建和维护任务将在下一章中介绍。


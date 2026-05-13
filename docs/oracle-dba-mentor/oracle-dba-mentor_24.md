# 14. `数据字典`

在上一章中，我们讨论了 Oracle 文档集。希望您已经有机会查阅了这些文档，对它越来越熟悉，从而也能让它对您更有用。在本章中，我们将转向另一种类型的文档——数据字典。我们还将探讨一些我们可以使用的各种动态性能视图。

我们在本章中看到的文档存在于您每一个 Oracle 数据库中。它能告诉您关于您数据库的信息，比任何一本书都多。每个数据库引擎，无论是 Oracle、SQL Server、Postgres 等，都有某种形式的数据字典。

本章的所有内容在您特定版本的 *Oracle Database Reference* 中都有记录。在上一章关于 Oracle 文档集的内容中没有提及 *Oracle Database Reference*，是因为我们打算在这里提到它。如果您对在这里看到的任何视图有疑问，请随时在 *Database Reference* 中进行更多探索。花点时间浏览一下这本书，看看它的布局。

## 数据字典

每个值得使用的数据库引擎都有某种形式的数据字典。有时它也被称为 *目录*、*系统目录* 或 *数据存储库*。当 Edgar Codd 开创关系数据库规则时，其中一条规则规定必须创建一个关系目录。每个数据库引擎如何实现数据字典会有所不同，但它们都有某种形式的数据字典。

那么，什么是数据字典？最常见的答案是，数据字典是关于数据本身的元数据。基本上，数据字典包含了您可能需要阅读的信息，以便了解更多关于数据库中内容的信息。

无论数据字典如何实现，您与它的交互方式与您与任何其他数据库表的交互方式相同。您查询它。在 Oracle 中，您通过 `SQL` 语句来查询数据字典。一些数据库引擎会提供存储过程来执行。您从数据字典中获取信息的方式，与您应用程序中其他数据的获取方式是相同的。

有趣的是，数据字典的更新方式也与您在应用程序表中更新数据的方式相同。当您创建一个表时，Oracle 会对一个包含数据库中所有表列表的数据字典表执行 `INSERT` 语句。Oracle 还会向另一个包含所有表的所有列列表的表中插入其他行。Oracle 与数据字典交互的方式，与您与自己的表交互的方式是相同的。

您绝不应该自己直接修改数据字典表中的数据。这样做可能会导致数据字典损坏，并使您的数据库处于无法操作的状态。让数据库引擎为您更新数据字典。您应该只查询数据字典。

## 基础表

数据字典由许多基础表组成。在当今的 Oracle 版本中，有超过一千个基础表。所有基础表都属于 `SYS` 用户。常听到的说法是 `SYS` 就是数据字典，或者 `SYS` 拥有数据字典。没有数据字典，数据库就无法运行，因此它必须始终受到保护。话虽如此，探索这些基础表并查看它们包含的内容是很有趣的。我还应该提到，我们几乎从不直接查询基础表。相反，我们使用 Oracle 为我们提供的数据字典视图。正如我们将在本节中看到的，基础表中的信息可能晦涩难懂，难以或无法破译。Oracle 没有公布太多关于基础表的信息。然而，开始探索 Oracle 数据库内部并了解数据库在底层如何工作是很有趣的。

让我们看看其中一个基础表。`SYS.TAB$` 表包含了数据库中所有表的列表。在基础表名称中看到美元符号是很常见的。在列表 14-1 中，我们可以看到 `SYS.TAB$` 表的描述。

```
SQL> desc sys.tab$
Name                       Null?       Type
--------------------------- ----------- --------------
OBJ#                       NOT NULL    NUMBER
DATAOBJ#                               NUMBER
TS#                        NOT NULL    NUMBER
FILE#                      NOT NULL    NUMBER
BLOCK#                     NOT NULL    NUMBER
BOBJ#                                  NUMBER
TAB#                                   NUMBER
COLS                       NOT NULL    NUMBER
CLUCOLS                                NUMBER
PCTFREE$                   NOT NULL    NUMBER
PCTUSED$                   NOT NULL    NUMBER
INITRANS                   NOT NULL    NUMBER
MAXTRANS                   NOT NULL    NUMBER
FLAGS                      NOT NULL    NUMBER
AUDIT$                     NOT NULL    VARCHAR2(38)
ROWCNT                                 NUMBER
BLKCNT                                 NUMBER
EMPCNT                                 NUMBER
AVGSPC                                 NUMBER
CHNCNT                                 NUMBER
AVGRLN                                 NUMBER
AVGSPC_FLB                             NUMBER
FLBCNT                                 NUMBER
ANALYZETIME                            DATE
SAMPLESIZE                             NUMBER
DEGREE                                 NUMBER
INSTANCES                              NUMBER
INTCOLS                    NOT NULL    NUMBER
KERNELCOLS                 NOT NULL    NUMBER
PROPERTY                   NOT NULL    NUMBER
TRIGFLAG                               NUMBER
SPARE1                                 NUMBER
SPARE2                                 NUMBER
SPARE3                                 NUMBER
SPARE4                                 VARCHAR2(1000)
SPARE5                                 VARCHAR2(1000)
SPARE6                                 DATE
列表 14-1
SYS.TAB$ 基础表
```

如果您曾花时间创建过 Oracle 表，那么这个基础表中的一些列名是有意义的。我们可以看到 `PCTFREE$`、`INITRANS` 和 `DEGREE`，这些都是您在创建表时可以指定的选项。缺失的一点是表的所有者和名称。要了解这部分信息，我们必须查看描述所有数据库对象的基础表 `SYS.OBJ$`。我们可以在列表 14-2 中看到 `SYS.OBJ$` 的描述。



```
SQL> desc sys.obj$
名称              是否为空?  类型
----------------- --------- ----------------------------
OBJ#              NOT NULL  NUMBER
DATAOBJ#                    NUMBER
OWNER#            NOT NULL  NUMBER
NAME              NOT NULL  VARCHAR2(128)
NAMESPACE         NOT NULL  NUMBER
SUBNAME                    VARCHAR2(128)
TYPE#             NOT NULL  NUMBER
CTIME             NOT NULL  DATE
MTIME             NOT NULL  DATE
STIME             NOT NULL  DATE
STATUS            NOT NULL  NUMBER
REMOTEOWNER                 VARCHAR2(128)
LINKNAME                    VARCHAR2(128)
FLAGS                       NUMBER
OID$                        RAW(16)
SPARE1                      NUMBER
SPARE2                      NUMBER
SPARE3                      NUMBER
SPARE4                      VARCHAR2(1000)
SPARE5                      VARCHAR2(1000)
SPARE6                      DATE
SIGNATURE                   RAW(16)
SPARE7                      NUMBER
SPARE8                      NUMBER
SPARE9                      NUMBER
清单 14-2
SYS.OBJ$ 基础表
```

立即显而易见的是，为了了解数据库中的任何表，我们至少需要两个表：`SYS.TAB$` 和 `SYS.OBJ$`。请注意，在 `SYS.OBJ$` 中，我们确实有对象的 NAME，但没有任何列告诉我们其所有者。相反，我们必须使用 `OWNER#` 列连接到 `SYS.USER$` 表的 `USER#` 列，这样我们就向组合中引入了第三个基础表。

基础表还可能包含一些对我们来说意义不大的列。在 `SYS.TAB$` 中，我们有名为 `FLAGS`、`PROPERTY`、`TRIGFLAG` 的列，以及多个 `SPARE` 列。如果你曾经想了解更多关于基础表的列信息，只需在 `$ORACLE_HOME/rdbms/admin` 目录下做一点侦探工作即可。正如我们稍后将看到的，我们通常使用的是 `DBA_TABLES` 视图，而不是 `SYS.TAB$` 表。那么，让我们进入该目录，看看哪个脚本创建了 `DBA_TABLES` 视图。清单 14-3 展示了如何找到创建 `DBA_TABLES` 视图的脚本。

```
[oracle@dbamentor ~]$ cd $ORACLE_HOME/rdbms/admin
[oracle@dbamentor admin]$ grep -i dba_tables *|grep -i create|grep -i view
catspace.sql:create or replace view DBA_TABLESPACES
catspace.sql:execute CDBView.create_cdbview(false,'SYS','DBA_TABLESPACES','CDB_TABLESPACES');
catspace.sql:create or replace view DBA_TABLESPACE_GROUPS
catspace.sql:execute CDBView.create_cdbview(false,'SYS','DBA_TABLESPACE_GROUPS','CDB_TABLESPACE_GROUPS');
catspace.sql:create or replace view DBA_TABLESPACE_USAGE_METRICS
catspace.sql:execute CDBView.create_cdbview(false,'SYS','DBA_TABLESPACE_USAGE_METRICS','CDB_TABLESPACE_USAGE_METRICS');
cdcore.sql:create or replace view DBA_TABLES
cdcore.sql:execute CDBView.create_cdbview(false,'SYS','DBA_TABLES','CDB_TABLES');
depssvrm.sql:  CREATE OR REPLACE VIEW DBA_TABLESPACE_THRESHOLDS
depssvrm.sql:execute CDBView.create_cdbview(false,'SYS','DBA_TABLESPACE_THRESHOLDS','CDB_TABLESPACE_THRESHOLDS');
e1002000.sql:create or replace view dba_tablespaces as select null nn from dual ;
清单 14-3
查找 DBA_TABLES 的创建位置
```

在清单 14-3 中，我使用 `grep` 来获取所有包含字符串 `DBA_TABLES` 的文件列表。然后，我筛选出那些在同一行中同时包含 `CREATE` 和 `VIEW` 的行。返回的某些行是针对其他数据库对象的，但我们可以看到文件 `cdcore.sql` 创建了视图 `DBA_TABLES`。在文本编辑器中打开该文件，查找创建此视图的那一行。不要对此文件做任何更改！在此文件中，我们可以看到 Oracle 如何使用九个不同的 SYS 所有表，外加一些神秘的 `X$` 表来创建这个视图，我稍后会讲到这些 `X$` 表。

希望在了解了 `DBA_TABLES` 是如何创建的，以及所有那些不同的连接之后，我们能明白为什么我们不直接使用数据字典基础表。正如我们将在本章探讨的，视图使事情对我们来说简单得多。使用视图而不是基础表的另一个原因是，Oracle 可能会在新版本中更改基础表的列和数据。当他们这样做时，他们会更新视图以处理这些更改。如果你直接查询基础表，你可能必须重写你的 SQL 语句。如果你查询视图，在新版本中事情应该能同样工作。

那么，`X$KSPPCV` 和 `X$KSPPI` 这些作为 `DBA_TABLES` 视图一部分的 `X$` 表又是什么呢？这些 `X$` 表是 Oracle 用来查看其自身内存内容的一种机制。这样，Oracle 可以通过一条简单的 SQL 语句来查询其内存中的内容。`DBA_TABLES` 视图从各种数据字典表中获取部分信息，其余信息则来自存储在 Oracle 内存中的数据。

对于你工作的大部分内容，这些基础表和 `X$` 内存表对于你的数据库管理任务并不重要。即使不花太多时间在它们上面，你也能拥有长久的职业生涯。我之所以提到它们，是因为随着你在 Oracle 领域不断进步，你可能会对幕后的工作原理产生更多兴趣，这是不可避免的。好奇心在这里是很好的。在很大程度上，我们将通过本章其余部分介绍的信息来与数据字典交互，但你应该继续培养一种健康的对 Oracle 幕后工作原理的好奇心。

## 静态字典视图

Oracle 将其许多数据字典视图归类为 *静态* 的。使用这个定义是因为那些视图中的数据保持静态，除非数据库中发生事务进行更改。例如，如果你反复查询 `DBA_TABLES`，它将返回完全相同的信息。如果有人创建了一个新表，该事务将导致 `DBA_TABLES` 返回的数据发生变化。在更改之后查询 `DBA_TABLES`，在有人删除或创建另一个表之前，都将返回相同的数据。



### DBA_、ALL_、USER_ 与 CDB_

大多数静态数据字典视图以 `DBA_`、`ALL_` 或 `USER_` 开头。其后跟随的部分通常在三个视图中都可见。我们已经讨论过 `DBA_TABLES`，但也有 `ALL_TABLES` 和 `USER_TABLES` 视图。这三个视图都允许我们查询数据字典以获取有关表的信息。不同之处在于我们可以从这些视图中看到的信息范围，如下所列：

*   `USER_`：这些视图用于我们拥有的对象。
*   `ALL_`：这些视图用于我们拥有且有权限访问的对象。
*   `DBA_`：这些视图用于整个数据库。
*   `CDB_`：这些视图用于 Oracle 多租户。`CDB_`视图显示与其对应的`DBA_`视图相同的信息，但范围是所有可插拔数据库。

从以上列表，我们可以正确推断出：`USER_TABLES` 显示我们拥有的表；`ALL_TABLES` 显示我们拥有且有权限访问的表；`DBA_TABLES` 显示整个数据库中的所有表。在清单 [14-4] 中，我们可以看到每个视图显示了我们在这些类别中可以看到的表的数量。

```sql
SQL> select count(*) From dba_tables;
COUNT(*)

SQL> select count(*) from all_tables;
COUNT(*)

SQL> select count(*) from user_tables;
COUNT(*)

Listing 14-4
Table Counts
```

发出上述 SQL 语句的用户只拥有一个表。他们可以访问 23,048 个表，整个数据库中有 23,048 个表。`ALL_TABLES` 和 `DBA_TABLES` 显示相同数量表的原因是因为发出清单 [14-4] 中查询的用户是 `DBA` 用户。他们可以访问数据库中的所有表。考虑清单 [14-5] 中那个不是 `DBA` 用户的用户。

```sql
SQL> select count(*) From dba_tables;
select count(*) From dba_tables
*
ERROR at line 1:
ORA-00942: table or view does not exist
SQL> select count(*) from all_tables;
COUNT(*)

SQL> select count(*) from user_tables;
COUNT(*)

Listing 14-5
Table Counts Non-DBA User
```

在上面的代码中，该用户拥有 5 个表并可以访问 4,360 个表。对 `DBA_TABLES` 的查询返回错误，因为该用户无权访问该视图。

需要注意的是，`USER_` 视图不包含名为 `OWNER` 的列。可以理解为，这些视图的输出归查询该视图的用户所有。`ALL_` 和 `DBA_` 表通常有一个 `OWNER` 列（在适用的情况下）。

作为一名 `DBA`，我经常查询 `DBA_` 视图。非 `DBA` 用户很少查询 `DBA_` 视图。我从不查询 `ALL_` 视图，因为作为 `DBA`，它们通常返回与相应 `DBA_` 视图相同的信息。

由于本书是面向数据库管理员的，代码将使用 `DBA_` 视图。如果你在*数据库参考手册*中查找 `DBA_` 视图，它很可能会引导你转到相应的 `ALL_` 视图。这不是问题，因为 `DBA_` 和 `ALL_` 视图显示相同的内容，只是范围不同。唯一的区别是返回哪些行，这取决于查询视图的用户。

### DICT

有大量的 `DBA_` 视图可供查询。多到可能很难找到你需要的那一个。每个新版本都会带来更多静态数据字典视图。在清单 [14-6] 中，我们可以看到一个 Oracle 12.1.0.2 数据库中 `DBA_` 视图总数的计数。

```sql
SQL> select count(*) From dba_objects where object_name like 'DBA_%' and object_type="VIEW";
COUNT(*)

Listing 14-6
Oracle 12.1.0.2 DBA View Count
```

在 Oracle 12.2.0.1 中，有 2,022 个 `DBA_` 视图。那么我们如何找到能提供所需信息的视图呢？值得庆幸的是，Oracle 创建了一个数据字典视图，名为 `DICT`（Dictionary 的缩写），它为我们提供数据字典视图的描述。当我很难记住哪个视图包含我想要的信息时，我会查询 `DICT` 来看看它提供了什么。`DICT` 有两列：表名和注释。“表”这个词是错误的，因为它很可能指的是一个视图，而不是真正的表。

我经常在搜索中使用通配符，因为我可能不知道确切的名称。例如，如果我想知道哪些视图以 `DBA` 开头并包含单词 `INDEX`，我会发出类似清单 [14-7] 的查询。

```sql
SQL> select * from dict where upper(table_name) like 'DBA%INDEX%';
TABLE_NAME                COMMENTS
------------------------- ---------------------------------------------
DBA_INDEXES               Description for all indexes in the database
DBA_INDEXTYPES            All indextypes
DBA_INDEXTYPE_ARRAYTYPES  All array types specified by the indextype
DBA_INDEXTYPE_COMMENTS    Comments for user-defined indextypes
DBA_INDEXTYPE_OPERATORS   All indextype operators
DBA_PART_INDEXES
DBA_XML_INDEXES           Description of all XML indexes in the database
7 rows selected.
Listing 14-7
Index Views
```

希望这七行中的一行包含我感兴趣的数据字典视图。我也可能查询 `COMMENTS` 列。在清单 [14-8] 的示例中，查询 `TABLE_NAME` 列没有给我描述物化视图的视图列表，但查询 `COMMENTS` 列却可以。

```sql
SQL> select * from dict where upper(table_name) like '%MATERIALIZED%';
no rows selected
SQL> select * from dict where upper(comments) like '%MATERIALIZED%';
TABLE_NAME             COMMENTS
---------------------- --------------------------------------------------
DBA_BASE_TABLE_MVIEWS  All materialized views with log(s) in the database
DBA_MVIEWS             All materialized views in the database
Listing 14-8
Materialized Views
```

我可以看到 `DBA_MVIEWS` 包含数据库中物化视图的信息。我也可以明白为什么在查询表名时没有找到它。上面的输出为了简洁而被截断了，因为在我的数据库中它返回了 42 行。

查询 `DICT` 是了解有哪些数据字典视图可用的好方法。另一种方法是阅读*数据库参考手册*。



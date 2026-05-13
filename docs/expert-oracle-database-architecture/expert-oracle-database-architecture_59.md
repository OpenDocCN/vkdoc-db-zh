# 临时表的适用场景

然而，在其他情况下，在一个流程中使用临时表是正确的方法。例如，我曾经编写过一个 Palm 同步应用程序，用于将 Palm Pilot 上的日程本与存储在 Oracle 中的 Calendar 信息进行同步。Palm 提供了一个自上次热同步以来所有被修改过的记录的列表。我必须获取这些记录，将它们与数据库中的实时数据进行比较，更新数据库记录，然后生成一个需要应用到 Palm 的变更列表。这是一个临时表非常有用且完美的示例。我使用一个临时表来在数据库中存储来自 Palm 的变更。然后，我运行一个存储过程，将 Palm 生成的变更与实时的（且非常庞大的）永久表进行比对，以发现需要对 Oracle 数据做的更改，然后找出需要从 Oracle 传回 Palm 的变更。我需要对这些数据进行几次遍历。首先，我找出所有仅在 Palm 上修改过的记录，并在 Oracle 中进行相应的更改。接下来，我找出自上次同步以来在 Palm 和我的数据库上都被修改过的所有记录，并进行调和。然后，我找出所有仅在数据库上修改过的记录，并将它们的变更放入临时表中。最后，Palm 同步应用程序从临时表中提取变更并应用到 Palm 设备本身。断开连接后，临时数据就会消失。



然而，我遇到的问题是，由于永久表已被分析，成本基优化器（CBO）被启用。临时表上没有任何统计信息（你可以分析临时表，但不会收集到统计信息），因此 CBO 会对其做出许多猜测。作为开发人员，我知道预期的平均行数、数据分布、索引的选择性等等。我需要一种方法将这些*更优的*猜测告知优化器。这通过为临时表生成统计信息来实现。这就引出了下一个主题：如何为临时表生成统计信息。

### 全局临时表统计信息

全局临时表统计信息的收集与使用方式如下：

*   默认情况下，为临时表收集统计信息时，生成的是会话级统计信息。
*   仍然可以收集共享统计信息，但必须先将 `DBMS_STATS.SET_TABLE_PREFS` 过程的 `GLOBAL_TEMP_TABLE_STATS` 参数设置为 `SHARED`。
*   对于定义为 `ON COMMIT DELETE ROWS` 的临时表，一些 `DBMS_STATS` 过程（如 `GATHER_TABLE_STATS`）不再发出隐式的 `COMMIT`；因此，可以为这类临时表生成具有代表性的统计信息。
*   对于定义为 `ON COMMIT PRESERVE ROWS` 的临时表，会为直接路径操作（如 CTAS 和直接路径 `INSERT` 语句）自动生成会话级统计信息；这消除了为这些特定操作调用 `DBMS_STATS` 来生成统计信息的需要。

我们将详细查看以上每一点，首先从会话统计信息开始。

#### 会话统计信息

当你为临时表生成统计信息时，这些统计信息仅对生成它的会话可见。这为 Oracle 优化器提供了更好的信息，以创建为每个会话生成的数据量身定制的执行计划。一个小例子将演示这一点；首先，创建一个临时表：

```
$ sqlplus eoda/foo@PDB1
SQL> create global temporary table gt(x number) on commit preserve rows;
Table created.
```

接下来，插入一些数据：

```
SQL> insert into gt select user_id from all_users;
51 rows created.
```

现在为该表生成统计信息：

```
SQL> exec dbms_stats.gather_table_stats( user, 'GT' );
PL/SQL procedure successfully completed.
```

我们可以通过查询 `USER_TAB_STATISTICS` 来验证会话级统计信息的存在：

```
SQL> select table_name, num_rows, last_analyzed, scope
from user_tab_statistics
where table_name like 'GT';
TABLE_NAME                  NUM_ROWS LAST_ANALYZED  SCOPE
------------------------- ---------- -------------- -------
GT                                                  SHARED
GT                                51 27-JUN-21      SESSION
```

我们可以通过 autotrace 进一步验证优化器对会话私有统计信息的感知：

```
SQL> set autotrace on;
SQL> select count(*) from gt;
```

输出底部附近有这样一条优化器注释：

```
Note
- Global temporary table session private statistics used
```

请记住，会话级统计信息仅在会话持续期间有效。如果你断开连接并重新连接，统计信息将消失：

```
SQL> disconnect
SQL> connect eoda/foo@PDB1
```

重新运行显示统计信息存在的查询，表明现在不存在任何会话统计信息：

```
SQL> select table_name, num_rows, last_analyzed, scope
from user_tab_statistics
where table_name like 'GT';
TABLE_NAME             NUM_ROWS LAST_ANALYZED  SCOPE
-------------------- ---------- -------------- -------
GT                                             SHARED
```

注意
 查询临时表时，如果存在会话级统计信息，优化器将使用它们。如果不存在会话级统计信息，则优化器会检查是否存在共享统计信息，如果存在，则使用它们。如果没有任何统计信息，优化器将使用动态统计信息（在 12c 之前，这被称为*动态采样*）。

#### 共享统计信息

如前一节所示，当你为临时表生成统计信息时，这些统计信息仅对生成它的会话可见。如果你需要多个会话共享同一临时表的相同统计信息，必须先使用 `DBMS_STATS.SET_TABLE_STATS` 过程将 `GLOBAL_TEMP_TABLE_STATS` 首选项设置为 `SHARED`（该首选项的默认值为 `SESSION`）。为了演示这一点，让我们创建一个临时表并插入一些数据：

```
SQL> create global temporary table gt(x number) on commit preserve rows;
Table created.
SQL> insert into gt select user_id from all_users;
51 rows created.
```

现在将 `GLOBAL_TEMP_TABLE_STATS` 首选项设置为 `SHARED`：

```
SQL> exec dbms_stats.set_table_prefs(user, -
> 'GT','GLOBAL_TEMP_TABLE_STATS','SHARED');
```

接下来，为临时表生成统计信息：

```
SQL> exec dbms_stats.gather_table_stats( user, 'GT' );
```

我们可以通过执行以下查询来验证是否已生成共享统计信息：

```
SQL> select table_name, num_rows, last_analyzed, scope
from user_tab_statistics
where table_name like 'GT';
TABLE_NAME        NUM_ROWS LAST_ANALYZED   SCOPE
--------------- ---------- --------------- -------
GT                      51 27-JUN-21       SHARED
```

全局临时表的共享统计信息会一直保留，直到被显式移除。你可以按如下方式删除共享统计信息：

```
SQL> exec dbms_stats.delete_table_stats( user, 'GT' );
```

我们可以通过运行以下查询来验证共享统计信息是否已被移除：

```
SQL> select table_name, num_rows, last_analyzed, scope
from user_tab_statistics
where table_name like 'GT';
TABLE_NAME             NUM_ROWS LAST_ANALYZED   SCOPE
-------------------- ---------- --------------- -------
GT                                              SHARED
```



## 关于“提交时删除行”的统计信息

如前所述，在运行诸如 `GATHER_TABLE_STATS` 之类的存储过程时，会发生一个隐式的 `COMMIT` 操作。因此，在为定义为 `ON COMMIT DELETE ROWS` 的临时表生成统计信息时，收集到的统计信息反映的是零行数据的表的情况（这种统计信息是无用的，因为你需要的统计信息应反映临时表在被 `COMMIT` 删除之前的数据状态）。

`DBMS_STATS` 中的几个存储过程（例如 `GATHER_TABLE_STATS`）在为定义为 `ON COMMIT DELETE ROWS` 的临时表收集统计信息后，**不会**发出隐式的 `COMMIT`。这意味着可以为这种类型的临时表收集有代表性的统计信息。下面通过一个简单示例来说明；首先，创建一个带有 `ON COMMIT DELETE ROWS` 的临时表：

```sql
SQL> create global temporary table gt(x number) on commit delete rows;
Table created.
```

接下来，插入一些数据：

```sql
SQL> insert into gt select user_id from all_users;
51 rows created.
```

现在，为该用户生成统计信息：

```sql
SQL> exec dbms_stats.gather_table_stats( user, 'GT' );
PL/SQL procedure successfully completed.
```

快速计数可以验证行数据仍然存在于 `GT` 表中：

```sql
SQL> select count(*) from gt;
COUNT(*)
----------
```

我们可以通过查询 `USER_TAB_STATISTICS` 来验证会话级统计信息的存在：

```sql
SQL> select table_name, num_rows, last_analyzed, scope
  2  from user_tab_statistics
  3  where table_name like 'GT';
TABLE_NAME                  NUM_ROWS LAST_ANALYZED  SCOPE
------------------------- ---------- -------------- -------
GT                                                  SHARED
GT                                51 27-JUN-21      SESSION
```

这使得你可以为希望在每次事务后删除行的临时表生成有用的统计信息。

**注意**
`DBMS_STATS` 的以下存储过程在为使用 `ON COMMIT DELETE ROWS` 创建的表收集临时表统计信息时，**不会**发出 `COMMIT`：`GATHER_TABLE_STATS`、`DELETE_TABLE_STATS`、`DELETE_COLUMN_STATS`、`DELETE_INDEX_STATS`、`SET_TABLE_STATS`、`SET_COLUMN_STATS`、`SET_INDEX_STATS`、`GET_TABLE_STATS`、`GET_COLUMN_STATS`、`GET_INDEX_STATS`。而之前的这些存储过程，对于定义为 `ON COMMIT PRESERVE ROWS` 的临时表，则**会**发出隐式的 `COMMIT`。

## 直接路径加载自动统计信息收集

当在临时表上执行直接路径操作（启用了 `ON COMMIT PRESERVE ROWS`）时，默认情况下会为正在加载的临时表收集会话级统计信息。两个典型的直接路径加载操作是 `CREATE TABLE AS SELECT` (CTAS) 和直接路径 `INSERT`（使用了 `/*+ append */` 提示的 `INSERT`）。

下面通过一个简单示例来说明。这里，我们创建一个 CTAS 表：

```sql
SQL> create global temporary table gt on commit preserve rows
  2  as select * from all_users;
Table created.
```

我们可以通过以下查询验证会话级统计信息是否已生成：

```sql
SQL> select table_name, num_rows, last_analyzed, scope
  2  from user_tab_statistics
  3  where table_name like 'GT';
TABLE_NAME   NUM_ROWS LAST_ANALYZED  SCOPE
---------- ---------- -------------- -------
GT                                   SHARED
GT                 51 27-JUN-21      SESSION
```

对于定义为 `ON COMMIT PRESERVE ROWS` 的临时表，当进行直接路径加载时，这就消除了调用 `DBMS_STATS` 来生成统计信息的需要。

## 私有临时表

Oracle 18c 引入了私有临时表。对于私有临时表，对象及其数据会在会话或事务结束时被删除。私有临时表仅存在于内存中，因此只对创建它的会话可见。

私有临时表的创建必须使用由 `PRIVATE_TEMP_TABLE_PREFIX` 初始化参数定义的前缀。该参数的默认值是 `ORA$PTT`。你可以通过以下方式查看该参数的定义：

```sql
SQL> show parameter PRIVATE_TEMP_TABLE_PREFIX
NAME                                 TYPE        VALUE
------------------------------------ ----------- -------------------------
private_temp_table_prefix            string      ORA$PTT_
```

创建私有临时表的基本语法如下：

```sql
CREATE PRIVATE TEMPORARY TABLE (
    column definition,
    column definition,
    ...
) ON COMMIT [DROP DEFINITION | PRESERVE DEFINITION];
```

`ON COMMIT` 子句用于定义私有临时表是仅持续一个事务，还是持续整个会话连接。换句话说，如果你的需求是在发出 `COMMIT` 或 `ROLLBACK` 语句后删除表，请使用 `ON COMMIT DROP DEFINITION`。使用 `ON COMMIT PRESERVE DEFINITION` 则使表在会话连接期间持续存在。

第一个示例演示创建一个仅持续当前事务的私有临时表：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create private temporary table ora$ptt_temp1(a int) on commit drop definition;
Table created.
```

现在我将插入一些数据并从表中查询：

```sql
SQL> insert into ora$ptt_temp1 values(1);
SQL> select * from ora$ptt_temp1;
         A
----------
```

现在，如果我发出 `COMMIT` 或 `ROLLBACK`（从而结束事务），该表将被自动删除：

```sql
SQL> commit;
SQL> desc ora$ptt_temp1;
ERROR:
ORA-04043: object ora$ptt_temp1 does not exist
```

下一个示例创建一个持续整个会话的私有临时表：

```sql
SQL> create private temporary table ora$ptt_temp2(a int) on commit preserve definition;
SQL> insert into ora$ptt_temp2 values(1);
SQL> commit;
SQL> select * from ora$ptt_temp2;
         A
----------
```

现在，如果我断开连接并重新连接，该表将被自动删除：

```sql
SQL> disconnect;
SQL> connect eoda/foo@PDB1
SQL> select * from ora$ptt_temp2;
select * from ora$ptt_temp2
*
ERROR at line 1:
ORA-00942: table or view does not exist
```

何时应该使用全局临时表而非私有临时表？表 10-3 对比了这些临时表的特性。

**表 10-3 全局临时表与私有临时表的特性对比**

| 特性 | 全局临时表 | 私有临时表 |
| --- | --- | --- |
| 命名规则 | 任意名称 | 前缀 (`ORA$PTT`) |
| 表可见性 | 可从其他会话可见 | 仅对创建表的会话可见 |
| 数据可见性 | 仅对创建数据的会话可见 | 仅对创建数据的会话可见 |
| 表定义持久性 | 是，表定义存储在数据字典中 | 否，表定义仅在事务或会话期间存在于内存中 |
| 数据持久性（事务或会话结束后） | 否 | 否 |
| 可在列上创建索引 | 是 | 否 |
| 可生成优化器统计信息 | 是 | 否 |

如果你需要一个跨连接持久的临时表，或者需要它被多个会话可见，那么请使用全局临时表。此外，查询全局临时表的性能可以通过索引和优化器统计信息来提升，而你无法为私有临时表创建索引或生成统计信息。



### 临时表小结

Oracle 提供两种类型的临时表：全局临时表和私有临时表。这两种临时表在应用中都很有用，当你需要为某个会话或事务临时存储一组行数据以便与其他表进行处理时，它们并非旨在用于将单个大型查询拆分为更小的结果集再合并回来（这似乎是其他数据库中最常见的临时表用法）。事实上，你会发现，在几乎所有情况下，将一个查询拆分成多个对临时表的查询在 Oracle 中执行速度都比执行单个查询要慢。我一次又一次地看到这种行为；当有机会将一系列向临时表的 `INSERT` 操作重写为一个大型 `SELECT` 查询时，最终得到的单个查询执行速度比原来的多步过程要快得多。

临时表生成的重做日志量极少，但它们仍然会生成一些重做日志。在 12c 版本之前，没有办法禁用此功能。重做日志是为回滚数据生成的，在大多数典型使用场景中，其量级可以忽略不计。如果你只是对临时表执行 `INSERT` 和 `SELECT` 操作，生成的重做日志量将不会明显。只有当你对临时表进行大量 `DELETE` 或 `UPDATE` 操作时，才会看到大量重做日志生成。

注意：你可以指示 Oracle 将撤销数据写入临时表空间，从而几乎消除所有重做日志的生成。这可以通过将 `TEMP_UNDO_ENABLED` 参数设置为 `TRUE` 来实现（详见章节）。

成本优化器（CBO）使用的统计信息可以小心地在全局临时表上生成；然而，更好的做法是使用 `DBMS_STATS` 包在临时表上设置一组统计信息，或者通过优化器在硬解析时使用动态采样来动态收集。从 Oracle 12c 开始，你可以生成特定于会话的统计信息（针对全局临时表）。这为优化器提供了更好的信息，以便为给定会话中加载的数据生成更优的执行计划。

## 对象表

我们已经通过嵌套表看到了对象表的部分示例。*对象表*是基于 `TYPE` 创建的表，而不是作为列的集合。通常，`CREATE TABLE` 语句如下所示：

```sql
create table t ( x int, y date, z varchar2(25) );
```

对象表的创建语句更像这样：

```sql
create table t of Some_Type;
```

`T` 的属性（列）源自 `SOME_TYPE` 的定义。让我们快速看一个涉及几个类型的例子，然后回顾生成的数据结构：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create or replace type address_type
as object
( city    varchar2(30),
street  varchar2(30),
state   varchar2(2),
zip     number)
/
Type created.
SQL> create or replace type person_type
as object
( name             varchar2(30),
dob              date,
home_address     address_type,
work_address     address_type)
/
Type created.
SQL> create table people of person_type
/
Table created.
SQL> desc people
Name                                     Null?    Type
---------------------------------------- -------- ------------------------
NAME                                              VARCHAR2(30)
DOB                                               DATE
HOME_ADDRESS                                      ADDRESS_TYPE
WORK_ADDRESS                                      ADDRESS_TYPE
```

简而言之，这就是全部内容。我们创建一些类型定义，然后就可以创建这些类型的表。该表看起来有四列，分别代表我们创建的 `PERSON_TYPE` 的四个属性。我们现在可以对该对象表执行 DML 操作以创建和查询数据了：

```sql
SQL> insert into people values ( 'Tom', '15-mar-1965',
address_type( 'Denver', '123 Main Street', 'Co', '12345' ),
address_type( 'Redwood', '1 Oracle Way', 'Ca', '23456' ) );
1 row created.
SQL> select name, dob, p.home_address Home, p.work_address work
from people p;
Tom                            15-MAR-65
ADDRESS_TYPE('Denver', '123 Main Street', 'Co', 12345)
ADDRESS_TYPE('Redwood', '1 Oracle Way', 'Ca', 23456)
SQL> select name, p.home_address.city from people p;
NAME                           HOME_ADDRESS.CITY
------------------------------ ------------------------------
Tom                            Denver
```

我们开始看到一些处理对象类型所必需的对象语法。例如，在 `INSERT` 语句中，我们必须用某种方式“包装” `HOME_ADDRESS` 和 `WORK_ADDRESS` 值，因为它们不是标量值，而是对象。我们通过使用 `ADDRESS_TYPE` 对象的默认构造函数来创建该行的 `ADDRESS_TYPE` 实例。

现在，就表的外部表现而言，我们的表有四列。到目前为止，在了解了嵌套表背后隐藏的机制后，我们大概能猜到这里还有其他情况。Oracle 将所有对象关系数据存储在普通的关系表中——归根结底，数据都是以行和列的形式存储的。如果我们深入查看真实的数据字典，就能看到这个表的真实样貌：

```sql
SQL> select name, segcollength
from sys.col$
where obj# = ( select object_id
from user_objects
where object_name = 'PEOPLE' )
/
NAME            SEGCOLLENGTH
--------------- ------------
SYS_NC_OID$               16
SYS_NC_ROWINFO$            1
NAME                      30
DOB                        7
HOME_ADDRESS               1
SYS_NC00006$              30
SYS_NC00007$              30
SYS_NC00008$               2
SYS_NC00009$              22
WORK_ADDRESS               1
SYS_NC00011$              30
SYS_NC00012$              30
SYS_NC00013$               2
SYS_NC00014$              22
14 rows selected.
```

这看起来与 `DESCRIBE` 告诉我们的信息大相径庭。显然，这个表有 14 列，而不是 4 列。在这些列中，它们是...



*   `SYS_NC_OID$`：这是表的系统生成对象 ID。它是一个唯一的 `RAW(16)` 列。它具有唯一约束，并且也为其创建了对应的唯一索引。

*   `SYS_NC_ROWINFO$`：这是我们在嵌套表中观察到的相同神奇函数。如果我们从表中选择它，它会将整行作为单个列返回：

*   `NAME`，`DOB`：这些是我们对象表的标量属性。它们的存储方式与我们预期的一致，即作为常规列。

*   `HOME_ADDRESS`，`WORK_ADDRESS`：这些也是神奇函数。它们将所代表的列集合返回为单个对象。除了表示实体的 `NULL` 或 `NOT NULL` 外，它们不消耗实际存储空间。

*   `SYS_NCnnnnn$`：这些是我们嵌入对象类型的标量实现。由于 `PERSON_TYPE` 中嵌入了 `ADDRESS_TYPE`，Oracle 需要在适当类型的列中为它们腾出空间存储。系统生成的名称是必要的，因为列名必须是唯一的，而且我们无法阻止像这里一样多次使用相同的对象类型。如果名称不是生成的，我们最终会得到两个 `ZIP` 列。

```
SQL> select sys_nc_rowinfo$ from people;
SYS_NC_ROWINFO$(NAME, DOB, HOME_ADDRESS(CITY, STREET, STATE, ZIP),
WORK_ADDRESS(CITY, STREET, STATE,

PERSON_TYPE('Tom', '15-MAR-65', ADDRESS_TYPE('Denver', '123 Main Street', 'Co', 12345), ADDRESS_TYPE('Redwood', '1 Oracle Way', 'Ca', 23456))
```

因此，就像嵌套表一样，这里发生了很多事情。添加了一个 16 字节的伪主键，存在虚拟列，并且为我们创建了一个索引。我们可以更改关于分配给对象的对象标识符值的默认行为，稍后将看到。首先，让我们看看生成我们的表的完整详细 SQL。这是使用 Data Pump 生成的，因为我希望轻松查看相关对象，包括重新创建此特定对象实例所需的所有 SQL。这是通过以下方式实现的：

```
$ expdp eoda directory=tk tables='PEOPLE' dumpfile=p.dmp logfile=p.log
$ impdp eoda directory=tk dumpfile=p.dmp logfile=pi.log sqlfile=people.sql
Master table "EODA"."SYS_SQL_FILE_FULL_01" successfully loaded/unloaded
Starting "EODA"."SYS_SQL_FILE_FULL_01":
eoda/******** directory=tk dumpfile=p.dmp logfile=pi.log sqlfile=people.sql
```

对生成的 `people.sql` 文件的审阅会显示如下内容：

```
-- new object type path: TABLE_EXPORT/TABLE/TABLE
CREATE TABLE "EODA"."PEOPLE" OF "EODA"."PERSON_TYPE"
OID 'F0484A73A93A7093E043B7D04F0A821B'
OIDINDEX  ( PCTFREE 10 INITRANS 2 MAXTRANS 255
STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
TABLESPACE "USERS" )
PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255
NOCOMPRESS LOGGING
STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
TABLESPACE "USERS" ;
```

这让我们更深入地了解了实际发生的情况。我们现在可以清楚地看到 `OIDINDEX` 子句，并且看到对 OID 列的引用，后跟一个十六进制数字。

`OID '<big hex number>'` 语法未在 Oracle 文档中记录。所有这一切都是为了确保在 `expdp` 和随后的 `impdp` 期间，底层类型 `PERSON_TYPE` 实际上是*相同*的类型。这将防止如果我们执行以下步骤时发生的错误：

1.  创建 `PEOPLE` 表。
2.  导出该表。
3.  删除该表和底层的 `PERSON_TYPE`。
4.  创建一个具有不同属性的新 `PERSON_TYPE`。
5.  导入旧的 `PEOPLE` 数据。

显然，此导出无法导入到新结构中——它将不匹配。此检查可防止这种情况发生。

如果您还记得，我提到过我们可以更改分配给对象实例的对象标识符的行为。我们可以使用对象的自然键，而不是让系统为我们生成伪主键。起初，这可能显得自相矛盾——`SYS_NC_OID$` 列仍会出现在 `SYS.COL$` 中的表定义中，并且事实上，与系统生成的列相比，它似乎消耗了大量存储空间。然而，这里再次出现了魔法。对于基于*主键*而非*系统*生成的对象表，`SYS_NC_OID$` 列是一个虚拟列，不消耗磁盘上的实际存储空间。

下面是一个示例，展示了数据字典中发生的情况，并证明 `SYS_NC_OID$` 列没有消耗物理存储。我们将从分析系统生成的 `OID` 表开始：

```
SQL> create table people of person_type
/
Table created.
SQL> select name, type#, segcollength
from sys.col$
where obj# = ( select object_id
from user_objects
where object_name = 'PEOPLE' )
and name like 'SYS\_NC\_%' escape '\'
/
NAME                      TYPE# SEGCOLLENGTH
-------------------- ---------- ------------
SYS_NC_OID$                  23           16
SYS_NC_ROWINFO$             121            1
SQL> insert into people(name) select rownum from all_objects;
72069 rows created.
SQL> exec dbms_stats.gather_table_stats( user, 'PEOPLE' );
PL/SQL procedure successfully completed.
SQL> select table_name, avg_row_len from user_object_tables;
TABLE_NAME           AVG_ROW_LEN
-------------------- -----------
PEOPLE                        24
```

我们在这里看到，平均行长度为 24 字节：`SYS_NC_OID$` 列 16 字节，`NAME` 列 8 字节。现在，让我们做同样的事情，但使用 `NAME` 列上的主键作为对象标识符：

```
SQL> CREATE TABLE "PEOPLE"
OF "PERSON_TYPE"
( constraint people_pk primary key(name) )
object identifier is PRIMARY KEY
/
Table created.
SQL> select name, type#, segcollength
from sys.col$
where obj# = ( select object_id
from user_objects
where object_name = 'PEOPLE' )
and name like 'SYS\_NC\_%' escape '\'
/
NAME                                TYPE# SEGCOLLENGTH
------------------------------ ---------- ------------
SYS_NC_OID$                            23           81
SYS_NC_ROWINFO$                       121            1
```

根据这个，我们得到的是一个大的 81 字节列，而不是一个小的 16 字节列！实际上，那里没有存储任何数据。它将为空。系统将根据对象表、其底层类型以及行本身的值生成一个唯一的 ID。我们可以在以下内容中看到这一点：

```
SQL> insert into people (name) values ( 'Hello World!' );
1 row created.
SQL> select sys_nc_oid$ from people p;
SYS_NC_OID$

F04931FE974478A7E043B7D04F0A082000000017260100010001002900000000000C07001E0100002A00078401FE00000014
0C48656C6C6F20576F726C6421000000000000000000000000000000000000
SQL> select utl_raw.cast_to_raw( 'Hello World!' ) data from dual;
DATA

48656C6C6F20576F726C6421
SQL> select utl_raw.cast_to_varchar2(sys_nc_oid$) data from people;
DATA

Hello World!
```

如果我们选择 `SYS_NC_OID$` 列并检查我们插入字符串的 `HEX` 转储，我们会看到行数据本身被嵌入到对象 ID 中。将对象 ID 转换为 `VARCHAR2` 后，我们可以直观地确认这一点。这是否意味着我们的数据被存储了两次并附带大量开销？不，并非如此——它只是在被检索时被纳入了 `SYS_NC_OID$` 列这个魔法之中。Oracle 在从表中选择时合成数据。


现在来谈谈我的观点。对象关系组件（嵌套表和对象表）主要是我称之为`语法糖`的东西。它们最终总是会被转换成传统的关系型行和列。就我个人而言，我更倾向于不将它们用作物理存储机制。其中发生了太多“魔法”——不明确的副作用。你会遇到隐藏列、额外索引、意外的伪列等等。`这并不意味着对象关系组件是浪费时间。`恰恰相反，我经常在 PL/SQL 中使用它们。我将它们与对象视图结合使用。我可以获得嵌套表结构的好处（例如，在主从关系中减少通过网络返回的数据量，概念上更易于处理等等），而无需担心任何物理存储问题。这是因为我可以使用对象视图，从我的关系型数据中合成对象。这解决了我对对象表/嵌套表的大部分担忧：物理存储由我决定，连接条件由我设置，并且表可以自然地作为关系型表使用（而这正是许多第三方工具和应用程序所要求的）。需要关系型数据对象视图的人可以得到它，需要关系型视图的人也可以得到它。由于对象表本质上是伪装的关系型表，我们实际上是在做 Oracle 在幕后为我们做的事情，只不过我们可以做得更高效，因为我们不必像 Oracle 那样进行通用处理。例如，使用之前定义的类型，我可以轻松地使用以下方式：

```sql
SQL> create table people_tab
(  name        varchar2(30) primary key,
dob         date,
home_city   varchar2(30),
home_street varchar2(30),
home_state  varchar2(2),
home_zip    number,
work_city   varchar2(30),
work_street varchar2(30),
work_state  varchar2(2),
work_zip    number
)
/
Table created.
SQL> create view people of person_type
with object identifier (name)
as
select name, dob,
address_type(home_city,home_street,home_state,home_zip) home_adress,
address_type(work_city,work_street,work_state,work_zip) work_adress
from people_tab
/
View created.
SQL> insert into people values ( 'Tom', '15-mar-1965',
address_type( 'Denver', '123 Main Street', 'Co', '12345' ),
address_type( 'Redwood', '1 Oracle Way', 'Ca', '23456' ) );
1 row created.
```

然而，我实现了几乎相同的效果；我确切地知道存储了什么、如何存储以及存储在哪里。对于更复杂的对象，我们可能需要在对象视图上编写`INSTEAD OF`触发器，以允许通过视图进行修改。

### 对象表总结

对象表用于在 Oracle 中实现对象关系模型。一个单独的对象表通常会创建许多物理数据库对象，并在您的模式中添加额外的列来管理所有内容。对象表涉及一定程度的“魔法”。对象视图则让您既能利用对象的语法和语义，又能完全控制数据的物理存储，并允许对底层数据进行关系型访问。这样，您就可以兼得关系型和对象关系型世界的优点。

## 区块链表

Oracle 包含多项安全功能来保护您的数据安全。您可能已经熟悉其中的大部分功能，例如密码、数据库角色、权限、加密功能和透明数据加密。然而，这些功能都无法阻止拥有表访问权限的人对数据进行有害的更改。例如，假设一位心怀不满但拥有`PAYROLL`表合法访问权限的经理，决定在该表中进行未经授权的加薪更新。或者一位拥有 SYS 权限并能修改数据库中任何表的恶意 DBA，决定截断`LEDGER`表。另一个威胁来源是犯罪分子，他们通过网络钓鱼攻击获得了数据库的访问权限，现在可以访问公司的财务数据。这些只是拥有数据库访问权限的人可能以对公司有害的方式修改数据库的几个例子。

Oracle 区块链技术可检测并防止对表数据进行非法的插入、更新和删除操作。区块链表是一种不可变的表，只能向其中插入数据。一旦插入，数据便无法修改。删除操作仅允许根据对已插入数据的保留策略执行。其思路是，您创建一行初始数据，然后如果需要更改该行中的数据，您需要插入一个新行，该新行链接到原始行。这样，您就拥有了数据如何变化的历史记录。什么样的应用程序会使用区块链表？当您需要为当前数据记录维护一个防篡改的账本，并为该记录保留一份不可变的交易历史时，这项技术就非常适用。

区块链技术被透明地集成到 Oracle 数据库中（从版本 19.10 开始）。它适用于 Oracle 的所有版本。创建和使用区块链表不需要额外的许可证或选件。在选择和查看数据方面，区块链表对应用程序而言看起来就像普通表。然而，区块链表与普通表的不同之处在于，它们是不可变的、仅可插入的表。在数据插入区块链表后，对其进行删除和更新操作是受到限制的。

您通过`CREATE BLOCKCHAIN TABLE`子句来指定何时可以删除数据的行为。在创建区块链表时，必须指定三个子句：

*   区块链删除表子句
*   区块链行保留子句
*   区块链哈希和数据格式子句

前面的子句控制数据在插入后的保护方式。以下各小节将详细说明这些子句。

### 区块链删除表子句

区块链删除表子句决定了何时可以删除该表。删除表子句的语法如下：

```sql
NO DROP [ UNTIL NUMBER DAYS IDLE ]
```

您可以通过两种方式指定此子句：

*   不带任何其他说明的`NO DROP`表示该区块链表永远不能被删除。以这种方式指定的区块链表，唯一的删除方式是删除拥有该表的模式。
*   `NO DROP UNTIL NUMBER DAYS IDLE`表示，直到最新插入的行存在时间超过`NUMBER DAYS IDLE`天数后，该区块链表才能被删除。

如果您指定了`NO DROP UNTIL 0 DAYS IDLE`，那么您可以在任何时候删除该表（我将在接下来的一个示例中使用此设置）。如果您在测试阶段，并且需要在将其投入使用前能够删除表，您可能需要这样做。

### 区块链行保留子句

区块链行保留子句控制何时可以从表中删除行。区块链行保留子句的语法如下：

```sql
NO DELETE { [ LOCKED ] | (UNTIL NUMBER DAYS AFTER INSERT [ LOCKED ]) }
```

如果您指定`NO DELETE`或`NO DELETE LOCKED`（没有`UNTIL`子句），那么您永远无法删除、更新或截断插入到此表中的任何行。删除数据的唯一方法是删除该表或删除拥有该表的用户。只有当该表不活跃的时间超过区块链删除表子句中指定的天数时，该表才能被删除。

`UNTIL NUMBER DAYS AFTER INSERT`值指定了行被插入后，经过多少天可以被删除。允许的最小值为 16。

如果您指定了`LOCKED`，那么您将无法使用`ALTER TABLE`命令更改`NUMBER DAYS`的保留期。如果您在`UNTIL NUMBER DAYS AFTER INSERT`子句中没有指定`LOCKED`，那么您可以使用`ALTER TABLE`命令更改保留期（但只能更改为高于先前保留期的值）。


### 区块链哈希与数据格式子句

此子句指定了为每个新插入行生成哈希签名时所使用的加密函数。该子句的语法如下：

```plaintext
HASHING USING sha2_512 VERSION v1
```

你必须将此子句指定在 `CREATE BLOCKCHAIN TABLE` 语句的最后（例如，必须放在所有其他子句之后）。此外，你无法通过 `ALTER TABLE` 语句修改此子句。

### 创建区块链表

现在你已经了解了什么是区块链表以及控制其行为的子句，让我们来看看如何创建一个区块链表。这是一个例子：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create blockchain table ledger
(id number,
username varchar2(30),
value number)
no drop until 0 days idle
no delete locked
hashing using SHA2_512 version v1;
Table created.
```

在前面的代码中，由于指定了 `DROP UNTIL 0 DAYS IDLE`，这意味着该表可以随时被删除。通常，你不会指定零天，但这允许你在测试此功能时删除此表。

`NO DELETE LOCKED` 意味着数据永远不能被删除、修改或截断。移除数据的唯一方法是删除该表或删除拥有该表的用户。例如，观察我们在插入数据并尝试删除后会发生什么：

```sql
SQL> insert into ledger values(1,'HEIDI',1);
1 row created.
SQL> delete from ledger
*
ERROR at line 1:
ORA-05715: operation not allowed on the blockchain table
```

你可以按如下方式验证区块链表的特性：

```sql
SQL> select * from user_blockchain_tables;
TABLE_NAME      ROW_RETENTION ROW TABLE_INACTIVITY_RETENTION HASH_ALG
--------------- ------------- --- -------------------------- --------
LEDGER                     16 NO                           0 SHA2_512
```

注意：除了对删除表或从区块链表中移除数据的限制外，你不能添加列、删除列，也不能将列标记为未使用。

除了被阻止的 DML 和 DDL（删除和更新）操作外，区块链表看起来和行为上都像一个常规表。你可以像访问任何其他表（你有访问权限的）一样访问此表。这使得公司能够识别并防止非法的数据更改。

提示：当你创建区块链表时，会在 `SYS` 拥有的字典表 `blockchain_table$` 中创建一个条目。

当你创建区块链表时，Oracle 会向表中添加隐藏列。这些列可以如下查看：

```sql
SQL> select column_name, data_type, hidden_column
from user_tab_cols
where table_name ='LEDGER'
order by column_id;
COLUMN_NAME                  DATA_TYPE                          HID
---------------------------- ---------------------------------- ---
ID                           NUMBER                             NO
USERNAME                     VARCHAR2                           NO
VALUE                        NUMBER                             NO
ORABCTAB_SPARE$              RAW                                YES
ORABCTAB_USER_NUMBER$        NUMBER                             YES
ORABCTAB_HASH$               RAW                                YES
ORABCTAB_SIGNATURE$          RAW                                YES
ORABCTAB_SIGNATURE_ALG$      NUMBER                             YES
ORABCTAB_SIGNATURE_CERT$     RAW                                YES
ORABCTAB_SEQ_NUM$            NUMBER                             YES
ORABCTAB_CHAIN_ID$           NUMBER                             YES
ORABCTAB_INST_ID$            NUMBER                             YES
ORABCTAB_CREATION_TIME$      TIMESTAMP(6) WITH TIME ZONE        YES
```

这些列用于跟踪表中的数据，以确保加密序列得以维护。你通常不需要查询这些列。相反，这些是 Oracle 用来存储信息以维护区块链表完整性的列。

### DBMS_BLOCKCHAIN_TABLE 包

Oracle 提供了一个内部 PL/SQL 包 `DBMS_BLOCKCHAIN_TABLE`，可用于执行以下操作：

-   删除区块链表中根据指定保留期允许被移除的行。
-   检索作为签名算法输入所需的字节，以便你可以对插入的行进行签名。
-   检索作为插入行的加密哈希输入的字节。这可用于验证该行的哈希值。
-   对插入表中的行进行签名。
-   验证表中行上的哈希值和签名。

有关 `DBMS_BLOCKCHAIN_TABLE` 的完整详细信息，请参阅 Oracle 的 *PL/SQL Packages and Types Reference* 指南。我将在这里展示几个例子，让你了解如何使用此包来验证数据是否未被任何方式篡改。下一个例子调用 `DBMS_BLOCKCHAIN_TABLE.VERIFY_ROWS` 过程来做到这一点：

```sql
$ sqlplus sys/foo@PB1
SQL> set serverout on
declare
actual_rows   number;
verified_rows number;
begin
select count(*)
into actual_rows
from eoda.ledger;
--
dbms_blockchain_table.verify_rows(
schema_name => 'EODA',
table_name => 'LEDGER',
number_of_rows_verified => verified_rows);
dbms_output.put_line('Actual rows: ' || actual_rows || ' Verified rows: ' || verified_rows);
end;
/
Actual rows: 1 Verified rows: 1
```

这允许你验证所有适用链上的行，以确保 `HASH` 列值的完整性。

你不能使用 `DELETE` 命令从区块链表中删除行。从区块链表中删除行的唯一方法是通过 `DBMS_BLOCKCHAIN_TABLE.DELETE_EXPIRED_ROWS` 过程。例如：

```sql
SQL> set serverout on
SQL> declare
number_rows number;
begin
dbms_blockchain_table.delete_expired_rows('EODA','LEDGER', null, number_rows);
dbms_output.put_line('Number of rows deleted: ' || number_rows);
end;
/
```

这允许你移除根据区块链行保留子句指定有资格被删除的行。

### 区块链表总结

许多企业将敏感的财务信息存储在数据库中。这可能以合同、银行对账单、会计信息等形式存在。如果没有区块链技术，有权访问这些表的用户有可能对数据进行非法更改。Oracle 引入了区块链表来防止这些不需要的更改发生。

区块链表是特殊的表，对移除或修改数据有严格限制。它们旨在成为仅可插入的表。随着数据的插入，新行会生成加密哈希签名，并作为行信息的一部分存储。



## 总结

希望阅读完本章后，您能得出这样的结论：并非所有的表都是相同的。Oracle 提供了丰富多样的表类型供您利用。在本章中，我们涵盖了表的一般性重要方面，并探讨了 Oracle 提供的多种不同表类型。

我们首先了解了与表相关的一些术语和存储参数。我们探讨了 `FREELIST`s 在多用户环境中的用处，特别是当一个表被许多人同时频繁插入/更新时，以及使用 ASSM 表空间如何让我们甚至无需考虑这个问题。我们研究了 `PCTFREE` 和 `PCTUSED` 的含义，并制定了一些正确设置它们的指南。

接着，我们介绍了不同类型的表，从普通的堆表开始。堆组织表是绝大多数 Oracle 应用中最常用的表类型，也是默认表类型。我们进而考察了索引组织表，它使我们能够将表数据存储在索引结构中，而不是堆表中。我们看到了它们如何适用于各种场景，例如查找表和倒排列表，在这些场景下堆表只会是数据的冗余副本。随后，我们了解到当与其他表类型（特别是嵌套表类型）结合使用时，索引组织表（IOTs）可以变得非常有用。

我们研究了簇对象，Oracle 有三种簇：索引簇、哈希簇和排序哈希簇。簇的目标是双重的：
- 使我们能够将来自多个表的数据存储在相同的数据库块上。
- 使我们能够基于某个簇键，强制相似的数据物理地存储在一起。通过这种方式，部门 `10` 的所有数据（来自多个表）可以存储在一起。

这些特性使我们能够以最小的物理 I/O 非常快速地访问相关数据。我们观察了索引簇和哈希簇之间的主要区别，并讨论了每种簇何时适用（何时不适用）。

接下来，我们介绍了嵌套表。我们回顾了嵌套表的语法、语义和用法。我们了解到，它们实际上是由系统生成和维护的父/子表对，并发现了 Oracle 为我们实现这一点的物理机制。我们研究了为嵌套表使用不同的表类型，默认情况下嵌套表使用基于堆的表。我们发现，很可能永远没有理由不使用索引组织表（IOT）来代替堆表作为嵌套表。

然后，我们深入探讨了临时表的方方面面，包括如何创建它们、它们从哪里获得存储，以及它们在运行时不会引入并发相关问题的事实。我们探讨了会话级和事务级临时表之间的区别，并讨论了在 Oracle 数据库中使用临时表的适当方法。

与嵌套表一样，我们发现 Oracle 中的对象表在底层也有很多处理。我们讨论了如何在关系表之上构建对象视图，这既能给我们提供对象表的功能，同时又能让我们轻松访问底层的关系数据。

本章最后介绍了区块链表。这是 Oracle 21c 引入并 backport 到 19.10 版本的新表类型。区块链表是专门的仅插入表，用于那些需要保证数据永不被删除或修改的安全性应用程序。


# 索引

为了加快返回记录的速度，你可以在表的列上定义一个或多个索引。这是一个独立的数据结构，只包含感兴趣的列的数据以及指向原始记录的指针。

假设你有一个包含过去 70 年 F1 车手的表。如果你想查找所有姓氏为 Verstappen 的车手，那么 Oracle 除了访问表中的每一行（进行全表扫描）并检查内容以满足此条件并排除不符合的记录外，别无他法。当你检查该 `select` 语句的解释计划（explain plan）时，可以看到这一点。使用以下语句可以创建一个解释计划。

```sql
explain plan for
select drv.forename
, drv.surname
from   drivers drv
where  drv.surname = 'Verstappen'
/
```

创建解释计划后，可以通过执行以下命令来查看它。

```sql
select *
From dbms_xplan.display( 'plan_table' )
/
```

让我们看看在 `drivers` 表中查找姓氏为 Verstappen 的车手是如何执行的。

```sql
select drv.forename
, drv.surname
from   drivers drv
where  drv.surname = 'Verstappen'
/
```

查询结果：

```
| Id  | Operation                   | Name    |

|   0 | SELECT STATEMENT            |         |
| * 1 |   TABLE ACCESS STORAGE FULL | DRIVERS |

Predicate Information (identified by operation id):

1 - storage("DRV"."SURNAME"='Verstappen')
filter("DRV"."SURNAME"='Verstappen')
```

当你在此列上添加一个 B-tree 索引时。

```sql
create index ix_driversurname on drivers (surname, forename)
/
```

解释计划会发生变化。

```
| Id  | Operation        | Name             |

|   0 | SELECT STATEMENT |                  |
|   1 |  INDEX RANGE SCAN| IX_DRIVERSURNAME |

Predicate Information (identified by operation id):

1 - access("DRV"."SURNAME"='Verstappen')
```

SQL 引擎现在使用创建的索引来满足此查询。尽管索引建立在两个列（surname 和 forename）上，但由于你查询的是索引的引导列（leading column），它仍然可以使用。在这个小表（少于 1000 行）上，你可能几乎看不到执行时间的差异，但如果表更大，就能看到显著的改善。

使用索引可以极大地加速 `select` 查询，但请注意这是有代价的。索引必须被维护。这是在执行 DML 的同时完成的，因此除了在表中插入数据外，还需要在索引中插入数据，这可能导致索引内容的重新排列。这需要花费时间；虽然不多，但总归是时间。如果一个表上定义了多个索引，那么所有这些索引都必须根据表的变化进行相应的更改。

#### B-tree

默认情况下，Oracle Database 中的索引被创建为 B-tree（或平衡树）索引。平衡树意味着从根（索引的起点）到叶（存储信息的地方）总是需要相同数量的步骤。它们可以是普通索引或唯一索引。使用唯一索引时，每个值在每个索引中最多只允许出现一次。这也是 Oracle Database 通过在表上创建唯一索引来强制执行主键的方式。

#### 位图索引

在位图索引中，为每个索引键存储一个位图。每个索引键指向多行。以下是位图索引通常使用的场景：

*   索引列中的不同值数量相对于表中的行数较少时，例如性别列。
*   表不经常通过 DML 语句修改或是只读表时。

位图索引主要用于数据仓库应用程序，在 OLTP 应用程序中并不常见。对于 OLTP 环境，不建议使用位图索引。对基础表的 DML 操作会对位图维护产生显著影响，从而直接影响应用程序。

### 索引组织表（IOT）

索引组织表在 Oracle Database 版本 8 中引入。它是索引和表的结合。普通的堆表没有隐式的排序，而这种表是一个有序的记录集合。B-tree 索引只包含索引列的信息以及指向完整记录的指针，而索引组织表则将完整记录存储在索引中，无需额外的表来存储记录的其余部分。这种表最适合作为查找表使用，并且列数有限且类型简单（如 VARCHAR2、NUMBER 和 DATE）。

要为 drivers 创建一个索引组织表，如下所示。

```sql
create table drivers_iot
(
driverid     number(11)    not null
, driverref    varchar2(255)
, drivernumber number(11)
, code         varchar2(3)
, forename     varchar2(255)
, surname      varchar2(255)
, dob          date
, nationality  varchar2(255)
, url          varchar2(255)
, primary key (surname, forename)
)
organization index
/
```

对于堆表，你可以在表创建后定义主键。但对于索引组织表，你必须在定义表时直接提供主键。

执行以下语句，用与 `drivers` 堆表相同的数据填充索引组织表。

```sql
insert into drivers_iot (select * from drivers)
/
```

在索引组织表上运行与之前相同的 `select` 语句。

```sql
select drv.forename
, drv.surname
from   drivers_iot drv
where  drv.surname = 'Verstappen'
/
```

然后你可以从解释计划中看到，所有需要的数据都已经在索引中了，无需在表中进行额外的查找。

```
| Id  | Operation        | Name           |

|   0 | SELECT STATEMENT |                |
|*  1 |  INDEX RANGE SCAN| PK_DRIVERS_IOT |

Predicate Information (identified by operation id):

1 - access("DRV"."SURNAME"='Verstappen')
```


### 簇

簇是另一种存储数据的方法。一组共享相同数据块的表就是一个簇。由于这些表拥有公共列，并且它们的数据经常被一起引用，因此将它们的数据存储在彼此靠近的位置是一个很好的理由。当你基于一个公共字段对表进行簇化时，Oracle 数据库会物理地将所有表中具有相同公共字段值的所有行存储在相同的数据块中。

由于不同表的相关行存储在同一个数据块中，你将获得两个主要好处。

- 当连接簇化表时，磁盘 `I/O` 减少，访问时间得到改善。
- 簇键（即簇中所有表共有的列或列组）只存储一次，无论不同表中有多少行包含该值。

因此，在簇中存储相关表和索引数据所需的存储空间可能比非簇化表格式更少。

所有数据都位于 `F1DATA` 模式中，但让我们在 `BOOK` 模式中创建簇，然后复制行以比较存储使用情况。

```
create cluster cl_f1laps
( raceid      number  (  11 )
, driverid    number  (  11 )
)
/
```

创建簇后，创建簇索引。

```
create index idx_cl_f1laps
on cluster cl_f1laps
/
```

对于创建簇索引，使用簇中指定的列。你不能指定不同的列集。使用簇不影响在簇化表上创建额外索引；它们可以像往常一样创建和删除。

要在簇中创建表，你必须指定它们应该被创建到哪个簇中。在簇中创建表时，至少应有一个列在簇中被指定。

让我们在同一个簇中创建 `RACES`、`DRIVERS` 和 `LAPTIMES` 表。

```
create table races (
raceid                number  (   11 ) not null
, year                  number  (   11 ) not null
, round                 number  (   11 ) not null
, circuitid             number  (   11 ) not null
, name                  varchar2(  255 )
, race_date             date             not null
, time                  date
, url                   varchar2(  255 )
, fp1_date              date
, fp1_time              date
, fp2_date              date
, fp2_time              date
, fp3_date              date
, fp3_time              date
, quali_date            date
, quali_time            date
, sprint_date           date
, sprint_time           date
, constraint pk_races
primary key (raceid)
, constraint uk_races_url
unique      (url)
)
cluster cl_f1laps( raceid )
/
create table drivers
(
driverid              number  (  11 ) not null
, driverref             varchar2( 255 )
, driver_number         number  (  11 )
, code                  varchar2(   3 )
, forename              varchar2( 255 )
, surname               varchar2( 255 )
, dob                   date
, nationality           varchar2( 255 )
, url                   varchar2( 255 )
, constraint pk_drivers
primary key ( driverid )
)
cluster cl_f1laps ( driverid )
/
create table laptimes (
raceid                number  (   11 ) not null
, driverid              number  (   11 ) not null
, lap                   number  (   11 ) not null
, position              number  (   11 ) default null
, time                  varchar2(  255 ) default null
, milliseconds          number  (   11 ) default null
, constraint pk_laptimes
primary key (raceid,driverid,lap)
, constraint fk_laptimes_races
foreign key (raceid)
references races (raceid)
, constraint fk_laptimes_drivers
foreign key (driverid)
references drivers (driverid)
)
cluster cl_f1laps ( raceid, driverid )
/
```

现在将数据从 `F1DATA` 模式复制到我们的表中，以比较查询计划。

```
insert into races
select raceid
, year
, round
, circuitid
, name
, race_date
, time
, url
, fp1_date
, fp1_time
, fp2_date
, fp2_time
, fp3_date
, fp3_time
, quali_date
, quali_time
, sprint_date
, sprint_time
from   f1data.races
/
insert into drivers
select driverid
, driverref
, driver_number
, code
, forename
, surname
, dob
, nationality
, url
from f1data.drivers
/
insert into laptimes
select raceid
, driverid
, lap
, position
, time
, milliseconds
from   f1data.laptimes
/
```

比较两个模式之间的磁盘大小时，你可以看到对于相同的信息量，簇化表占用的空间更小。

以下是 `BOOK` 模式中的情况。

```
select sum(bytes) bytes
from   user_segments
where  segment_type = 'TABLE'
and    segment_name in ( 'DRIVERS'
, 'LAPTIMES'
, 'RACES'
)
/
BYTES

```

以下是 `F1DATA` 模式中的情况。

```
select sum(bytes) bytes
from   user_segments
where  segment_type = 'TABLE'
and    segment_name in ( 'DRIVERS'
, 'LAPTIMES'
, 'RACES'
)
/
BYTES

```

当查询这些表时，优化器可以减少访问的块数，因为簇中表的数据位于磁盘上的同一个块中。

你不应该对频繁单独访问的表使用簇。那会违背其目的。

### 临时表

Oracle 数据库 `8i` 引入了 `temporary table`，它在一个事务**或**一个会话期间保存数据，并且其数据对并发运行在数据库中的其他会话是不可见的。它被称为全局的，但这是因为表的定义是全局的。Oracle 数据库 `18c` 版本引入了私有临时表，其表的定义仅限于创建它的会话。

#### 全局临时表

`全局临时表`是“全局可用”的。你必须像对待任何其他表一样，授予其他方案（schema）对此表的访问权限。此表中的数据仅对创建该数据的会话可用。当你使用两个不同的会话连接到同一个方案时，一个会话无法看到另一个会话放入此表中的任何数据。因此，数据对于会话是私有的，但表的定义是全局可用的。

##### 事务级持续时间

定义全局临时表时，你必须决定行数据是仅对事务可用，还是对整个会话的持续时间可用。若要使行数据仅对事务可用，你需要告诉表在提交时删除行。以下语句创建了一个事务级持续时间的全局临时表。

```sql
create global temporary table gtt_drivers
( driverid     number  (  11 ) not null
, driverref    varchar2( 255 )
, drivernumber number  (  11 )
, code         varchar2(   3 )
, forename     varchar2( 255 )
, surname      varchar2( 255 )
, dob          date
, nationality  varchar2( 255 )
, url          varchar2( 255 )
) on commit delete rows
/
```

向表中填充一些数据后，可以像往常一样查询和操作这些数据。

```sql
insert into gtt_drivers
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
select code
, forename
, surname
, dob
, nationality
from   gtt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
Huub            Rothengatter         08/10/1954 Dutch
Michael         Bleekemolen          02/10/1949 Dutch
Boy             Lunger               03/05/1949 Dutch
Roelof          Wunderink            12/12/1948 Dutch
Gijs            van Lennep           16/03/1942 Dutch
ALB  Christijan      Albers               16/04/1979 Dutch
DOO  Robert          Doornbos             23/09/1981 Dutch
Jos             Verstappen           04/03/1972 Dutch
Jan             Lammers              02/06/1956 Dutch
Dries           van der Lof          23/08/1919 Dutch
Jan             Flinterman           02/10/1919 Dutch
VDG  Giedo           van der Garde        25/04/1985 Dutch
VER  Max             Verstappen           30/09/1997 Dutch
Carel Godin     de Beaufort          10/04/1934 Dutch
Ernie           de Vos               01/07/1941 Dutch
Ben             Pon                  09/12/1936 Dutch
Rob             Slotemaker           13/06/1929 Dutch
17 rows selected
```

然而，当事务结束（通过`commit`或`rollback`）时，表中的数据将被移除。

```sql
commit
/
select code
, forename
, surname
, dob
, nationality
from   gtt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
```

##### 会话级持续时间

另一个选项是保留行，这意味着行在会话期间可用。以下语句重新创建了一个会话持续时间的全局临时表。

```sql
create global temporary table gtt_drivers
( driverid     number  (  11 ) not null
, driverref    varchar2( 255 )
, drivernumber number  (  11 )
, code         varchar2(   3 )
, forename     varchar2( 255 )
, surname      varchar2( 255 )
, dob          date
, nationality  varchar2( 255 )
, url          varchar2( 255 )
) on commit preserve rows
/
```

用示例数据填充表，这些数据可以像往常一样进行查询和操作。

```sql
insert into gtt_drivers
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
select code
, forename
, surname
, dob
, nationality
from   gtt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
Huub            Rothengatter         08/10/1954 Dutch
Michael         Bleekemolen          02/10/1949 Dutch
Boy             Lunger               03/05/1949 Dutch
Roelof          Wunderink            12/12/1948 Dutch
Gijs            van Lennep           16/03/1942 Dutch
ALB  Christijan      Albers               16/04/1979 Dutch
DOO  Robert          Doornbos             23/09/1981 Dutch
Jos             Verstappen           04/03/1972 Dutch
Jan             Lammers              02/06/1956 Dutch
Dries           van der Lof          23/08/1919 Dutch
Jan             Flinterman           02/10/1919 Dutch
VDG  Giedo           van der Garde        25/04/1985 Dutch
VER  Max             Verstappen           30/09/1997 Dutch
Carel Godin     de Beaufort          10/04/1934 Dutch
Ernie           de Vos               01/07/1941 Dutch
Ben             Pon                  09/12/1936 Dutch
Rob             Slotemaker           13/06/1929 Dutch
17 rows selected
```

在发出`commit`语句后，行被保留。

```sql
commit
/
select code
, forename
, surname
, dob
, nationality
from   gtt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
Huub            Rothengatter         08/10/1954 Dutch
Michael         Bleekemolen          02/10/1949 Dutch
Boy             Lunger               03/05/1949 Dutch
Roelof          Wunderink            12/12/1948 Dutch
Gijs            van Lennep           16/03/1942 Dutch
ALB  Christijan      Albers               16/04/1979 Dutch
DOO  Robert          Doornbos             23/09/1981 Dutch
Jos             Verstappen           04/03/1972 Dutch
Jan             Lammers              02/06/1956 Dutch
Dries           van der Lof          23/08/1919 Dutch
Jan             Flinterman           02/10/1919 Dutch
VDG  Giedo           van der Garde        25/04/1985 Dutch
VER  Max             Verstappen           30/09/1997 Dutch
Carel Godin     de Beaufort          10/04/1934 Dutch
Ernie           de Vos               01/07/1941 Dutch
Ben             Pon                  09/12/1936 Dutch
Rob             Slotemaker           13/06/1929 Dutch
17 rows selected
```

数据将保留在这个全局临时表中，直到会话结束。

##### 索引与错误处理

你可以为全局临时表添加索引，但只能在表为空时进行。如果你尝试为设置了`on commit preserve rows`的全局临时表添加索引，可能会遇到此错误。

```
ORA-14452: attempt to create, alter or drop an index on temporary table already in use
```

如果发生这种情况，并且你的会话是唯一使用该表的会话，你可以通过断开并重新连接或发出`truncate table`语句来解决这个问题。

##### 使用 CTAS 创建

你也可以使用`create table as select`结构来创建全局临时表。

```sql
create global temporary table gtt_drivers
as
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
```

由于默认选项是`on commit delete rows`，并且任何 DDL 操作都会执行隐式提交，因此表定义被创建，但行会立即被移除。要保留行，你必须使用`on commit preserve rows`选项创建表。

```sql
create global temporary table gtt_drivers
on commit preserve rows
as
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
```

#### 私有临时表

Oracle 数据库 18c 引入了`私有临时表`。它是一种仅对当前会话私有的临时表。私有临时表必须使用前缀：`ora$ptt_`。以下语句创建了一个私有临时表。

```sql
create private temporary table ora$ptt_drivers
( driverid     number  (  11 )
, driverref    varchar2( 255 )
, drivernumber number  (  11 )
, code         varchar2(   3 )
, forename     varchar2( 255 )
, surname      varchar2( 255 )
, dob          date
, nationality  varchar2( 255 )
, url          varchar2( 255 )
) on commit drop definition
/
```

##### 在提交时填充与删除

填充私有临时表。

```sql
insert into ora$ptt_drivers
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
select code
, forename
, surname
, dob
, nationality
from   ora$ptt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
Huub            Rothengatter         08/10/1954 Dutch
Michael         Bleekemolen          02/10/1949 Dutch
Boy             Lunger               03/05/1949 Dutch
Roelof          Wunderink            12/12/1948 Dutch
Gijs            van Lennep           16/03/1942 Dutch
ALB  Christijan      Albers               16/04/1979 Dutch
DOO  Robert          Doornbos             23/09/1981 Dutch
Jos             Verstappen           04/03/1972 Dutch
Jan             Lammers              02/06/1956 Dutch
Dries           van der Lof          23/08/1919 Dutch
Jan             Flinterman           02/10/1919 Dutch
VDG  Giedo           van der Garde        25/04/1985 Dutch
VER  Max             Verstappen           30/09/1997 Dutch
Carel Godin     de Beaufort          10/04/1934 Dutch
Ernie           de Vos               01/07/1941 Dutch
Ben             Pon                  09/12/1936 Dutch
Rob             Slotemaker           13/06/1929 Dutch
17 rows selected
```

结束事务会从数据库中移除私有临时表。

```sql
commit
/
select code
, forename
, surname
, dob
, nationality
from   ora$ptt_drivers
/
```

```
ORA-00942: table or view does not exist
```

##### 保留定义

私有临时表也可能需要存在更长时间。

```sql
create private temporary table ora$ptt_drivers
( driverid     number  (  11 )
, driverref    varchar2( 255 )
, drivernumber number  (  11 )
, code         varchar2(   3 )
, forename     varchar2( 255 )
, surname      varchar2( 255 )
, dob          date
, nationality  varchar2( 255 )
, url          varchar2( 255 )
) on commit preserve definition
/
insert into ora$ptt_drivers
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
select code
, forename
, surname
, dob
, nationality
from   ora$ptt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
Huub            Rothengatter         08/10/1954 Dutch
Michael         Bleekemolen          02/10/1949 Dutch
Boy             Lunger               03/05/1949 Dutch
Roelof          Wunderink            12/12/1948 Dutch
Gijs            van Lennep           16/03/1942 Dutch
ALB  Christijan      Albers               16/04/1979 Dutch
DOO  Robert          Doornbos             23/09/1981 Dutch
Jos             Verstappen           04/03/1972 Dutch
Jan             Lammers              02/06/1956 Dutch
Dries           van der Lof          23/08/1919 Dutch
Jan             Flinterman           02/10/1919 Dutch
VDG  Giedo           van der Garde        25/04/1985 Dutch
VER  Max             Verstappen           30/09/1997 Dutch
Carel Godin     de Beaufort          10/04/1934 Dutch
Ernie           de Vos               01/07/1941 Dutch
Ben             Pon                  09/12/1936 Dutch
Rob             Slotemaker           13/06/1929 Dutch
17 rows selected
```

结束事务不再移除私有临时表或其内容。

```sql
commit
/
select code
, forename
, surname
, dob
, nationality
from   ora$ptt_drivers
/
```

```
CODE FORENAME        SURNAME              DOB        NATIONALITY
---- --------------- -------------------- ---------- -----------
Huub            Rothengatter         08/10/1954 Dutch
Michael         Bleekemolen          02/10/1949 Dutch
Boy             Lunger               03/05/1949 Dutch
Roelof          Wunderink            12/12/1948 Dutch
Gijs            van Lennep           16/03/1942 Dutch
ALB  Christijan      Albers               16/04/1979 Dutch
DOO  Robert          Doornbos             23/09/1981 Dutch
Jos             Verstappen           04/03/1972 Dutch
Jan             Lammers              02/06/1956 Dutch
Dries           van der Lof          23/08/1919 Dutch
Jan             Flinterman           02/10/1919 Dutch
VDG  Giedo           van der Garde        25/04/1985 Dutch
VER  Max             Verstappen           30/09/1997 Dutch
Carel Godin     de Beaufort          10/04/1934 Dutch
Ernie           de Vos               01/07/1941 Dutch
Ben             Pon                  09/12/1936 Dutch
Rob             Slotemaker           13/06/1929 Dutch
17 rows selected
```

##### 使用 CTAS 创建

你也可以使用 `create table as select` 结构创建私有临时表。

```sql
create private temporary table ora$ptt_drivers
as
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
```

创建临时表的默认选项是 `on commit drop definition`。但由于这是一个仅内存的结构，没有隐式提交，因此行数据在表中是可用的。你可以添加选项以在提交后保留定义。

```sql
create private temporary table ora$ptt_drivers
on commit preserve definition
as
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
/
```

私有临时表的定义仅对当前会话私有；因此 `on commit preserve definition` 意味着该定义将保留到会话结束，然后消失。你不能在私有临时表上创建索引。

##### 表前缀

私有临时表必须使用标准前缀。默认情况下，此前缀为 `ORA$PTT_`，因为它是在参数中设置的。

```sql
select *
from   v$parameter
where  name = 'private_temp_table_prefix'
/
```

##### 临时表的限制

尽管临时表看起来和行为上像常规表，但仍有一些限制。

*   临时表不能被分区、集群或组织为索引。
*   你不能在临时表上指定任何外键约束。
*   临时表不能包含嵌套表的列。
*   你不能指定 `LOB_storage_clause` 的以下子句：`TABLESPACE`、`storage_clause` 或 `logging_clause`。
*   不支持对临时表进行并行 `UPDATE`、`DELETE` 和 `MERGE` 操作。
*   对于临时表，`segment_attributes_clause` 中唯一可以指定的部分是 `TABLESPACE`，它允许你指定一个单一的临时表空间。
*   不支持对临时表进行分布式事务。
*   临时表不能包含 `INVISIBLE` 列。

所有这些限制既适用于全局临时表，也适用于私有临时表。私有临时表在上述列表基础上还有更多限制。

*   私有临时表的名称必须始终以 `init.ora` 参数 `PRIVATE_TEMP_TABLE_PREFIX` 定义的任何内容为前缀。默认是 `ORA$PTT_`。
*   你不能在私有临时表上创建索引、物化视图或区域映射。
*   不允许在私有临时表上定义主键或任何需要索引的约束。
*   你不能定义带有默认值的列。
*   你不能在任何永久对象（例如视图或触发器）中引用私有临时表。
*   私有临时表通过数据库链接不可见。

尽管看起来临时表有很多限制，但它们可能非常有用。你可能不会受到这些限制的阻碍。


### 外部表

#### 外部表基本概念

外部表允许您像读取常规表一样从平面文件中读取数据。文件必须放置在 Oracle 数据库可以访问的位置。如果您使用的是文件系统上的文件，则需要创建一个目录对象并告诉 Oracle 数据库查找位置。

```sql
create or replace directory f1db_csv
as '/media/sf_Ergast_F1/f1db_csv'
/
```

确保您对此位置拥有读取权限。如果使用`logfile`/`badfile`/`discardfile`，则还需要写入权限。

现在，您可以使用`create table … organization external`语法开始定义外部表。

#### 创建外部表

定义的第一部分类似于常规表，但需要通过`organization external`关键字指定其为外部表。

```sql
create table drivers_ext
( driverid     number  (  11 )
, driverref    varchar2( 255 )
, drivernumber varchar2( 255 )
, code         varchar2(   3 )
, forename     varchar2( 255 )
, surname      varchar2( 255 )
, dob          date
, nationality  varchar2( 255 )
, url          varchar2( 255 )
) organization external
/
```

#### 外部表属性

现在，您需要指定外部表的属性。

*   `type` 指定外部表的类型。每种类型都由其自己的访问驱动程序支持。
    *   `oracle_loader` 是默认的访问驱动程序。它将数据从外部表加载到内部表。它只能读取文本数据文件，不能写入文本文件，即不能用于将内部表卸载到外部表中。
    *   `oracle_datapump` 可以执行加载和卸载操作。数据必须位于二进制转储文件中。您只能作为使用`create table as select`语句创建外部表的一部分来使用此驱动程序写入转储文件。这是一次写入、多次读取的操作。您无法在外部表上执行任何 DML 操作。
    *   `oracle_hdfs` 提取存储在 Hadoop 分布式文件系统 (HDFS) 中的数据。
    *   `oracle_hive` 提取存储在 Apache HIVE 中的数据。
*   `default directory` 指定用于所有输入和输出文件的默认目录（如果您没有显式指定目录对象）。它是一个目录对象，而不是目录路径。
*   `access parameters` 描述外部数据源并实现外部表的类型。每种类型的外部表都有自己的访问驱动程序，需要自己的访问参数。访问参数是可选的。部分访问参数包括以下内容。
    *   `records delimited by newline` 指示记录结束的字符。在 Unix 或 Linux 操作系统上，`NEWLINE` 被假定为`'\n'`。在 Microsoft Windows 操作系统上，`NEWLINE` 被假定为`'\r\n'`。如果您在 Unix 环境中使用在 Windows 系统上创建的文件，换行符会被错误地解析。要解决此问题，可以使用`records delimited by detected newline`。
    *   注：`Detected newline` 是自 Oracle Database 19c 起提供的选项。
    *   `skip <n>` 指定在加载前要跳过的记录数。主要用于文件包含头信息的情况。
    *   `badfile <filename>` 指定当记录因错误而无法加载时，将记录写入的文件名。
    *   `logfile <filename>` 指定在访问数据时将所有消息写入的文件名。
    *   `discardfile <filename>` 指定将所有未能满足`load when`子句条件的记录写入的文件名。
    *   `fields terminated by` 指定字段分隔符。
    *   `optionally enclosed by` 指定字段包围符。

在访问参数中，您还可以指定文件中发现的列。它必须是表定义中列的超集。指定列时，可以对列执行有限的操作，例如将日期字符串转换为`date`数据类型。

```
dob char( 10 ) date_format date mask "YYYY-MM-DD"
```

有关所有其他访问参数，请查阅 Oracle 文档。

*   `location` 指定外部表的数据文件。对于`oracle_loader`和`oracle_datapump`，使用`directory:file`格式。目录部分是可选的。如果省略，则使用默认目录。使用`oracle_loader`驱动程序时，可以在文件名中使用通配符。星号（`*`）表示 0 个或多个字符，问号（`?`）表示单个字符。
*   对于`oracle_hdfs`，它是目录或文件的 URI（统一资源标识符）列表。目录对象与 URI 无关。
*   对于`oracle_hive`，不使用此子句。而是使用 Hadoop HCatalog 表。
*   `reject limit <n>` 指定在加载完全停止之前可以拒绝的记录数。

除了为日志文件、错误文件和废弃文件指定文件名外，您还可以指定`nologfile`、`nobadfile`和`nodiscardfile`选项。不会创建任何文件。

日志文件可能非常有用并包含有价值的信息，但它们可能变得非常大。在提升到生产环境时，您可能希望使用`nologfile`选项。如果希望使用`logfile`/`badfile`/`discardfile`，最好将它们放在不同的位置，以便包含这些文件的目录对象可以被授予只读权限。

清单 20-1 是创建外部表的完整脚本。

```sql
create table drivers_ext
( driverid      number  (  11 )
, driverref     varchar2( 256 )
, driver_number varchar2( 256 )
, code          varchar2(   4 )
, forename      varchar2( 256 )
, surname       varchar2( 256 )
, dob           date
, nationality   varchar2( 256 )
, url           varchar2( 256 )
) organization external
(
type oracle_loader
default directory f1db_csv
access parameters
( -- you can use comments,
-- but only at the beginning the parameters
records delimited by newline
skip 1
badfile 'drivers.bad'
logfile 'drivers.log'
fields terminated by ','
optionally enclosed by '"'
( driverid
, driverref
, driver_number
, code
, forename
, surname
, dob          char( 10 ) date_format date mask "YYYY-MM-DD"
, nationality
, url
)
)
location ( 'drivers.csv' )
)
reject limit 0
/
```

清单 20-1
创建 `drivers_ext` 外部表的脚本

#### 修改与运行时操作

创建外部表后，您可以修改设置，例如将其指向另一个文件。

```sql
alter table drivers_ext location ( 'drivers100.csv' )
```

如果更改访问参数，请注意，您未提供的每个参数都会恢复为其默认值。它不会保留您之前提供的设置。

Oracle Database 12.2 引入了在运行时修改某些设置的选项。

*   您可以在`access`参数中更改`badfile`、`logfile`和`discardfile`的文件名。其他`access`参数保留其值。
*   更改`default directory`；它必须是一个字面值。
*   更改`location`；它可以是一个字面值或绑定变量。
*   更改`reject limit`；它可以是一个字面值或绑定变量。

```sql
select count(*)
from   drivers_ext external modify (location ('drivers100.csv'))
/
COUNT(*)

select count(*)
from   drivers_ext
/
COUNT(*)
```

#### 内联外部表

自 Oracle Database 18c 起，您无需创建外部表即可访问外部表。通常位于外部表 DDL 中的所有代码现在都可以放入 SQL 语句中。

```sql
select * from external (
( constructorId  number(11)
, constructorRef varchar2(256)
, name           varchar2(256)
, nationality    varchar2(256)
, url            varchar2(256)
)
type oracle_loader
default directory f1db_csv
access parameters
( records delimited by newline
skip 1
badfile 'constructors.bad'
logfile 'constructors.log'
fields terminated by ','
optionally enclosed by '"'
( constructorId
, constructorRef
, name
, nationality
, url
)
)
location ( 'constructors.csv' )
reject limit 0
)
```



如果您之前使用 SQL*loader 实用程序将数据加载到 Oracle Database 中，可以通过 `EXTERNAL_TABLE=GENERATE_ONLY` 选项来生成所需的外部表定义。

### 不可变表

Oracle Database 21c 引入了不可变表的概念。不可变表是仅可插入的表，其中的现有数据无法更改。除非插入操作发生在指定的天数之前，否则禁止从不可变表中删除行。创建表后，无法执行任何 DDL 来更改表的布局。但是，可以添加和移除约束及索引。

以下语句用于创建不可变表。

```sql
create immutable table dutchdrivers
( driverid     number  (  11 ) not null
, driverref    varchar2( 255 )
, drivernumber number  (  11 )
, code         varchar2(   3 )
, forename     varchar2( 255 )
, surname      varchar2( 255 )
, dob          date
, nationality  varchar2( 255 )
, url          varchar2( 255 )
)
```

您必须指定删除整个表和删除行的保留期。

首先，指定删除表的条件。

```sql
no drop [ until days idle ]
```

您可以为 idle 选项指定的最小天数为 0。

接下来，指定在能够删除行之前的记录保留时间。

```sql
no delete ( [ locked ] | ( until days after insert [ locked ] ) )
```

您可以为 delete 选项指定的最小天数为 16 天。

您可以选择永久保留数据；那么，您就不必在这两个子句中指定天数。

**提示**

在尝试这些选项时，请确保将 drop 子句中的 idle 天数设置为一个较低的值，最好为 0；否则，您最终可能会得到一个充满无法删除的表的模式。除了完全删除您的数据库之外，没有其他选项可以移除仍处于保留期内的表。

创建表的完整脚本可以在清单 20-2 中找到。

```sql
create immutable table dutchdrivers
( driverid      number  (  11 ) not null
, driverref     varchar2( 255 )
, driver_number number  (  11 )
, code          varchar2(   3 )
, forename      varchar2( 255 )
, surname       varchar2( 255 )
, dob           date
, nationality   varchar2( 255 )
, url           varchar2( 255 )
)
no drop until 0 days idle
no delete until 16 days after insert
/
Listing 20-2
创建不可变表 dutchdrivers
```

现在您已经创建了表，可以向其中插入数据。

```sql
insert into dutchdrivers
select *
from f1data.drivers drv
where drv.nationality = 'Dutch'
/
```

到目前为止，这与普通表完全相同。当您添加一条您稍后想要更改或移除的记录时，这就不可能了。

```sql
insert into dutchdrivers
select *
from f1data.drivers drv
where drv.drivernumber = 44
/
```

如果您尝试更新该记录，将会遇到错误。

```sql
update dutchdrivers ddv
set ddv.nationality = 'Dutch'
where ddv.drivernumber = 44
/
ORA-05715: operation not allowed on the blockchain or immutable table
```

如果您尝试删除该记录，将会遇到相同的错误。

```sql
delete from dutchdrivers ddv
where ddv.drivernumber = '44'
/
ORA-05715: operation not allowed on the blockchain or immutable table
```

当您想要移除该记录时，如果您指定了 `no delete until`，则必须等待指定的天数。

此功能在 Oracle Database 21c 中引入，但已向后移植到 Oracle Database 19.11。

当存储不应被篡改的数据（例如合同数据）时，此功能非常有用。如果您想要比仅仅确保记录无法更改更高的安全性，可以使用区块链表，其中记录使用哈希算法链接在一起。

### 区块链表

区块链表是仅可追加的表，其中只允许插入操作。删除行要么被禁止，要么根据时间受到限制。特殊的排序和链接算法使区块链表中的行具有防篡改性。用户可以验证行是否未被篡改。行元数据的一部分哈希值用于链接和验证行。

区块链表的语法与不可变表几乎相同。区块链表必须在表定义中包含一个额外的子句。

```sql
hashing using sha2_512 version v1
```

哈希算法在记录添加到表中时基于现有数据计算，这与不可变表相比增加了加载时间。

清单 20-3 是一个创建的区块链表，类似于之前创建的不可变表。

```sql
create blockchain table dutchdrivers
( driverid      number  (  11 ) not null
, driverref     varchar2( 255 )
, driver_number number  (  11 )
, code          varchar2(   3 )
, forename      varchar2( 255 )
, surname       varchar2( 255 )
, dob           date
, nationality   varchar2( 255 )
, url           varchar2( 255 )
)
no drop until 0 days idle
no delete until 16 days after insert
hashing using sha2_512 version v1
/
Listing 20-3
创建区块链表 dutchdrivers
```

此表具有与不可变表相同的限制，因此在插入并提交数据后，您无法修改或删除行。但是，如果保留时间已过，则可以进行删除操作。

此功能在 Oracle Database 21c 中引入，但已向后移植到 Oracle Database 19.10。

### 总结

如您所见，Oracle Database 远不止是一个“表容器”。某些表，如堆表，在 OLTP 环境中很有用。相比之下，其他表，如索引组织表，在 OLTP 和数据仓库环境中作为查找表非常有用。B 树索引在 OLTP 环境中更有用，而位图索引在数据仓库环境中更有用。有时，您可能希望将数据从内存卸载到数据库中，以充分利用 SQL 的功能，同时节省可用内存，这时临时表就派上用场了。如果您想要无法篡改的表，可以使用不可变表或区块链表。

Oracle Database 为您提供了所有选项，现在您可以理解为什么一种类型会更适合。




# 13. `分区`

`分区` 是将一个表或索引在物理上分解成许多更小、更易管理片段的过程。就访问数据库的应用程序而言，逻辑上只有一个表或一个索引，但物理上该表或索引可能由数十个物理分区组成。每个分区都是一个独立的对象，可以单独操作或作为更大对象的一部分进行操作。

> **注意**
>
> 分区是 Oracle 数据库企业版的额外付费选项。标准版中不可用。

在本章中，我们将探讨为什么你可能会考虑使用分区。原因包括数据可用性提高、管理（DBA）负担减轻，以及在某些情况下性能提升。一旦你充分理解了使用分区的原因，我们将看看如何对表及其对应的索引进行分区。本讨论的目标不是教你管理分区的细节，而是提供一个关于在应用程序中实施分区的实用指南。

我们还将讨论一个重要事实：对表和索引进行分区并不能保证数据库的 `fast = true` 设置。根据我的经验，许多开发人员和 DBA 认为分区对象会自动带来性能提升。分区只是一个工具，当你对索引或表进行分区时，会发生以下三种情况之一：使用这些分区表的应用程序可能运行得更慢，可能运行得更快，或者可能不受影响。我认为，如果你只是应用分区而不理解它的工作原理以及你的应用程序如何利用它，那么仅仅开启它很可能对性能产生负面影响。

最后，我们将探讨当今世界中分区的一种非常常见的用途：在 OLTP 和其他操作系统中支持大型联机审计跟踪。我们将讨论如何结合使用分区和段空间压缩，以高效地在线存储大型审计跟踪，并提供以最小工作量将旧记录从该审计跟踪中归档出去的能力。

## 分区概述

分区利用 `分而治之` 的逻辑，便于管理超大型表和索引。分区引入了 `分区键` 的概念，用于根据某个范围值、特定值列表或哈希函数的值来隔离数据。如果要我将分区的好处按某种顺序排列，那将如下：

1.  `提高数据可用性`：此属性适用于所有系统类型，无论是 OLTP 系统还是数据仓库系统。
2.  `通过将大型段移出数据库来简化其管理`：对一个 100GB 的表执行管理操作（例如重组以清除迁移行或回收因清除旧信息而在表中留下的“空白空间”），要比对十个单独的 10GB 表分区执行相同操作繁重得多。此外，使用分区，我们或许能够执行清除例程而根本不留下空白空间，从而完全免去重组的需要！
3.  `提高某些查询的性能`：这主要在大型仓库环境中有益，我们可以利用分区来排除需要考虑的大量数据范围，从而完全避免访问这些数据。这在事务处理系统中不太适用，因为我们在那种系统中访问的数据量本来就不大。
4.  `可能通过将修改分散到多个独立分区来减少高流量 OLTP 系统的争用`：如果你有一个争用度很高的段，将其拆分为多个段可能会产生按比例减少该争用的副作用。

让我们逐一看看使用分区的这些潜在好处。

### 可用性提升

可用性提升源于每个分区的独立性。对象中单个分区的可用（或不可用）并不意味着对象本身不可用。优化器知晓当前的分区方案，并会据此从查询计划中移除未被引用的分区。如果一个大型对象中某个分区不可用，而你的查询能够排除对此分区的考虑，那么 Oracle 将成功处理该查询。

为了演示这种提升的可用性，我们将设置一个包含两个分区的哈希分区表，每个分区位于独立的表空间中。我们将创建一个 `EMP` 表，并在 `EMPNO` 列上指定分区键；`EMPNO` 将作为我们的分区键。在这种情况下，此结构意味着对于插入此表的每一行，`EMPNO` 列的值会被哈希计算以确定该行将被放置到哪个分区（从而确定表空间）。首先，我们创建两个表空间 (`P1` 和 `P2`)，然后创建一个包含两个分区 (`PART_1` 和 `PART_2`) 的分区表，每个分区位于一个表空间中：

**注意**

此示例中的表空间使用 Oracle 托管文件，初始化参数 `DB_CREATE_FILE_DEST` 设置为 `/opt/oracle/oradata`。

```
$ sqlplus eoda/foo@PDB1
SQL> create tablespace p1 datafile size 1m autoextend on next 1m;
Tablespace created.
SQL> create tablespace p2 datafile size 1m autoextend on next 1m;
Tablespace created.
SQL> CREATE TABLE emp
( empno   int,
ename   varchar2(20)
)
PARTITION BY HASH (empno)
( partition part_1 tablespace p1,
partition part_2 tablespace p2
)
/
Table created.
```

接下来，我们向 `EMP` 表插入一些数据，然后使用分区扩展表名来检查每个分区的内容：

```
SQL> insert into emp select empno, ename from scott.emp;
14 rows created.
SQL> select * from emp partition(part_1);
EMPNO ENAME
---------- --------------------
7369 SMITH
7499 ALLEN
7654 MARTIN
7698 BLAKE
7782 CLARK
7839 KING
7876 ADAMS
7934 MILLER
8 rows selected.
SQL> select * from emp partition(part_2);
EMPNO ENAME
---------- --------------------
7521 WARD
7566 JONES
7788 SCOTT
7844 TURNER
7900 JAMES
7902 FORD
6 rows selected.
```

你应该注意到数据是随机分配的。这是此处的设计意图。使用哈希分区，我们要求 Oracle 随机地——但希望是均匀地——将数据分布到多个分区中。我们无法控制数据进入哪个分区；Oracle 根据哈希键值本身的哈希计算来决定。稍后，当我们查看范围分区和列表分区时，将了解如何控制哪些分区接收哪些数据。

现在，我们将其中一个表空间离线（模拟，例如，磁盘故障），从而使该分区中的数据不可用：

```
SQL> alter tablespace p1 offline;
Tablespace altered.
```

接下来，我们运行一个访问每个分区的查询，看到此查询失败了：

```
SQL> select * from emp;
select * from emp
*
ERROR at line 1:
ORA-00376: file 22 cannot be read at this time
ORA-01110: data file 22:
'/opt/oracle/oradata/CDB/C217E68DF48779E1E0530101007F73B9/datafile/o1_mf_p1_jc8b
g9nm_.dbf'
```

但是，一个不访问离线表空间的查询将正常运行；Oracle 会从考虑范围中排除离线的分区。我在此特定示例中使用绑定变量是为了证明，即使 Oracle 在查询优化时不知道将访问哪个分区，它仍然能够在运行时执行这种消除：

```
SQL> variable n number
SQL> exec :n := 7844;
PL/SQL procedure successfully completed.
SQL> select * from emp where empno = :n;
EMPNO ENAME
---------- --------------------
7844 TURNER
```

总之，当优化器能够从计划中消除分区时，它就会这样做。这一事实为那些在查询中使用分区键的应用程序提高了可用性。

分区还通过减少停机时间来提高可用性。例如，如果你有一个 100GB 的表，并将其划分为 50 个 2GB 的分区，那么你就可以更快地从错误中恢复。如果一个 2GB 的分区损坏，恢复所需的时间就是恢复一个 2GB 分区所需的时间，而不是一个 100GB 的表。因此，可用性通过两种方式得到提升：

*   优化器进行分区消除意味着许多用户甚至可能不会注意到部分数据不可用。

*   由于恢复所需的工作量显著减少，发生错误时的停机时间得以缩短。

## 减轻管理负担

管理负担的减轻源于一个基本事实：对小对象进行操作通常比对大对象进行相同操作更容易、更快且资源消耗更少。

例如，假设数据库中有一个 10GB 的索引。如果需要重建此索引且它未被分区，那么你将不得不把整个 10GB 索引作为一个工作单元来重建。虽然可以在线重建索引，但这需要海量资源来完全重建整个 10GB 索引。你至少需要 10GB 的额外可用空间来保存两个索引的副本，需要一个临时事务日志表来记录重建索引期间对基表所做的更改，等等。

另一方面，如果索引本身被划分为十个 1GB 的分区，那么你可以逐个重建每个索引分区。现在，你需要的可用空间只是之前的百分之十。同样地，单个索引重建会快得多（也许快十倍），因此在线索引重建期间需要合并到新索引中的事务更改也会少得多，等等。

另外，考虑一下在即将完成 10GB 索引重建时，系统或软件发生故障的情况。整个工作将全部丢失。通过分解问题并将索引分区为 1GB 的分区，你最多只会损失重建索引所需总工作的百分之十。

最后但同样重要的是，可能你只需要重建总聚合索引的百分之十——例如，只有“最新”数据（活动数据）需要重组，而所有“较旧”数据（相对静态）则不受影响。

另一个例子可能是，你发现表中 50% 的行是“迁移”行（有关迁移行的详细信息，请参见第 10 章），并且你想修复这个问题。拥有一个分区表将便于此操作。要“修复”迁移行，通常必须重建对象——在本例中是表。如果你有一个 100GB 的表，你将需要以单个非常大的块串行执行此操作，使用 `ALTER TABLE MOVE`。另一方面，如果你有 25 个分区，每个分区大小为 4GB，那么你可以逐个重建每个分区。或者，如果你在非高峰时间执行此操作并且有充足的资源，你甚至可以在单独的会话中并行执行 `ALTER TABLE MOVE` 语句，从而可能减少整个操作所需的时间。几乎任何可以对非分区对象执行的操作，都可以对分区对象的单个分区执行。你甚至可能会发现你的迁移行集中在非常小的一部分分区中；因此，你可以重建一两个分区，而不是整个表。

下面是一个快速示例，演示如何重建一个包含许多迁移行的表。`BIG_TABLE1` 和 `BIG_TABLE2` 都是从一个包含 10,000,000 行的 `BIG_TABLE` 实例创建的（有关 `BIG_TABLE` 的创建脚本，请参见本书开头的“设置环境”部分）。`BIG_TABLE1` 是一个常规的、未分区的表，而 `BIG_TABLE2` 是一个包含八个分区的哈希分区表（我们将在后续章节中详细介绍哈希分区；简单来说，它将数据相当均匀地分布到了八个分区中）。此示例创建两个表空间，然后创建两个表：

```sql
SQL> create tablespace big1 datafile size 1500M;
Tablespace created.
SQL> create tablespace big2 datafile size 1500m;
Tablespace created.
SQL> create table big_table1
( ID, OWNER, OBJECT_NAME, SUBOBJECT_NAME,
OBJECT_ID, DATA_OBJECT_ID,
OBJECT_TYPE, CREATED, LAST_DDL_TIME,
TIMESTAMP, STATUS, TEMPORARY,
GENERATED, SECONDARY )
tablespace big1
as
select ID, OWNER, OBJECT_NAME, SUBOBJECT_NAME,
OBJECT_ID, DATA_OBJECT_ID,
OBJECT_TYPE, CREATED, LAST_DDL_TIME,
TIMESTAMP, STATUS, TEMPORARY,
GENERATED, SECONDARY
from big_table;
Table created.
SQL> create table big_table2
( ID, OWNER, OBJECT_NAME, SUBOBJECT_NAME,
OBJECT_ID, DATA_OBJECT_ID,
OBJECT_TYPE, CREATED, LAST_DDL_TIME,
TIMESTAMP, STATUS, TEMPORARY,
GENERATED, SECONDARY )
partition by hash(id)
(partition part_1 tablespace big2,
partition part_2 tablespace big2,
partition part_3 tablespace big2,
partition part_4 tablespace big2,
partition part_5 tablespace big2,
partition part_6 tablespace big2,
partition part_7 tablespace big2,
partition part_8 tablespace big2
)
as
select ID, OWNER, OBJECT_NAME, SUBOBJECT_NAME,
OBJECT_ID, DATA_OBJECT_ID,
OBJECT_TYPE, CREATED, LAST_DDL_TIME,
TIMESTAMP, STATUS, TEMPORARY,
GENERATED, SECONDARY
from big_table;
Table created.
```

现在，每个表都在自己的表空间中，因此我们可以轻松查询数据字典以查看每个表空间的已分配和可用空间：

```sql
SQL> select b.tablespace_name,
mbytes_alloc,
mbytes_free
from ( select round(sum(bytes)/1024/1024) mbytes_free,
tablespace_name
from dba_free_space
group by tablespace_name ) a,
( select round(sum(bytes)/1024/1024) mbytes_alloc,
tablespace_name
from dba_data_files
group by tablespace_name ) b
where a.tablespace_name (+) = b.tablespace_name
and b.tablespace_name in ('BIG1','BIG2')
/
TABLESPACE_NAME                MBYTES_ALLOC MBYTES_FREE
------------------------------ ------------ -----------
BIG1                                   1500         219
BIG2                                   1500         219
```

`BIG1` 和 `BIG2` 的大小约为 1500MB，每个大约有 219MB 的可用空间。我们将尝试重建第一个表 `BIG_TABLE1`：

```sql
SQL> alter table big_table1 move;
alter table big_table1 move
*
ERROR at line 1:
ORA-01652: unable to extend temp segment by 1024 in tablespace BIG1
```

操作失败——我们需要在表空间 `BIG1` 中有足够的可用空间，以便在旧副本还在的情况下同时保存整个 `BIG_TABLE1` 的副本——简而言之，我们需要大约两倍于表大小的存储空间（可能更多，也可能更少，这取决于重建后表的大小）。我们现在尝试在 `BIG_TABLE2` 上执行相同的操作：

```sql
SQL> alter table big_table2 move;
alter table big_table2 move
*
ERROR at line 1:
ORA-14511: cannot perform operation on a partitioned object
```

这是 Oracle 告诉我们不能对*表*执行 `MOVE` 操作；我们必须改为对表的每个*分区*执行该操作。我们可以逐个移动（从而重建和重组）每个分区：

```sql
SQL> alter table big_table2 move partition part_1;
Table altered.
SQL> alter table big_table2 move partition part_2;
Table altered.
SQL> alter table big_table2 move partition part_3;
Table altered.
SQL> alter table big_table2 move partition part_4;
Table altered.
SQL> alter table big_table2 move partition part_5;
Table altered.
SQL> alter table big_table2 move partition part_6;
Table altered.
SQL> alter table big_table2 move partition part_7;
Table altered.
SQL> alter table big_table2 move partition part_8;
Table altered.
```


每一次移动操作只需要足够的可用空间来保存数据八分之一的副本！因此，在与之前相同数量的可用空间下，这些命令就能成功执行。我们需要的临时资源显著减少，而且，更进一步地，如果系统在我们移动 `PART_4` 之后、但 `PART_5` 移动完成之前发生故障（例如，由于断电），我们也不会丢失所有已完成的工作。系统恢复时，前四个分区仍将被移动，我们可以从分区 `PART_5` 继续处理。

有些人看到这里可能会说：“哇，八个语句——这需要输入好多内容”，这确实如此，如果你有数百个或更多分区，这种事情会变得不切实际。幸运的是，编写脚本实现解决方案非常容易，之前的八个语句可以简化为

```sql
SQL> begin
for x in ( select partition_name
from user_tab_partitions
where table_name = 'BIG_TABLE2' )
loop
execute immediate
'alter table big_table2 move partition ' ||
x.partition_name;
end loop;
end;
/
PL/SQL procedure successfully completed.
```

所有你需要的信息都存在于 Oracle 数据字典中，大多数已经实施分区的站点也拥有一系列存储过程，用于轻松管理大量分区。此外，像 Enterprise Manager 这样的许多 GUI 工具也内置了执行这些操作的能力，无需你手动输入单个命令。

关于分区和管理需要考虑的另一个因素是在数据仓库和归档中使用数据的 `sliding windows`（滑动窗口）。在许多情况下，你需要保留最近 N 个时间单位内的在线数据。例如，假设你需要保留最近 12 个月或最近 5 年的数据在线。如果没有分区，这通常是一次大规模的 `INSERT` 操作后跟一次大规模的 `DELETE` 操作，导致产生两个大型事务。大量的 DML 操作，以及生成大量的 redo 和 undo。现在有了分区，你可以简单地执行以下步骤：

1.  将新月份（或年份，或任何时间单位）的数据加载到一个单独的表中。

2.  对该表进行完全索引。（这些步骤甚至可以在另一个实例中完成，然后传输到该数据库）。

3.  使用快速的 DDL 命令 `ALTER TABLE EXCHANGE PARTITION`，将这个新加载并索引好的表附加到分区表的末尾。

4.  从分区表的另一端分离出最旧的分区。

因此，你现在可以非常容易地支持包含时间敏感信息的超大对象。旧数据可以很容易地从分区表中移除，如果你不需要它，可以简单地 `dropped`（删除），或者可以将其归档到其他地方。新数据可以加载到单独的表中，这样在加载、索引等操作完成之前不会影响分区表。稍后我们将查看一个完整的滑动窗口示例。

简而言之，分区可以使原本艰巨的、或者在某些情况下不可行的操作，变得像在小型数据库中一样容易。

## 提升语句性能

分区的第三个普遍（潜在）益处在于增强语句（`SELECT`、`INSERT`、`UPDATE`、`DELETE`、`MERGE`）性能的领域。我们将考察两类语句——那些修改信息的语句和那些只读取信息的语句——并讨论在每种情况下我们可能期望从分区中获得什么好处。

## 并行 DML

修改数据库中数据的语句有可能执行 `parallel DML`（并行 DML）（`PDML`）。在 PDML 期间，Oracle 使用多个线程或进程来执行你的 `INSERT`、`UPDATE`、`DELETE` 或 `MERGE` 操作，而不是单个串行进程。在具有充足 I/O 带宽的多 CPU 机器上，对于大规模 DML 操作，潜在的提速可能很大。

注意

我们将在第 14 章更详细地讨论并行操作。

### 查询性能

在严格的只读查询性能（`SELECT` 语句）领域，分区通过两种特殊类型的操作发挥作用：

*   `Partition elimination`（分区消除）：数据的某些分区在查询处理过程中不被考虑。我们已经看到了一个分区消除的例子。

*   `Parallel operations`（并行操作）：例如并行全表扫描和并行索引范围扫描。

然而，你能从中获得多少益处在很大程度上取决于你所使用的系统类型。

#### OLTP 系统

你不应该指望分区能大幅提高 OLTP 系统中的查询性能。事实上，在传统的 OLTP 系统中，你必须谨慎应用分区，以免对运行时性能产生*负面*影响。在传统的 OLTP 系统中，大多数查询期望几乎瞬间返回，并且大多数从数据库的检索期望是通过非常小的索引范围扫描完成的。因此，前面列出的主要性能优势不会显现出来。分区消除在对大对象进行全表扫描时很有用，因为它允许你避免扫描对象的大部分。然而，在 OLTP 环境中，*你并不会*对大对象进行全表扫描（如果你这么做了，说明存在严重的设计缺陷）。即使你对索引进行了分区，通过扫描更小的索引所获得的任何性能提升都将是微乎其微的——如果你真的获得了速度提升的话。如果你的某些查询使用索引*并且*它们无法排除除一个分区以外的所有分区，你可能会发现你的查询在分区后实际上运行得更慢，因为现在你需要探测 5、10 或 20 个小索引，而不是 1 个较大的索引。稍后在我们研究可用的分区索引类型时，我们将更详细地探讨这一点。

至于并行操作，正如我们将在下一章更详细探讨的，你不希望在 OLTP 系统中进行并行查询。你会将并行操作的使用保留给 DBA 用于重建、创建索引、收集统计信息等。事实是，在 OLTP 系统中，你的查询应该已经以非常快的索引访问为特征，而分区不会（或几乎不会）加速这一点。这并不意味着你应该避免在 OLTP 中使用分区；它的意思是，你不应该期望分区能带来性能上的巨大提升。大多数 OLTP 应用程序无法利用分区能够提升查询性能的场景，但你仍然可以从其他可能的分区优势中获益：管理便捷、更高可用性和减少竞争。



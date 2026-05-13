# 第一部分：高级基础

### 1. 未充分利用的功能与增强特性

随着每个 Oracle 数据库版本的发布，越来越多的功能被添加进来，以使数据库开发更轻松、更简单和/或性能更高。其中一些增强功能对于普通开发人员来说仍然相对陌生，这很可惜。Oracle 数据库功能丰富，在你自行构建解决方案之前，应该充分发挥其潜力。本章的重点是那些值得更多关注的功能，例如批量操作、复合触发器和错误日志等。本章简要介绍并举例说明如何使用它们，以提高意识并供你进一步探索。有关示例中使用的表，请参阅引言。


### 合并

假设你想根据 `RESULTS`（结果）表来更新 `CONSTRUCTORRESULTS`（ constructors 结果）表。你可以创建一个 PL/SQL 块来判断哪些行已存在需要更新、哪些需要新建。但你可以用一条 `merge`（合并）语句来完成这一切。

`merge` 语句（在 Oracle 数据库 9i 中引入）根据数据在表中的存在情况，有条件地向表中插入或更新数据。如果记录不存在，则插入；如果记录已存在，则更新。可以在 `matched`（匹配）子句中添加一个可选的 `delete where`（删除条件）子句。这仅删除那些同时符合 `on`（关联）子句和 `delete where` 子句的行。此技术在代码清单 1-1 中展示。

```
merge into constructorresults tgt
using (select rsl.raceid          as raceid
, rsl.constructorid   as constructorid
, sum( rsl.points )   as points
from   f1data.results rsl
where  rsl.raceid = g_raceids(indx).raceid
and    rsl.constructorid = g_raceids(indx).constructorid
group  by rsl.raceid
, rsl.constructorid) src
on (    tgt.raceid = src.raceid
and tgt.constructorid = src.constructorid )
when matched
then
update
set    tgt.points = src.points
when not matched
then
insert
( constructorresultsid
, raceid
, constructorid
, points)
values
( f1data.constructorresults_seq.nextval
, src.raceid
, src.constructorid
, src.points)
/
代码清单 1-1
将 RESULTS 表合并到 CONSTRUCTORRESULTS 表
```

### 集合

集合（Collections）在 PL/SQL 中自其被引入 Oracle 数据库起就已存在。如今，你可以选择三种类型的集合。

*   `associative array`（关联数组）
*   `nested table`（嵌套表）
*   `varray`（可变数组）

`associative array` 仅在 PL/SQL 中可用。其他两种在 SQL 中也可用，因此它们可以作为表中的列数据类型或独立的 SQL 对象使用。

这些集合有许多相似之处，但也有一些重要区别，如表 1-1 所述。

表 1-1
集合比较

|   | `关联数组` | `嵌套表` | `可变数组` |
| --- | --- | --- | --- |
| **使用的语言** | 仅 PL/SQL | SQL 和 PL/SQL | SQL 和 PL/SQL |
| **索引类型** | （字母）数字 | 数字 | 数字 |
| **范围** | 不受限制 – 2³¹+1 到 2³¹–1 | 不受限制 1 到 2³¹–1 | 受限制 |
| **稀疏性** | 稀疏 | 初始密集/删除后稀疏 | 密集 |
| **有序性** | 无序 | 无序 | 有序 |
| **存储在数据库表中时** | 不适用 | 行外存储 | 行内存储 |
| **用途** | 任何数据集 | 任何数据集 | 小型数据集 |

当将数据从数据库选取到 PL/SQL 变量中或将数据从 PL/SQL 变量写回数据库时，你可以逐行进行（也称为慢速逐行处理）。但是，自 Oracle 数据库 8i 起，你就有了批量操作可用。要实现这一点，集合至关重要。

#### 批量收集

你可以使用 `bulk collect`（批量收集）选项从数据库中检索数据。与其从数据库中选择单条记录，不如从数据库中选择整个记录集，并在一次数据库访问中将它们存储在一个集合中。

如果你想仅从数据库中检索荷兰籍车手，你必须创建一个集合类型来保存数据。在这种情况下，基于游标的结构创建一个记录的集合。

```
declare
cursor c_dutch_drivers
is
select *
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
;
type dutch_drivers_tt is table of f1data.drivers%rowtype
index by pls_integer;
```

然后基于此集合类型创建一个变量。

```
l_dutch_drivers dutch_drivers_tt;
```

接着在游标中，使用 `bulk collect` 批量收集到集合变量中，而不是提取到单个记录。

```
begin
open  c_dutch_drivers;
fetch c_dutch_drivers
bulk  collect
into  l_dutch_drivers;
close c_dutch_drivers;
```

这样，整个结果集在一次数据库访问中被提取到本地变量中。

此语句在找不到数据时不会生成错误；相反，它返回一个空集合。因此，在开始处理集合之前，良好的做法是先检查是否有内容需要处理。

```
if l_dutch_drivers.count > 0
then
for indx in l_dutch_drivers.first ..
l_dutch_drivers.last
loop
dbms_output.put_line(  l_dutch_drivers( indx ).forename
|| ' '
|| l_dutch_drivers( indx ).surname);
end loop;
end if;
end;
```

代码清单 1-2 是完整的匿名块。

```
declare
cursor c_dutch_drivers
is
select drv.*
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
;
type dutch_drivers_tt is table of f1data.drivers%rowtype
index by pls_integer;
l_dutch_drivers dutch_drivers_tt;
begin
open  c_dutch_drivers;
fetch c_dutch_drivers
bulk  collect
into  l_dutch_drivers;
close c_dutch_drivers;
if l_dutch_drivers.count > 0
then
for indx in l_dutch_drivers.first ..
l_dutch_drivers.last
loop
dbms_output.put_line(  l_dutch_drivers( indx ).forename
|| ' '
|| l_dutch_drivers( indx ).surname);
end loop;
end if;
end;
/
代码清单 1-2
显示荷兰籍车手的姓名
```

##### 限制

如果你将结果集 `bulk collect` 批量收集到一个集合变量中，请注意所有数据都会传输到内存中。这可能会占用大量内存，并且由于可能有多个进程正在运行并执行类似的查询，你可能会耗尽可用内存。Oracle 数据库提供了 `limit`（限制）子句来限制使用的内存量。

这需要多写一点代码，因为你必须构建一个循环来分多次获取所有记录。代码清单 1-3 实现了相同的代码，但现在使用了 `limit` 子句。

```
declare
cursor c_dutch_drivers
is
select drv.*
from   f1data.drivers drv
where  drv.nationality = 'Dutch'
;
type dutch_drivers_tt is table of f1data.drivers%rowtype
index by pls_integer;
l_dutch_drivers dutch_drivers_tt;
begin
open  c_dutch_drivers;
loop
fetch c_dutch_drivers
bulk  collect
into  l_dutch_drivers
limit 5;
dbms_output.put_line(  '----- '
|| to_char( l_dutch_drivers.count )
|| ' -----');
if l_dutch_drivers.count > 0
then
for indx in l_dutch_drivers.first ..
l_dutch_drivers.last
loop
dbms_output.put_line(  l_dutch_drivers( indx ).forename
|| ' '
|| l_dutch_drivers( indx ).surname);
end loop;
else
exit;
end if;
end loop;
close c_dutch_drivers;
end;
/
----- 5 -----
Michael Bleekemolen
Boy Lunger
Roelof Wunderink
Gijs van Lennep
Christijan Albers
----- 5 -----
Robert Doornbos
Jos Verstappen
Jan Lammers
Huub Rothengatter
Dries van der Lof
----- 5 -----
Jan Flinterman
Giedo van der Garde
Max Verstappen
Carel Godin de Beaufort
Ernie de Vos
----- 2 -----
Ben Pon
Rob Slotemaker
----- 0 -----
PL/SQL 过程已成功完成。
代码清单 1-3
使用 limit 子句显示荷兰籍车手的姓名
```

## forall

既然你可以从数据库中批量获取数据，你也可以使用 `forall` 语句将数据批量写回数据库。

`forall` 语句的语法看起来很像循环语句，但它不是将单个语句发送到 SQL 引擎，而是将它们打包在一起一次性发送。

你可以用三种方式使用 `forall` 语句。最简单的使用方法是为密集集合指定一个范围。如果你的集合是稀疏的，你可以使用 `indices of`（索引）子句。如果你想将你集合的值用作指向另一个集合的指针，你可以使用 `values of`（值）子句。对于这些示例，创建一个表来保存拥有固定车手编号的车手。

```
create table drivers_with_number
as
select drv.*
from   f1data.drivers drv
where  1=2
/
```

注意
所有这些示例都可以使用单条 SQL 语句完成，但此处是为了展示不同的可能用法。


#### 范围

当你在语句中指定要使用的集合的起始和结束索引时，会使用到 `forall` 语句的（隐式）`range` 选项。该集合必须是一个密集集合。清单 1-4 展示了一个在 `forall` 语句中使用 `range` 子句的示例。如果你指定的范围中缺少某个元素，将会引发异常。

```sql
declare
  cursor c_drivers
  is
    select drv.*
    from   f1data.drivers drv
    where  drv.driver_number is not null
    order  by drv.dob
  ;
  type drivers_tt is table of f1data.drivers%rowtype
    index by pls_integer;
  l_drivers drivers_tt;
begin
  open  c_drivers;
  fetch c_drivers
    bulk  collect into l_drivers;
  close c_drivers;
  if l_drivers.count > 0
  then
    forall indx in l_drivers.first .. l_drivers.last
      insert into drivers_with_number
        values l_drivers( indx );
  end if;
end;
/
```

清单 1-4
在 `forall` 中使用 `range` 选项

```sql
ORA-22160: element at index [xxx] does not exist
```

### indices of

如果你有一个稀疏集合（即，在该范围中缺少某些元素），你可以使用 `indices of` 选项。与之前的例子（首先就没有选择没有固定 `driver_number` 的车手）不同，你现在将所有车手从数据库选择到我们的集合中。

```sql
open c_drivers;
fetch c_drivers
  bulk  collect into l_drivers;
close c_drivers;
```

然后从集合中删除没有固定 `driver_number` 的车手，创建一个稀疏集合。

```sql
for indx in l_drivers.first .. l_drivers.last
loop
  if l_drivers( indx ).driver_number is null
  then
    l_drivers.delete( indx );
  end if;
end loop;
```

现在你有了一个稀疏集合，使用 `forall` 语句的 `indices of` 选项将这些行从稀疏集合插入到数据库中。

```sql
forall indx in indices of l_drivers
  insert into drivers_with_number values l_drivers( indx );
```

清单 1-5 是完整的脚本。

```sql
declare
  cursor c_drivers
  is
    select drv.*
    from   f1data.drivers drv
  ;
  type drivers_tt is table of f1data.drivers%rowtype
    index by pls_integer;
  l_drivers drivers_tt;
begin
  open c_drivers;
  fetch c_drivers
    bulk  collect into l_drivers;
  close c_drivers;
  if l_drivers.count > 0
  then
    for indx in l_drivers.first .. l_drivers.last
    loop
      if l_drivers( indx ).driver_number is null
      then
        l_drivers.delete( indx );
      end if;
    end loop;
  end if;
  if l_drivers.count > 0
  then
    forall indx in indices of l_drivers
      insert into drivers_with_number values l_drivers( indx );
  end if;
end;
/
```
```sql
select dwn.driverid      as id
, dwn.driverref     as ref
, dwn.driver_number as num
from   drivers_with_number dwn
order  by dwn.driver_number
/
```
```
ID REF                  NUM
----- -------------------- ---
838 vandoorne              2
817 ricciardo              3
846 norris                 4
820 chilton                4
20 vettel                 5
3 rosberg                6
849 latifi                 6
8 raikkonen              7
154 grosjean               8
853 mazepin                9
828 ericsson               9
155 kobayashi             10
842 gasly                 10
815 perez                 11
831 nasr                  12
813 maldonado             13
4 alonso                14
844 leclerc               16
824 jules_bianchi         17
840 stroll                18
13 massa                 19
825 kevin_magnussen       20
821 gutierrez             21
18 button                22
852 tsunoda               22
848 albon                 23
855 zhou                  24
818 vergne                25
826 kvyat                 26
807 hulkenberg            27
843 brendon_hartley       28
829 stevens               28
835 jolyon_palmer         30
839 ocon                  31
830 max_verstappen        33
845 sirotkin              35
1 hamilton              44
827 lotterer              45
854 mick_schumacher       47
850 pietro_fittipaldi     51
834 rossi                 53
832 sainz                 55
847 russell               63
822 bottas                77
9 kubica                88
837 haryanto              88
851 aitken                89
836 wehrlein              94
833 merhi                 98
841 giovinazzi            99
16 sutil                 99
51 rows selected
```
清单 1-5
在 `forall` 中使用 `indices of` 选项



## `values of` 子句的用法

如果你想将一个集合用作另一个集合中的一组指针，可以使用 `values of` 子句。

将表中的所有行获取到我们的集合后，用具有 `driver_number` 值的记录填充另一个集合 (`l_drivers_with_number`)。该集合的索引就是 `driver_number`。同时，将 `driver_number` 保存到另一个不同的集合 (`l_driver_numbers`) 中。这些就是我们指向 `l_drivers_with_number` 集合的指针。

```sql
for indx in l_drivers.first .. l_drivers.last
loop
    if l_drivers( indx ).driver_number is not null
    then
        l_drivers_with_number(l_drivers( indx ).driver_number) :=
            l_drivers( indx );
        l_driver_numbers( l_driver_numbers.count + 1 ) :=
            l_drivers( indx ).driver_number;
    end if;
end loop;
```

使用这两个集合，你可以利用 `values of` 子句将行插入数据库。`l_driver_numbers` 集合的值就是 `l_drivers_with_number` 集合的索引。

```sql
forall indx in values of l_driver_numbers
    insert into drivers_with_number
    values l_drivers_with_number( indx );
```

代码清单 1-6 展示了完整的脚本。

```sql
declare
    cursor c_drivers
    is
        select drv.*
        from   f1data.drivers drv
    ;
    type drivers_tt is table of f1data.drivers%rowtype
        index by pls_integer;
    type driver_numbers_tt is table of pls_integer
        index by pls_integer;
    l_drivers             drivers_tt;
    l_drivers_with_number drivers_tt;
    l_driver_numbers      driver_numbers_tt;
begin
    open c_drivers;
    fetch c_drivers
    bulk  collect into l_drivers;
    close c_drivers;
    if l_drivers.count > 0
    then
        for indx in l_drivers.first .. l_drivers.last
        loop
            if l_drivers( indx ).driver_number is not null
            then
                l_drivers_with_number(l_drivers( indx ).driver_number) :=
                    l_drivers( indx );
                l_driver_numbers( l_driver_numbers.count + 1 ) :=
                    l_drivers( indx ).driver_number;
            end if;
        end loop;
    end if;
    if l_driver_numbers.count > 0
    then
        forall indx in values of l_driver_numbers
            insert into drivers_with_number
            values l_drivers_with_number( indx );
    end if;
end;
/
select dwn.driverid      as id
     , dwn.driverref     as ref
     , dwn.driver_number as num
from   drivers_with_number dwn
order  by dwn.driver_number
/
```

```
ID REF                  NUM
----- -------------------- ---
  838 vandoorne              2
  817 ricciardo              3
  846 norris                 4
  846 norris                 4
   20 vettel                 5
  849 latifi                 6
  849 latifi                 6
    8 raikkonen              7
  154 grosjean               8
  853 mazepin                9
  853 mazepin                9
  842 gasly                 10
  842 gasly                 10
  815 perez                 11
  831 nasr                  12
  813 maldonado             13
    4 alonso                14
  844 leclerc               16
  824 jules_bianchi         17
  840 stroll                18
   13 massa                 19
  825 kevin_magnussen       20
  821 gutierrez             21
  852 tsunoda               22
  852 tsunoda               22
  848 albon                 23
  855 zhou                  24
  818 vergne                25
  826 kvyat                 26
  807 hulkenberg            27
  843 brendon_hartley       28
  843 brendon_hartley       28
  835 jolyon_palmer         30
  839 ocon                  31
  830 max_verstappen        33
  845 sirotkin              35
    1 hamilton              44
  827 lotterer              45
  854 mick_schumacher       47
  850 pietro_fittipaldi     51
  834 rossi                 53
  832 sainz                 55
  847 russell               63
  822 bottas                77
  837 haryanto              88
  837 haryanto              88
  851 aitken                89
  836 wehrlein              94
  833 merhi                 98
  841 giovinazzi            99
  841 giovinazzi            99

51 行已选择。
```

代码清单 1-6：使用带有 `values of` 选项的 `forall`

如你所见，这导致了重复记录，因为 `driver_number` 在 `l_drivers_with_number` 集合中被用作索引，因此可能会覆盖现有记录。例如，索引 99 处的条目可能首先获得 Sutil 的值，但后来被 Giovinazzo 覆盖。这导致 `l_driver_numbers` 集合中有两个条目，但它们都指向 `l_drivers_with_number` 集合中的同一条记录，从而导致 `forall` 语句两次插入同一条记录。

#### 批量异常

本章中的“DML 错误日志记录”一节讨论了批量异常。

### 复合触发器

在对表（或视图）执行 DML 时，Oracle 数据库可以在语句执行的四个不同时间点触发触发器。

*   触发语句执行前
*   触发语句影响的每一行之前
*   触发语句影响的每一行之后
*   触发语句执行后

可以为每个 DML 操作（`insert`、`update` 和 `delete`）构建不同的触发器。

复合触发器的概念是在 Oracle Database 11g 中引入的。所有操作都可以构建到一个复合触发器中。这不仅限制了所需对象（从而也是源文件）的数量，而且复合触发器允许使用变量在所有触发点之间共享状态。复合触发器有一个声明部分，类似于包的声明部分。公共状态在触发语句开始时创建，并在触发器完成时销毁。

在引入复合触发器之前，相同的（并且现在仍然可以）通过使用包来共享数据来实现，只是你必须自己负责数据结构的创建和销毁。以下是复合触发器的框架，并附有每个部分的简短说明。

```sql
create or replace trigger compound_trigger_name
for [insert | delete | update] [of column] on table
compound trigger
    -- 声明部分（可选）
    -- 此处声明的变量具有触发语句持续时间。
    -- 在 DML 语句之前执行
    before statement is
    begin
        null;
    end before statement;
    -- 在每一行更改之前执行 - :new, :old 可用
    before each row is
    begin
        null;
    end before each row;
    -- 在每一行更改之后执行 - :new, :old 可用
    after each row is
    begin
        null;
    end after each row;
    -- 在 DML 语句之后执行
    after statement is
    begin
        null;
    end after statement;
end compound_trigger_name;
/
```

存在两个常见用例，其中复合触发器优于传统的触发器和包组合。第一个是表上的自定义日志记录，第二个是变通解决变异表问题。这两个用例将在接下来的章节中更详细地描述。


### 日志记录

当需要对某个表上的所有事务保留日志表时，使用复合触发器可以通过批量处理来加速这个过程。你可以在复合触发器中将受影响行的数据累积到集合中，然后使用批量操作将这些数据推送到日志表中，而不是在每一行处理后都向日志表插入一行。

首先，创建一个日志表。

```sql
create table constructors_jn
as
select constructorid
, constructorref
, name
, nationality
, url
, CAST( null as varchar2( 1 ) ) jn_action
, systimestamp jn_date
from   f1data.constructors
where  1 = 2
/
```

你可以为插入、更新和删除事件创建一个单一的复合触发器。

```sql
create or replace trigger tr_constructors_cti
for insert or update or delete on constructors
compound trigger
```

在复合触发器中，为日志记录声明一个集合类型。

```sql
-- 声明部分（可选）
type constructors_tt is table of constructors_jn%rowtype
index by pls_integer;
```

接下来，声明一个全局变量来保存该集合。

```sql
-- 此处声明的变量具有触发语句的持续时间。
g_constructors_jn constructors_tt;
```

然后，在 `after each row` 部分记录日志表所需的所有信息。

```sql
-- 在每行变更后执行 - :new, :old 可用
after each row is
l_constructor constructors_jn%rowtype;
begin
if inserting
or updating
then
l_constructor.constructorid  := :new.constructorid;
l_constructor.constructorref := :new.constructorref;
l_constructor.name := :new.name;
l_constructor.nationality    := :new.nationality;
l_constructor.url  := :new.url;
else
l_constructor.constructorid  := :old.constructorid;
l_constructor.constructorref := :old.constructorref;
l_constructor.name := :old.name;
l_constructor.nationality    := :old.nationality;
l_constructor.url  := :old.url;
end if;
if inserting
then
l_constructor.jn_action := 'I';
elsif updating
then
l_constructor.jn_action := 'U';
else
l_constructor.jn_action := 'D';
end if;
l_constructor.jn_date := systimestamp;
g_constructors_jn(g_constructors_jn.count + 1) :=
l_constructor;
end after each row;
```

最后，在 `after statement` 部分，执行批量操作将数据推送到数据库。

```sql
-- 在 DML 语句之后执行
after statement is
begin
forall indx in g_constructors_jn.first ..
g_constructors_jn.last
insert
into   constructors_jn
values g_constructors_jn( indx );
end after statement;
```

清单 1-7 是完整的复合触发器。

**提示**

如果你预计日志集合会变得非常大，可以在复合触发器的 `AFTER EACH ROW` 部分，当集合大小达到某个阈值时，构建一个 `FORALL BULK INSERT`，然后清空该集合。

```sql
create or replace trigger tr_constructors_cti
for insert or update or delete on constructors
compound trigger
-- 声明部分（可选）
type constructors_tt is table of constructors_jn%rowtype
index by pls_integer;
-- 此处声明的变量具有触发语句的持续时间。
g_constructors_jn constructors_tt;
-- 在每行变更后执行 - :new, :old 可用
after each row is
l_constructor constructors_jn%rowtype;
begin
if inserting
or updating
then
l_constructor.constructorid  := :new.constructorid;
l_constructor.constructorref := :new.constructorref;
l_constructor.name           := :new.name;
l_constructor.nationality    := :new.nationality;
l_constructor.url            := :new.url;
else
l_constructor.constructorid  := :old.constructorid;
l_constructor.constructorref := :old.constructorref;
l_constructor.name           := :old.name;
l_constructor.nationality    := :old.nationality;
l_constructor.url            := :old.url;
end if;
if inserting
then
l_constructor.jn_action := 'I';
elsif updating
then
l_constructor.jn_action := 'U';
else
l_constructor.jn_action := 'D';
end if;
l_constructor.jn_date := systimestamp;
g_constructors_jn(g_constructors_jn.count + 1) :=
l_constructor;
end after each row;
-- 在 DML 语句之后执行
after statement is
begin
forall indx in g_constructors_jn.first ..
g_constructors_jn.last
insert
into   constructors_jn
values g_constructors_jn( indx );
end after statement;
end tr_constructors_cti;
/
```

**清单 1-7**
复合触发器 `tr_constructors_cti`

### 变异表

假设你想在将比赛结果添加到 `RESULTS` 表时更新 `CONSTRUCTORRESULTS` 表。如果你尝试在触发器中执行此操作，如清单 1-8 所示，当你尝试在 `RESULTS` 表中插入或更新行时，会收到错误。

```sql
create or replace trigger tr_results_ariu
after insert or update on f1data.results
for each row
begin
merge into f1data.constructorresults tgt
using (select rsl.raceid        as raceid
, rsl.constructorid as constructorid
, sum( rsl.points ) as points
from   f1data.results rsl
where  rsl.raceid        = :new.raceid
and    rsl.constructorid = :new.constructorid
group  by rsl.raceid
, rsl.constructorid) src
on (    tgt.raceid        = src.raceid
and tgt.constructorid = src.constructorid)
when matched then
update
set    tgr.points = src.points
when not matched then
insert
( constructorresultsid
, raceid
, constructorid
, points)
values
( f1data.constructorresults_seq.nextval
, src.raceid
, src.constructorid
, src.points);
end tr_results_ariu;
/
```

**清单 1-8**
在 `RESULTS` 上的行后触发器，用于更新 `CONSTRUCTORRESULTS`

```
ORA-04091: table F1DATA.RESULTS is mutating, trigger/function may not see it
```

你可以通过创建一个复合触发器来解决这个问题，该触发器在 after row 部分将不同的构造函数添加到一个集合中，然后在 after statement 触发器中使用该集合来执行实际的合并语句。

```sql
create or replace trigger f1data.tr_results_cti
for insert or update or delete on f1data.results
compound trigger
-- 声明部分（可选）
type raceid_t is record
( raceid        number
, constructorid number);
type raceids_tt is table of raceid_t
index by pls_integer;
-- 此处声明的变量具有触发语句的持续时间。
g_raceids raceids_tt;
-- 在 DML 语句之前执行
before statement is
begin
null;
end before statement;
-- 在每行变更后执行 - :new, :old 可用
after each row is
begin
g_raceids(:new.constructorid).raceid :=
:new.raceid;
g_raceids(:new.constructorid).constructorid :=
:new.constructorid;
end after each row;
-- 在 DML 语句之后执行
after statement is
begin
forall indx in indices of g_raceids
merge into f1data.constructorresults tgt
using (select rsl.raceid
, rsl.constructorid
, sum( rsl.points ) as points
from   f1data.results rsl
where  rsl.raceid =
g_raceids( indx ).raceid
and    rsl.constructorid =
g_raceids(indx).constructorid
group  by rsl.raceid
,rsl.constructorid) src
on (    tgt.raceid        = src.raceid
and tgt.constructorid = src.constructorid)
when matched then
update
set    tgt.points = src.points
when not matched then
insert
( constructorresultsid
, raceid
, constructorid
, points)
values
( f1data.constructorresults_seq.nextval
, src.raceid
, src.constructorid
, src.points);
end after statement;
end tr_results_cti;
/
```

**清单 1-9**
在 `RESULTS` 上的复合触发器，用于更新 `CONSTRUCTORRESULTS`

### DML 错误日志

执行 DML 时，可能会出现问题。例如，当违反约束时，操作无法完成，事务会被回滚。如果你正在处理单条记录，这不是问题：要么修复数据并重新运行语句，要么将错误记录到某处。但是，如果你运行一条插入一百万行的语句，而第一百万行失败了，整个事务都会被回滚。Oracle 数据库提供了一种机制来解决这个问题。

#### 记录错误

创建一个表，用于存放所有被分配了永久编号的车手。

```
create table drivers_with_number
as
select drv.*
from   f1data.drivers drv
where  1=2
/
```

为确保永久编号的唯一性，需要为该表添加一个唯一性约束。

```
alter table drivers_with_number add constraint un_driver_number
unique (driver_number)
/
```

要使用此功能，必须创建一个表来保存有问题的记录。`dbms_errlog` 包提供了一个名为 `create_error_log` 的过程来简化此操作。

```
procedure create_error_log
( dml_table_name varchar2
, err_log_table_name  varchar2 default NULL
, err_log_table_owner varchar2 default NULL
, err_log_table_space varchar2 default NULL
, skip_unsupported    boolean  default FALSE
);
```

使用此过程时，可以指定所有默认值，如下所示。

```
begin
dbms_errlog.create_error_log
(dml_table_name => 'drivers_with_number');
end;
/
```

这将在当前模式中创建一个名为 `ERR$_DRIVERS_WITH_NUMBER` 的表。该表包含五个描述错误的列。

```
desc err$_drivers_with_number
Name            Type            Nullable Default Comments
--------------- --------------- -------- ------- --------
ORA_ERR_NUMBER$ NUMBER          Y
ORA_ERR_MESG$   VARCHAR2(2000)  Y
ORA_ERR_ROWID$  UROWID          Y
ORA_ERR_OPTYP$  VARCHAR2(2)     Y
ORA_ERR_TAG$    VARCHAR2(2000)  Y
```

并且，对于源表中的每一列，它都会创建一个 `VARCHAR2` 类型的列。如果原始列的类型是 `VARCHAR2`，则新列也是 `VARCHAR2` 类型，但长度为最大字符串大小。其他所有列的类型都是 `VARCHAR2(4000)`。在 `max_string_size=extended` 的数据库中，您会看到类似下面这样类型为 `VARCHAR2(32767)` 的列。

```
DRIVERID        VARCHAR2(4000)  Y
DRIVERREF       VARCHAR2(32767) Y
DRIVER_NUMBER   VARCHAR2(4000)  Y
CODE            VARCHAR2(32767) Y
FORENAME        VARCHAR2(32767) Y
SURNAME         VARCHAR2(32767) Y
DOB             VARCHAR2(4000)  Y
NATIONALITY     VARCHAR2(32767) Y
URL             VARCHAR2(32767) Y
```

要将此表用作错误日志，必须在 DML 语句中添加 `log errors` 子句。

- `into` 子句允许您指定错误日志表的名称。默认为当前模式下的 `err$_<tablename>`。
- `error_tag` 允许您指定一个字符串，用于标识与此语句相关的记录。
- `reject limit` 允许您在引发异常之前指定希望接受的错误数量。

```
insert into drivers_with_number
select *
from   f1data.drivers drv
where  drv.driver_number is not null
log errors into err$_drivers_with_number ('with limit unlimited') reject limit unlimited
/
```

```
log errors
[into [schema_name.]table_name]
[('error_tag')]
[reject limit integer|unlimited]
```

所有错误都被写入错误日志表。表 1-2 展示了其中一个错误。

表 1-2

一条错误记录

| **ORA_ERR_NUMBER$** | 1 |
| **ORA_ERR_MESG$** | ORA-00001: unique constraint (BOOK.UN_DRIVER_NUMBER) violated |
| **ORA_ERR_ROWID$** |  |
| **ORA_ERR_OPTYP$** | I |
| **ORA_ERR_TAG$** | with limit unlimited |
| **DRIVERID** | 846 |
| **DRIVERREF** | Norris |
| **DRIVER_NUMBER** | 4 |
| **CODE** | NOR |
| **FORENAME** | Lando |
| **SURNAME** | Norris |
| **DOB** | 13-NOV-99 |
| **NATIONALITY** | British |
| **URL** | [`http://en.wikipedia.org/wiki/Lando_Norris`](http://en.wikipedia.org/wiki/Lando_Norris) |

目标表包含未引发错误的记录。所有有问题的记录都被记录在错误日志表中。错误日志表中的数据会被持久化保存，无论您是 `commit` 还是 `rollback` 该事务。

> **注意：** `ORA_ERR_ROWID$` 指向原始记录的行 ID；因此，在执行插入操作时该字段为空。当执行更新操作时，它会包含一个值。

#### 保存异常

在 PL/SQL 中使用 `forall` 语句执行批量 DML 操作时，有一个类似的选择。在这种情况下，错误不会被记录到表中，而是记录到内存中的一个集合里。

Oracle 数据库会引发一个特殊的异常，您可以捕获并处理它。

```
failure_in_forall exception;
pragma exception_init( failure_in_forall, -24381 );
```

所有异常都被保存到一个伪集合中：`sql%bulk_exceptions`。此集合中包含以下可用属性。

- `error_code`: 保存相应的错误代码。
- `error_index`: 保存当前 `forall` 语句的迭代编号。
- `count`: 保存异常的总数量。

```
declare
cursor c_drivers is
select drv.*
from   f1data.drivers drv
where  drv.driver_number is not null
order  by drv.dob;
type drivers_tt is table of f1data.drivers%rowtype
index by pls_integer;
l_drivers drivers_tt;
failure_in_forall exception;
pragma exception_init( failure_in_forall, -24381 );
begin
open  c_drivers;
fetch c_drivers
bulk  collect into l_drivers;
close c_drivers;
if l_drivers.count > 0
then
begin
forall indx in l_drivers.first .. l_drivers.last
save exceptions
insert
into drivers_with_number values l_drivers(indx);
exception
when failure_in_forall then
for indx in 1 .. sql%bulk_exceptions.count
loop
dbms_output.put_line(  'Error '
|| indx
|| ' occurred on index '
|| sql%bulk_exceptions( indx ).
error_index
|| '.'
);
dbms_output.put_line(  'Oracle error is '
|| sqlerrm( -1 *
sql%bulk_exceptions( indx ).
error_code)
);        end loop;
end;
end if;
end;
/
Error 1 occurred on index 25.
Oracle error is ORA-00001: unique constraint (.) violated
Error 2 occurred on index 31.
Oracle error is ORA-00001: unique constraint (.) violated
Error 3 occurred on index 32.
Oracle error is ORA-00001: unique constraint (.) violated
Error 4 occurred on index 36.
Oracle error is ORA-00001: unique constraint (.) violated
Error 5 occurred on index 39.
Oracle error is ORA-00001: unique constraint (.) violated
Error 6 occurred on index 47.
Oracle error is ORA-00001: unique constraint (.) violated
Error 7 occurred on index 50.
Oracle error is ORA-00001: unique constraint (.) violated
Error 8 occurred on index 51.
Oracle error is ORA-00001: unique constraint (.) violated
PL/SQL procedure successfully completed
Listing 1-10
Using forall with save exceptions
```

> **注意：** 错误代码是一个正数，但 Oracle 错误代码是负数，因此必须将代码乘以 -1 才能获得正确的错误消息。

### 总结

本章探讨了如何利用集合和批量操作从数据库检索数据并将其写回数据库表。集合与复合触发器结合用于日志记录，或者用于规避 *变异表* 问题时也非常有用。

本章还涵盖了针对直接 DML（插入、更新、删除）和批量操作的错误处理。

## 2. 分析函数与（逆）透视

数据库应用程序的最终用户经常要求导出 Excel 文件，以便他们能够操作数据来获取做出业务决策所需的信息，这种情况太过常见了。通常，所需的信息可以通过使用 SQL 以最终用户期望的格式检索到，从而消除了导出后手动操作数据的需要。您对 SQL 的了解越深入，就越有能力满足最终用户的各种需求。本章讨论两种重要的技术：分析函数和透视。

分析函数是一个需要掌握的重要工具，尽管它已有超过 23 年的历史，但仍然相对不为人知。

将结果集进行透视是将行转换为列的常见需求，同样地，逆透视则是将列转换为行。


### 分析函数

分析函数在 Oracle 数据库 8.1.6 企业版中被引入，但至今仍鲜为人知。分析函数使得访问多行数据的值成为可能，而无需使用自连接，也不需要 `group by` 子句即可创建聚合值。分析函数大放异彩的另一个例子是确定一组值内的排名。

包含分析函数的查询，其处理顺序分为三个阶段。在第一阶段，解析连接、`where` 条件、`group by` 和 `having` 子句。分析函数在第二阶段应用于结果集。最后，`order by` 作用于完整的结果集。处理顺序之所以重要，是因为有时使用标准的聚合函数可能就足够了。一个典型的迹象是查询同时结合了分析函数和 `distinct` 操作符。这种查询很可能可以用普通的聚合函数来编写。

聚合函数会减少最终结果集的行数——例如 `count(*)` 只返回一行——而分析函数则不然。分析函数会为最终结果集中的每一行返回一个值。

#### 构成要素

使用分析函数的第一步是决定需要哪个函数，即“做什么”。许多聚合函数都有对应的分析函数，例如 `count`、`sum` 或 `avg`。以下示例展示了 `avg` 函数对应的分析函数的骨架。

```
avg () over ()
```

关键字 `over` 表明该函数确实是一个分析函数。

在决定了使用哪个函数之后，下一步是确定该函数应应用于结果集的哪些部分。结果集中的行被“分组”在一起，称为*分区*，分析函数被应用于这些分区。

注意

分析函数上下文中的分区与表分区无关。分析分区是数据的一种逻辑分组。

分区可以小到只有一行，也可以大到整个结果集。分区子句中的标准决定了有多少个不同的组——每个记录至少一个分区，每个结果集至少一个分区。当省略分区子句时，整个结果集被视为单个分区。所选函数的结果被应用于单个分区。它不能跨分区工作。以下示例展示了包含分区子句的分析函数的骨架。

```
avg () over (partition by )
```

在分区内部，可以指定一个*窗口*。窗口决定了当前分区中哪些行范围可用于执行函数。分析函数总是从当前行的角度执行。窗口大小可以基于物理行数，也可以基于逻辑区间，例如数值或时间。

目前有三种选项来定义开窗子句。Oracle 数据库 21c 引入了 groups 窗口。

*   `rows` 指定相对于当前行的行数。可以是当前行之前或之后的行。
*   `range` 指定相对于当前行的值范围。行的数量取决于当前行的值。排序键必须支持加法或减法运算，例如数值、日期或间隔数据类型。
*   `groups` 是根据有序值将数据进行划分的方式。值相等的行属于同一组。指定的偏移量指的是前一组或后一组。

由于开窗子句依赖于有序的结果集，因此必须在分析子句中包含 `order by`。当存在 `order by` 时，开窗子句默认为 `range unbounded preceding and current row`。清单 2-2 是 `rows`、`range` 和 `groups` 之间差异的示例。

除非另有说明，示例中使用 `DRIVERS`、`RACES` 和 `LAPTIMES` 表来演示分析函数的工作原理。

#### 累计总和

使用分析函数创建累计总和（即随着行的推进累积总量）是轻而易举的。`laptimes` 表记录了完成一圈赛道所需的时间（以毫秒为单位）。以下示例筛选了 2022 赛季巴林大奖赛中 Charles Leclerc 的数据，仅使用前 10 圈。

`sum` 函数的分析函数版本被用来创建累计总和。关键字 `over` 表明它是一个分析函数。下面的例子中缺少分区子句，表明整个结果集就是一个分区。然而，由于分析子句中的 `order by` 子句，存在一个随每行扩展的窗口。在这种情况下，隐式的窗口是默认窗口：`range unbounded preceding and current row`，所有前面的圈速都会加到当前行的值上，形成累计总和，同时也包括那些与当前行值相同的圈速。尽管在此情况下它*看起来*有效，但这仅仅是因为圈速值是唯一的。当存在重复值时，所有前面的圈速以及任何与当前行值相同的后续圈速都会被加进来。一个真正的累计总和需要以下显式窗口：`rows between unbounded preceding and current row`。

在清单 2-1 中，结果集中的 `running_total` 列显示了 Charles Leclerc 前十圈的累计总时间。第三行（第 3 圈）显示了前三行（99070 + 97853 + 98272 = 295195）的总时间（以毫秒为单位）。

```
select ltm.lap
, ltm.milliseconds
, sum (ltm.milliseconds) over (
    order by ltm.lap
    rows between unbounded preceding
             and current row
    ) as running_total
from   f1data.laptimes ltm
join   f1data.races rcs
  on   rcs.raceid = ltm.raceid
where  rcs.race_date between trunc (sysdate, 'yy') and sysdate
  and    ltm.driverid = 844 -- Charles Leclerc
  and    rcs.raceid = 1074 -- Bahrain Grand Prix
  and    ltm.lap between 1 and 10
order  by ltm.lap
/
LAP MILLISECONDS RUNNING_TOTAL
---------- ------------ -------------
         1        99070         99070
         2        97853        196923
         3        98272        295195
         4        98414        393609
         5        98471        492080
         6        98712        590792
         7        98835        689627
         8        98951        788578
         9        98807        887385
        10        99123        986508
Listing 2-1
单一车手的累计总和
```

要为每位车手创建累计总和，需要添加分区子句，如下例所示。结果被筛选为仅四位车手的前三圈数据。

```
select ltm.lap
, drv.forename||' '||drv.surname as driver
, ltm.milliseconds
, sum (ltm.milliseconds)
    over (partition by ltm.driverid
          order by ltm.lap) as running_total
from   f1data.laptimes ltm
join   f1data.races rcs
  on   rcs.raceid = ltm.raceid
join   f1data.drivers drv
  on   drv.driverid = ltm.driverid
where  rcs.race_date between trunc (sysdate, 'yy') and sysdate
  and    rcs.raceid = 1074 -- Bahrain Grand Prix
  and    ltm.lap between 1 and 3
  and    ltm.driverid in (1, 815, 830, 844)
order  by ltm.lap
        ,ltm.milliseconds
        ,ltm.driverid
/
LAP DRIVER          MILLISECONDS RUNNING_TOTAL
---------- --------------- ------------ -------------
         1 Charles Leclerc        99070         99070
         1 Max Verstappen        100236        100236
         1 Lewis Hamilton        101555        101555
         1 Sergio Pérez          102993        102993
         2 Charles Leclerc        97853        196923
         2 Max Verstappen         97880        198116
         2 Lewis Hamilton         99002        200557
         2 Sergio Pérez           99092        202085
         3 Charles Leclerc        98272        295195
         3 Max Verstappen         98357        296473
         3 Lewis Hamilton         99075        299632
         3 Sergio Pérez           99473        301558
12 rows selected.
```



最终的`order by`子句（上述查询的最后一行）决定了结果集的排序方式。运行总计可能不会立即显现，但当聚焦于查尔斯·勒克莱尔的运行总计时，其结果与清单 2-1 中第一个查询的结果一致。

为了帮助可视化窗口，可以使用`first_value`和`last_value`函数。在以下查询中，`first_value`和`last_value`与`sum`函数使用相同的分析子句，并分别显示窗口开始（在`win_start`列中）和窗口结束（在`win_end`列中）的圈数。

```sql
select lt.lap
, lt.milliseconds
, sum (lt.milliseconds)
over (partition by lt.driverid
order by lap) as running_total
, first_value (lt.lap)
over (partition by lt.driverid
order by lt.lap) as win_start
, last_value (lt.lap)
over (partition by lt.driverid
order by lt.lap) as win_end
from   f1data.laptimes lt
join   f1data.races r
on   r.raceid = lt.raceid
where  race_date between trunc (sysdate, 'yy') and sysdate
and    r.raceid = 1074 -- 巴林大奖赛
and    lt.lap between 1 and 3
and    lt.driverid = 844 -- 查尔斯·勒克莱尔
order  by lt.lap
,lt.milliseconds
/
LAP MILLISECONDS RUNNING_TOTAL  WIN_START    WIN_END
---------- ------------ ------------- ---------- ----------
1        99070         99070          1          1
2        97853        196923          1          2
3        98272        295195          1          3
```

在上述结果集中，`win_start`列保持值 1 作为窗口起点，而`win_end`列显示该值随行数增加而递增。

注意

当你想知道的是分区（`partition`）中的最后一个值，而非窗口中当前的最后一行时，请指定`unbounded following`。这会将窗口从当前行扩展到最后一行。

自 Oracle Database 21c 起，可以定义可重用的窗口子句。以下查询是前述查询的一个变体，其中窗口可通过`first_value`和`last_value`函数进行可视化。这个可重用的窗口子句被命名为“win”，并在所有三个分析函数中使用。

```sql
select ltm.lap
, ltm.milliseconds
, sum (ltm.milliseconds) over win as running_total
, first_value (ltm.lap) over win as win_start
, last_value (ltm.lap) over win as win_end
from   f1data.laptimes ltm
join   f1data.races rcs
on   rcs.raceid = ltm.raceid
where  rcs.race_date between trunc (sysdate, 'yy') and sysdate
and    rcs.raceid = 1074 -- 巴林大奖赛
and    ltm.lap between 1 and 3
and    ltm.driverid = 844 -- 查尔斯·勒克莱尔
window win as (partition by ltm.driverid
order by ltm.lap)
order  by ltm.lap
, ltm.milliseconds
/
LAP MILLISECONDS RUNNING_TOTAL  WIN_START    WIN_END
---------- ------------ ------------- ---------- ----------
1        99070         99070          1          1
2        97853        196923          1          2
3        98272        295195          1          3
有关可重用窗口子句的更多内容，请参见“可重用窗口子句”一节。
```

#### 访问其他行的值

有几个函数允许访问结果集中其他行的值，而无需自连接。在前面的例子中，已经展示了`first_value`和`last_value`有助于可视化窗口的开始和结束位置。这些函数的一个变体是`nth_value`。

`first_value`从窗口中的第一行检索值，`last_value`从窗口中的最后一行检索值，而`nth_value`允许从窗口中的任意行检索值。

`nth_value`函数的第一个参数指定需要检索的值。第二个参数指定需要返回的行的偏移量。这可以是一个常量、绑定变量、列或涉及它们的表达式（如果解析为正整数）。

在以下示例中，比较了每位车手的当前圈（当前行）与第二圈所用时间之间的差异。与第二圈的差异显示在标记为`diff`的列中。

在此示例中，每一行都与结果集中的第二行进行比较，包括第一行。窗口`rows between unbounded preceding and unbounded following`不仅限于“向后看”，它也可以“向前看”。

刘易斯·汉密尔顿和塞尔吉奥·佩雷斯在第四圈比第二圈更快。

```sql
select ltm.lap
, drv.forename||' '||drv.surname as driver
, ltm.milliseconds
, ltm.milliseconds
- nth_value (ltm.milliseconds, 2)
over (partition by ltm.driverid
order by ltm.lap
rows between unbounded preceding
and unbounded following)
as diff
from   f1data.laptimes ltm
join   f1data.races rcs
on   rcs.raceid = ltm.raceid
join   f1data.drivers drv
on   drv.driverid = ltm.driverid
where  rcs.race_date between trunc (sysdate, 'yy') and sysdate
and    rcs.raceid = 1074 -- 巴林大奖赛
and    ltm.lap between 1 and 4
and    ltm.driverid in (1, 815, 830, 844)
order  by ltm.lap
, ltm.milliseconds
/
LAP DRIVER          MILLISECONDS       DIFF
---------- --------------- ------------ ----------
1 查尔斯·勒克莱尔        99070       1217
1 马克斯·维斯塔潘        100236       2356
1 刘易斯·汉密尔顿        101555       2553
1 塞尔吉奥·佩雷斯          102993       3901
2 查尔斯·勒克莱尔        97853          0
2 马克斯·维斯塔潘         97880          0
2 刘易斯·汉密尔顿         99002          0
2 塞尔吉奥·佩雷斯           99092          0
3 查尔斯·勒克莱尔        98272        419
3 马克斯·维斯塔潘         98357        477
3 刘易斯·汉密尔顿         99075         73
3 塞尔吉奥·佩雷斯           99473        381
4 查尔斯·勒克莱尔        98414        561
4 马克斯·维斯塔潘         98566        686
4 塞尔吉奥·佩雷斯           98741       -351
4 刘易斯·汉密尔顿         98892       -110
已选择 16 行。
```

比较一圈与下一圈的圈速，可以使用`lag`和`lead`函数。使用`lag`可以从前一行检索值，而`lead`则从结果集中的后续行检索值。

以下示例将单一车手的圈速与前一圈的圈速进行比较。因为筛选器只选择了一位车手，所以分析函数中省略了分区子句。从结果集可见，第二圈明显比第一圈快。

```sql
select ltm.lap
, ltm.milliseconds
, ltm.milliseconds
- lag (ltm.milliseconds)
over (partition by ltm.driverid
order by ltm.lap)
as diff
from   f1data.laptimes ltm
join   f1data.races rcs
on   rcs.raceid = ltm.raceid
join   f1data.drivers drv
on   drv.driverid = ltm.driverid
where  rcs.race_date between trunc (sysdate, 'yy') and sysdate
and    rcs.raceid = 1074 -- 巴林大奖赛
and    ltm.lap between 1 and 4
and    ltm.driverid = 830 -- 马克斯·维斯塔潘
order  by ltm.lap
, ltm.milliseconds
/
LAP MILLISECONDS       DIFF
---------- ------------ ----------
1       100236
2        97880      -2356
3        98357        477
4        98566        209
```



# 信息包定义与调用

```
create or replace package information is
procedure display(refcursor_in in sys_refcursor);
end information;
/
create or replace package body information is
type rec_t is record
( id          number(11)
, ref         varchar2(255)
, name        varchar2(255)
, nationality varchar2(255)
);
procedure printit(rec_in in rec_t)
is
begin
dbms_output.put_line
( rec_in.id
||'('
||rec_in.ref
||')'
||rec_in.name
||'-'
||rec_in.nationality
);
end;
procedure display(refcursor_in in sys_refcursor)
is
begin
for rec rec_t in values of refcursor_in loop
printit(rec_in => rec);
end loop;
end display;
end information;
/
代码清单 7-12
显示信息包
```

你可以使用任何 ref 游标来调用此过程，前提是返回的记录类型符合定义的记录结构。

```
declare
rc sys_refcursor;
begin
open rc for select d.driverid
, d.driverref
, d.forename || ' ' || d.surname
, d.nationality
from   f1data.drivers d;
information.display(rc);
end;
/
1(hamilton)Lewis Hamilton-British
2(heidfeld)Nick Heidfeld-German
3(rosberg)Nico Rosberg-German
4(alonso)Fernando Alonso-Spanish
5(kovalainen)Heikki Kovalainen-Finnish
6(nakajima)Kazuki Nakajima-Japanese
7(bourdais)Sébastien Bourdais-French
8(raikkonen)Kimi Räikkönen-Finnish
9(kubica)Robert Kubica-Polish
10(glock)Timo Glock-German
>
declare
rc sys_refcursor;
begin
open rc for select c.constructorid
, c.constructorref
, c.name
, c.nationality
from   f1data.constructors c;
information.display(rc);
end;
/
1(mclaren)McLaren-British
2(bmw_sauber)BMW Sauber-German
3(williams)Williams-British
4(renault)Renault-French
5(toro_rosso)Toro Rosso-Italian
6(ferrari)Ferrari-Italian
7(toyota)Toyota-Japanese
8(super_aguri)Super Aguri-Japanese
9(red_bull)Red Bull-Austrian
10(force_india)Force India-Indian
>
```

#### 可变迭代器

你曾多少次试图给你的迭代器赋值？例如，你想在迭代中跳过任意数量的步骤。考虑代码清单 7-13 中所示的脚本。

```
begin
for i in 1 .. 10
loop
dbms_output.put_line(i);
i := i + 3;
end loop;
end;
/
ORA-06550: line 7, column 7:
PLS-00363: expression 'I' cannot be used as an assignment target
ORA-06550: line 7, column 7:
PL/SQL: Statement ignored
代码清单 7-13
尝试给迭代器赋值
```

Oracle Database 21c 引入了一个新选项：`mutable` 迭代器。在迭代器名称后使用关键字 `mutable` 使其在循环执行期间可被修改，这样代码清单 7-14 中的代码就能运行了。

```
begin
for i mutable in 1 .. 10
loop
dbms_output.put_line(i);
i := i + 3;
end loop;
end;
/

代码清单 7-14
使迭代器可变
```

默认行为是使用 `immutable`（不可变）迭代器，也可以用更冗长的方式指定，如代码清单 7-15 所示。

```
begin
for i immutable in 1 .. 10
loop
dbms_output.put_line(i);
i := i + 3;
end loop;
end;
/
ORA-06550: line 7, column 7:
PLS-00363: expression 'I' cannot be used as an assignment target
ORA-06550: line 7, column 7:
PL/SQL: Statement ignored
代码清单 7-15
使迭代器不可变
```

#### 增强迭代器

数值迭代器除非你明确指定 `mutable` 关键字，否则不是 `mutable` 的，而游标 `for` 循环中的记录却是可变的；也就是说，尽管迭代器本身未被声明为可变，但其各个属性的值是可变的。假设你有一个用于反转字符串的过程，如代码清单 7-16 所示。

```
create or replace procedure
reversestring(string_inout in out varchar2)
is
l_reversed varchar2( 32767 );
l_strlen number;
begin
l_strlen := length( string_inout );
for i in reverse 1..l_strlen
loop
l_reversed := l_reversed || substr( string_inout, i, 1 );
end loop;
string_inout := l_reversed;
end;
/
代码清单 7-16
反转字符串过程
```

如果你创建一个游标 `for` 循环，其中的迭代器是来自游标的一条记录，那么即使该迭代器未定义为可变，你仍然可以给记录的不同元素赋值。这从 Oracle Database 版本 7 开始就已可行。

```
begin
for driver in (select *
from   f1data.drivers d
where  d.nationality = 'Dutch')
loop
dbms_output.put_line( driver.forename
|| ' '
|| driver.surname );
reversestring( driver.forename );
reversestring( driver.surname );
dbms_output.put_line( driver.surname
|| ' '
|| driver.forename );
end loop;
end;
/
```

#### 数组中的循环

你不必像代码清单 7-17 中那样，先创建一个 `for` 循环，然后在循环内部给关联数组赋值；你可以在赋值语句内部定义循环，如代码清单 7-18 所示。

```
declare
type num_list is table of int index by pls_integer;
s1 num_list;
s2 num_list;
begin
s1 := num_list( for i in 1 .. 10 => i );
s2 := num_list( for i in 2 .. 10 by 2 => s1( i ) );
end;
/
代码清单 7-18
在 21c 中于数组内部使用循环
```

```
declare
type num_list is table of int index by pls_integer;
s1 num_list;
s2 num_list;
begin
for i in 1 .. 10
loop
s1(i) := i * 10;
end loop;
for i in 1 .. 10
loop
if mod(i, 2) = 0
then
s2(i) := s1(i);
end if;
end loop;
end;
/
代码清单 7-17
在 21c 之前使用循环赋值
```

##### 数组的索引

与用于单次从数据库检索数据的 `bulk collect` 构造语句互补的是 `forall` 语句，它用于单次将更改写回数据库。`forall` 语句除了对密集集合使用 `<collection>.first .. <collection>.last` 构造外，还可以使用 `indices of` 和 `values of` 构造。从 Oracle Database 21c 开始，可以在 `for` 循环中使用这些构造。你可以使用 `indices of` 构造，如代码清单 7-20 所示，而不必像代码清单 7-19 中展示的那样编写 `while` 循环。

```
declare
type num_list is table of int index by pls_integer;
s2 num_list;
begin
s2 := num_list(for i in 2 .. 10 by 2 => i * 10);
for idx in indices of s2
loop
dbms_output.put_line(idx || '=' || s2(idx));
end loop;
end;
/
2=20
4=40
6=60
8=80
10=100
代码清单 7-20
自 21c 起显示稀疏集合
```

```
declare
type num_list is table of int index by pls_integer;
s2 num_list;
idx int;
begin
s2 := num_list(2  => 20
,4  => 40
,6  => 60
,8  => 80
,10 => 100);
idx := s2.first;
while idx is not null
loop
dbms_output.put_line(idx || '=' || s2(idx));
idx := s2.next(idx);
end loop;
end;
/
2=20
4=40
6=60
8=80
10=100
代码清单 7-19
在 21c 之前显示稀疏集合
```

提示

`s2` 变量是在迭代控制中使用限定表达式初始化的。更多内容请参阅本章后面的限定表达式部分。



##### 数组的值

如果您只对数组的值感兴趣，可以使用 `values of` 操作符调用 `for` 循环，如代码清单 7-21 所示。

```sql
declare
  type num_list is table of int index by pls_integer;
  s2 num_list;
begin
  s2 := num_list(for i in 2 .. 10 by 2 => i * 10);
  for idx in values of s2
  loop
    dbms_output.put_line(idx);
  end loop;
end;
/
```
代码清单 7-21
自 21c 起显示集合的值

迭代器不一定必须是标量；它也可以是一个记录，如代码清单 7-22 所示。

```sql
declare
  type driver_aa is table of f1data.drivers%rowtype
    index by pls_integer;
  l_drivers driver_aa;
  cursor drivers is
    select d.driverid
         , d.driverref
         , d.driver_number
         , d.code
         , d.forename
         , d.surname
         , d.dob
         , d.nationality
         , d.url
    from   f1data.drivers d
    where  d.nationality = 'Dutch';
begin
  l_drivers := driver_aa( for i in drivers
                           index i.driverid => i );
  for driver in values of l_drivers
  loop
    dbms_output.put_line( 'l_drivers['
                        || driver.driverid
                        || ']='
                        || driver.driverref
                        );
  end loop;
end;
/
l_drivers[27]=albers
l_drivers[38]=doornbos
l_drivers[50]=verstappen
l_drivers[136]=lammers
l_drivers[179]=rothengatter
l_drivers[247]=bleekemolen
l_drivers[257]=hayje
l_drivers[295]=wunderink
l_drivers[298]=lennep
l_drivers[430]=beaufort
l_drivers[450]=vos
l_drivers[457]=pon
l_drivers[458]=slotemaker
l_drivers[758]=lof
l_drivers[759]=flinterman
l_drivers[823]=garde
l_drivers[830]=max_verstappen
```
代码清单 7-22
在记录中使用 `values of`

注意

集合的初始化将在本章后面的限定表达式部分进行解释。

##### 数组的对

在 Oracle Database 21c 中，您可以使用 `pairs of` 构造。您无需创建索引迭代器并用它从集合中获取值（参见代码清单 7-23），而是可以在循环中同时获取索引和值作为迭代器（参见代码清单 7-24）。这也适用于稀疏集合。

```sql
declare
  type num_list is table of int index by pls_integer;
  s1 num_list;
  s2 num_list;
begin
  s1 := num_list(10, 20, 30, 40, 50, 60, 70, 80, 90, 1000);
  s2 := num_list(for i in 2 .. 10 by 2 => s1(i));
  for x, y in pairs of s2
  loop
    dbms_output.put_line(x || ',' || y);
  end loop;
end;
/
2,20
4,40
6,60
8,80
10,1000
```
代码清单 7-24
使用 `pairs of` 获取索引和值

```sql
declare
  type num_list is table of int index by pls_integer;
  s1 num_list;
  s2 num_list;
begin
  s1 := num_list(10, 20, 30, 40, 50, 60, 70, 80, 90, 1000);
  s2 := num_list(for i in 2 .. 10 by 2 => s1(i));
  for idx in values of s2
  loop
    dbms_output.put_line(idx);
  end loop;
end;
/
```
代码清单 7-23
使用索引迭代器获取值

值不一定必须是标量值；它也可以是一个记录，如代码清单 7-25 所示。

```sql
declare
  type driver_aa is table of f1data.drivers%rowtype
    index by pls_integer;
  l_drivers driver_aa;
  cursor drivers is
    select d.driverid
         , d.driverref
         , d.driver_number
         , d.code
         , d.forename
         , d.surname
         , d.dob
         , d.nationality
         , d.url
    from   f1data.drivers d
    where  d.nationality = 'Dutch';
begin
  l_drivers := driver_aa( for i in drivers
                           index i.driverid => i );
  for driverid, driver in pairs of l_drivers
  loop
    dbms_output.put_line( 'l_drivers[' || driverid || ']=' ||
                           driver.driverref );
  end loop;
end;
/
l_drivers[27]=albers
l_drivers[38]=doornbos
l_drivers[50]=verstappen
l_drivers[136]=lammers
l_drivers[179]=rothengatter
l_drivers[247]=bleekemolen
l_drivers[257]=hayje
l_drivers[295]=wunderink
l_drivers[298]=lennep
l_drivers[430]=beaufort
l_drivers[450]=vos
l_drivers[457]=pon
l_drivers[458]=slotemaker
l_drivers[758]=lof
l_drivers[759]=flinterman
l_drivers[823]=garde
l_drivers[830]=max_verstappen
```
代码清单 7-25
在记录中使用 `pairs of`

### PL/SQL 限定表达式

自 Oracle Database 18c 起，可以使用限定表达式。限定表达式看起来很像使用命名参数调用过程或函数。

#### 记录

在之前的 Oracle Database 版本中，您可以通过初始化变量然后为记录的所有属性提供值来初始化记录（参见代码清单 7-26）。

```sql
declare
  type circuit_t is record
    ( circuitid  number(11)
    , circuitref varchar2(255)
    , name       varchar2(255)
    , location   varchar2(255)
    , country    varchar2(255)
    , lat        float
    , lng        float
    , alt        number(11)
    , url        varchar2(255));
  l_circuit circuit_t;
begin
  l_circuit := circuit_t();
  l_circuit.circuitid  := 2912;
  l_circuit.circuitref := 'tt_circuit';
  l_circuit.name       := 'TT Circuit Assen';
  l_circuit.location   := 'Assen';
  l_circuit.country    := 'Netherlands';
  l_circuit.lat        := 52.961667;
  l_circuit.lng        := 6.523333;
  l_circuit.url        :=
    'https://en.wikipedia.org/wiki/TT_Circuit_Assen';
  dbms_output.put_line( 'Name     : ' || l_circuit.name );
  dbms_output.put_line( 'Location : ' || l_circuit.location );
  dbms_output.put_line( 'Latitude : ' || l_circuit.lat );
  dbms_output.put_line( 'Lngitude : ' || l_circuit.lng );
  dbms_output.put_line( 'Altitude : ' || l_circuit.alt );
end;
/
Name     : TT Circuit Assen
Location : Assen
Latitude : 52.961667
Lngitude : 6.523333
Altitude :
```
代码清单 7-26
18c 之前初始化变量

也可以通过在初始化时提供这些值作为参数来完成赋值。如果值未知，您可以提供 `NULL`（参见代码清单 7-27）。

```sql
declare
  type circuit_t is record
    ( circuitid  number(11)
    , circuitref varchar2(255)
    , name       varchar2(255)
    , location   varchar2(255)
    , country    varchar2(255)
    , lat        float
    , lng        float
    , alt        number(11)
    , url        varchar2(255));
  l_circuit circuit_t;
begin
  l_circuit := circuit_t
    ( 2912
    , 'tt_circuit'
    , 'TT Circuit Assen'
    , 'Assen'
    , 'Netherlands'
    , 52.961667
    , 6.523333
    , null
    , 'https://en.wikipedia.org/wiki/TT_Circuit_Assen'
    );
  dbms_output.put_line( 'Name     : ' || l_circuit.name );
  dbms_output.put_line( 'Location : ' || l_circuit.location );
  dbms_output.put_line( 'Latitude : ' || l_circuit.lat );
  dbms_output.put_line( 'Lngitude : ' || l_circuit.lng );
  dbms_output.put_line( 'Altitude : ' || l_circuit.alt );
end;
/
Name     : TT Circuit Assen
Location : Assen
Latitude : 52.961667
Lngitude : 6.523333
Altitude :
```
代码清单 7-27
18c 之前在初始化中初始化变量

自 Oracle Database 18c 起，您可以使用限定表达式来初始化记录（参见代码清单 7-28）。

```sql
declare
  type circuit_t is record
    ( circuitid  number(11)
    , circuitref varchar2(255)
    , name       varchar2(255)
    , location   varchar2(255)
    , country    varchar2(255)
    , lat        float
    , lng        float
    , alt        number(11)
    , url        varchar2(255));
  l_circuit circuit_t;
begin
  l_circuit := circuit_t
    ( circuitid  => 2912
    , circuitref => 'tt_circuit'
    , name       => 'TT Circuit Assen'
    , location   => 'Assen'
    , country    => 'Netherlands'
    , lat        => 52.961667
    , lng        => 6.523333
    , url        => 'https://en.wikipedia.org/wiki/TT_Circuit_Assen'
    );
  dbms_output.put_line( 'Name     : ' || l_circuit.name );
  dbms_output.put_line( 'Location : ' || l_circuit.location );
  dbms_output.put_line( 'Latitude : ' || l_circuit.lat );
  dbms_output.put_line( 'Lngitude : ' || l_circuit.lng );
  dbms_output.put_line( 'Altitude : ' || l_circuit.alt );
end;
/
Name     : TT Circuit Assen
Location : Assen
Latitude : 52.961667
Lngitude : 6.523333
Altitude :
```
代码清单 7-28
在 18c 中初始化变量


### 集合

与记录类似，您也可以使用 `qualified expressions` 来初始化集合。在之前的 Oracle 数据库版本中，您必须逐项初始化集合（参见清单 7-29）。

```
declare
  type numbers_tt is table of number index by pls_integer;
  l_numbers numbers_tt;
begin
  l_numbers(72) := 29;
  l_numbers(76) := 6;
  l_numbers(98) := 18;
  l_numbers(69) := 5;
  for indx in indices of l_numbers loop
    dbms_output.put_line( '('
                        ||indx
                        ||') '
                        ||l_numbers(indx)
                        );
  end loop;
end;
/
(69) 5
(72) 29
(76) 6
(98) 18
清单 7-29
18c 版本之前的集合初始化
```

使用限定表达式后，相同的脚本变得更加清晰和简洁（参见清单 7-30）。

```
declare
  type numbers_tt is table of number index by pls_integer;
  l_numbers numbers_tt;
begin
  l_numbers := numbers_tt( 72 => 29
                         , 76 => 6
                         , 98 => 18
                         , 69 => 5
                         );
  for indx in indices of l_numbers loop
    dbms_output.put_line( '('
                        ||indx
                        ||') '
                        ||l_numbers(indx)
                        );
  end loop;
end;
/
(69) 5
(72) 29
(76) 6
(98) 18
清单 7-30
18c 版本中的集合初始化
```

如果您想初始化一个索引从数字 1 开始的稠密集合，Oracle 数据库 21c 不需要使用限定表达式（参见清单 7-31）。

```
declare
  type numbers_tt is table of number index by pls_integer;
  l_numbers numbers_tt;
begin
  l_numbers := numbers_tt( 29, 6, 18, 5 );
  for indx in indices of l_numbers loop
    dbms_output.put_line( '('
                        ||indx
                        ||') '
                        ||l_numbers(indx)
                        );
  end loop;
end;
/
(1) 29
(2) 6
(3) 18
(4) 5
清单 7-31
21c 版本中的稠密集合初始化
```

您还可以直接在构造函数中使用游标。有两种方法可以使用游标填充集合。您可以像使用 `bulk collect` 填充集合那样，将其填充为一个稠密集合。在这种情况下，您在集合中使用 `sequence` 关键字作为索引（参见清单 7-32）。

```
declare
  type circuit_tt is table of f1data.circuits%rowtype
    index by pls_integer;
  l_circuits circuit_tt;
begin
  l_circuits := circuit_tt
    (for rec in
      (select c.circuitid
             , c.circuitref
             , c.name
             , c.location
             , c.country
             , c.lat
             , c.lng
             , c.alt
             , c.url
       from   f1data.circuits c
       order  by c.circuitref
      ) sequence => rec);
  dbms_output.put_line( l_circuits(1).name
                      ||' ('
                      ||l_circuits(1).location
                      ||' ['
                      ||l_circuits(1).lat
                      ||','
                      ||l_circuits(1).lng
                      ||'])'
                      );
end;
/
Circuit Park Zandvoort (Zandvoort [52.3888,4.54092])
清单 7-32
在 21c 版本中使用游标初始化稠密集合
```

有时，使用表的主键作为索引，或者像本例中使用唯一键 `circuitref` 字段更有意义。在这里，您使用 `index` 关键字将游标中的一个字段指定为索引（参见清单 7-33）。

```
declare
  type circuit_tt is table of f1data.circuits%rowtype
    index by varchar2(256);
  l_circuits circuit_tt;
begin
  l_circuits := circuit_tt
    (for rec in
      (select c.circuitid
             , c.circuitref
             , c.name
             , c.location
             , c.country
             , c.lat
             , c.lng
             , c.alt
             , c.url
       from   f1data.circuits c
      ) index rec.circuitref => rec);
  dbms_output.put_line( l_circuits('zandvoort').name
                      ||' ('
                      ||l_circuits('zandvoort').location
                      ||' ['
                      ||l_circuits('zandvoort').lat
                      ||','
                      ||l_circuits('zandvoort').lng
                      ||'])'
                      );
end;
/
Circuit Park Zandvoort (Zandvoort [52.3888,4.54092])
清单 7-33
在 21c 版本中使用游标初始化稀疏集合
```

### 总结

自 Oracle 数据库 21c 起，您在循环中拥有了广泛的可用性。迭代器不必是整数，也可以是非整数或字符串值。迭代器现在可以声明为可变的，因此您可以在循环内部更改其值。迭代控制可以被步进和堆叠。

使用 `qualified expressions` 使您的代码清晰且更具自描述性。


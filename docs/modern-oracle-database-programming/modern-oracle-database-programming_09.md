# DBMS_DB_VERSION 包

Oracle 提供了一个包，其中包含一个用于当前 RDBMS 版本号的常量和一个用于当前发行号的常量。该包的名称为 `DBMS_DB_VERSION`。它还包含一组布尔值，可用于确定当前版本。代码清单 6-1 显示了随 Oracle Database 21c 提供的当前 `DBMS_DB_VERSION` 包。

```sql
package sys.dbms_db_version is
version constant pls_integer :=
21; -- RDBMS 版本号
release constant pls_integer := 0;  -- RDBMS 发行号
/* 以下的布尔常量遵循一种命名约定。
   每个常量为一个布尔表达式命名。
   例如，
   ver_le_9_1  表示 version <=  9 且 release <= 1
   ver_le_10_2 表示 version <= 10 且 release <= 2
   ver_le_10   表示 version <= 10
   引用这些布尔常量（而不是直接引用 version 和 release）的代码，
   将随着 version 和 release 值的变化而受益于细粒度失效。
   这些布尔常量的典型用法是：
   $if dbms_db_version.ver_le_10 $then
   version 10 及更早版本的代码
   $elsif dbms_db_version.ver_le_11 $then
   version 11 代码
   $else
   version 12 及之后版本的代码
   $end
   这种代码结构将保护对 version 12 代码的任何引用。
   它还防止在 version 10 下编译程序时，引用控制包常量 dbms_db_version.ver_le_11。
   对 version 11 的观察也类似。
   即使在 version 10 数据库中未定义静态常量 ver_le_11，此方案也有效，
   因为如果 dbms_db_version.ver_le_10 为 TRUE，条件编译会保护 $elsif 子句不被求值。
*/
/* 弃用针对不支持版本的布尔常量 */
ver_le_9_1    constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_9_1);
ver_le_9_2    constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_9_2);
ver_le_9      constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_9);
ver_le_10_1   constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_10_1);
ver_le_10_2   constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_10_2);
ver_le_10     constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_10);
ver_le_11_1   constant boolean := FALSE;
PRAGMA DEPRECATE(ver_le_11_1);
ver_le_11_2   constant boolean := FALSE;
ver_le_11     constant boolean := FALSE;
ver_le_12_1   constant boolean := FALSE;
ver_le_12_2   constant boolean := FALSE;
ver_le_12     constant boolean := FALSE;
ver_le_18     constant boolean := FALSE;
ver_le_19     constant boolean := FALSE;
ver_le_20     constant boolean := FALSE;
ver_le_21     constant boolean := TRUE;
end dbms_db_version;
代码清单 6-1
21c 版本的 DBMS_DB_VERSION 包
```

在所有较低版本的数据库中，这些标志均为 `FALSE`。只有当前版本被设置为 `TRUE`。

如果您使用 `DBMS_DB_VERSION` 常量，请从可用的最低版本号开始查询。如果某个查询指令求值为 true，则 `$then` 块中的代码将被包含，而其余代码将被忽略，因此不会被包含。由于此代码不再被求值，因此该块是否引用对象（例如包中不存在的常量）并不重要。

如果您的应用程序当前运行在 Oracle 12c 实例上，您的条件编译块应开始查询版本是否为 12（或更早）。

```sql
$if dbms_db_version.ver_le_12 $then
dbms_output.put_line( '这将运行 12c 代码。' );
```

如果您希望程序包含多个版本的代码，以使用每个数据库版本中可用的最佳功能，请使用 `$elsif` 指令来处理多个查询指令，类似于 case 语句。

```sql
$elsif dbms_db_version.ver_le_18 $then
dbms_output.put_line( '这将运行 18c 代码。' );
$elsif dbms_db_version.ver_le_19 $then
dbms_output.put_line( '这将运行 19c 代码。' );
$elsif dbms_db_version.ver_le_20 $then
dbms_output.put_line( '这将运行 20c 代码。' );
$elsif dbms_db_version.ver_le_21 $then
dbms_output.put_line( '这将运行 21c 代码。' );
```

如果所有查询指令都不为 true，则包含一个捕获所有情况的 `$else` 指令。

```sql
$else
dbms_output.put_line( '此版本不受支持' );
```

代码清单 6-2 显示了完整的脚本。

```sql
begin
$if dbms_db_version.ver_le_12 $then
dbms_output.put_line( '这将运行 12c 代码。' );
$elsif dbms_db_version.ver_le_18 $then
dbms_output.put_line( '这将运行 18c 代码。' );
$elsif dbms_db_version.ver_le_19 $then
dbms_output.put_line( '这将运行 19c 代码。' );
$elsif dbms_db_version.ver_le_20 $then
dbms_output.put_line( '这将运行 20c 代码。' );
$elsif dbms_db_version.ver_le_21 $then
dbms_output.put_line( '这将运行 21c 代码。' );
$else
dbms_output.put_line( '此版本不受支持' );
$end
end;
代码清单 6-2
正在运行哪个版本？
```

如果您构建一个包含多个条件的块（即类 case 语句），请确保包含所有可能的标志，以便您的代码能在每个版本的数据库上编译。

以下代码无法在 Oracle 20c 实例上编译，因为尽管 `ver_le_21` 表示“版本小于或等于 21”，但该变量在 Oracle 20c 数据库的 `DBMS_DB_VERSION` 包中并未声明。因此，编译会抛出错误。

```sql
begin
$if dbms_db_version.ver_le_12 $then
dbms_output.put_line( '这将运行 12c 代码。' );
$elsif dbms_db_version.ver_le_18 $then
dbms_output.put_line( '这将运行 18c 代码。' );
$elsif dbms_db_version.ver_le_19 $then
dbms_output.put_line( '这将运行 19c 代码。' );
$elsif dbms_db_version.ver_le_21 $then
dbms_output.put_line( '这将运行 21c 代码。' );
$else
dbms_output.put_line( '此版本不受支持' );
$end
end;
```

此外，条件编译不进行短路求值，因此请确保不要在一个语句中混合多个可能非法的条件。

```sql
begin
$if dbms_db_version.ver_le_12
or dbms_db_version.ver_le_18
or dbms_db_version.ver_le_19 $then
dbms_output.put_line( '这将运行 20c 之前的代码。' );
$elsif dbms_db_version.ver_le_20 $then
dbms_output.put_line( '这将运行 20c 代码。' );
$elsif dbms_db_version.ver_le_21 $then
dbms_output.put_line( '这将运行 21c 代码。' );
$else
dbms_output.put_line( '此版本不受支持' );
$end
end;
```

例如，此代码无法在 Oracle 12c 实例上编译。同样，`DBMS_DB_VERSION.VER_LE_18` 和 `DBMS_DB_VERSION.VER_LE_19` 常量根本还不存在。

#### 预定义的 CCFLAGS

对于任何程序单元，都自动定义了一些 `CCFLAGS`。这些标志作为选择指令中的查询指令可能不是很有用，但它们可以提供有关程序的有用信息。例如，在对代码进行插桩时，了解操作发生的行号可能会很有帮助。表 6-1 显示了预定义 `CCFLAGS` 的列表。

**表 6-1 预定义的 CCFLAGS**

| 标志 | 描述 |
| --- | --- |
| `$$PLSQL_LINE` | 一个 `PLS_INTEGER` 字面量，其值为该指令在当前 PL/SQL 单元中出现的源代码行号。 |
| `$$PLSQL_UNIT` | 一个 `VARCHAR2` 字面量，包含当前 PL/SQL 单元的名称。如果当前 PL/SQL 单元是匿名块，则 `$$PLSQL_UNIT` 包含 `NULL` 值。仅限顶级对象，不包括子程序。 |
| `$$PLSQL_UNIT_OWNER` | 一个 `VARCHAR2` 字面量，包含当前 PL/SQL 单元所有者的名称。如果当前 PL/SQL 单元是匿名块，则 `$$PLSQL_UNIT_OWNER` 包含 `NULL` 值。 |
| `$$PLSQL_UNIT_TYPE` | 一个 `VARCHAR2` 字面量，包含当前 PL/SQL 单元的类型——`ANONYMOUS BLOCK`（匿名块）、`FUNCTION`（函数）、`PACKAGE`（包）、`PACKAGE BODY`（包体）、`PROCEDURE`（过程）、`TRIGGER`（触发器）、`TYPE`（类型）或 `TYPE BODY`（类型体）。在匿名块或非 DML 触发器内部，`$$PLSQL_UNIT_TYPE` 的值为 `ANONYMOUS BLOCK`。 |



### 条件编译指令

#### 错误指令

要完全停止编译（例如，因为某些条件未满足），你可以使用 `error` 指令。`error` 指令由这个构造块组成。

```sql
$error 'error message' $end
```

一旦这段代码被包含在编译的程序中，它就保持在一个无效状态。清单 6-3 展示了一个无法编译的包。

```sql
create or replace package willnotcompile as
the_answer constant number := 42;
$error 'This package should not be compiled' $end
wear_towel constant boolean := true;
end willnotcompile;
```

这段代码将无法成功编译。

清单 6-3 中的代码会导致编译失败。

```sql
Warning: Package created with compilation errors
Errors for PACKAGE BOOK.CCWILLNOTCOMPILE:
LINE/COL ERROR
-------- ------------------------------------------------------
3/3      PLS-00179: $ERROR: This package should not be compiled
```

编译在此指令后立即终止。其余的代码甚至不会被解析。由于程序处于无效状态，你无法从中引用任何内容，甚至包括在错误指令之前定义的常量。

#### DEPRECATE 编译指示指令

如果 `error` 指令过于严厉，因为它会完全阻止代码编译，那么使用 `DEPRECATE` 编译指示指令可能是个好主意。这样，代码仍然可以编译，但会有一个警告，提示这部分代码已弃用，并将在未来的版本中移除。`DEPRECATE` 编译指示应紧接在声明之后。如果你在代码的其他位置添加编译指示，它会引发 `plw-06021` 警告。你可以弃用过程、函数或包。如果你尝试在包体中弃用包，则会引发 `plw-06022` 警告。

清单 6-4 展示了可能发出的一些警告。

```sql
create or replace package deprecatedwarnings is
procedure deprecate_this_later;
procedure deprecate_this_now;
pragma deprecate( deprecate_this_now,
'This will raise PLW-6019, when this program is compiled' );
pragma deprecate( deprecate_this_later,
'This will raise PLW-6021, because this pragma is misplaced' );
procedure call_the_deprecated_procedure;
end deprecatedwarnings;
```

弃用警告。

在调用程序中会引发 `plw-06020` 异常。如果使用以下语句启用，则在编译时会引发警告。

```sql
alter session set plsql_warnings='enable:(6019,6020,6021,6022)';
```

可以发出的警告如表 6-2 所示。

表 6-2. 弃用警告

| 警告 | 含义 |
| --- | --- |
| 6019 | 该实体已被弃用，并可能在未来的版本中移除。请勿使用已弃用的实体。 |
| 6020 | 引用的实体已被弃用，并可能在未来的版本中移除。请勿使用已弃用的实体。如果警告中提供了具体说明，请遵循这些说明。 |
| 6021 | 编译指示位置错误。`DEPRECATE` 编译指示应紧接在被弃用实体的声明之后。将编译指示放在被弃用实体声明的紧后方。 |
| 6022 | 此实体不能被弃用。弃用仅适用于可在包或类型规范中声明的实体，以及顶级过程和函数定义。请移除此编译指示。 |

要查看发出的警告，编译器警告必须在编译时启用。

```sql
alter session set plsql_warnings='enable:(6019,6020,6021,6022)'
```

或者你可能希望启用所有警告。

```sql
alter session set plsql_warnings='enable:all'
```

在编译特定对象时也可以进行设置。

```sql
alter package deprecatedwarnings compile
plsql_warnings = 'enable:(6019,6020,6021,6022)'
```

#### 使用指令

条件编译块通常以 `$if` 选择指令开始。

```sql
$if dbms_db_version.version <= 19 $then
```

如果此条件评估为真，则包含 `$then` 指令之后的代码。包含所有代码，直到找到下一个指令。

```sql
dbms_output.put_line( 'pre Oracle 19c code' );
```

如果你想包含不同的代码部分，可以包含一个 `$else` 指令和不同的代码块。

```sql
$else
dbms_output.put_line( 'post Oracle 19c code' );
```

条件编译块以 `$end` 指令结束。

清单 6-5 展示了完整的脚本。

```sql
begin
$if dbms_db_version.version <= 19 $then
dbms_output.put_line( 'pre Oracle 19c code' );
$else
dbms_output.put_line( 'post Oracle 19c code' );
$end
end;
```

根据数据库版本包含代码。

如果你在 Oracle 19c 实例上运行此匿名块，会得到以下输出。

```sql
pre Oracle 19c code
```

如果你在 Oracle 21c 实例上运行相同的匿名块，会得到以下输出。

```sql
post Oracle 19c code
```

### 用例

有无数情况可以使条件编译变得有用。例如，你可以使用条件编译来为新的数据库版本准备你的应用程序，测试私有程序，检查数据库是用于开发、测试还是生产，检查编译器优化级别，并根据环境包含或排除代码。可能还有更多你能想到的用例。

#### 检查新功能的存在

如果你可以访问沙箱数据库（例如，免费套餐），你可能知道你想在应用程序中使用的新功能。然而，由于你的生产环境仍在运行旧版本的数据库，你无法实现它们。在这种情况下，你有（至少）两个选择。

*   你记录该功能、你想使用它的地方以及在哪里可以找到文档，这样当你的生产环境升级到新版本时，你就可以开始开发新的解决方案。现在，你必须记住在数据库升级时你想使用此功能。

*   你可以在当前代码库中实现新的解决方案，并使用条件编译将此代码对旧数据库“隐藏”。

让我们看一个简单的例子。Oracle 21c 附带了更简单的步进迭代器实现。首先，检查你为其编译程序的实例版本。

```sql
$if dbms_db_version.version >= 21 $then
```

如果运行至少版本 21 的数据库，你可以使用新的迭代器结构。

```sql
-- code using the iterators available from 21c
for n in 2 .. 10 by 2 loop
dbms_output.put_line(n);
end loop;
```

如果你运行的是旧版本，则必须使用自 Oracle 7 以来可用的构造。

```sql
-- code using the iterators as available before 21c
for n in 2 .. 10 loop
if mod(n, 2) = 0 then
dbms_output.put_line(n);
end if;
end loop;
```

完整的脚本可以在清单 6-6 中找到。

```sql
begin
$if dbms_db_version.version >= 21 $then
-- code using the iterators available from 21c
for n in 2 .. 10 by 2 loop
dbms_output.put_line(n);
end loop;
$else
-- code using the iterators as available before 21c
for n in 2 .. 10 loop
if mod(n, 2) = 0 then
dbms_output.put_line(n);
end if;
end loop;
$end
end;
```

如果可用，则使用新的迭代器。


#### 显示调试和测试代码的输出

条件编译还能够暴露和隐藏包中的过程与函数。如果希望测试包中的私有过程或函数，必须修改包规范以公开该程序。随后，在发布代码时，又必须再次将其隐藏。

通过条件编译，你可以根据 `CCFLAGS` 设置中的标志值来使这些程序可见或隐藏。如果该标志未被定义，其查询求值结果为 `false`，因此你无需在编译源代码之前定义它。

使用你自定义的标志，在选择指令中定义程序是否可见。

```
$if $$expose_for_test $then
procedure private_procedure;
$end
```

你可以将此代码放在包规范中的任何位置。如果在编译包源代码时未定义该标志或将其设置为 `false`，则该代码将不会被包含在编译后的程序中。

完整的包规范如列表 6-7 所示。

```
create or replace package cctestdemo as
$if $$expose_for_test $then
procedure private_procedure;
$end
procedure public_procedure;
end cctestdemo;
Listing 6-7
包含条件暴露程序的包规范
```

列表 6-8 展示了此包的实现。

```
create or replace package body cctestdemo as
procedure private_procedure
is
begin
dbms_output.put_line('=> This is the private_procedure  This is the public_procedure <=');
end public_procedure;
end cctestdemo;
Listing 6-8
实现了所有程序的包体
```

如果你描述当前包，只会看到定义的公共过程。

```
SQL> desc cctestdemo
PROCEDURE PUBLIC_PROCEDURE
```

如果你想执行私有过程，将会收到一个错误，提示该过程必须被声明。

```
SQL> exec cctestdemo.private_procedure
BEGIN ccdemo.private_procedure; END;
*
ERROR at line 1:
ORA-06550: line 1, column 14:
PLS-00302: component 'PRIVATE_PROCEDURE' must be declared
ORA-06550: line 1, column 7:
PL/SQL: Statement ignored
```

你可以使用 `alter session` 语句定义查询指令的值，从而在不更改源代码的情况下暴露代码进行测试。

```
SQL> alter session set PLSQL_CCFLAGS = 'expose_for_test:true'
```

然后重新编译数据库中的现有包。

```
SQL> alter package cctestdemo compile
```

通过此操作，你可以暴露该过程，使其可用于测试。

或者，你也可以在编译时直接指定 `CCFLAGS`。

```
alter package cctestdemo compile plsql_ccflags='expose_for_test:true'
```

如果你现在描述该包，会看到私有过程已经被暴露。

```
SQL> desc cctestdemo
PROCEDURE PRIVATE_PROCEDURE
PROCEDURE PUBLIC_PROCEDURE
```

你现在可以访问私有过程来测试其内容。

```
SQL> exec cctestdemo.private_procedure
=> This is the private_procedure <=
```

要使用条件编译标志的最后设置来重新编译程序，请在编译语句中使用 `reuse settings`。

```
alter package cctestdemo compile reuse settings
```

如果你的程序在不同的环境中依赖于不同的设置，此功能将特别有用。

使用以下语句查找程序编译时使用了哪些设置。

```
select plsql_ccflags
from   all_plsql_object_settings
where  name  = :object_name
and    owner = :object_owner
and    type  = :object_type
```

你可以测试私有程序，并检查你的程序是否优雅地处理了所有异常，即使是那些不太可能发生的异常。

例如，如果你依赖于唯一约束的存在，你不太可能遇到 `too_many_rows` 异常。如果该约束缺失或被禁用，你希望你的代码能优雅地处理此异常，而不是抛出错误。

首先，你必须实现代码并包含异常处理器。

```
function getdobfordriver( driverid_in in number )
return date
is
l_dob date;
begin
begin
select drv.dob
into   l_dob
from   f1data.drivers drv
where  drv.driverid = driverid_in;
exception
when no_data_found
then
l_dob := null;
when too_many_rows
then
l_dob := to_date('19721229', 'YYYYMMDD');
end;
return l_dob;
end getdobfordriver;
```

然后，使用查询指令，你可以包含测试异常所需的所有代码。

```
begin
$if $$testexceptions $then
case
when mod( driverid_in, 2 ) = 0
then raise no_data_found;
else raise too_many_rows;
end case;
$else
select drv,dob
into   l_dob
from   f1data.drivers drv
where  drv.driverid = driverid_in;
$end
```

如果你想查看引发了哪个异常以及处理了哪个异常，在异常处理器中扩展以显示它们处理了哪种异常可能是个好主意。

```
exception
when no_data_found
then
$if $$testexceptions $then
dbms_output.put_line( 'no_data_found' );
$end
l_dob := null;
when too_many_rows
then
$if $$testexceptions $then
dbms_output.put_line( 'too_many_rows' );
$end
l_dob := to_date('19721229', 'YYYYMMDD');
end;
```

列表 6-9 显示了包规范，列表 6-10 显示了包体。

```
create or replace package body ccexception as
function getdobfordriver( driverid_in in number )
return date
is
l_dob date;
begin
begin
$if $$testexceptions $then
case
when mod( driverid_in, 2 ) = 0
then raise no_data_found;
else raise too_many_rows;
end case;
$else
select drv.dob
into   l_dob
from   f1data.drivers drv
where  drv.driverid = driverid_in;
$end
exception
when no_data_found
then
$if $$testexceptions $then
dbms_output.put_line( 'no_data_found' );
$end
l_dob := null;
when too_many_rows
then
$if $$testexceptions $then
dbms_output.put_line( 'too_many_rows' );
$end
l_dob := to_date('19721229', 'YYYYMMDD');
end;
return l_dob;
end getdobfordriver;
end ccexception;
/
Listing 6-10
测试异常的包体
```

```
create or replace package ccexception as
function getdobfordriver( driverid_in in number )
return date;
end ccexception;
/
Listing 6-9
测试异常的包规范
```

如果你在不设置查询指令的情况下运行 `getdobfordriver` 函数，你会得到预期的输出。

```
begin
dbms_output.put_line( 'Date or birth for unknown: '
|| ccexception.getdobfordriver ( -1 )
);
dbms_output.put_line( 'Date or birth for Hamilton: '
|| ccexception.getdobfordriver ( 1 )
);
dbms_output.put_line( 'Date or birth for Verstappen: '
|| ccexception.getdobfordriver ( 830 )
);
end;
/
Date or birth for unknown:
Date or birth for Hamilton: 07-JAN-85
Date or birth for Verstappen: 30-SEP-97
```

为了得到不同的结果，在重新编译包体时设置查询指令。

```
alter package ccexception compile
plsql_ccflags = 'testexceptions:true'
```

现在，当你运行代码时，你还会看到可能引发的不同异常。

```
begin
dbms_output.put_line( 'Date or birth for unknown: '
|| ccexception.getdobfordriver ( -1 )
);
dbms_output.put_line( 'Date or birth for Hamilton: '
|| ccexception.getdobfordriver ( 1 )
);
dbms_output.put_line( 'Date or birth for Verstappen: '
|| ccexception.getdobfordriver ( 830 )
);
end;
/
too_many_rows
Date or birth for unknown: 29-DEC-72
too_many_rows
Date or birth for Hamilton: 29-DEC-72
no_data_found
Date or birth for Verstappen:
```

#### 检查正确的数据库环境

使用条件编译指令，你可以防止代码在不同环境（例如你的生产环境）中被编译。你可以创建一些不应在生产环境中编译的代码，因为它们包含的工具在开发环境中可能非常有用但被认为不安全。

你可以使用选择指令和查询指令来检查你的环境。

```
$if environment_pkg.development $then
```

如果你的环境正确，就包含该代码。

```
procedure still_in_development;
```

如果环境不正确，则使用 `$error` 指令来阻止代码编译。

```
$else
$error 'This package is for the development environment only'
$end
$end
```

清单 6-11 展示了完整的实现。

```
create or replace package only_for_dev is
$if environment_pkg.development $then
procedure still_in_development;
$else
$error 'This package is for the development environment only'
$end
$end
end only_for_dev;
Listing 6-11
仅用于开发环境的代码
```

通过使用包含不同标志的环境包，你可以根据环境来编译代码。清单 6-12 创建了具有正确标志值的环境包。

```
declare
l_pack_spec varchar2( 32767 );
l_env       varchar2( 128 );
function tochar(
p_val in boolean)
return varchar2
as
begin
return case p_val
when true then 'TRUE'
when false then 'FALSE'
else null
end;
end tochar;
begin
l_env       := sys_context( 'userenv', 'db_name' );
l_pack_spec := 'create or replace package environment_pkg'
|| chr( 10 );
l_pack_spec := l_pack_spec || 'is' || chr( 10 );
l_pack_spec := l_pack_spec || '   --==' || chr( 10 );
l_pack_spec := l_pack_spec
|| '   -- Environment Information'
|| chr( 10 );
l_pack_spec := l_pack_spec
|| '   development constant boolean := '
|| lower( tochar( l_env like '%DEV%' ) )
|| ';' || chr( 10 );
l_pack_spec := l_pack_spec
|| '   test        constant boolean := '
|| lower( tochar( l_env like '%TST%' ) )
|| ';' || chr( 10 );
l_pack_spec := l_pack_spec
|| '   acceptance  constant boolean := '
|| lower( tochar( l_env like '%ACC%' ) )
|| ';' || chr( 10 );
l_pack_spec := l_pack_spec
|| '   production  constant boolean := '
|| lower( tochar( l_env like '%PRD%' ) )
|| ';' || chr( 10 );
l_pack_spec := l_pack_spec || '   --==' || chr( 10 );
l_pack_spec := l_pack_spec || 'end environment_pkg;'
|| chr( 10 );
execute immediate l_pack_spec;
end;
/
Listing 6-12
创建环境包
```

如果你在部署脚本中包含这个匿名块，你就可以确保标志包含正确的值，并且你的代码将根据这些标志的设置进行编译。你必须创建一个包来保存这些值，因为查询指令必须包含静态常量，而 `SYS_CONTEXT` 不被视为静态布尔表达式。

使用环境检查还可以防止代码在不同环境中执行。一个很好的例子是防止程序在非生产环境中发送电子邮件。

#### 展示已编译内容

你可以执行以下命令来查看根据你的 `CCFLAGS` 设置、数据库版本或环境设置编译后的代码。

```
dbms_preprocessor.print_post_processed_source
```

查看代码编译后的样子对于分析代码的行为非常有用。

清单 6-13 展示了实际的源代码。

```
create or replace package postprocesseddemo as
$if $$exposeprocedure $then
procedure private_procedure;
$end
procedure public_procedure;
end postprocesseddemo;
Listing 6-13
预处理演示
```

当你使用正确的参数调用该过程时。

```
begin
dbms_preprocessor.print_post_processed_source
(object_type => 'PACKAGE'
,schema_name => 'BOOK'
,object_name => 'POSTPROCESSEDDEMO');
end;
```

你会得到类似下面的结果。

```
package ccdemo as
procedure public_procedure;
end ccdemo;
```

注意输出中的空行。这些空行是为所有未进入编译源代码的行插入的，包括包含指令的行。这样做是为了在报告错误时保留行号。

如果你重新编译包并将标志设置为 `TRUE`。

```
alter package postprocesseddemo compile plsql_ccflags = 'exposeprocedure:true'
```

你会得到不同的结果。

```
package postprocesseddemo as
procedure private_procedure;
procedure public_procedure;
end postprocesseddemo;
```

**提示**

使用此实用程序时，请确保在设置 `server output` 时启用了格式包装；否则，空行会被删除。`set serveroutput on size unlimited format wrapped`

### 总结

本章介绍了选择、查询和错误指令，这些是条件编译的构建块。通过条件编译，你可以为不同的 Oracle 数据库版本创建单一代码库，确保代码被部署到正确的环境（例如，仅部署到开发环境而绝不部署到生产环境），以及出于测试目的暴露私有过程。文中还讨论了条件编译的几个用例和一些技巧。

## 7. 迭代和限定表达式

自 Oracle 首次在数据库中引入 PL/SQL 以来，循环语句就一直存在。通过一些编码和创建特殊逻辑，所有类型的循环都是可能的。许多这些“自制”的循环类型，如步进循环和有条件退出点的循环，现在已通过标准代码实现。

### 迭代

`for` 循环指定了一个迭代器及其迭代控制。如果需要，可以堆叠迭代控制，从而将多个循环浓缩为一个循环。迭代器的作用域仅限于循环内部。循环外的语句不能引用迭代器，迭代器在循环结束后是未定义的。

### 迭代控制

在 Oracle Database 21c 之前发布的版本中，`for` 循环具有静态签名 `for <iterator> in [reverse] <iteration_start> .. <iteration_end>`，这是唯一的迭代控制。循环变量始终是整数，从起始值运行到结束值，或者如果指定了 `reverse`，则从结束值运行到起始值，访问起始值和结束值之间的所有值。当你想跳过值或想做一些特殊处理时，必须由你自己来构建。

##### 多重迭代控制

以前，一个迭代器只能有一个迭代控制，因此如果你想对多个集合做相同的事情，你就必须设计多个循环并复制逻辑。当然，你会为此创建一个（局部）过程，但即便如此，你仍然在创建代码，如清单 7-1 所示。

```
begin
for i in 1 .. 3
loop
dbms_output.put_line(i);
end loop;
for i in 100 .. 104
loop
dbms_output.put_line(i);
end loop;
for i in 200 .. 202
loop
dbms_output.put_line(i);
end loop;
end;
/

Listing 7-1
21c 之前的多重迭代
```

从 Oracle Database 21c 开始，你可以在一个循环内堆叠多个迭代控制。为了实现相同的逻辑，你可以编写如清单 7-2 所示的代码。

```
begin
for i in 1 .. 3, 100 .. 104, 200 .. 202
loop
dbms_output.put_line(i);
end loop;
end;
/

Listing 7-2
21c 及更高版本的多重迭代
```


##### 步进迭代控制

如果您有一个需要步进控制的迭代循环，则必须自行构建。您可以构建一个简短的 `for` 循环，然后对迭代器值进行一些算术运算，或者构建一个长循环，然后跳过您不想使用的迭代器值。后一种解决方案如清单 7-3 所示。

```
begin
for i in 1 .. 20
loop
if mod(i, 3) = 1
then
dbms_output.put_line(i);
end if;
end loop;
end;
/
```

**清单 7-3** 21c 版本之前的步进迭代

自 Oracle Database 21c 起，您可以将步长直接写入迭代中，如清单 7-4 所示。

```
begin
for i in 1 .. 20 by 3
loop
dbms_output.put_line(i);
end loop;
end;
/
```

**清单 7-4** 21c 及以上版本的步进迭代

如果您将迭代器声明为非整数变量而非整数，则甚至可以使用分数步长。清单 7-5 是一个示例。

```
begin
for i number(3, 1) in 1 .. 10 by 0.5
loop
dbms_output.put_line(i);
end loop;
end;
/
```

```
1.5
2.5
3.5
4.5
5.5
6.5
7.5
8.5
9.5
```

**清单 7-5** 21c 及以上版本的分数步进迭代

## 迭代控制中的 while 循环

在之前的 Oracle Database 版本中，您可以构建 `while` 循环，但这意味着要声明一个变量、初始化它并构建循环。问题始终在于，如何初始化变量，以及在哪里更改其值？清单 7-6 展示了 21c 版本之前数据库中的一个 `while` 循环。

```
declare
power2 number;
begin
power2 := 1;
while power2 <= 1024 loop
dbms_output.put_line( power2 );
power2 := power2 * 2;
end loop;
end;
/
```

**清单 7-6** while 循环

在 Oracle Database 21c 中，您可以将 `while` 循环包含在 `for` 循环中，如清单 7-7 所见。

```
begin
for power2 in 1, repeat power2 * 2 while power2 <= 1024
loop
dbms_output.put_line(power2);
end loop;
end;
/
```

**清单 7-7** 迭代控制中的 while 循环

## 迭代控制中的 when

在通常情况下，您会运行循环中的所有迭代，并在每次迭代中检查指定条件是否满足，然后才执行某些逻辑。现在，这些条件可以直接在迭代控制本身中处理。

如果您创建一个返回 `BOOLEAN` 值的函数，如清单 7-8 所示，您可以将其用作迭代控制中的条件的一部分，如清单 7-9 所用。

```
create or replace function is_prime(number_in in number)
return boolean is
l_returnvalue boolean := true;
begin
for indx in 2 .. number_in / 2
loop
if mod(number_in, indx) = 0
then
l_returnvalue := false;
exit;
end if;
end loop;
return l_returnvalue;
end;
/
```

**清单 7-8** `is_prime` 函数

使用此函数，您可以确保仅在条件满足时才执行循环。清单 7-9 中的示例展示了如何使用此函数来确定执行代码块内逻辑的条件。

```
begin
for i in 1 .. 100 when is_prime(i) and i not in (19, 37)
loop
dbms_output.put_line(i);
end loop;
end;
/
```

**清单 7-9** 迭代控制中的 `when`

该循环仅针对满足 `when` 子句中指定条件的值执行。

##### 迭代控制中的游标

自 Oracle 数据库首次实现 PL/SQL 以来，游标 `for loop` 语句就已可用。您可以使用迭代器记录来处理游标的数据。清单 7-10 展示了这种游标 `for loop` 的实现。

```
begin
dbms_output.put_line( 'Drivers:');
for rec in (select d.driverid
, d.driverref
, d.forename || ' ' || d.surname as name
, d.nationality
from   f1data.drivers d
) loop
dbms_output.put_line( rec.driverid
||'('
||rec.driverref
||')'
||rec.name
||'-'
||rec.nationality
);
end loop;
dbms_output.put_line( 'Constructors:');
for rec in (select c.constructorid
, c.constructorref
, c.name
, c.nationality
from   f1data.constructors c
) loop
dbms_output.put_line( rec.constructorid
||'('
||rec.constructorref
||')'
||rec.name
||'-'
||rec.nationality
);
end loop;
end;
/
```

```
Drivers:
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
Constructors:
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

**清单 7-10** 显示信息的游标 for 循环

如您所见，要显示信息，您必须在两个游标 `for loop` 语句之间复制逻辑。在 Oracle Database 21c 中，可以将迭代器定义为预定义的记录类型，您可以将其用作显示过程的参数。清单 7-11 是此用法的示例；它声明了一个记录类型 `info_t`，并使用此类型作为迭代器，而不是标准的标量值。

```
declare
type info_t is record
( id          number(11)
, ref         varchar2(255)
, name        varchar2(255)
, nationality varchar2(255)
);
procedure printit(rec_in in info_t)
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
begin
dbms_output.put_line( 'Drivers:');
for rec info_t in (select d.driverid
, d.driverref
, d.forename || ' ' || d.surname
, d.nationality
from   f1data.drivers d
where  rownum <= 10) loop
printit(rec);
end loop;
dbms_output.put_line( 'Constructors:');
for rec info_t in (select c.constructorid
, c.constructorref
, c.name
, c.nationality
from   f1data.constructors c
where  rownum <= 10) loop
printit(rec);
end loop;
end;
/
```

```
Drivers:
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
Constructors:
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

**清单 7-11** 使用预定义记录类型的游标 for 循环

此技术还可以创建将 ref 游标作为参数之一的过程或函数。清单 7-12 在包中实现了这一点。您不是创建游标 `for loop`，而是基于游标变量的 `values of` 创建循环。




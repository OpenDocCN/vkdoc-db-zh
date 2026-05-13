# PL/SQL 包体 `unloader` 文档

## 包体声明
```sql
SQL> create or replace package body unloader
as
g_theCursor     integer default dbms_sql.open_cursor;
g_descTbl       dbms_sql.desc_tab;
g_nl            varchar2(2) default chr(10);
```

这些是此包体中使用的一些全局变量。全局游标在首次引用此包时打开，并将保持打开状态直到我们登出。这避免了每次调用此包时获取新游标的开销。`g_descTbl` 是一个 PL/SQL 表，用于保存 `DBMS_SQL.DESCRIBE` 调用的输出。`g_nl` 是一个换行符。我们将其用于需要嵌入换行符的字符串中。我们不需要为 Windows 调整此设置——`UTL_FILE` 会在字符字符串中看到 `CHR(10)` 并自动将其转换为回车/换行符。

## 辅助函数

接下来，我们有一个用于将字符转换为十六进制的小型便捷函数。它使用内置函数来完成此操作：

```sql
function to_hex( p_str in varchar2 ) return varchar2
is
begin
return to_char( ascii(p_str), 'fm0x' );
end;
```

最后，我们再创建一个便捷函数 `is_windows`，它根据我们是否在 Windows 平台上返回 `TRUE` 或 `FALSE`，因此行尾是一个双字符字符串，而不是大多数其他平台上的单个字符。我们使用内置的 `DBMS_UTILITY` 函数 `GET_PARAMETER_VALUE`，它可以读取大多数任何参数。我们检索 `CONTROL_FILES` 参数并查找其中是否存在 `\`——如果找到，我们就在 Windows 上：

```sql
function is_windows return boolean
is
l_cfiles varchar2(4000);
l_dummy  number;
begin
if (dbms_utility.get_parameter_value( 'control_files', l_dummy, l_cfiles )>0)
then
return instr( l_cfiles, '\' ) > 0;
else
return FALSE;
end if;
end;
```

> **注意**
> `is_windows` 函数确实依赖于你在 `CONTROL_FILES` 参数中使用 `\`。请注意，你也可以使用 `/`，但这非常不寻常。

## 控制文件生成过程

以下是一个过程，用于创建一个控制文件以重新加载卸载的数据，它使用 `DBMS_SQL.DESCRIBE_COLUMNS` 生成的 `DESCRIBE` 表。它为我们处理了操作系统细节，例如操作系统是否使用回车/换行符（用于 `STR` 属性）：

```sql
procedure  dump_ctl( p_dir        in varchar2,
p_filename   in varchar2,
p_tname      in varchar2,
p_mode       in varchar2,
p_separator  in varchar2,
p_enclosure  in varchar2,
p_terminator in varchar2 )
is
l_output        utl_file.file_type;
l_sep           varchar2(5);
l_str           varchar2(5) := chr(10);
begin
if ( is_windows )
then
l_str := chr(13) || chr(10);
end if;
l_output := utl_file.fopen( p_dir, p_filename || '.ctl', 'w' );
utl_file.put_line( l_output, 'load data' );
utl_file.put_line( l_output, 'infile ''' ||
p_filename || '.dat'' "str x''' ||
utl_raw.cast_to_raw( p_terminator ||
l_str ) || '''"' );
utl_file.put_line( l_output, 'into table ' || p_tname );
utl_file.put_line( l_output, p_mode );
utl_file.put_line( l_output, 'fields terminated by X''' ||
to_hex(p_separator) ||
''' enclosed by X''' ||
to_hex(p_enclosure) || ''' ' );
utl_file.put_line( l_output, '(' );
for i in 1 .. g_descTbl.count
loop
if ( g_descTbl(i).col_type = 12 )
then
utl_file.put( l_output, l_sep || g_descTbl(i).col_name ||
' date ''ddmmyyyyhh24miss'' ');
else
utl_file.put( l_output, l_sep || g_descTbl(i).col_name ||
' char(' ||
to_char(g_descTbl(i).col_max_len*2) ||' )' );
end if;
l_sep := ','||g_nl ;
end loop;
utl_file.put_line( l_output, g_nl || ')' );
utl_file.fclose( l_output );
end;
```

## 引号处理函数

这是一个简单的函数，用于使用选定的围栏字符返回带引号的字符串。请注意，它不仅用围栏字符包围字符，而且如果字符串中存在围栏字符，还会将其加倍，以便保留它们：

```sql
function quote(p_str in varchar2, p_enclosure in varchar2)
return varchar2
is
begin
return p_enclosure ||
replace( p_str, p_enclosure, p_enclosure||p_enclosure ) ||
p_enclosure;
end;
```

## 主函数 `run`

接下来，我们有主函数 `run`。由于它相当大，我将边讲解边注释：

```sql
function run( p_query      in varchar2,
p_tname      in varchar2,
p_mode       in varchar2 default 'REPLACE',
p_dir        in varchar2,
p_filename   in varchar2,
p_separator  in varchar2 default ',',
p_enclosure  in varchar2 default '"',
p_terminator in varchar2 default '|' ) return number
is
l_output        utl_file.file_type;
l_columnValue   varchar2(4000);
l_colCnt        number default 0;
l_separator     varchar2(10) default '';
l_cnt           number default 0;
l_line          long;
l_datefmt       varchar2(255);
l_descTbl       dbms_sql.desc_tab;
begin
```

### 日期格式处理与异常块设置
我们将把 `NLS_DATE_FORMAT` 保存到一个变量中，以便将其更改为在将数据转储到磁盘时保留日期和时间的格式。通过这种方式，我们将保留日期的时间组件。然后我们设置一个异常块，以便在任何错误时重置 `NLS_DATE_FORMAT`：

```sql
select value
into l_datefmt
from nls_session_parameters
where parameter = 'NLS_DATE_FORMAT';
/*
将日期格式设置为一个长数字字符串。避免了
所有 NLS 问题并同时保存了时间和日期。
*/
execute immediate
'alter session set nls_date_format=''ddmmyyyyhh24miss'' ';
/*
设置一个异常块，以便在发生任何错误时，
我们至少可以重置日期格式。
*/
begin
```

### 查询解析、描述与控制文件创建
接下来，我们将解析和描述查询。将 `g_descTbl` 设置为 `l_descTbl` 是为了重置全局表；否则，它可能包含来自先前 `DESCRIBE` 的数据以及当前查询的数据。完成后，我们调用 `dump_ctl` 来实际创建控制文件：

```sql
/*
解析并描述查询。我们将 descTbl 重置为空表，
这样在其上使用 .count 才会可靠。
*/
dbms_sql.parse( g_theCursor, p_query, dbms_sql.native );
g_descTbl := l_descTbl;
dbms_sql.describe_columns( g_theCursor, l_colCnt, g_descTbl );
/*
创建一个控制文件，用于将此数据重新加载到目标表中。
*/
dump_ctl( p_dir, p_filename, p_tname, p_mode, p_separator,
p_enclosure, p_terminator );
/*
将每一列都绑定到一个 varchar2(4000)。我们不关心
我们提取的是数字、日期还是其他类型。
所有东西都可以是字符串。
*/
```

### 数据提取准备
我们准备将实际数据转储到磁盘。我们首先将每个列定义为用于提取的 `VARCHAR2(4000)`。所有 `NUMBER`、`DATE`、`RAW`——每种类型都将转换为 `VARCHAR2`。在此之后，我们立即执行查询以准备提取阶段：

```sql
for i in 1 .. l_colCnt loop
dbms_sql.define_column( g_theCursor, i, l_columnValue, 4000);
end loop;
/*
运行查询 - 忽略执行的输出。它只在
DML 是插入/更新或删除时才有效。
*/
```

### 数据提取与写入文件
现在我们打开数据文件进行写入，从查询中获取所有行，并将其打印到数据文件：

```sql
l_cnt := dbms_sql.execute(g_theCursor);
/*
打开文件以写入输出，然后将
分隔的数据写入其中。
*/
l_output := utl_file.fopen( p_dir, p_filename || '.dat', 'w',
32760 );
loop
exit when ( dbms_sql.fetch_rows(g_theCursor) <= 0 );
l_separator := '';
l_line := null;
for i in 1 .. l_colCnt loop
dbms_sql.column_value( g_theCursor, i,
l_columnValue );
l_line := l_line || l_separator ||
quote( l_columnValue, p_enclosure );
l_separator := p_separator;
end loop;
l_line := l_line || p_terminator;
utl_file.put_line( l_output, l_line );
l_cnt := l_cnt+1;
end loop;
utl_file.fclose( l_output );
```

最后，我们将日期格式设置回来（如果前面的代码因任何原因失败，异常块也会执行相同的操作）并返回：



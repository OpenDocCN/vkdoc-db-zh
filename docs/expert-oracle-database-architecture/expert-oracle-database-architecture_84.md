# SQLLDR 总结

在本节中，我们探讨了数据加载的诸多领域。我们涵盖了将会遇到的典型日常问题：加载分隔符文件、加载定长文件、加载充满图像文件的目录、使用函数处理输入数据以进行转换，等等。我们没有详细涉及使用直接路径加载器进行海量数据加载的内容；相反，我们只是浅尝辄止。我们的目标是解答使用 SQLLDR 时经常出现并影响最广泛用户群的问题。

## 平面文件导出

有许多实用程序可用于将数据导出到平面文件。例如，以下工具可以使用数据库表作为源生成文本文件：APEX、SQL Workshop、SQL Developer、TOAD 和 SQL\*Plus。或者，如果你需要更好的性能，可以编写 PL/SQL、Pro\*C、C++ 或 Java 程序来导出数据。

从上述工具列表中，我将展示如何使用 SQL\*Plus 和 PL/SQL 将数据导出到平面文件。我主要关注这两种工具，因为大多数开发人员和 DBA 都熟悉 SQL\*Plus 和 PL/SQL，并且这些实用程序相当易于使用。

### 使用 SQL\*Plus

如果数据量较小（例如小于 100GB），那么可以使用 SQL\*Plus 通过 `SET MARKUP CSV ON` 设置轻松地导出数据。例如，将以下代码放在一个名为 `unload_big_table.sql` 的文件中：

```sql
set verify off feedback off termout off echo off trimspool on pagesize 0 head off
set markup csv on
spo /tmp/big_table.dat
select * from big_table;
spo off;
```

然后你可以从操作系统命令行执行该脚本，如下所示：

```bash
$ sqlplus eoda/foo@PDB1 @unload_big_table.sql
```

你也可以从 SQL\*Plus 命令行启用 CSV 选项。此示例执行时启用 CSV 输出，分隔符为竖线，且字符列没有引号包围：

```bash
$ sqlplus -m 'csv on delimiter | quote off' eoda/foo@PDB1
SQL> select * from dept;
DEPTNO|DNAME|LOC
10|Sales|Virginia
20|Accounting|Virginia
30|Consulting|Virginia
40|Finance|Virginia
```

前述技术为你提供了一种简单易用的方法，可以快速将表中的大量数据导出到平面文件。此外，文本文件是在你的 SQL\*Plus 客户端所在的位置生成的。例如，如果你从笔记本电脑上的客户端连接到数据库，输出文件就会在你的笔记本电脑上创建。这使得最终用户和客户端对生成平面文件有更大的控制权。

### 使用 PL/SQL

在本节中，我们将开发一个小的 PL/SQL 实用程序，可用于在服务器上以 SQLLDR 友好的格式导出数据。该 PL/SQL 实用程序在大多数小型场景中都能正常工作，但使用 Pro\*C 会获得更好的性能。请注意，如果你需要在客户端而不是服务器上生成文件，Pro\*C 和 SQL\*Plus 也很有用，而 PL/SQL 会在服务器上创建文件。

我们将创建的包规范如下：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create or replace package unloader
AUTHID CURRENT_USER
as
/* Function run -- unloads data from any query into a file
and creates a control file to reload that
data into another table
p_query      = SQL query to "unload". May be virtually any query.
p_tname      = Table to load into. Will be put into control file.
p_mode       = REPLACE|APPEND|TRUNCATE -- how to reload the data
p_dir        = directory we will write the ctl and dat file to.
p_filename   = name of file to write to. I will add .ctl and .dat
to this name
p_separator  = field delimiter. I default this to a comma.
p_enclosure  = what each field will be wrapped in
p_terminator = end of line character. We use this so we can unload
and reload data with newlines in it. I default to
"|\n" (a pipe and a newline together) and "|\r\n" on NT.
You need only to override this if you believe your
data will have that default sequence of characters in it.
I ALWAYS add the OS "end of line" marker to this sequence, you should not
*/
function run( p_query      in varchar2,
p_tname      in varchar2,
p_mode       in varchar2 default 'REPLACE',
p_dir        in varchar2,
p_filename   in varchar2,
p_separator  in varchar2 default ',',
p_enclosure  in varchar2 default '"',
p_terminator in varchar2 default '|' )
return number;
end;
/
Package created.
```

注意 `AUTHID CURRENT_USER` 的使用。这允许该包在数据库中安装`一次`，并可由任何人用来导出数据。该人员只需要对其想要导出的表拥有 `SELECT` 权限，并对本包拥有 `EXECUTE` 权限即可。如果在此情况下我们不使用 `AUTHID CURRENT_USER`，那么该包的所有者将需要对要导出的所有表拥有直接的 `SELECT` 权限。此外，由于此例程完全容易受到“SQL 注入”攻击（它接受任意 SQL 语句作为输入），`CREATE` 语句必须指定 `AUTHID CURRENT_USER`。如果它是一个默认的 `definer` 权限例程，那么任何拥有其 `EXECUTE` 权限的人都将能够使用所有者的权限执行*任何* SQL 语句。如果你知道一个例程容易受到 SQL 注入攻击，那它最好是一个调用者权限例程！

**注意**
SQL 将使用调用此例程的用户的权限执行。但是，所有 PL/SQL 调用将使用被调用例程的 `definer` 的权限运行；因此，使用 `UTL_FILE` 写入目录的能力被隐式授予任何拥有对此包执行权限的人。

包体如下。我们使用 `UTL_FILE` 来写入控制文件和数据文件。`DBMS_SQL` 用于动态处理任何查询。我们在查询中只使用一种数据类型：`VARCHAR2(4000)`。这意味着我们不能使用此方法导出 LOB，如果 LOB 大于 4000 字节，情况确实如此。然而，我们可以使用此方法通过 `SUBSTR` 导出最多 4000 字节的任何 LOB。此外，由于我们使用 `VARCHAR2` 作为唯一的输出数据类型，我们可以处理长度达 2000 字节的 `RAW`（4000 个十六进制字符），这对于除 `LONG RAW` 和 `LOB` 之外的所有内容都足够了。另外，任何引用非标量属性（复杂对象类型、嵌套表等）的查询都无法与此简单实现一起使用。以下是一个解决 90% 问题的方案，意味着它在 90% 的情况下都能解决问题：


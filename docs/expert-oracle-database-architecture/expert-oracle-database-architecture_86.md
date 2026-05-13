# PL/SQL 数据卸载与加载示例

## PL/SQL 代码

```sql
/*
现在重置日期格式并返回写入输出文件的行数。
*/
execute immediate
'alter session set nls_date_format=''' || l_datefmt || '''';
return l_cnt;
exception
/*
在发生任何错误时，重置日期格式并重新引发错误。
*/
when others then
execute immediate
'alter session set nls_date_format=''' || l_datefmt || '''';
RAISE;
end;
end run;
end unloader;
/
```

`Package body created.`

## 使用方法

要运行此程序，我们可以简单地使用以下命令：

```sql
SQL> create table emp as select * from scott.emp;
SQL> set serveroutput on
SQL> create or replace directory my_dir as '/tmp';
Directory created.
SQL> declare
l_rows    number;
begin
l_rows := unloader.run
( p_query      => 'select * from emp',
p_tname      => 'emp',
p_mode       => 'replace',
p_dir        => 'MY_DIR',
p_filename   => 'emp',
p_separator  => ',',
p_enclosure  => '"',
p_terminator => '~' );
dbms_output.put_line( to_char(l_rows) || ' rows extracted to ascii file' );
end;
/
14 rows extracted to ascii file
PL/SQL procedure successfully completed.
```

## 控制文件解析

由上述代码生成的控制文件内容如下（注意：右侧括号中的数字并非文件实际内容，仅作参考）：

```
load data                                         (1)
infile 'emp.dat' "str x'7E0A'"                    (2)
into table emp                                    (3)
replace                                           (4)
fields terminated by X'2c' enclosed by X'22'      (5)
(                                                 (6)
EMPNO char(44),                                   (7)
ENAME char(20),                                   (8)
JOB char(18),                                     (9)
MGR char(44),                                    (10)
HIREDATE date 'ddmmyyyyhh24miss' ,               (11)
SAL char(44),                                    (12)
COMM char(44),                                   (13)
DEPTNO char(44),                                 (14)
)                                                (15)
```

此控制文件需要注意的要点如下：

*   **行 (2)**: 我们使用了 SQLLDR 的 `STR` 特性。我们可以指定用于终止记录的字符或字符串。这允许我们轻松加载包含嵌入换行符的数据。字符串 `x'7E0A'` 就是一个波浪号后跟一个换行符。

*   **行 (5)**: 我们使用了分隔符和包围符。我们没有使用 `OPTIONALLY ENCLOSED BY`，因为我们会将原始数据中包围符的任何出现加倍后，再用包围符包裹每一个字段。

*   **行 (11)**: 我们使用了大数字日期格式。这样做有两点好处：避免了数据相关的 NLS 问题，并保留了日期字段的时间部分。

## 数据文件内容

由前述代码生成的原始数据 (`.dat`) 文件内容如下：

```
"7369","SMITH","CLERK","7902","17121980000000","800","","20"~
"7499","ALLEN","SALESMAN","7698","20021981000000","1600","300","30"~
"7521","WARD","SALESMAN","7698","22021981000000","1250","500","30"~
"7566","JONES","MANAGER","7839","02041981000000","2975","","20"~
"7654","MARTIN","SALESMAN","7698","28091981000000","1250","1400","30"~
"7698","BLAKE","MANAGER","7839","01051981000000","2850","","30"~
"7782","CLARK","MANAGER","7839","09061981000000","2450","","10"~
"7788","SCOTT","ANALYST","7566","19041987000000","3000","","20"~
"7839","KING","PRESIDENT","","17111981000000","5000","","10"~
"7844","TURNER","SALESMAN","7698","08091981000000","1500","0","30"~
"7876","ADAMS","CLERK","7788","23051987000000","1100","","20"~
"7900","JAMES","CLERK","7698","03121981000000","950","","30"~
"7902","FORD","ANALYST","7566","03121981000000","3000","","20"~
"7934","MILLER","CLERK","7782","23011982000000","1300","","10"~
```

在 `.dat` 文件中需要注意以下几点：

*   每个字段都被我们的包围符包裹。
*   `DATES` 被卸载为大数字。
*   该文件中的每一行数据都以要求的 `~` 结尾。

我们现在可以使用 SQLLDR 轻松地重新加载这些数据。你可以根据需要向 SQLLDR 命令行添加选项。

如前所述，卸载包的逻辑可以用多种语言和工具实现。PL/SQL 是一个很好的全能型实现（无需在客户端工作站上编译和安装），但它总是写入服务器文件系统。SQL*Plus 是一个很好的折中方案，提供了不错的性能和写入客户端文件系统的能力。

## 总结

在本章中，我们涵盖了数据加载和卸载的许多细节。首先，我们讨论了外部表相对于 SQLLDR 的优势。接着，我们探讨了轻松上手外部表的简单技术。我们还展示了使用 `PREPROCESSOR` 指令在加载数据前执行操作系统命令的示例。

然后，我们研究了外部表卸载功能，以及轻松创建和在不同数据库间移动数据提取物的能力。最后，我们通过研究如何将表中的数据卸载到转储文件中来结束本章，这种文件可用于在数据库之间移动数据。

我们讨论了在大多数情况下，你应该使用外部表而不是 SQLLDR。然而，有些情况可能需要使用 SQLLDR，例如通过网络加载数据。随后，我们检查了加载分隔数据、固定宽度数据、LOB 等的许多基本技术。

最后，我们探讨了逆向过程——数据卸载，以及如何以电子表格或其他工具可以使用的格式将数据从数据库中导出。在该讨论过程中，我们审视了使用 SQL*Plus 的方法。之后，我们开发了一个 PL/SQL 工具来演示该过程——它能以 SQLLDR（或外部表）友好的格式卸载数据，但可以轻松修改以满足你的需求。




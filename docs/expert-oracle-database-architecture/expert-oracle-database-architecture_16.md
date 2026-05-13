# 从 SQL*Plus 生成文件

## 生成 CSV 文件

你可以轻松地从 SQL*Plus 生成 CSV 平面文件。你可以使用操作系统提示符下的 `-m 'csv on'` 开关，也可以在启动 SQL*Plus 会话后使用 `SET MARKUP CSV`，例如：

```
$ sqlplus scott/tiger@PDB1
SQL> set markup csv on delimiter , quote off
SQL> SELECT * FROM dept;
DEPTNO,DNAME,LOC
10,ACCOUNTING,NEW YORK
20,RESEARCH,DALLAS
30,SALES,CHICAGO
40,OPERATIONS,BOSTON
```

以下是在命令行使用 `-m 'csv on'` 开关的示例：

```
$ sqlplus -markup 'csv on quote off' scott/tiger@PDB1
SQL> select * from emp where rownum < 3;
EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO
7369,SMITH,CLERK,7902,17-DEC-80,800,,20
7499,ALLEN,SALESMAN,7698,20-FEB-81,1600,300,30
```

**提示**

当你使用 `-markup 'csv on'` 开关时，SQL*Plus 会设置诸如 `ROWPREFETCH`、`STATEMENTCACHE` 和 `PAGESIZE` 等变量以优化 I/O 性能。

## 生成 HTML

与生成 CSV 文件类似，你可以在运行查询前设置 `SET MARKUP HTML ON` 来从 SQL*Plus 生成 HTML 文件：

```
SQL> set markup html on
SQL> select * from dept;

DEPTNO

DNAME
... 
```

你也可以在命令行指定 `-markup 'html on'` 开关来生成 HTML 输出：

```
$ sqlplus -markup 'html on' scott/tiger@PDB1
```

通过这种方式，你可以轻松地基于表内容生成 HTML 输出。

## 生成 JSON 文件

`JSON_OBJECT` 函数用于将表数据转换为 JSON。例如：

```
SQL> set heading off feedback off
SQL> select json_object('deptNO' is d.deptno, 'dName' is d.dname) departments from dept d;
{"deptNO":10,"dName":"ACCOUNTING"}
{"deptNO":20,"dName":"RESEARCH"}
{"deptNO":30,"dName":"SALES"}
{"deptNO":40,"dName":"OPERATIONS"}
```

上述技术为你提供了一种从 SQL*Plus 直接生成 JSON 输出的快速高效的方法。

**注意**

你还可以使用许多其他工具来生成平面文件，例如 TOAD、APEX、SQL Developer 和 Enterprise Manager。

## 总结

在本章中，我们探讨了 Oracle 数据库使用的重要文件类型，从低级的参数文件（没有它你甚至无法启动）到至关重要的重做日志和数据文件。我们检查了 Oracle 的存储结构，从表空间到段和区，最后到数据库块（最小的存储单元）。我们简要回顾了数据库中检查点的工作原理，甚至开始展望 Oracle 的某些物理进程或线程的作用。我们还介绍了很多可选的文件类型，如密码文件、变更跟踪文件、`Data Pump` 文件等。在下一章中，我们将准备探讨 Oracle 的内存结构。


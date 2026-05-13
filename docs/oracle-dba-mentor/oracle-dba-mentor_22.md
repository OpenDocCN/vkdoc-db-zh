# 监听器日志

另一个有用的跟踪文件是监听器日志。如果监听器遇到任何问题，它会将条目写入其日志文件。清单 12-12 显示了监听器日志的位置。此目录中应该只有一个文件，即监听器日志。

## 监听器日志位置

```
[oracle@dbamentor ~]$ cd /u01/app/oracle/diag/tnslsnr/dbamentor/listener/trace/
[oracle@dbamentor trace]$ ls -l
total 12
-rw-r-----. 1 oracle oinstall 11514 Jul 30 16:02 listener.log
```

**清单 12-12** 监听器日志位置

查看监听器日志，有点难以看出它从哪里开始。我们必须查找写着“Started with pid”的那一行。该消息之后是有关监听器正在使用的主机、端口和网络协议的信息。在最后一行，我们可以看到实例已将服务“orcl”注册到监听器。清单 12-13 显示了监听器启动和将服务注册到监听器的那部分日志。

## 监听器启动

```
[oracle@dbamentor trace]$ cat listener.log
2018-07-30T16:15:52.354897-05:00
Log messages written to /u01/app/oracle/diag/tnslsnr/dbamentor/listener/alert/log.xml
Trace information written to /u01/app/oracle/diag/tnslsnr/dbamentor/listener/trace/ora_29203_140321849041280.trc
Started with pid=29203
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521)))
30-JUL-2018 16:16:46 * (ADDRESS=(PROTOCOL=tcp)(HOST=::1)(PORT=33928)) * service_register * orcl * 12542
```

**清单 12-13** 监听器启动

当监听器关闭时，日志文件将包含少量表明这一点信息，如清单 12-14 所示。

## 监听器关闭

```
2018-07-30T16:19:38.049565-05:00
No longer listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=localhost)(PORT=1521)))
```

**清单 12-14** 监听器关闭

不幸的是，监听器的日志文件没有明确说明有人启动或停止了监听器。在清单 12-14 中，我们只看到监听器不再监听该主机和端口。

通常我们不会使用监听器日志来监控监听器的启动和关闭。前面的例子是为了说明如何在监听器的日志文件中查找信息。如果你想要真正监控监听器，请使用像 Enterprise Manager 这样的产品。


## 后台进程跟踪文件

Oracle 数据库引擎由多个在后台运行的进程组成。为使 Oracle 能够运行，每个进程都有不同的职责。您可能已经接触过其中一些进程，如 `PMON`、`SMON` 和 `CKPT`。如果您想了解更多关于这些进程的信息，第 21 章将提供更多内容。

这些进程中的每一个都可以写入自己的跟踪文件。后台进程跟踪文件位于与**警告日志**相同的目录中。文件通常命名为 `instance_process_pid`.trc，其格式为实例名，后跟进程名，再后跟操作系统的进程标识符。例如，我有一个名为 `orcl_pmon_25978.trc` 的跟踪文件。这个文件对应我的 `orcl` 实例的 `PMON` 进程，当时它运行时的操作系统进程 ID 为 `25978`。

很多时候，即使没有问题发生，您也会找到这些文件。进程启动这一简单动作就会生成其中一个跟踪文件。通常，其内容只是信息性的。只有当您遇到特定后台进程的问题时，您才需要检查其跟踪文件。请回想本章前面部分，警告日志显示实例终止是因为 `SMON` 已死亡。`PMON` 检测到此情况并终止了实例。`PMON` 的跟踪文件记录了此事件，正如我们在清单 12-15 中看到的。

```
[oracle@dbamentor trace]$ cat orcl_pmon_25978.trc
Trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_pmon_25978.trc
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
Build label:    RDBMS_12.2.0.1.0_LINUX.X64_170125
ORACLE_HOME:    /u01/app/oracle/product/12.2.0.1
System name: Linux
Node name:   dbamentor.localdomain
Release:     4.1.12-103.6.1.el7uek.x86_64
Version:     #2 SMP Wed Sep 20 12:15:11 PDT 2017
Machine:     x86_64
Instance name: orcl
Redo thread mounted by this instance: 1
Oracle process number: 2
Unix process pid: 25978, image: oracle@dbamentor.localdomain (PMON)
*** 2018-07-30T15:40:21.285277-05:00
*** SESSION ID:(2.57305) 2018-07-30T15:40:21.285291-05:00
*** CLIENT ID:() 2018-07-30T15:40:21.285294-05:00
*** SERVICE NAME:(SYS$BACKGROUND) 2018-07-30T15:40:21.285297-05:00
*** MODULE NAME:() 2018-07-30T15:40:21.285300-05:00
*** ACTION NAME:() 2018-07-30T15:40:21.285302-05:00
*** CLIENT DRIVER:() 2018-07-30T15:40:21.285304-05:00
Instance Critical Process (pid: 22, ospid: 26021, SMON) died unexpectedly
Background process SMON found dead
...
*** 2018-07-30T15:40:21.301322-05:00
PMON (ospid: 25978): terminating the instance due to error 474
ksuitm: waiting up to [5] seconds before killing DIAG(25999)
清单 12-15
PMON 跟踪文件
```

跟踪文件的第一部分只是信息性的，是在进程启动时生成的。在清单 12-15 的末尾，我们可以看到 `PMON` 知道 `SMON` 意外死亡。在一段大部分看起来像乱码的冗长堆栈跟踪（为简洁起见已省略）之后，我们可以看到 `PMON` 终止了该实例。

## 用户跟踪文件

任何最终用户的数据库会话也可以创建跟踪文件。会话的跟踪文件非常有用，因为它可能包含在跟踪期间发送到数据库的每条 SQL 语句的记录。有时最终用户会报告一个问题，但无论是最终用户还是数据库管理员都不知道是哪条 SQL 语句导致了问题。DBA 可以在用户的会话中启动跟踪，并尝试弄清楚用户正在对数据库执行什么操作。

默认情况下，会话不会生成跟踪文件。启动跟踪最简单的方法是将会话的 `SQL_TRACE` 参数修改为 `TRUE` 值，如清单 12-16 所示。

```
SQL> alter session set sql_trace=true;
Session altered.
SQL> select 'this is my trace',sysdate from dual;
'THISISMYTRACE'  SYSDATE
---------------- ---------
this is my trace 31-JUL-18
SQL> alter session set sql_trace=false;
Session altered.
SQL> select spid as os_pid from v$process
2  where addr in (select paddr from v$session
3                 where sid=sys_context('USERENV','SID'));
OS_PID

清单 12-16
启动 SQL 跟踪
```

在前面的例子中，用户启动了 SQL 跟踪，向数据库发出了一条 SQL 语句，然后结束了跟踪。最后，用户使用了 `V$SESSION` 和 `V$PROCESS` 来了解会话的操作系统进程标识符。所有用户跟踪文件的命名格式为 `instance`_ora_`pid`.trc，其中 `ora` 是常量。既然我们知道实例名为 `ORCL`，并且现在知道了操作系统进程 ID，我们就知道要在 RDBMS 跟踪目录（该目录也包含警告日志）中查找一个名为 `orcl_ora_5765.trc` 的文件。

如果我们用文本编辑器打开跟踪文件，可以看到一些介绍性信息，如果向下滚动，可以找到发送给数据库的 SQL 语句。清单 12-16 生成的跟踪文件如清单 12-17 所示。

```
=====================
PARSING IN CURSOR #139855599918944 len=43 dep=0 uid=0 oct=3 lid=0 tim=8201176343
6 hv=2979390255 ad='70aa7700' sqlid='az5q2d6stbstg'
select 'this is my trace',sysdate from dual
END OF STMT
PARSE #139855599918944:c=0,e=567,p=0,cr=0,cu=0,mis=1,r=0,dep=0,og=1,plh=13887349
53,tim=82011763436
EXEC #139855599918944:c=520,e=2029,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=1388734953,tim=82011765502
FETCH #139855599918944:c=0,e=7,p=0,cr=0,cu=0,mis=0,r=1,dep=0,og=1,plh=1388734953,tim=82011765583
STAT #139855599918944 id=1 cnt=1 pid=0 pos=1 obj=0 op='FAST DUAL  (cr=0 pr=0 pw=0 str=1 time=2 us cost=2 size=0 card=1)'
FETCH #139855599918944:c=0,e=1,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=0,plh=1388734953,tim=82011766311
*** 2018-07-31T09:16:25.618519-05:00
CLOSE #139855599918944:c=0,e=7,dep=0,type=0,tim=82020618587
=====================
清单 12-17
原始跟踪文件内容
```

输出内容有点难以阅读，但我们可以清楚地辨认出 SQL 语句。如果我们查看 SQL 语句下面的行，可以看到解析阶段、执行阶段和获取阶段。SQL 语句通常会被解析以确保其语法正确，并且用户对语句中的对象拥有权限。然后该语句被执行。所有 SQL 语句都经历这三个阶段。如果 SQL 语句返回行，则结果会被获取并返回给用户。

训练有素的 Oracle DBA 可能能够理解后面的内容。例如，`c=` 值告诉我们该阶段在 CPU 上处理花费了多少时间。`e=` 值告诉我们该阶段花费了多少流逝时间。


幸运的是，我们无需直接阅读这个原始跟踪文件。第 11 章介绍过，Oracle 提供了一个实用的工具`tkprof`，它能将跟踪文件格式化为更便于人类阅读的形式。我们可以通过向`tkprof`工具传递两个参数（即跟踪文件和输出文件）来调用它。清单 12-18 展示了如何对我们的原始跟踪文件使用`tkprof`工具。

```
[oracle@dbamentor trace]$ tkprof orcl_ora_5765.trc tkprof.out
TKPROF: Release 12.2.0.1.0 - Development on Tue Jul 31 09:40:13 2018
Copyright (c) 1982, 2017, Oracle and/or its affiliates. All rights reserved.
清单 12-18
tkprof 执行
```

在上面的例子中，输出将被放置在名为`tkprof.out`的文件中。如果我们查看该文件的内容，可以看到信息以一种*更易阅读*的形式呈现，如清单 12-19 所示。

```
SQL ID: az5q2d6stbstg Plan Hash: 1388734953
select 'this is my trace',sysdate
from
dual
调用     次数     CPU 时间    耗时       磁盘读    查询量    当前量       行数
------- ------  ------ ---------- -------- --------- ----------  ---------
解析        1    0.00       0.00        0         0          0          0
执行        1    0.00       0.00        0         0          0          0
获取        2    0.00       0.00        0         0          0          1
------- ------  ------ ---------- -------- --------- ----------  ---------
总计        4    0.00       0.00        0         0          0          1
清单 12-19
tkprof 输出
```

从上面的输出中，我们可以轻松地看到 SQL 语句的三个阶段。

`tkprof`工具有许多选项，可以帮助格式化跟踪文件以满足你的特定需求。只需输入不带任何额外参数的`tkprof`，该工具就会显示一些有用的信息。我最喜欢的一种使用`tkprof`的方式是，按语句的**运行耗时**排序 SQL 语句，这样运行时间最长的 SQL 语句就会出现在输出的最后。如果用户遇到响应缓慢的问题，通常正是这个运行时间最长的 SQL 语句给他们带来了最大的痛苦。

最终用户无法发出`ALTER SESSION`命令来启动 SQL 跟踪，这种情况非常常见。最终用户通常通过某个应用程序访问数据库，而该应用程序不允许他们输入任意的 SQL 语句。Oracle 允许 DBA 在该用户的会话中启动跟踪。DBA 只需要确定该会话的标识符和序列号。在清单 12-20 的例子中，DBA 通过查询`V$SESSION`获取了这些值。一旦获知，DBA 就调用`DBMS_MONITOR`包来启动该会话的跟踪。经过一段时间后，DBA 结束跟踪。

```
SQL> select sid,serial# from v$session
2  where username='PEASLAND';
SID    SERIAL#
---------- ----------
61      21753
SQL> exec dbms_monitor.session_trace_enable(session_id=>61,serial_num=>21753);
PL/SQL 过程已成功完成。
SQL> exec dbms_monitor.session_trace_disable(session_id=>61,serial_num=>21753);
PL/SQL 过程已成功完成。
清单 12-20
在其他会话中启动 SQL 跟踪
```

生成的跟踪文件与用户自行发出`ALTER SESSION`命令所生成的文件没有区别。

本节介绍了 SQL Trace 和`tkprof`工具。能够跟踪用户操作及其执行的 SQL 语句，是数据库管理员一个有价值的工具。大多数情况下，SQL Trace 用于尝试解决性能问题。性能调优是一个很大的话题，本书不作讨论。如果你想了解更多关于性能调优的知识，Oracle 文档是一个很好的起点，我们将在下一章讨论。

当你无法访问应用程序代码，却又需要确定是哪条 SQL 语句导致了错误时，SQL Trace 文件也很有用。例如，应用程序告知用户出现了`ORA-00942`错误。数据库管理员可以在该用户的会话中启动跟踪，然后在生成的跟踪文件中搜索`942`，以查看是哪条 SQL 语句出现了这个错误。

## ADRCI

在 Oracle 11*g*之前，数据库管理员需要管理所有的跟踪文件。Oracle 不会自动删除这些文件，因此 DBA 需要一个定期执行的作业，来删除超过特定时间的旧文件。这些跟踪文件的另一个问题是，如果你需要寻求 Oracle 支持的帮助来解决问题，他们通常会要求提供一两个跟踪文件，而 DBA 很难确定支持部门需要哪些文件。于是 DBA 就把所有的跟踪文件都发送给 Oracle 支持。Oracle 现在包含了自动诊断资料库命令解释器（`adrci`），这是一个用来管理这些跟踪文件的命令行实用程序。

让我们首先看看`adrci`如何帮助打包 Oracle 支持可能需要用于解决问题的跟踪文件。在清单 12-21 中，我们首先要求`adrci`列出它在我们的 Oracle 数据库中跟踪到的问题。

```
adrci> show problem
ADR Home = /u01/app/oracle/diag/rdbms/orcl/orcl:
*************************************************************************
问题 ID   问题键     最后事件 ID  最后事件时间
------------ -----------   -------------  ---------------------------------
3            ORA 445       222031         2016-12-10 04:30:31.338000 -06:00
清单 12-21
adrci 问题列表
```

这里只记录了一个问题。让我们创建一个包发送给 Oracle 支持，如清单 12-22 所示。

```
adrci> set home diag/rdbms/mspstg/mspstg
adrci> ips create package problem 3 correlate all
基于问题 ID 3 创建了包 1，关联级别为全部
adrci> ips generate package 1 in "/home/oracle"
在文件 /home/oracle/IPSPKG_20180731101513_COM_1.zip 中生成了包 1，模式为完整
清单 12-22
adrci 创建包
```

`adrci`工具已经为我们创建了一个 zip 文件，可以上传给 Oracle 支持。如果所有这些`adrci`命令听起来令人困惑，请不要担心。如果 Oracle 支持需要你以这种方式发送跟踪文件，他们会提供所有必要的指导。

在`adrci`出现之前，数据库管理员必须自己动手管理跟踪文件。将文件留在服务器上太久，存储空间就会被占满。在 Unix/Linux 系统上，数据库管理员经常使用`crontab`中的命令来安排删除超过一定时间的跟踪文件。问题在于 DBA 需要从多个子目录中删除跟踪文件。如果 Oracle 发布了新版本，用于清理旧文件的例程也需要更新。

`adrci`可以删除跟踪文件，并能处理未来版本可能带来的任何变化。删除跟踪文件的命令非常简单，如清单 12-23 所示。

```
adrci> purge -age 10080 -type trace
清单 12-23
ADRCI 清理跟踪文件
```

上面的命令将删除所有超过 10,080 分钟（即 7 天）的文件。DBA 需要做的就是将这个命令放入一个脚本中，并在数据库服务器上安排定时执行。


## 继续前行

本章向我们展示了数据库服务器上各种诊断文件和跟踪文件的位置。我们通过一些示例了解了如何利用告警日志和其他跟踪文件来协助解决问题。本章还向我们展示了如何启动 SQL 跟踪，以便 DBA 能够获取最终用户提交给数据库的 SQL 语句信息。跟踪文件可能难以阅读，但解析它们对于获取诊断数据、帮助解决问题是必要的。至少，它们会被发送给 Oracle 支持人员供分析师阅读。

下一章将聚焦于一个似乎被许多 Oracle 专业人士所鄙视的主题：Oracle 文档。请不要跳过下一章，因为它包含了对您发展 Oracle DBA 职业生涯至关重要的信息。许多人不喜欢文档的原因在于其篇幅浩繁，导致难以找到所需内容。下一章将为您梳理清楚，并帮助您学会如何查找需要了解的信息。


# 警报日志

```
2018-07-30T15:40:21.276295-05:00
Instance Critical Process (pid: 22, ospid: 26021, SMON) died unexpectedly
PMON (ospid: 25978): terminating the instance due to error 474
2018-07-30T15:40:21.323557-05:00
System state dump requested by (instance=1, osid=25978 (PMON)), summary=[abnormal instance termination].
System State dumped to trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_diag_25999_20180730154021.trc
2018-07-30T15:40:21.790622-05:00
Dumping diagnostic data in directory=[cdmp_20180730154021], requested by (instance=1, osid=25978 (PMON)), summary=[abnormal instance termination].
2018-07-30T15:40:22.882068-05:00
Instance terminated by PMON, pid = 25978
```

**清单 12-9** 警报日志中的实例崩溃

我们知道发生了什么。一个必需的后台进程死亡。我们仍然不知道`SMON`为何死亡。本章后面，我们将查看各个后台进程生成的跟踪文件。

在某些情况下，能够向警报日志写入自定义消息是很有帮助的。由于实例正在写入文件，你不能用文本编辑器打开它来添加文本。可以使用操作系统命令来追加到文件，类似于下面这样：

```
cat "text to alert log" >> alert_orcl.log
```

这样做会面临在实例写入文件的同时尝试写入文件的风险。更好的方法是在实例中运行一个命令来写入消息。清单 12-10 中的示例使用`DBMS_SYSTEM`包向警报日志写入一个条目。随后显示了警报日志的相关部分。

## 写入警报日志

```
SQL> exec dbms_system.ksdwrt(2,'Shutting down for db maintenance - BP');
PL/SQL procedure successfully completed.
SQL> !tail -2 alert_orcl.log
2018-07-30T15:47:18.866067-05:00
Shutting down for db maintenance – BP
```

**清单 12-10** 写入警报日志

在上面的示例中，我写了一条自定义消息，表示我为了数据库维护而关闭实例。我在消息末尾加上了我的姓名首字母，以便团队中的其他数据库管理员知道是谁执行的操作。

我已经习惯使用操作系统工具来查看警报日志的内容，就像查看任何文本文件一样。在 Oracle 11g 中，引入了一个新的固定视图，允许你使用任何 SQL 语句查询警报日志的内容。在清单 12-11 中，我们可以看到如何查找警报日志中任何出现的`ALTER`命令。

## 查询警报日志

```
SQL> select originating_timestamp,message_text
  2  from x$dbgalertext
  3  where upper(message_text) like '%ALTER%'
  4  order by originating_timestamp;
ORIGINATING_TIMESTAMP              MESSAGE_TEXT
---------------------------------- ----------------------------------------
29-SEP-17 10.51.16.369 AM -05:00   ALTER DATABASE DEFAULT TEMPORARY TABLESPACE TEMP
29-SEP-17 11.04.28.706 AM -05:00   ALTER DATABASE CLOSE NORMAL
29-SEP-17 11.04.30.011 AM -05:00   Completed: ALTER DATABASE CLOSE NORMAL
29-SEP-17 11.04.30.012 AM -05:00   ALTER DATABASE DISMOUNT
29-SEP-17 11.04.30.018 AM -05:00   Completed: ALTER DATABASE DISMOUNT
29-SEP-17 11.04.46.162 AM -05:00   ALTER DATABASE MOUNT
29-SEP-17 11.04.50.247 AM -05:00   Completed: ALTER DATABASE MOUNT
29-SEP-17 11.04.50.264 AM -05:00   ALTER DATABASE OPEN
29-SEP-17 11.04.50.950 AM -05:00   Completed: ALTER DATABASE OPEN
```

**清单 12-11** 查询警报日志

现在我们可以利用 SQL 的强大功能，可以非常容易地进行过滤。例如，如果我们知道问题发生的时间，就可以搜索该时间段内发生的任何消息。


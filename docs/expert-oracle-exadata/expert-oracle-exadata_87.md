# 在数据库中使用 cellcli XML 输出

以表格形式解析 `cellcli` 的输出可能有点挑战。幸运的是，性能分析师还有另一种输出方法——XML。结合使用 `–e` 和 `–xml` 标志，您可以将 `cellcli` 输出读入数据库并使用 SQL 进行处理。以下是一个如何将 XML 格式的 `cellcli` 信息加载到数据库中的示例。

第一步是在 SQL*Plus 中使用 `create directory` 命令在数据库中创建一个目录对象。后续段落中使用的目录是 `ORADIR`。有了目录后，您可以创建一个 `XMLType` 表。存储 XML 信息还有其他方法，但作者发现这个方法可行。

```
SQL> create table metrics_xmltab of xmltype;

Table created.
```

这个特定的表将保存 `cellcli` 性能数据的 XML 表示。在加载 XML 信息之前，需要先将其从单元中提取出来。为了使示例简单，只加载一个单元的性能数据。或者，您也可以使用 `dcli` 来同时捕获信息。可以通过 SSH 读取与 `SMARTIO` 类别相关的性能指标的 XML 文件，并将其放入目录 `ORADIR` 中。这只是一个示例。您可以对单元发起任何查询：

```
ssh cellmonitor@enkcel04 "cellcli -xml -e list metriccurrent where objectType = 'SMARTIO'" \
> > oradir/metrics.xml
```

XML 文件将包含以下内容；后面要显示的重要部分位于 `<metric>` 标签中：

```
[oracle@enkdb03 ~]$ head -n20 oradir/metrics.xml
<?xml version="1.0" encoding="utf-8" ?>
<cli-output>
<context cell="enkcel04" realm="" ossStartTimestamp="1430673687318" iormResetTimestamp="782230633"/>
<metric> <name>SIO_IO_EL_OF</name>
<alertState>normal</alertState>
<collectionTime>1431455744000</collectionTime>
<metricObjectName>SMARTIO</metricObjectName>
<metricType>Cumulative</metricType>
<metricValue>634659.296875</metricValue>
<objectType>SMARTIO</objectType>
</metric>
<metric> <name>SIO_IO_EL_OF_SEC</name>
<alertState>normal</alertState>
```

格式良好的 XML——*正是您所需要的*。在下一步中，这可以加载到数据库中：

```
SQL> insert into metrics_xmltab values (xmltype
  2   (bfilename('ORADIR','metrics.xml'), nls_charset_id('AL32UTF8')));

1 row created.

SQL> commit;

Commit complete.

SQL> select count(*) from metrics_xmltab;

  COUNT(*)
----------
         1
```

现在剩下要做的就是使用一些 XMLDB 技巧将 XML 信息转换为表格形式，以便人眼更容易理解。`XMLTABLE` 结构可用于生成报告。之后，当插入更多数据时，您可以根据 `collectionTime` 来区分记录。XML 文档以 UNIX 纪元时间报告时间。

```
SQL> select x.name, TO_DATE('1970-01-01', 'YYYY-MM-DD') + x.collectionTime/86400000
  2    as collectedAt, x.metricType, x.metricValue, x.objectType
  3   from metrics_xmltab m,
  4  xmltable('//metric'
  5   passing object_value
  6   columns
  7     name           varchar2(25)          path 'name',
  8     collectionTime number                path 'collectionTime',
  9     metricType     varchar2(20)          path 'metricType',
 10     metricValue    number                path 'metricValue',
 11     objectType     varchar2(50)          path 'objectType') x;

NAME                      COLLECTEDAT          METRICTYPE           METRICVALUE OBJECTTYPE
------------------------- -------------------- -------------------- ----------- -------------
SIO_IO_EL_OF              12.05.2015 18:35:44 Cumulative            634659.297 SMARTIO
SIO_IO_EL_OF_SEC          12.05.2015 18:35:44 Rate                           0 SMARTIO
SIO_IO_OF_RE              12.05.2015 18:35:44 Cumulative            14336.6719 SMARTIO
SIO_IO_OF_RE_SEC          12.05.2015 18:35:44 Rate                           0 SMARTIO
SIO_IO_PA_TH              12.05.2015 18:35:44 Cumulative                     0 SMARTIO
SIO_IO_PA_TH_SEC          12.05.2015 18:35:44 Rate                           0 SMARTIO
SIO_IO_RD_FC              12.05.2015 18:35:44 Cumulative            60078.5391 SMARTIO
SIO_IO_RD_FC_HD           12.05.2015 18:35:44 Cumulative            3831.76563 SMARTIO
```

您可以通过创建另一个 XML 表来扩展此示例，该表包含从每个单元上找到的 `metricdefinition` 中获取的指标名称及其定义，然后将两者连接起来以获得更易于阅读的输出。视图有助于减少提取相同信息所需的输入量。

### 配置和管理存储单元

`cellcli` 还以多种方式用于配置从磁盘存储到单元警报的所有内容。您还可以将 `cellcli` 用于管理任务，如启动和关闭。以下是一些如何使用 `cellcli` 配置和管理存储单元的示例。

可以逐个或全部关闭单元服务。以下命令用于关闭单元服务：

```
-- 逐个关闭单元服务 --
CellCLI> alter cell shutdown services cellsrv
CellCLI> alter cell shutdown services ms
CellCLI> alter cell shutdown services rs

-- 关闭所有单元服务 --
CellCLI> alter cell shutdown services all
```

单元服务也可以逐个或全部启动。请注意，`RS` 进程必须首先启动，否则 `cellcli` 将抛出如下错误：

```
CellCLI> alter cell startup services cellsrv
Starting CELLSRV services...
CELL-01509: Restart Server (RS) not responding.
```

以下命令用于启动单元服务：

```
-- 逐个启动单元服务 --
CellCLI> alter cell startup services rs
CellCLI> alter cell startup services ms
CellCLI> alter cell startup services cellsrv

-- 启动所有单元服务 --
CellCLI> alter cell startup services all
```

要显示 `cellsrv` 的当前状态，请使用 `LIST CELL` 命令：

```
CellCLI> list cell attributes name,cellsrvStatus,cellVersion,flashCacheMode,-
> msStatus,rsStatus,status detail
          name:                       enkcel04
          cellVersion:                OSS_12.1.2.1.0_LINUX.X64_141206.1
          flashCacheMode:             WriteBack
          status:                     online
          cellsrvStatus:              running
          msStatus:                   running
          rsStatus:                   running
```

在 `list cell detail` 输出中看到的某些设置可以使用 `ALTER CELL` 命令进行更改。这些设置可以一次配置一个，也可以通过用逗号分隔一起配置。例如：

```
-- 配置警报的通知级别 --
CellCLI> ALTER CELL notificationPolicy='critical,warning,clear'

-- 配置单元的电子邮件通知 --
CellCLI> ALTER CELL smtpServer='smtp.example.com', -
                    smtpFromAddr='exa01@example.com', -
                    smtpFrom='exa01', -
                    smtpToAddr='all_dba@example.com', -
                    notificationPolicy='critical,warning,clear', -
                    notificationMethod='mail'
```

顺便说一句，如果您还没有发现这个功能，`cellcli` 会存储类似于 bash shell 的命令历史记录。您可以使用上下箭头键滚动浏览历史记录并使用箭头键编辑命令。多亏了查询中的正则表达式支持，您拥有了一个*非常强大*的模式匹配能力可供使用。`cellcli` 语法对系统管理员和 DBA 来说可能有些陌生，但一旦您理解了其逻辑，掌握起来真的不难。

## `dcli` 简介

`dcli` 是一种工具，通过它，你可以在所有存储单元或计算节点上执行单条命令。多年来，在多种集群系统上工作后，我们深刻体会到保持所有节点上的脚本（以及某些配置文件）一致的重要性。能够有一个工具在集群的所有节点上一致地执行相同命令，也是非常方便的。Oracle 提供了 `dcli` 命令来实现这一点——可惜，它不适用于普通的集群系统。除其他功能外，`dcli` 命令允许你执行以下操作：

*   在所有存储单元和/或数据库服务器之间配置 `SSH` 等效性
*   将文件分发到集群中所有服务器/单元的相同位置
*   在集群中的服务器/单元上分发并执行脚本
*   在集群中的服务器/单元上执行命令和脚本

`dcli` 使用 `SSH` 等效性来验证你在远程服务器上的会话。如果你没有在服务器/单元之间建立 `SSH` 等效性，你仍然可以使用它，但它会在执行命令前提示你输入每个远程系统的密码。`dcli` 并行执行所有命令，将来自每个服务器的输出聚合成一个列表，并在本地机器上显示输出。

与刚刚讨论的 `cellcli` 不同，`dcli` 不是一个 `bash` 脚本。它仍然是一个脚本，但是用 `python` 编写的。该脚本可以接受以下命令行选项（取自一个 `12.1.2.1.0` 计算节点）：

```
[oracle@enkdb03 ∼]$ dcli -h
Distributed Shell for Oracle Storage
[...]
Usage: dcli [options] [command]
Options:
  --version            show program's version number and exit
  --batchsize=MAXTHDS  limit the number of target cells on which to run the
                       command or file copy in parallel
  -c CELLS             comma-separated list of cells
  -d DESTFILE          destination directory or file
  -f FILE              files to be copied
  -g GROUPFILE         file containing list of cells
  -h, --help           show help message and exit
  --hidestderr         hide stderr for remotely executed commands in ssh
  -k                   push ssh key to cell's authorized_keys file
  -l USERID            user to login as on remote cells (default: celladmin)
  --maxlines=MAXLINES  limit output lines from a cell when in parallel
                       execution over multiple cells (default: 100000)
  -n                   abbreviate non-error output
  -r REGEXP            abbreviate output lines matching a regular expression
  -s SSHOPTIONS        string of options passed through to ssh
  --scp=SCPOPTIONS     string of options passed through to scp if different
                       from sshoptions
  --serial             serialize execution over the cells
  --showbanner         show banner of the remote node in ssh
  -t                   list target cells
  --unkey              drop keys from target cells' authorized_keys file
  -v                   print extra messages to stdout
  --vmstat=VMSTATOPS   vmstat command options
  -x EXECFILE          file to be copied and executed
```

在大多数情况下，你会发现你要么对所有存储单元，要么对所有计算节点执行命令。该工具依赖于一个简单的文本文件，其中包含以回车符分隔的机器名称，并使用 `–g` 参数传递。通常这些文件分别被称为 `cell_group`、`dbs_group` 和 `all_group`，这正是 `onecommand` 工具在安装后留下的。这些文件可以描述如下：

*   `dbs_group`：此文件包含你的 Exadata 配置中所有数据库服务器的管理主机名。它提供了一种在数据库服务器上执行 `dcli` 命令的便捷方式。
*   `cell_group`：此文件包含你的 Exadata 配置中所有存储单元的管理主机名。它提供了一种将 `dcli` 命令限制在存储单元上执行的便捷方式。
*   `all_group`：此文件是 `dbs_group` 和 `cell_group` 文件的组合，包含了你的 Exadata 配置中所有数据库服务器和存储单元的管理主机名列表。使用此文件，你可以谨慎地在所有数据库服务器和存储单元上执行 `dcli` 命令。

以下是一个四分之一机架的示例：

```
[root@enkdb03 ∼]# cat cell_group
enkcel04
enkcel05
enkcel06
```

除了 `–g` 参数外，你最常做的可能是使用 `dcli` 的 `–l` 参数来提供用户 ID，以指定应使用哪个用户连接到组文件中指定的系统。请记住，连接到存储单元时可以使用 `root`、`celladmin` 和 `cellmonitor`。当调用 `dcli` 与其他计算节点交互时，`root` 和 `oracle` 是最明显的用户 ID 候选者。

当你想使用 `cellcli` 命令从所有存储单元收集信息时，`dcli` 特别有用。以下示例展示了如何结合使用 `dcli` 和 `cellcli` 命令来报告四分之一机架集群中所有存储单元的状态：

```
[oracle@enkdb03 ∼] $ dcli -g cell_group -l cellmonitor cellcli -e list cell
enkcel04: enkcel04       online
enkcel05: enkcel05       online
enkcel06: enkcel06       online
```

本附录讨论的任何 `cellcli` 命令都可以使用 `dcli` 从一个中心位置执行。事实上，唯一的限制是该命令不能是交互式的（例如，在执行期间需要用户输入）。例如，下面的列表说明了如何从存储单元收集当前性能指标：

```
[oracle@enkdb03 ∼]$ dcli -l cellmonitor -g cell_group \
> cellcli -e "LIST METRICCURRENT  ATTRIBUTES name,metricValue, \
> collectionTime where objecttype=’FLASHLOG’ and name like ’.*FIRST’"
enkcel04: FL_DISK_FIRST          525,232,661 IO requests         2015-05-12T14:33:45-05:00
enkcel04: FL_FLASH_FIRST         11,914,368 IO requests         2015-05-12T14:33:45-05:00
enkcel05: FL_DISK_FIRST          555,655,999 IO requests         2015-05-12T14:34:06-05:00
enkcel05: FL_FLASH_FIRST         12,418,630 IO requests         2015-05-12T14:34:06-05:00
enkcel06: FL_DISK_FIRST          555,696,351 IO requests         2015-05-12T14:34:08-05:00
enkcel06: FL_FLASH_FIRST         12,086,961 IO requests         2015-05-12T14:34:08-05:00
```

如你所见，从 `dcli` 使用 `cellcli` 需要在 `where` 子句中进行创造性的引用。

## 总结

`dcli` 和 `cellcli` 的用途远比我们在此介绍的要多。系统管理员会发现它对于使用 `useradd` 和 `groupadd` 命令在数据库服务器上创建新用户账户等任务非常有用。数据库管理员（DBA）会发现 `dcli` 对于将脚本和其他文件分发到集群中的其他服务器非常有用。而结合使用 `dcli` 和 `cellcli` 提供了一种管理、提取和报告来自存储单元的关键性能指标的便捷方式。将这些任务安排为 `cron` 作业，并如那个小示例所示将其加载到数据库中，可以让你为后续分析保留一个很好的性能相关信息存储库。


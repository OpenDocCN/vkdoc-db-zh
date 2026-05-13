# CELLCLI 与 DCLI

在前几章中，您已经多次看到对 `dcli` 和 `cellcli` 的引用。尽管其语法看起来相当直观，但更深入地探讨这些工具的功能显然是必要的。您可能是通过另一章的引用来到这里。本附录并非旨在全面讨论您能用这些工具做什么——您有《Exadata 存储服务器软件用户指南》的第 8 章（使用 CellCLI 实用程序）和第 9 章（使用 dcli 实用程序）来负责这部分内容。本附录更像是您入门并了解 Exadata 系统两个最实用配置工具的指南。`cellcli` 部分稍长一些，以体现这两个实用程序中功能更强者的地位。

`cellcli` 是一个命令解释器，您可以通过它来管理存储单元。可以理解，它在计算节点上不可用。`cellcli` 之于存储单元，就如同 `SQL*Plus` 之于数据库实例。本附录将更详细介绍的另一个实用程序是 `dcli`。它是一个可以一次性向所有数据库服务器和/或存储单元发送单一命令的工具。其他功能还包括将文件复制到多个位置以及复制 SSH 密钥。任何在支持 Exadata 之前曾担任过 RAC 管理员的人，可能都会和作者一样认为，`dcli` 是每个 RAC 系统都应该具备的工具。在八分之一或四分之一机架的情况下，它可能听起来没那么有用，但一旦您开始管理半机架或全机架，您就会开始欣赏这种能够一次在所有集群节点上执行命令的能力。

## CellCLI 简介

顾名思义，Exadata 存储软件使用 `cellcli` 实用程序作为其命令行界面。我们在本书第一版中曾抱怨缺乏 `cellcli` 的语法参考。当时，您唯一的帮助是在线帮助或本附录。值得庆幸的是，情况已经好转，《Exadata 存储管理员指南》有一章专门介绍 `cellcli` 的使用。不过，由于您可能不是在电子设备上阅读本书（或者身边可能没有设备），我们还是决定再包含一点参考内容。在编写本附录时，我们还发现文档在某些地方有些滞后，所以我们认为应该包含一些我们在使用过程中学到的内容。

有趣的是，Oracle 选择编写一个全新的命令行工具来管理存储单元。Oracle 本可以使用 `SQL*Plus`，它已成为管理数据库和 ASM 最著名的工具。尽管如此，`cellcli` 仍是您将用于管理存储单元的工具。其语法与 `SQL*Plus` 有些不同，但也存在相似之处，特别是在 `LIST` 命令上。`LIST` 用于执行查询，它看起来与 DBA 已经习惯的 `SELECT` 命令非常相似。和 `SELECT` 一样，它有 `WHERE` 和 `LIKE` 关键字，允许您从输出中过滤掉不需要的信息。

以下是我们总结的关于 `cellcli` 的十大要点：

*   `cellcli` 确实实现了一些 `SQL*Plus` 命令（`START (@)`、`SET ECHO ON`、`SPOOL`、`DESCRIBE`、`REM` 和 `HELP`）。
*   `SELECT` 被 `LIST` 取代，它必须是命令行的第一个关键字。
*   没有 `FROM` 关键字（`LIST` 关键字后必须紧跟 `ObjectType`，它相当于表名）。
*   列名使用 `ATTRIBUTES` 关键字指定，后跟您希望显示的列。
*   存在一个 `DESCRIBE` 命令，用于显示构成 `ObjectType`（表）的属性（列）。
*   每个 `ObjectType` 都有一组默认列，如果不指定 `ATTRIBUTES` 关键字，将返回这些列。
*   有一个 `WHERE` 子句可以应用于任何属性，并且多个条件可以用 `AND` 组合；但是，不支持 `OR`。
*   与本书第一版不同，在 12.1.2.1.0 及更高版本中，存在一个等同于 `ORDER BY` 的功能，并且您可以将输出限制为最多 200 行。
*   `DETAIL` 关键字可以附加到任何 `LIST` 命令，以将输出从按列显示更改为按行显示。
*   `LIKE` 运算符有效，但它不是使用标准的 SQL 通配符 `%`，而是使用一种简单的正则表达式形式，因此我们熟悉的 `SQL*Plus` 中的 `%` 变成了 `.*`，匹配任意字符零次或多次。

稍加练习后，您就会对 `cellcli` 感到得心应手。在大多数情况下，您将使用 `LIST` 命令来查询存储单元的各个（性能）方面。


### 调用 cellcli

当你在命令行执行 `cellcli` 时，实际启动的是一个 bash 脚本。该脚本是一个封装器，包含健全性检查、加载环境变量，并最终执行 Java 代码。默认情况下，在连接到存储单元时，于命令行输入 `cellcli` 将进入一个交互式会话。你在会话中享有的权限取决于你用于连接存储单元的账户。你可以选择以 `root`、`celladmin` 或 `cellmonitor` 的身份连接，其中 `root` 权限最高（请小心！），`cellmonitor` 权限最低。

调用时，你可以传递一些命令行选项。最相关的是 `` `-e` `` 用于在非交互式会话中执行命令，以及 `` `-xml` `` 用于生成 XML 输出。如果遇到问题，你可以使用 `` `-v` `` 到 `` `-vvv` `` 来生成更详细的日志记录。还存在其他选项，但与本章内容无关。

由于 `cellcli` 将输出写到 `STDOUT`，你也可以使用你喜欢的 UNIX 工具：只需将 `cellcli` 的输出通过管道传递给 `sed`、`awk`、`grep`、`sort`、`uniq`、`nl` 或任何其他众多能帮助你处理文本的 UNIX 工具。这些工具为 12.1.2.1.0 版本之前的存储单元提供了一种执行数据切片和切块的好方法。下面是一个如何模拟缺失的 `order by` 子句的示例——首先在 12.1.2.1.0 版本上演示，向你展示如何在当前的 Exadata 软件版本中实现：

```
# cellcli -e list metriccurrent where objectType = 'SMARTIO' order by metricValue limit 30
SIO_IO_EL_OF_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RD_FC_HD_SEC            SMARTIO         0.000 MB/sec
SIO_IO_RD_RQ_FC_HD_SEC         SMARTIO         0.0 IO/sec
SIO_IO_RD_RQ_FC_SEC            SMARTIO         0.0 IO/sec
SIO_IO_OF_RE_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RD_RQ_HD_SEC            SMARTIO         0.0 IO/sec
SIO_IO_RV_OF                   SMARTIO         0.000 MB
SIO_IO_RV_OF_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RD_FC_SEC               SMARTIO         0.000 MB/sec
SIO_IO_SI_SV_SEC               SMARTIO         0.000 MB/sec
SIO_IO_PA_TH                   SMARTIO         0.000 MB
SIO_IO_WR_FC_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RD_HD_SEC               SMARTIO         0.000 MB/sec
SIO_IO_WR_HD_SEC               SMARTIO         0.000 MB/sec
SIO_IO_PA_TH_SEC               SMARTIO         0.000 MB/sec
SIO_IO_WR_RQ_FC_SEC            SMARTIO         0.0 IO/sec
SIO_IO_WR_RQ_HD_SEC            SMARTIO         0.0 IO/sec
SIO_IO_RD_FC_HD                SMARTIO         577 MB
SIO_IO_RD_RQ_FC_HD             SMARTIO         582 IO requests
SIO_IO_WR_FC                   SMARTIO         106,126 MB
SIO_IO_WR_RQ_FC                SMARTIO         106,564 IO requests
SIO_IO_OF_RE                   SMARTIO         128,723 MB
SIO_IO_WR_HD                   SMARTIO         225,426 MB
SIO_IO_WR_RQ_HD                SMARTIO         226,354 IO requests
SIO_IO_SI_SV                   SMARTIO         235,694 MB
SIO_IO_RD_FC                   SMARTIO         408,460 MB
SIO_IO_RD_RQ_FC                SMARTIO         412,032 IO requests
SIO_IO_RD_HD                   SMARTIO         573,819 MB
SIO_IO_RD_RQ_HD                SMARTIO         574,816 IO requests
SIO_IO_EL_OF                   SMARTIO         1,481,339 MB
```

在较旧版本的存储单元软件中，你会发现该语法不受支持。另一方面，如第二个示例所示，可以使用 UNIX `sort` 命令作为替代。

```
# cellcli -e cellcli -e list metriccurrent where objectType = 'SMARTIO' order by metricValue
CELL-01504: Invalid command syntax.
# cellcli -e list metriccurrent where objectType = 'SMARTIO' | sort -n -k 3 | head –n30
SIO_IO_EL_OF_SEC               SMARTIO         0.000 MB/sec
SIO_IO_OF_RE_SEC               SMARTIO         0.000 MB/sec
SIO_IO_PA_TH_SEC               SMARTIO         0.000 MB/sec
SIO_IO_PA_TH                   SMARTIO         0.000 MB
SIO_IO_RD_FC_HD_SEC            SMARTIO         0.000 MB/sec
SIO_IO_RD_FC_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RD_HD_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RD_RQ_FC_HD_SEC         SMARTIO         0.0 IO/sec
SIO_IO_RD_RQ_FC_SEC            SMARTIO         0.0 IO/sec
SIO_IO_RD_RQ_HD_SEC            SMARTIO         0.0 IO/sec
SIO_IO_RV_OF_SEC               SMARTIO         0.000 MB/sec
SIO_IO_RV_OF                   SMARTIO         0.000 MB
SIO_IO_SI_SV_SEC               SMARTIO         0.000 MB/sec
SIO_IO_WR_FC_SEC               SMARTIO         0.000 MB/sec
SIO_IO_WR_HD_SEC               SMARTIO         0.000 MB/sec
SIO_IO_WR_RQ_FC_SEC            SMARTIO         0.0 IO/sec
SIO_IO_WR_RQ_HD_SEC            SMARTIO         0.0 IO/sec
SIO_IO_RD_FC_HD                SMARTIO         577 MB
SIO_IO_RD_RQ_FC_HD             SMARTIO         582 IO requests
SIO_IO_WR_FC                   SMARTIO         106,126 MB
SIO_IO_WR_RQ_FC                SMARTIO         106,564 IO requests
SIO_IO_OF_RE                   SMARTIO         128,723 MB
SIO_IO_WR_HD                   SMARTIO         225,426 MB
SIO_IO_WR_RQ_HD                SMARTIO         226,354 IO requests
SIO_IO_SI_SV                   SMARTIO         235,694 MB
SIO_IO_RD_FC                   SMARTIO         408,460 MB
SIO_IO_RD_RQ_FC                SMARTIO         412,032 IO requests
SIO_IO_RD_HD                   SMARTIO         573,819 MB
SIO_IO_RD_RQ_HD                SMARTIO         574,816 IO requests
SIO_IO_EL_OF                   SMARTIO         1,481,339 MB
```

请善用这些 UNIX 工具并发挥创意！现在或许正是重温 `sed`、`awk` 和 `perl` 教程的好时机。


## 熟悉 cellcli

像任何优秀的命令行实用程序一样，`cellcli` 提供了在线帮助。这个内置的帮助系统比官方文档更准确，而且，作为一个提示，你应该对照帮助命令的输出来检查官方的 HTML 文档。为了向你展示在 Exadata 12.1.2.1.0 中使用 `cellcli` 能做些什么，以下是其输出：

```
CellCLI> help

HELP [主题]

可用主题：

ALTER
ALTER ALERTHISTORY
ALTER CELL
ALTER CELLDISK
ALTER FLASHCACHE
ALTER GRIDDISK
ALTER IBPORT
ALTER IORMPLAN
ALTER LUN
ALTER PHYSICALDISK
ALTER QUARANTINE
ALTER THRESHOLD
ASSIGN KEY
CALIBRATE
CREATE
CREATE CELL
CREATE CELLDISK
CREATE FLASHCACHE
CREATE FLASHLOG
CREATE GRIDDISK
CREATE KEY
CREATE QUARANTINE
CREATE THRESHOLD
DESCRIBE
DROP
DROP ALERTHISTORY
DROP CELL
DROP CELLDISK
DROP FLASHCACHE
DROP FLASHLOG
DROP GRIDDISK
DROP QUARANTINE
DROP THRESHOLD
EXPORT CELLDISK
IMPORT CELLDISK
LIST
LIST ACTIVEREQUEST
LIST ALERTDEFINITION
LIST ALERTHISTORY
LIST CELL
LIST CELLDISK
LIST DATABASE
LIST FLASHCACHE
LIST FLASHCACHECONTENT
LIST FLASHLOG
LIST GRIDDISK
LIST IBPORT
LIST IORMPLAN
LIST KEY
LIST LUN
LIST METRICCURRENT
LIST METRICDEFINITION
LIST METRICHISTORY
LIST PHYSICALDISK
LIST QUARANTINE
LIST THRESHOLD
SET
SPOOL
START
```

每个命令都可以使用 `HELP` 命令进一步描述，例如 `HELP LIST` 或 `HELP LIST CELL`。

```
CellCLI> help list

用法: LIST <对象类型> [<名称> | <筛选条件>] [<属性列表>] [DETAIL] \
[ORDER BY <排序属性列表>] [LIMIT 整数]

目的: LIST 命令显示 Oracle Exadata 服务器软件对象的属性。
显示的对象通过名称或筛选条件来标识。
每个对象显示的属性由指定的属性列表决定。

参数:
<对象类型>:   要显示的现有对象的类型。
<名称>:   要显示的活动请求的名称。
<筛选条件>:   确定应显示哪些活动请求的表达式。
<属性列表>: 要显示的属性。
ATTRIBUTES {ALL | attr1 [, attr2]... }
<排序属性列表>: 要排序的属性。
{attr1 [asc|desc] [, attr2 [asc|desc]]}

选项:
[DETAIL]: 将显示格式化为每行一个属性，并在每个值前加上属性描述符。
[ORDER BY]: 按属性对对象进行升序或降序排序。默认为升序。
[LIMIT]: 设置显示对象的数量。

输入 HELP LIST <对象类型> 获取具体的帮助语法。
<对象类型>:   {ACTIVEREQUEST | ALERTHISTORY | ALERTDEFINITION | CELL
| CELLDISK | DATABASE | FLASHCACHE | FLASHLOG | FLASHCACHECONTENT
| GRIDDISK | IBPORT | IORMPLAN | KEY | LUN
| METRICCURRENT | METRICDEFINITION | METRICHISTORY
| PHYSICALDISK | QUARANTINE | THRESHOLD }
```

```
CellCLI> help list cell

用法: LIST CELL [<属性列表>] [DETAIL]

目的: 显示单元的指定属性。

参数:
<属性列表>: 要显示的属性。
ATTRIBUTES {ALL | attr1 [, attr2]... }

选项:
[DETAIL]: 将显示格式化为每行一个属性，并在每个值前加上属性描述符。

示例:
LIST CELL attributes status, cellnumber
LIST CELL DETAIL
```

前面清单中显示的输出取自一个运行 Exadata 软件版本 12.1.2.1.0 的单元。如你所见，帮助系统允许你查看每个命令的语法。你可能还注意到了几个 SQL*Plus 的**遗留命令**。`SET`、`SPOOL` 和 `START` 的功能与预期基本一致。注意 `@` 字符等同于 SQL*Plus 的 `START` 命令，而你能用 `SET` 设置的只有 `ECHO` 和 `DATEFORMAT`。现在，这里有几个使用 `LIST` 命令的查询示例：

```
CellCLI> describe metriccurrent
name
alertState
collectionTime
metricObjectName
metricType
metricValue
objectType
```

## CellCLI 命令示例

以下展示了一系列 `CellCLI` 命令及其输出结果。

```cellcli
CellCLI> list metriccurrent where objectType = 'FLASHCACHE' and name not like '.*SEC' -
> attributes name, metricType, metricValue
FC_BYKEEP_OVERWR                        Cumulative      0.000 MB
FC_BYKEEP_USED                          Instantaneous   0.062 MB
FC_BY_ALLOCATED                         Instantaneous   337,591 MB
FC_BY_DIRTY                             Instantaneous   166,816 MB
FC_BY_STALE_DIRTY                       Instantaneous   373 MB
FC_BY_USED                              Instantaneous   363,063 MB
[many more skipped]
FC_IO_RQ_W_SKIP_NCMIRROR                Cumulative      0 IO requests
```

```cellcli
CellCLI> list metriccurrent where objectType = 'FLASHCACHE' and name not like '.*SEC' -
> and metricValue not like '0.*' attributes name, metricType, metricValue -
> order by metricValue desc limit 20
FC_IO_RQ_W                              Cumulative      1,325,800,847 IO requests
FC_IO_RQ_R                              Cumulative      1,323,518,596 IO requests
FC_IO_RQ_W_OVERWRITE                    Cumulative      1,288,444,158 IO requests
FC_IO_RQ_W_SKIP                         Cumulative      545,358,635 IO requests
FC_IO_RQ_W_FIRST                        Cumulative      35,716,955 IO requests
FC_IO_BY_W                              Cumulative      11,196,640 MB
FC_IO_BY_W_OVERWRITE                    Cumulative      10,702,060 MB
FC_IO_BY_R                              Cumulative      10,518,906 MB
FC_IO_RQ_R_SKIP                         Cumulative      4,422,859 IO requests
FC_IO_RQ_REPLACEMENT_ATTEMPTED          Cumulative      4,051,159 IO requests
FC_IO_RQ_R_SKIP_NCMIRROR                Cumulative      4,029,287 IO requests
FC_IO_RQ_R_DW                           Cumulative      2,614,954 IO requests
FC_IO_RQ_W_SKIP_LG                      Cumulative      2,558,104 IO requests
FC_IO_BY_W_SKIP                         Cumulative      2,467,692 MB
FC_IO_RQ_W_POPULATE                     Cumulative      1,639,734 IO requests
FC_IO_RQ_R_MISS                         Cumulative      1,581,764 IO requests
FC_IO_BY_W_SKIP_LG                      Cumulative      989,725 MB
FC_IO_RQ_DISK_WRITE                     Cumulative      800,551 IO requests
FC_IO_RQ_REPLACEMENT_FAILED             Cumulative      455,425 IO requests
FC_IO_BY_R_DW                           Instantaneous   404,121 MB
```

```cellcli
CellCLI> list metriccurrent where objectType = 'FLASHCACHE' and name not like '.*SEC' -
> and metricValue not like '0.*' attributes name, metricType, metricValue -
> order by metricType, metricValue desc limit 20
FC_IO_RQ_W                              Cumulative      1,325,801,132 IO requests
FC_IO_RQ_R                              Cumulative      1,323,520,259 IO requests
FC_IO_RQ_W_OVERWRITE                    Cumulative      1,288,444,442 IO requests
FC_IO_RQ_W_SKIP                         Cumulative      545,359,973 IO requests
FC_IO_RQ_W_FIRST                        Cumulative      35,716,956 IO requests
FC_IO_BY_W                              Cumulative      11,196,644 MB
FC_IO_BY_W_OVERWRITE                    Cumulative      10,702,065 MB
FC_IO_BY_R                              Cumulative      10,518,931 MB
FC_IO_RQ_R_SKIP                         Cumulative      4,424,593 IO requests
FC_IO_RQ_REPLACEMENT_ATTEMPTED          Cumulative      4,051,159 IO requests
FC_IO_RQ_R_SKIP_NCMIRROR                Cumulative      4,030,357 IO requests
FC_IO_RQ_R_DW                           Cumulative      2,614,954 IO requests
FC_IO_RQ_W_SKIP_LG                      Cumulative      2,558,201 IO requests
FC_IO_BY_W_SKIP                         Cumulative      2,467,735 MB
FC_IO_RQ_W_POPULATE                     Cumulative      1,639,734 IO requests
FC_IO_RQ_R_MISS                         Cumulative      1,581,764 IO requests
FC_IO_BY_W_SKIP_LG                      Cumulative      989,754 MB
FC_IO_RQ_DISK_WRITE                     Cumulative      800,551 IO requests
FC_IO_RQ_REPLACEMENT_FAILED             Cumulative      455,425 IO requests
FC_IO_BY_W_FIRST                        Cumulative      378,110 MB
```

## 命令语法说明

`DESCRIBE` 动词的工作方式与 `SQL*Plus` 中类似，但必须完整拼写；不能使用熟悉的 `DESC` 作为缩写。请注意，列式输出没有标题。许多 `LIST` 命令通过使用续行符 (`-`) 被串接在多行上。`LIST` 命令看起来很像 `SQL`，只是用 `LIST` 代替了 `SELECT`，并且在使用 `LIKE` 关键词时使用了正则表达式进行匹配。你可以看到 `ATTRIBUTES` 和 `WHERE` 关键词可以在命令行上出现在 `LIST ObjectType` 关键词之后的任何位置。换句话说，这两个关键词不是位置相关的；任何一个都可以先使用。

前面列表中的第一个示例可以在任何 `Exadata` 单元版本上执行。`LIST` 命令用于显示与单元的 `FLASHCACHE` 性能指标相关的一组特定属性。更具体地说，那些计算“每秒”值的指标被排除了。

第二个示例在前一个示例的基础上进行了扩展，指定了一个 `ORDER BY`，这对于 `12.1.2.1.0` 及更高版本是新的。通常，当指定了 `ORDER BY` 时，还必须提供一个 `LIMIT` 子句，否则会引发类似以下的错误：

```cellcli
CellCLI> list metriccurrent where objectType = 'FLASHCACHE' and name not like '.*SEC' -
> and metricValue not like '0.*' attributes name, metricType, metricValue -
> order by metricValue desc
CELL-02026: The LIMIT parameter is mandatory for "LIST METRICCURRENT" command when using the ORDER BY option.
```

避免此限制的一种替代方法是如前所示将输出通过管道传输到 `UNIX` `sort` 命令。请注意，你可以按升序 (`ASC`，默认) 或降序 (`DESC`) 排序。你不仅限于只按一列排序。最后一个示例显示了可以按多列排序。

### 从操作系统发送命令

除了你刚才在这些示例中看到的交互式运行 `cellcli` 外，你在介绍中已经了解到，你可以指定 `-e` 选项，从操作系统提示符传入 `cellcli` 命令（甚至可以通过 `dcli`，你稍后会看到）。例如，以下列表显示了如何使用 `-e` 选项直接从 `OS` 命令行查询 `cellsrv` 的状态：

```bash
[root@enkcel04 ∼]# cellcli -e "list cell detail"
          name:                       enkcel04
          buStatus:                   normal
          cellVersion:                OSS_12.1.2.1.0_LINUX.X64_141206.1
          cpuCount:                   24
          diagHistoryDays:            7
          fanCount:                   12/12
          fanStatus:                  normal
          flashCacheMode:             WriteBack
[...]
          usbStatus:                  normal
          cellsrvStatus:              running
          msStatus:                   running
          rsStatus:                   running
```

在其他方面，当你想要从操作系统 `shell` 脚本内部调用 `cellcli` 时，`-e` 选项很有帮助。使用 `-e` 参数向 `cellcli` 传递命令时需要小心——转义引号会变得非常重要，正如你将在本章后面看到的那样。


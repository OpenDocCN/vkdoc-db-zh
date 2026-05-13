# Oracle 数据泵任务与操作指南

## 排除和包含导入对象

在导入时，可以使用 `exclude` 参数排除特定对象类型，如触发器、存储过程等。
例如，在参数文件中设置 `exclude=TRIGGER,PROCEDURE`。

可以通过添加 SQL 子句来进一步细化排除条件。例如，如果不想导入以字母 `B` 开头的触发器，参数文件内容如下：
```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
schemas=HEERA
exclude=trigger:"like 'B%'"
```

### 包含对象导入

可以使用 `INCLUDE` 参数来限制导入的内容。假设需要从某个模式中导入所有以字母 `A` 开头的表，参数文件如下：
```
userid=mv_maint/foo
directory=dp_dir
dumpfile=inv.dmp
schemas=HEERA
include=table:"like 'A%'"
```

将上述内容保存到名为 `h.par` 的文件后，可以按如下方式调用参数文件：
```
$ impdp parfile=h.par
```

在此示例中，`HEERA` 模式必须已存在。只有以字母 `A` 开头的表会被导入。

### 常见数据泵任务

以下各节介绍数据泵的常用功能。其中许多功能是数据泵的标准功能，例如创建一致性导出以及处理导入对象已存在于数据库中的情况。其他功能，如压缩和加密，则需要 Oracle 企业版或额外许可，或两者兼需。在介绍相关数据泵元素时，我会指出这些要求（如适用）。

### 估算导出作业大小

如果即将导出大量数据，可以在运行导出前估算数据泵创建的文件大小。这样做可能是因为担心导出作业所需的空间量。

要估算大小，请使用 `ESTIMATE_ONLY` 参数。此示例估算整个数据库导出文件的大小：
```
$ expdp mv_maint/foo estimate_only=y full=y logfile=n
```

部分输出示例如下：
```
Estimate in progress using BLOCKS method...
Total estimation using BLOCKS method: 6.75 GB
```

同样，可以指定模式名来估算导出特定用户所需的空间大小：
```
$ expdp mv_maint/foo estimate_only=y schemas=star2 logfile=n
```

以下示例演示了估算两张表所需大小：
```
$ expdp mv_maint/foo estimate_only=y tables=star2.f_configs,star2.f_installations \
logfile=n
```

### 列出转储文件内容

数据泵提供了一种非常强大的方法来创建包含导入作业运行时执行的所有 SQL 语句的文件。数据泵使用 `DBMS_METADATA` 包来创建可用于在数据泵转储文件中重建对象的 DDL。

使用数据泵导入的 `SQLFILE` 选项可以列出数据泵导出文件的内容。此示例创建一个名为 `expfull.sql` 的文件，其中包含导入过程将调用的 SQL 语句（文件放置在由 `DPUMP_DIR2` 目录对象定义的目录中）：
```
$ impdp hr/hr DIRECTORY=dpump_dir1 DUMPFILE=expfull.dmp \
SQLFILE=dpump_dir2:expfull.sql
```

如果未指定单独的目录（例如上例中的 `dpump_dir2`），则 SQL 文件将写入 `DIRECTORY` 选项指定的位置。

```
提示
```
必须使用具有 DBA 权限的用户或执行数据泵导出的模式用户来运行上述命令。否则，将得到一个不包含预期 SQL 语句的空 SQL 文件。

当对导入使用 `SQLFILE` 选项时，`impdp` 进程不会导入任何数据；它仅创建一个包含导入过程将运行的 SQL 命令的文件。

有时生成 SQL 文件非常方便，原因如下：
* 在运行导入前预览和验证 SQL 语句
* 手动运行 SQL 以预创建数据库对象
* 捕获重建数据库对象（用户、表、索引等）所需的 SQL

关于最后一点，有时签入源代码控制仓库的内容与实际应用于生产数据库的内容并不匹配。此过程对于故障排除或记录数据库在某个时间点的状态非常方便。

### 克隆用户

假设需要将用户对象和数据移动到新数据库。作为迁移的一部分，希望重命名该用户。首先，创建一个包含要克隆用户的模式级导出文件。在此示例中，用户名为 `INV`：
```
$ expdp mv_maint/foo directory=dp_dir schemas=inv dumpfile=inv.dmp
```

现在，可以使用数据泵导入来克隆用户。如果要将用户移动到不同的数据库，请将转储文件复制到远程数据库，并使用 `REMAP_SCHEMA` 参数创建用户的副本。在此示例中，`INV` 用户被克隆为 `INV_DW` 用户：
```
$ impdp mv_maint/foo directory=dp_dir remap_schema=inv:inv_dw dumpfile=inv.dmp
```

此命令将 `INV` 用户中的所有结构和数据复制到 `INV_DW` 用户。最终的 `INV_DW` 用户在对象方面与 `INV` 用户相同。复制的模式也包含与源模式相同的密码。

如果只想将元数据从一个模式复制到另一个模式，请使用带有 `METADATA_ONLY` 选项的 `CONTENT` 参数：
```
$ impdp mv_maint/foo directory=dp_dir remap_schema=inv:inv_dw \
content=metadata_only dumpfile=inv.dmp
```

`REMAP_SCHEMA` 参数提供了一种高效复制模式的方法，无论是否包含数据。在模式复制操作期间，如果希望更改对象所在的表空间，还可以使用 `REMAP_TABLESPACE` 参数。这允许您复制模式并将对象放置在与源对象不同的表空间中。

也可以无需先创建转储文件就将用户从一个数据库复制到另一个数据库。为此，请使用 `NETWORK_LINK` 参数。有关直接从一个数据库复制数据的详细信息，请参阅本章前面的“直接通过网络导出和导入”部分。

### 创建一致性导出

一致性导出意味着导出文件中的所有数据都截至某个时间点或 SCN 一致。当导出具有许多父-子表的活动数据库时，应确保获得数据的一致性快照。

```
提示
```
如果使用的是 Oracle 11g Release 2 或更高版本，可以通过调用传统模式参数 `CONSISTENT=Y` 来进行一致性导出。有关详细信息，请参阅本章后面的“数据泵传统模式”部分。

通过使用 `FLASHBACK_SCN` 或 `FLASHBACK_TIME` 参数可以创建一致性导出。此示例使用 `FLASHBACK_SCN` 参数进行导出。要确定数据集的当前 SCN 值，请发出以下查询：
```
SQL> select current_scn from v$database;
```

典型输出如下：
```
 CURRENT_SCN
-----------
     5715397
```

以下命令使用 `FLASHBACK_SCN` 参数对数据库进行一致性完全导出：
```
$ expdp mv_maint/foo directory=dp_dir full=y flashback_scn=5715397 \
dumpfile=full.dmp
```

前面的导出命令确保所有导出的数据与指定 SCN 时数据库中已提交的任何事务保持一致。

当使用 `FLASHBACK_SCN` 参数时，数据泵确保导出文件中的数据截至指定的 SCN 是一致的。



这意味着在指定的 SCN 之后提交的任何事务都不会包含在导出文件中。

`注意`：如果将 `NETWORK_LINK` 参数与 `FLASHBACK_SCN` 结合使用，则导出将使用与数据库链接中引用的数据库一致的 SCN 进行。

## 使用 FLASHBACK_TIME 指定导出一致性

您还可以使用 `FLASHBACK_TIME` 来指定导出文件应基于指定时间点的、一致的已提交事务来创建。使用 `FLASHBACK_TIME` 时，Oracle 会确定与指定时间最匹配的 SCN，并使用该 SCN 来生成一致的导出。使用 `FLASHBACK_TIME` 的语法如下：

```
FLASHBACK_TIME="TO_TIMESTAMP{<value>}"
"
```

对于某些操作系统，直接在命令行上出现的双引号必须用反斜杠 (`\`) 转义，因为操作系统将它们视为特殊字符。因此，使用参数文件要直接得多。以下是一个使用 `FLASHBACK_TIME` 的参数文件内容：

```
directory=dp_dir
content=metadata_only
dumpfile=inv.dmp
flashback_time="to_timestamp('24-jan-2014 07:03:00','dd-mon-yyyy hh24:mi:ss')"
```

根据您的操作系统，前述示例的命令行版本必须指定如下：

```
flashback_time=\"to_timestamp\(\'24-jan-2014 07:03:00\', \'dd-mon-yyyy hh24:mi:ss\'\)\"
```

此行代码应在一行上指定。这里，为了适应页面，代码被放在了两行上。

`注意`：执行导出时，不能同时指定 `FLASHBACK_SCN` 和 `FLASHBACK_TIME`；这两个参数是互斥的。如果您尝试同时使用这两个参数，Data Pump 将抛出以下错误消息并停止导出作业：

```
ORA-39050: parameter FLASHBACK_TIME is incompatible with parameter FLASHBACK_SCN
```

## 当对象已存在时进行导入

在导出和导入数据时，您经常需要导入到其中对象（表、索引等）已创建的模式中。在这种情况下，您应该导入数据，但指示 Data Pump 尝试不创建已存在的对象。

您可以使用 `TABLE_EXISTS_ACTION` 和 `CONTENT` 参数来实现这一点。下一个示例通过 `TABLE_EXISTS_ACTION=APPEND` 选项指示 Data Pump 将数据附加到任何已存在的表中。同时还使用了 `CONTENT=DATA_ONLY` 选项，该选项指示 Data Pump 不要运行任何 DDL 来创建对象（仅加载数据）：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
table_exists_action=append content=data_only
```

现有对象不会以任何方式被修改，而转储文件中存在的任何新数据都会插入到相应的表中。

您可能想知道如果只使用 `TABLE_EXISTS_ACTION` 选项而不将其与 `CONTENT` 选项结合会发生什么：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp \
table_exists_action=append
```

唯一的区别是，如果对象存在，Data Pump 会尝试运行 DDL 命令来创建它们。这不会阻止作业运行，但您会在输出中看到一条错误消息，指示对象已存在。以下是前述命令输出的一个片段：

```
Table "MV_MAINT"."INV" exists. Data will be appended ...
```

`TABLE_EXISTS_ACTION` 参数的默认值是 `SKIP`，除非您还指定了 `CONTENT=DATA_ONLY` 参数。如果您使用 `CONTENT=DATA_ONLY`，那么 `TABLE_EXISTS_ACTION` 的默认值是 `APPEND`。

### TABLE_EXISTS_ACTION 选项

`TABLE_EXISTS_ACTION` 参数接受以下选项：

*   `SKIP`（默认，如果未与 `CONTENT=DATA_ONLY` 结合使用）
*   `APPEND`（默认，如果与 `CONTENT=DATA_ONLY` 结合使用）
*   `REPLACE`
*   `TRUNCATE`

`SKIP` 选项告诉 Data Pump，如果对象存在则不处理该对象。`APPEND` 选项指示 Data Pump 不要删除现有数据，而是向表中添加数据，而不修改任何现有数据。

`REPLACE` 选项指示 Data Pump 删除并重新创建对象；当 `CONTENT` 参数与 `DATA_ONLY` 选项一起使用时，此参数无效。`TRUNCATE` 参数告诉 Data Pump 通过 `TRUNCATE` 语句删除表中的行。

### CONTENT 选项

`CONTENT` 参数接受以下选项：

*   `ALL`（默认）
*   `DATA_ONLY`
*   `METADATA_ONLY`

`ALL` 选项指示 Data Pump 加载转储文件中包含的数据和元数据；这是默认行为。`DATA_ONLY` 选项告诉 Data Pump 仅将表数据加载到现有表中；不创建任何数据库对象。`METADATA_ONLY` 选项仅创建对象；不加载任何数据。

## 重命名表

从 Oracle 11g 开始，您可以在导入操作期间选择重命名表。导入时重命名表的原因有很多。例如，目标模式中可能有一个表与您要导入的表同名。您可以使用 `REMAP_TABLE` 参数在导入时重命名表。此示例将 `HEERA` 用户的 `INV` 表从转储文件导入为 `HEERA` 用户的 `INVEN` 表：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp tables=heera.inv \
remap_table=heera.inv:inven
```

以下是重命名表的一般语法：

```
REMAP_TABLE=[schema.]old_tablename[.partition]:new_tablename
```

`注意`：此语法不允许您将表重命名到不同的模式中。如果不小心，您可能会尝试执行以下操作（认为您是在一个操作中移动并重命名表）：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp tables=heera.inv \
remap_table=heera.inv:scott.inven
```

在前面的示例中，您最终会在 `HEERA` 模式中得到一个名为 `SCOTT` 的表。这可能会引起混淆。

`注意`：在 Oracle 11g Release 1 中，重命名表的过程并非完全没有错误，但在 Oracle 11g Release 2 中已得到纠正。有关更多详细信息，请参阅 MOS Note 886762.1。

## 重映射数据

从 Oracle 11g 开始，在导出或导入时，您可以应用 PL/SQL 函数来更改列值。例如，您可能有一位审计员需要查看数据，其中一个要求是对敏感列应用简单的混淆函数。数据不需要加密；只需要进行足够改变，使得审计员无法轻易确定 `CUSTOMERS` 表中 `LAST_NAME` 列的值。

此示例首先创建一个用于混淆数据的简单包：

```
create or replace package obfus is
  function obf(clear_string varchar2) return varchar2;
  function unobf(obs_string varchar2) return varchar2;
end obfus;
/
create or replace package body obfus is
  fromstr varchar2(62) := '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ' ||
             'abcdefghijklmnopqrstuvwxyz';
  tostr varchar2(62)   := 'defghijklmnopqrstuvwxyzabc3456789012' ||
             'KLMNOPQRSTUVWXYZABCDEFGHIJ';
  function obf(clear_string varchar2) return varchar2 is
  begin
    return translate(clear_string, fromstr, tostr);
  end obf;
  function unobf(obs_string varchar2) return varchar2 is
  begin
    return translate(obs_string, tostr, fromstr);
  end unobf;
end obfus;
/
```

现在，当您将数据导入数据库时，您将混淆函数应用于 `CUSTOMERS` 表的 `LAST_NAME` 列：

```
$ impdp mv_maint/foo directory=dp_dir dumpfile=cust.dmp tables=customers  \
remap_data=customers.last_name:obfus.obf
```

从 `CUSTOMERS` 表中选择 `LAST_NAME` 显示它已被混淆的方式导入：

```
SQL> select last_name from customers;
LAST_NAME
------------------
yYZEJ
tOXXSMU
xERX
```

您可以手动应用包的 `UNOBF` 函数来查看列的真实值：

```
SQL> select obfus.unobf(last_name) from customers;
OBFUS.UNOBF(LAST_NAME)
------------------------
Smith
Johnson
Williams
```



## 抑制日志文件
默认情况下，Data Pump 在执行导出或导入时会创建一个日志文件。如果您知道不需要生成日志文件，可以通过指定 `NOLOGFILE` 参数来抑制它。这是一个示例：

```bash
$ expdp mv_maint/foo directory=dp_dir tables=inv nologfile=y
```

如果您选择不创建日志文件，Data Pump 仍然会在输出设备上显示状态消息。一般来说，我建议您为每个 Data Pump 操作都创建一个日志文件。这为您提供了操作的审计跟踪。

## 使用并行
使用 `PARALLEL` 参数来并行化一个 Data Pump 作业。例如，如果您知道一台服务器上有四个 CPU，并希望将并行度设置为 4，可以如下使用 `PARALLEL`：

```bash
$ expdp mv_maint/foo parallel=4 dumpfile=exp.dmp directory=dp_dir full=y
```

为了充分利用并行功能，请确保在导出时指定多个文件。以下示例为每个并行线程创建一个文件：

```bash
$ expdp mv_maint/foo parallel=4 dumpfile=exp1.dmp,exp2.dmp,exp3.dmp,exp4.dmp
```

您也可以使用 `%U` 替代变量来指示 Data Pump 自动创建与并行度匹配的转储文件。`%U` 变量从值 01 开始，并在分配额外的转储文件时递增。此示例使用了 `%U` 变量：

```bash
$ expdp mv_maint/foo parallel=4 dumpfile=exp%U.dmp
```

现在，假设您需要从由导出创建的转储文件中导入。您可以单独指定转储文件，或者，如果转储文件是使用 `%U` 变量创建的，则在导入时也使用它：

```bash
$ impdp mv_maint/foo parallel=4 dumpfile=exp%U.dmp
```

在上面的示例中，导入进程首先查找名为 `exp01.dmp` 的文件，然后是 `exp02.dmp`，依此类推。

![Image](img/sq.jpg) **提示** Oracle 建议并行度不要设置为服务器上可用 CPU 数量的两倍以上。

您也可以在作业运行时修改并行度。首先，以交互命令模式附加到要修改并行度的作业（请参阅本章后面的“交互命令模式”部分），然后使用 `PARALLEL` 选项。此示例中附加到的作业是 `SYS_IMPORT_TABLE_01`：

```bash
$ impdp mv_maint/foo attach=sys_import_table_01
Import> parallel=6
```

您可以通过 `STATUS` 命令检查并行度：

```bash
Import> status
```

这是一些示例输出：

```
Job: SYS_IMPORT_TABLE_01
  Operation: IMPORT
  Mode: TABLE
  State: EXECUTING
  Bytes Processed: 0
  Current Parallelism: 6
```

![Image](img/sq.jpg) **注意** `PARALLEL` 功能仅在 Oracle 企业版中可用。

## 指定额外的转储文件
如果主数据泵位置空间不足，您可以动态指定额外的数据泵位置。使用交互式命令提示符下的 `ADD_FILE` 命令。以下是添加额外文件的基本语法：

```
ADD_FILE=[directory_object:]file_name [,...]
```

此示例向已存在的 Data Pump 导出作业添加另一个输出文件：

```bash
Export> add_file=alt2.dmp
```

您也可以指定一个单独的数据库目录对象：

```bash
Export> add_file=alt_dir:alt3.dmp
```

## 重用输出文件名
默认情况下，Data Pump 不会覆盖现有的转储文件。例如，第一次运行此作业时，它会正常运行，因为所使用的目录中没有名为 `inv.dmp` 的转储文件：

```bash
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp
```

如果您尝试使用相同的目录和数据泵名称再次运行上述命令，将会抛出此错误：

```
ORA-31641: unable to create dump file "/oradump/inv.dmp"
```

您可以为导出作业指定一个新的数据泵名称，或者使用 `REUSE_DUMPFILES` 参数来指示 Data Pump 覆盖现有的转储文件；例如：

```bash
$ expdp mv_maint/foo directory=dp_dir dumpfile=inv.dmp reuse_dumpfiles=y
```

现在，无论输出目录中是否存在同名的现有转储文件，您都应该能够运行 Data Pump 导出。当您将 `REUSE_DUMPFILES` 设置为 `y` 时，如果 Data Pump 找到同名的转储文件，它将覆盖该文件。

![Image](img/sq.jpg) **注意** `REUSE_DUMPFILES` 的默认值为 `n`。`REUSE_DUMPFILES` 参数仅在 Oracle 11g 及更高版本中可用。

## 创建每日 DDL 文件
有时，在数据库环境中，数据库对象会以意想不到的方式发生变化。您可能有一位开发人员以某种方式获得了生产用户密码，并决定在未经任何人告知的情况下即时进行更改。或者，DBA 在排查问题时可能决定不遵循标准发布流程而更改对象。这些情况对于生产支持 DBA 来说可能是令人沮丧的。每当出现问题时，首先被提出的问题是：“什么发生了变化？”

当您使用 Data Pump 时，创建一个包含用于重新创建数据库中所有对象的 DDL 的文件相当简单。您可以通过 `CONTENT=METADATA_ONLY` 选项指示 Data Pump 仅导出或导入元数据。

例如，在生产环境中，您可以设置一个每日作业来捕获此 DDL。如果对发生了什么变化以及何时发生变化有疑问，您可以回溯并比较每日转储文件中的 DDL。

下面列出的是一个简单的 shell 脚本，它首先从数据库导出元数据内容，然后使用 Data Pump 导入从该导出创建 DDL 文件：

```bash
#!/bin/bash
export ORACLE_SID=O12C
export ORACLE_HOME=/orahome/app/oracle/product/12.1.0.1/db_1
export PATH=$PATH:$ORACLE_HOME/bin
#
DAY=$(date +%Y_%m_%d)
SID=DWREP
#---------------------------------------------------
# 首先创建仅包含元数据的导出转储文件
expdp mv_maint/foo dumpfile=${SID}.${DAY}.dmp content=metadata_only \
directory=dp_dir full=y logfile=${SID}.${DAY}.log
#---------------------------------------------------
# 现在从导出转储文件创建 DDL 文件。
impdp mv_maint/foo directory=dp_dir dumpfile=${SID}.${DAY}.dmp \
SQLFILE=${SID}.${DAY}.sql logfile=${SID}.${DAY}.sql.log
#
# exit 0
```

此代码清单依赖于已创建一个数据库目录对象，该对象指向您希望每日转储文件写入的位置。您可能还想设置另一个作业，该作业定期删除超过一定时间的文件。

## 压缩输出
当您使用 Data Pump 创建大文件时，应该考虑压缩输出。从 Oracle 11g 开始，`COMPRESSION` 参数可以是以下值之一：`ALL`、`DATA_ONLY`、`METADATA_ONLY` 或 `NONE`。如果您指定 `ALL`，则输出中的数据和元数据都将被压缩。此示例导出一个表并压缩输出文件中的数据和元数据：

```bash
$ expdp dbauser/foo tables=locations directory=datapump \
dumpfile=compress.dmp compression=all
```

如果您使用的是 Oracle 10g，则 `COMPRESSION` 参数只有 `METADATA_ONLY` 和 `NONE` 值。

![Image](img/sq.jpg) **注意** `COMPRESS` 参数的 `ALL` 和 `DATA_ONLY` 选项需要 Oracle 高级压缩选项的许可。

Oracle 12c 的新功能是，您可以指定一种压缩算法。选择有 `BASIC`、`LOW`、`MEDIUM` 和 `HIGH`。这是一个使用 `MEDIUM` 压缩的示例：

```bash
$ expdp mv_maint/foo dumpfile=full.dmp compression=medium
```



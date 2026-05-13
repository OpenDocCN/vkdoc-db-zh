# 第 17 章 ■ 数据库管理

## 工作原理

我们建议您在创建所有表空间时，都使用 `EXTENT MANAGEMENT LOCAL` 来指定每个表空间在本地管理其区。本地管理的表空间在数据文件中使用位图映像，以高效地确定一个区是否被使用。对于本地管理的表空间，存储参数 `NEXT`、`PCTINCREASE`、`MINEXTENTS`、`MAXEXTENTS` 和 `DEFAULT STORAGE` 对区选项无效。

在创建表空间时，您还应使用 `UNIFORM SIZE`，它告诉 Oracle 使每个区的大小相同（当对象，如表和索引，需要空间时）。

`SEGMENT SPACE MANAGEMENT AUTO` 子句指示 Oracle 管理块内的空间。使用此子句时，无需指定诸如 `PCTUSED`、`FREELISTS` 和 `FREELIST GROUPS` 之类的参数。

如果您创建具有 `AUTOEXTEND` 功能的表空间，我们还建议您指定一个 `MAXSIZE`，这样失控的 SQL 进程就不会无意中填满表空间，进而填满挂载点。以下是一个创建自动扩展表空间并限制其最大大小的示例：

```
create tablespace users
datafile '/ora01/oradata/INVREP/users_01.dbf'
size 100m
autoextend on maxsize 1000m
extent management local
uniform size 128k
segment space management auto;
```

在不同的环境中使用 `CREATE TABLESPACE` 脚本时，能够对脚本的部分进行参数化是很有用的。例如，在开发中，您可能将数据文件大小设为 100MB，而在生产环境中，数据文件可能为 5,000MB。使用 `&` 变量可以使 `CREATE` 脚本更易于移植。下一个列表在脚本顶部定义了 `&` 变量，这些变量决定了为表空间创建的数据文件的大小：

```
define tbsp_large=1000M
define tbsp_med=500M
--
create tablespace exp_data
datafile '/ora01/oradata/INVREP/exp_data01.dbf'
size &&tbsp_large
autoextend on maxsize 1000m
extent management local
uniform size 128k
segment space management auto;
--
create tablespace reg_data
datafile '/ora01/oradata/INVREP/reg_data01.dbf'
size &&tbsp_med
autoextend on maxsize 1000m
extent management local
uniform size 128k
segment space management auto;
```

使用 `&` 变量允许您修改脚本一次，并在整个脚本中重用这些变量。

您还可以从 SQL*Plus 命令行将值传递给 `&` 变量到 `CREATE TABLESPACE` 脚本中。为此，首先在脚本顶部定义 `&` 变量以接受传入的值：

```
define tbsp_large=&1
define tbsp_med=&2
-- 脚本的其余部分在此处...
```

现在您可以从 SQL*Plus 命令行向脚本传递变量。以下示例执行一个名为 `cretbsp.sql` 的脚本并传递两个值，这些值将分别将 `&` 变量设置为 `1000m` 和 `500m`：

```
SQL> @cretbsp 1000m 500m
```

##### 17-5. 删除表空间

**问题**

您想要删除一个表空间。

**解决方案**

如果需要删除表空间，请使用 `DROP TABLESPACE` 语句。我们建议您在删除表空间之前先将其脱机。将表空间脱机可确保没有 SQL 语句正在操作该表空间中的对象。

此示例首先将 `INV_DATA` 表空间脱机，然后使用 `INCLUDING CONTENTS AND DATAFILES` 子句将其删除：

```
alter tablespace inv_data offline;
drop tablespace inv_data including contents and datafiles;
```

**■ 注意** 无论表空间是在线还是脱机，您都可以删除它。`SYSTEM` 表空间是个例外，它无法被删除。

**工作原理**

使用 `INCLUDING CONTENTS AND DATAFILES` 删除表空间将永久删除该表空间及其任何数据文件。在删除表空间之前，请确保该表空间不包含您想要保留的任何数据。

如果要删除的表空间包含一个具有主键的表，而该主键被另一个不同表空间中的表的外键约束引用，您将无法删除该表空间。您可以选择删除约束，或者使用 `DROP TABLESPACE` 语句的 `CASCADE CONSTRAINTS` 子句：

```
drop tablespace inv_data including contents and datafiles cascade constraints;
```

此语句将删除被删除表空间之外的、引用被删除表空间内表的所有参照完整性约束。

##### 17-6. 调整表空间大小

**问题**

当尝试向表中插入数据时，出现此错误：`ORA-01653: unable to extend table INVENTORY by 128 in tablespace INV_IDX`

您确定需要向与该表关联的表空间添加更多空间。

**解决方案**

调整表空间大小有两种常用技术：
* 增加（或减少）现有数据文件的大小
* 添加一个额外的数据文件

在调整数据文件大小之前，您应首先使用类似于以下的查询验证其当前位置和大小：

```
select name, bytes from v$datafile;
```

确定要调整大小的数据文件后，使用 `ALTER DATABASE DATAFILE ... RESIZE` 命令增加其大小。此示例将数据文件大小调整为 1GB：

```
alter database datafile '/ora01/oradata/INVREP/reg_data01.dbf' resize 1g;
```

要向现有表空间添加数据文件，请使用 `ALTER TABLESPACE ... ADD DATAFILE` 语句：

```
alter tablespace reg_data add datafile '/ora01/oradata/INVREP/reg_data02.dbf' size 100m;
```

**工作原理**

在管理具有高事务负载的数据库时，调整数据文件大小可能是一项日常任务。增加现有数据文件的大小允许您在不添加更多数据文件的情况下向表空间添加空间。如果包含现有数据文件的存储设备上没有足够的磁盘空间，您可以将一个数据文件添加到现有表空间的不同位置。

您还可以修改现有数据文件以打开或关闭 `AUTOEXTEND`。下一个示例修改一个数据文件以启用 `AUTOEXTEND`，最大大小为 `1000MB`：

```
alter database datafile '/ora01/oradata/INVREP/reg_data02.dbf'
autoextend on maxsize 1000m;
```

许多 DBA 不愿意启用 `AUTOEXTEND`，因为他们担心如果错误的 SQL `INSERT` 语句执行多次，存储设备可能会被填满。DBA 会选择实现监控脚本，在表空间即将变满时提醒他们，而不是启用 `AUTOEXTEND`。

如果您想向临时表空间添加空间，首先查询 `V$TEMPFILE` 视图以验证临时数据文件的当前位置和大小：

```
select name, bytes from v$tempfile;
```

接下来使用 `ALTER DATABASE` 语句的 `TEMPFILE` 选项：

```
alter database tempfile '/ora01/oradata/INVREP/temp01.dbf' resize 500m;
```

您也可以通过 `ALTER TABLESPACE` 语句向临时表空间添加文件：

```
alter tablespace temp add tempfile '/ora01/oradata/INVREP/temp02.dbf' size 5000m;
```

##### 17-7. 限制每个会话的数据库资源

**问题**

您希望限制用户在您的数据库中可以消耗的资源量。

**解决方案**

首先，您需要确保数据库的 `RESOURCE_LIMIT` 初始化参数设置为 `TRUE`。您可以在初始化文件（`init.ora` 或 `spfile`）中完成此操作，也可以使用 `ALTER SYSTEM` 命令。



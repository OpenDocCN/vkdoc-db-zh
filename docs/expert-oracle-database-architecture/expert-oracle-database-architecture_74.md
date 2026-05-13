# 大型对象的读一致性

在前几章中，我们讨论了读一致性、多版本机制以及撤销在其中所扮演的角色。那么，当涉及到大型对象时，读一致性的实现方式就发生了变化。`LOBSEGMENT` 不使用撤销来记录其更改；相反，它直接在 `LOBSEGMENT` 本身中对信息进行版本控制。`LOBINDEX` 会像任何其他段一样生成撤销，但 `LOBSEGMENT` 则不会。取而代之的是，当你修改一个大型对象时，Oracle 会分配一个新的 `CHUNK`，并让旧的 `CHUNK` 保留在原处。如果你回滚事务，对大型对象索引的更改也会被回滚，索引将再次指向旧的 `CHUNK`。因此，撤销维护操作是在 `LOBSEGMENT` 自身内部完成的。当你修改数据时，旧数据被保留在原地，而新数据被创建出来。

这一点在读取大型对象数据时也发挥作用。大型对象具有一致性，就像所有其他段一样。如果你在上午 9:00 获取了一个大型对象定位器，那么你从中检索到的大型对象数据将是“截至上午 9:00 的状态”。就像如果你在上午 9:00 打开一个游标（结果集），它产生的行也将是该时间点的状态一样。即使其他人修改了大型对象数据并提交（或未提交），你的大型对象定位器也将是“截至上午 9:00 的状态”，就像你的结果集一样。在这里，Oracle 结合使用 `LOBSEGMENT` 和 `LOBINDEX` 的读一致性视图来撤销对大型对象的更改，以便向你呈现获取大型对象定位器时该数据存在的状态。它不会使用 `LOBSEGMENT` 的撤销信息，因为 `LOBSEGMENT` 本身并未生成这些信息。

我们可以很容易地看出大型对象具有一致性。考虑下面这个包含一个非行内存储的大型对象（它存储在 `LOBSEGMENT` 中）的小表：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t
( id int   primary key,
txt      clob )
lob( txt) store as ( disable storage in row );
Table created.
SQL> insert into t values ( 1, 'hello world' );
1 row created.
SQL> commit;
Commit complete.
```

如果我们获取大型对象定位器并按如下方式在表上打开一个游标：

```
SQL> declare
l_clob  clob;
cursor c is select id from t;
l_id    number;
begin
select txt into l_clob from t;
open c;
```

然后我们修改该行并提交

```
update t set id = 2, txt = 'Goodbye';
commit;
```

在操作大型对象定位器和已打开的游标时，我们将看到数据是“截至我们获取或打开它们的时间点的状态”：

```
dbms_output.put_line( dbms_lob.substr( l_clob, 100, 1 ) );
fetch c into l_id;
dbms_output.put_line( 'id = ' || l_id );
close c;
end;
/
hello world
id = 1
PL/SQL procedure successfully completed.
```

但数据在数据库中无疑已经被更新/修改了：

```
SQL> select * from t;
ID TXT
---------- ---------------
2 Goodbye
```

游标 `C` 的读一致映像来源于撤销段，而大型对象的读一致映像则来源于 `LOBSEGMENT` 本身。那么，这就给了我们一个担忧的理由：如果撤销段不用于存储大型对象的回滚信息，*并且*大型对象支持读一致性，我们如何防止可怕的 `ORA-01555: 快照过旧` 错误发生呢？同样重要的是，我们如何控制这些旧版本所使用的空间量？这就是 `RETENTION` 以及 `PCTVERSION` 发挥作用的地方。

## BasicFiles 的 RETENTION

`RETENTION` 指示数据库根据数据库的 `UNDO_RETENTION` 设置，在 LOB 段中保留已修改的 LOB 段数据。如果你将 `UNDO_RETENTION` 设置为两天，Oracle 将尝试不重用由修改操作释放的 LOB 段空间。也就是说，如果你删除了所有指向 LOB 的行，Oracle 将尝试在 LOB 段中保留被删除的数据两天，以满足你的 `UNDO_RETENTION` 策略，就像它会尝试在 `UNDO` 表空间中为结构化数据（你的关系型行和列）保留撤销信息两天一样。你需要理解这一点：LOB 段中释放的空间不会被后续的 `INSERT` 或 `UPDATE` 立即重用。这是经常导致诸如“为什么我的 LOB 段不断增长？”这类问题的原因。大规模清理后重新加载信息往往会导致 LOB 段持续增长，因为保留期尚未到期。

或者，BasicFiles LOB 存储子句可以使用 `PCTVERSION`，它控制应分配的（曾被 LOB 使用过且位于 `LOBSEGMENT` 的 HWM 之下的）LOB 空间中，有多少百分比应用于 LOB 数据的版本控制。默认的百分之十对于许多用途来说是足够的，因为很多时候你只是 `INSERT` 和检索 LOB（通常不进行 LOB 的更新；LOB 往往是一次插入，多次检索）。因此，不需要预留太多空间（如果有的话）用于 LOB 版本控制。

但是，如果你的应用程序确实频繁修改 LOB，并且你经常在读取 LOB 的同时，有其他会话正在修改它们，那么默认的百分之十可能太小了。如果你在处理 LOB 时遇到 `ORA-22924` 错误，解决方案不是增加撤销表空间的大小，或增加撤销保留时间，或者如果你使用的是手动撤销管理，则添加更多的回滚段。相反，你应该使用以下语句：

```
ALTER TABLE tabname MODIFY LOB (lobname) ( PCTVERSION n );
```

并增加在该 `LOBSEGMENT` 中用于数据版本控制的空间量。

## SecureFiles 的 RETENTION

SecureFiles 使用 `RETENTION` 来控制读一致性（与 BasicFiles 类似）。在 SecureFiles LOB 的 `DBMS_METADATA` 的 `CREATE TABLE` 输出中，没有 `RETENTION` 子句。这是因为默认的 `RETENTION` 被设置为 `AUTO`，它指示 Oracle 保留撤销足够长的时间以用于读一致性目的。

如果你想更改默认的 `RETENTION` 行为，可以通过以下参数进行调整：

*   使用 `MAX` 表示撤销应保留，直到 LOB 段达到存储子句中指定的 `MAXSIZE`（因此，`MAX` 必须与存储子句中的 `MAXSIZE` 子句结合使用）。
*   如果启用了闪回数据库，设置 `MIN N` 以将 LOB 的撤销持续时间限制为 `N` 秒。
*   如果一致读或闪回操作不需要撤销，则设置 `NONE`。

如果你没有为 SecureFiles 设置 `RETENTION` 参数，或者指定了 `RETENTION` 但不带任何参数，那么它将被设置为 `DEFAULT`（这等同于 `AUTO`）。


#### CACHE 子句

当 `CREATE TABLE` 语句从 `DBMS_METADATA` 返回时，之前对于 SecureFiles 和 BasicFiles 都会包含以下内容：

```sql
LOB ("TXT") STORE AS (...   NOCACHE ... )
```

`NOCACHE` 的替代选项是 `CACHE` 或 `CACHE READS`。此子句控制 LOBSEGMENT 数据是否存储在缓冲区缓存中。默认的 `NOCACHE` 意味着每次访问都将从磁盘直接读取，每次写入/修改也将从磁盘直接读取。`CACHE READS` 允许从磁盘读取的 LOB 数据被缓冲，但 LOB 数据的写入将直接写入磁盘。`CACHE` 则允许在读取和写入时都缓存 LOB 数据。

在许多情况下，默认设置可能并非你所期望的。如果你有中小型 LOB（例如，仅用于存储几千字节的描述性字段），缓存它们是非常有意义的。如果不缓存，当用户更新描述字段时，用户还必须等待将数据写入磁盘的 I/O 操作（将执行大小为 `CHUNK` 的 I/O，用户将等待此 I/O 完成）。如果你正在执行大量 LOB 的加载，你将不得不在加载每一行时等待 I/O 完成。为这些 LOB 启用缓存是合理的。你可以轻松地开启和关闭缓存：

```sql
ALTER TABLE tabname MODIFY LOB (lobname) ( CACHE );
ALTER TABLE tabname MODIFY LOB (lobname) ( NOCACHE );
```

以查看这可能对你的系统产生的影响。对于大型初始加载，启用 LOB 的缓存并让 `DBWR` 在后台将 LOB 数据写出到磁盘，同时你的客户端应用程序继续加载更多数据，是有意义的。对于频繁访问或修改的中小型 LOB，缓存是有意义的，这样最终用户就不必实时等待物理 I/O 完成。然而，对于一个 50MB 大小的 LOB，将其放在缓存中可能没有意义。

> 提示
>
> 请记住，你可以在此处充分利用 `keep` 或 `recycle` 池（在第 4 章讨论）。你可以使用 `keep` 或 `recycle` 池将 LOBSEGMENT 数据与所有常规数据分开，而不是将其缓存在默认缓存中。通过这种方式，你可以在不影响系统中现有数据缓存的情况下，实现缓存 LOB 数据的目标。

#### LOB STORAGE 子句

最后，当 `CREATE TABLE` 语句从 `DBMS_METADATA` 返回时，之前对于 SecureFiles 会包含以下内容：

```sql
LOB ("TXT") STORE AS SECUREFILE (...
STORAGE(INITIAL 106496 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
PCTINCREASE 0
BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT))
```

以下是 BasicFiles 对应的输出：

```sql
LOB ("TXT") STORE AS BASICFILE ( ...
STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT))
```

SecureFiles 和 BasicFiles 都有一个完整的存储子句，你可以用来控制物理存储特性。应该注意的是，此存储子句同时应用于 LOBSEGMENT 和 LOBINDEX——对一个的设置也用于另一个。

SecureFiles 的存储管理比 BasicFiles 简单。回想一下，SecureFiles LOB 必须在 ASSM 管理的表空间中创建，因此以下属性不再适用：`FREELISTS`、`FREELIST GROUPS` 和 `FREEPOOLS`。

对于 BasicFiles LOB，相关的 LOB 设置是 `FREELISTS` 和 `FREELIST GROUPS`（当不使用 ASSM 时，如第 10 章所讨论）。同样的规则适用于 LOBINDEX 段，因为 LOBINDEX 的管理方式与任何其他索引段相同。如果你有高度并发的 LOB 修改操作，建议在索引段上使用多个 `FREELISTS`。

如前一节所述，对 LOB 段使用 `keep` 或 `recycle` 池可能是一种有用的技术，它允许你缓存 LOB 数据，而不会损害你现有的默认缓冲区缓存。与其让 LOB 从普通表中淘汰块缓冲区，你可以在 SGA 中为这些对象预留一块专用的内存。`BUFFER_POOL` 子句可用于实现此目的。

### BFILEs

最后要讨论的 LOB 类型是 `BFILE` 类型。`BFILE` 类型简单来说就是一个指向操作系统中文件的指针。它用于提供对这些操作系统的只读访问。

> 注意
>
> 内置包 `UTL_FILE` 也提供了对操作系统文件的读写访问。然而，它不使用 `BFILE` 类型。

当你使用 `BFILE` 时，你还将使用 Oracle 的 `DIRECTORY` 对象。`DIRECTORY` 对象只是将操作系统目录映射到数据库中的字符串或名称（提供了可移植性；你在 `BFILE` 中引用的是一个字符串，而不是操作系统特定的文件命名约定）。因此，举个简单的例子，让我们创建一个带有 `BFILE` 列的表，创建一个 `DIRECTORY` 对象，并插入一行引用文件系统中的文件：

```sql
$ sqlplus eoda/foo@PDB1
SQL> create table t
( id       int primary key,
os_file  bfile);
Table created.
SQL> create or replace directory my_dir as '/tmp/';
Directory created.
SQL> insert into t values ( 1, bfilename( 'MY_DIR', 'test.dmp' ) );
1 row created.
```

在这个例子中，我将使用 UNIX/Linux 的 `dd` 命令在 `/tmp` 目录中创建一个 test.dmp 文件：

```bash
dd if=/dev/zero of=/tmp/test.dmp bs=1056768 count=1
```

现在，`BFILE` 可以像 LOB 一样对待——因为它就是。例如：

```sql
SQL> select dbms_lob.getlength(os_file) from t;
DBMS_LOB.GETLENGTH(OS_FILE)
---------------------------
                   1056768
```

我们可以看到指向的文件是 1MB 大小。注意，在 `INSERT` 语句中使用 `MY_DIR` 是故意的。如果我们使用混合大小写或小写，我们将得到以下结果：

```sql
SQL> update t set os_file = bfilename( 'my_dir', 'test.dmp' );
1 row updated.
SQL> select dbms_lob.getlength(os_file) from t;
select dbms_lob.getlength(os_file) from t
*
ERROR at line 1:
ORA-22285: non-existent directory or file for GETLENGTH operation
ORA-06512: at "SYS.DBMS_LOB", line 850
```

这个例子指出，Oracle 中的 `DIRECTORY` 对象是标识符，而标识符默认以大写形式存储。`BFILENAME` 内置函数接受一个字符串，这个字符串的大小写必须与数据字典中存储的 `DIRECTORY` 对象的大小写完全匹配。因此，我们必须在 `BFILENAME` 函数中使用大写，或者在创建 `DIRECTORY` 对象时使用带引号的标识符：

```sql
SQL> create or replace directory "my_dir" as '/tmp/';
Directory created.
SQL> select dbms_lob.getlength(os_file) from t;
DBMS_LOB.GETLENGTH(OS_FILE)
---------------------------
                   1056768
```

我建议不要使用带引号的标识符；而是在 `BFILENAME` 调用中使用大写名称。带引号的标识符不常见，并且往往会为后续操作带来混淆。

`BFILE`（数据库中的指针对象，而不是磁盘上的实际二进制文件）在磁盘上占用的空间量是变化的，这取决于 `DIRECTORY` 对象名称和文件名的长度。在前面的例子中，生成的 `BFILE` 长度约为 35 字节。通常，你会发现 `BFILE` 消耗大约 20 字节的开销，加上 `DIRECTORY` 对象名称的长度，再加上文件名本身的长度。

> 注意
>
> `BFILE` 数据不像其他 LOB 数据那样具有读一致性。由于 `BFILE` 是在数据库外部管理的，当你解引用 `BFILE` 时，文件中恰好是什么内容，你就会得到什么。因此，对同一个 `BFILE` 的重复读取可能会产生不同的结果——这与用于 `CLOB`、`BLOB` 或 `NCLOB` 的 LOB 定位符不同。


## ROWID/UROWID 类型

最后要讨论的数据类型是 `ROWID` 和 `UROWID` 类型。`ROWID` 是表中一行的地址（请回想第 10 章，唯一标识数据库中的一行需要一个 `ROWID` 和一个表名）。`ROWID` 中编码了足够的信息来定位磁盘上的行，并识别 `ROWID` 所指向的对象（表等）。`ROWID` 的近亲 `UROWID` 是一个通用 `ROWID`，用于诸如索引组织表（IOT）以及通过网关访问的异构数据库中的表等没有固定 `ROWID` 的表。`UROWID` 表示行的主键值，因此其大小会根据其指向的对象而变化。

每个表中的每一行都关联一个 `ROWID` 或 `UROWID`。当从表中检索时，它们被视为*伪列*，这意味着它们实际上并不与行一起存储，而是行的派生属性。`ROWID` 是根据行的物理位置生成的；它并不与行一起存储。`UROWID` 是根据行的主键生成的，所以在某种意义上它是随行存储的，但又不完全是，因为 `UROWID` 并不是作为一个离散的列存在，而是作为现有列的一个函数。

过去，对于拥有 `ROWID` 的行（Oracle 中最常见的行类型；除了 IOT 中的行，所有行都有 `ROWID`），`ROWID` 是不可变的。当一行被插入时，它会与一个 `ROWID`（一个地址）相关联，并且该 `ROWID` 将一直与该行关联，直到它被删除，即从数据库中物理移除。随着时间的推移，这种情况变得不那么真实了，因为现在有些操作可能导致行的 `ROWID` 发生变化，例如：

*   更新分区表中行的分区键，导致该行必须从一个分区移动到另一个分区
*   使用 `FLASHBACK` 表命令将数据库表恢复到之前的某个时间点
*   `MOVE` 操作以及许多分区操作，如拆分或合并分区
*   使用 `ALTER TABLE SHRINK SPACE` 命令执行段收缩

现在，由于 `ROWID` 会随时间变化（因为它们不再是不可变的），因此不建议将它们作为列物理存储在数据库表中。也就是说，使用 `ROWID` 作为数据库列的数据类型被认为是糟糕的做法，应该避免。应改为使用行的主键（应该是不可变的），并且可以实施参照完整性来确保数据完整性得以保留。你无法对 `ROWID` 类型执行此操作——你不能通过 `ROWID` 从子表到父表创建外键，也无法像那样跨表强制完整性。你必须使用主键约束。

那么，`ROWID` 类型有什么用呢？它在允许最终用户与数据交互的应用程序中仍然有用——`ROWID` 作为行的物理地址，是访问任何表中单行的最快方式。从数据库读取数据并呈现给最终用户的应用程序可以在尝试更新该行时使用 `ROWID`。应用程序必须将 `ROWID` 与其他字段或校验和结合使用（有关应用程序锁定的更多信息，请参阅第 7 章）。通过这种方式，你可以以最少的工作量（例如，无需再次查找索引来定位该行）更新目标行，并通过验证列值是否未更改来确保该行就是你最初读取的行。因此，在采用乐观锁定的应用程序中，`ROWID` 很有用。

## JSON 类型

Oracle 21c 新增了 `JSON` 数据类型。此数据类型允许你以二进制格式在数据库中原生存储 `JSON` 数据。它是一种优化的二进制存储，使用 `OSON` 格式。`OSON` 格式是 Oracle 优化的二进制 `JSON` 格式。使用此格式，查询性能得到极大提升，因为不再需要解析文本 `JSON` 数据。`JSON` 数据类型可以为表、物化视图和 PL/SQL 参数声明。

以下是如何创建带有 `JSON` 数据类型列的表示例：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t (
id         number generated always as identity,
json_data  json,
constraint t_pk primary key (id));
SQL> desc t;
Name                                      Null?    Type
----------------------------------------- -------- -----------------------
ID                                        NOT NULL NUMBER
JSON_DATA                                          UNDEFINED
```

`VARCHAR2`、`CLOB` 和 `BLOB` 数据类型支持 JSON 数据。为了说明这一点，我将使用一些 PL/SQL 代码，该代码定义了三个变量：一个 `VARCHAR2`、一个 `CLOB` 和一个 `BLOB`。这些变量将被填充值并用于插入到 `JSON` 数据类型中。例如：

```
declare
l_varchar2  varchar2(32767);
l_clob      clob;
l_blob      blob;
begin
l_varchar2 := '{"car":"tesla","quantity":100}';
l_clob     := '{"car":"chevy","quantity":2}';
l_blob     := utl_raw.cast_to_raw('{"car":"volvo","quantity":10}');
--
insert into t (json_data) values (json(l_varchar2));
insert into t (json_data) values (json(l_clob));
insert into t (json_data) values (json(l_blob));
end;
/
```

你可以以与访问数据库中其他数据相同的方式检索 `JSON` 数据（包括 OCI 和 JDBC）。`JSON` 数据以二进制格式存储，因此如果不使用 `JSON` 函数来选择数据，将显示原始数据：

```
SQL> select * from t;
ID JSON_DATA
---------- ----------------------------------------------------------------
1 7B22636172223A227465736C61222C227175616E74697479223A3130307D
2 7B22636172223A226368657679222C227175616E74697479223A327D
3 7B22636172223A22766F6C766F222C227175616E74697479223A31307D
```

接下来，我将使用 `JSON_SERIALIZE` 函数将 `JSON` 数据转换为文本格式：

```
SQL> select id, json_serialize(json_data) as json_data from t;
ID JSON_DATA
---------- ----------------------------------------------------------------
1 {"car":"tesla","quantity":100}
2 {"car":"chevy","quantity":2}
3 {"car":"volvo","quantity":10}
```

其他 JSON 函数可用于检索和显示文本数据。此示例使用 `JSON_VALUE` 函数：

```
SQL> select t.id,
json_value(t.json_data, '$.car') car,
json_value(t.json_data, '$.quantity' returning number) quantity
from t;
ID CAR               QUANTITY
---------- --------------- ----------
1 tesla                  100
2 chevy                    2
3 volvo                   10
```

通过这种方式，你可以在数据库中以其原生二进制格式存储和检索 `JSON` 数据。当处理 `JSON` 数据时，这允许改进查询性能。这是因为数据不是以文本形式存储的，因此不需要被解析。


## 总结

在本章中，我们考察了 Oracle 提供的许多基本数据类型；了解了它们的物理存储方式以及每种类型的可用选项。我们从最基本的字符串类型开始，探讨了围绕多字节字符和原始二进制数据的注意事项。然后我们讨论了扩展数据类型（在 Oracle 12c 及以上版本可用），以及这一特性如何允许你将 `VARCHAR2`、`NVARCHAR2` 和 `RAW` 数据类型的大小定义为最大 32,727 字节。接下来，我们研究了数值类型，包括非常精确的 Oracle `NUMBER` 类型以及 Oracle 提供的浮点类型。

我们还考虑了遗留的 `LONG` 和 `LONG RAW` 类型，重点讨论了如何规避它们的存在，因为这些类型提供的功能远远不及 `LOB` 类型。接着，我们查看了能够存储日期和时间的数据类型。我们涵盖了日期运算的基础知识，这是一个在理解之前令人困惑的问题。最后，在关于日期和时间戳的部分，我们考察了 `INTERVAL` 类型以及如何最佳地使用它。

从物理存储的角度来看，本章最详细的部分是 `LOB` 部分。`LOB` 类型经常被开发人员和 DBA 误解，因此这部分的大部分内容都用于探讨它们的物理实现方式、某些性能考虑因素以及 `SecureFiles` 与 `BasicFiles` 之间的区别。

我们还介绍了 `ROWID`/`UROWID` 类型。出于现在应该显而易见的原因，你不应将此数据类型用作数据库列，因为 `ROWID` 不是不可变的，并且没有完整性约束能够强制执行父/子关系。相反，如果你需要指向另一行，应该存储主键。

我们考察的最后一个数据类型是 `JSON` 数据类型。这个新数据类型在 Oracle 21c 中引入。它允许你在数据库中以原生二进制格式存储 `JSON` 数据。这提高了查询性能，因为检索数据时你无需解析它（如果以文本存储则需要解析）。


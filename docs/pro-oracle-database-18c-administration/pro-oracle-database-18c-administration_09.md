# 公共同义词的使用与管理

共享同一个数据库的多个应用程序，如果都使用在数据库内非唯一的公共同义词，可能会导致对象名称冲突。

安全应根据需要进行管理，而非一刀切。

我通常尽量避免使用公共同义词。然而，有些场景可能值得使用它们。例如，Oracle 创建数据字典时，就使用公共同义词来简化对内部数据库对象的访问管理。要显示数据库中的任何公共同义词，请运行此查询：

```sql
SQL> select owner, synonym_name
from dba_synonyms
where owner='PUBLIC';
```

### 动态生成同义词

有时，为需要私有同义词的某个模式（schema）的所有表或视图动态生成同义词是很有用的。以下脚本使用 `SQL*Plus` 命令来格式化和捕获一个 SQL 脚本的输出，该脚本为模式内的所有表生成同义词：

```sql
SQL> CONNECT &&master_user/&&master_pwd
--
SQL> SET LINESIZE 132 PAGESIZE 0 ECHO OFF FEEDBACK OFF
SQL> SET VERIFY OFF HEAD OFF TERM OFF TRIMSPOOL ON
--
SQL> SPO gen_syns_dyn.sql
--
SQL> select 'create or replace synonym ' || table_name ||
' for ' || '&&master_user..' ||
table_name || ';'
from user_tables;
--
SQL> SPO OFF;
--
SQL> SET ECHO ON FEEDBACK ON VERIFY ON HEAD ON TERM ON;
```

请看 `SELECT` 语句中带有两个点的 `&master_user` 变量：双点语法的目的是什么？在 `&` 变量末尾放置一个点，会指示 `SQL*Plus` 将该点之后的任何内容连接到该 `&` 变量。当你放两个点在一起时，那就告诉 `SQL*Plus` 将一个点连接到 `&` 变量所包含的字符串。

### 显示同义词元数据

`DBA_SYNONYMS`、`ALL_SYNONYMS` 和 `USER_SYNONYMS` 视图包含数据库中同义词的信息。使用以下 SQL 查看当前连接用户的同义词元数据：

```sql
SQL> select synonym_name, table_owner, table_name, db_link
from user_synonyms
order by 1;
```

`ALL_SYNONYMS` 视图显示所有私有同义词、所有公共同义词，以及不同用户拥有的、当前连接用户对其基础表具有 SELECT 访问权限的任何私有同义词。你可以通过查询 `DBA_SYNONYMS` 视图来显示数据库中所有私有和公共同义词的信息。

`DBA_SYNONYMS`、`ALL_SYNONYMS` 和 `USER_SYNONYMS` 视图中的 `TABLE_NAME` 列名有点误导，因为 `TABLE_NAME` 可以引用多种类型的数据库对象，如同义词、视图、包、函数、过程或物化视图。类似地，`TABLE_OWNER` 指的是对象的所有者（而该对象不一定是一张表）。

当你诊断数据完整性问题时，有时首先要确定正在访问的是哪张表或哪个对象。你可能从看似是表的东西进行选择，但实际上，它可能是一个指向视图的同义词，而该视图又从另一个同义词进行选择，后者最终指向另一个数据库中的一张表。

以下查询通常是确定一个对象是同义词、视图还是表的起点：

```sql
SQL> select owner, object_name, object_type, status
from dba_objects
where object_name like upper('&object_name%');
```

请注意，在此查询中使用通配符百分号 (`%`) 允许你输入对象名称的一部分。因此，该查询有可能返回与你输入的文本字符串部分匹配的任何对象的信息。

你也可以使用 `DBMS_METADATA` 包的 `GET_DDL()` 函数来显示同义词元数据。如果你想为当前连接用户显示所有同义词的 DDL，请运行此 SQL：

```sql
SQL> set long 5000
SQL> select dbms_metadata.get_ddl('SYNONYM', synonym_name) from user_synonyms;
```

你也可以显示特定用户的 DDL。你必须为 `GET_DDL()` 函数提供对象类型、对象名称和模式作为输入。



## 重命名同义词

您可能希望重命名一个同义词，以便使其符合命名规范，或者确定它是否正在被使用。使用 `RENAME` 语句来更改同义词的名称：

```sql
SQL> rename inv_s to inv_st;
```

请注意，输出会显示此消息：

```
Table renamed.
```

先前的提示有些误导性。它指示一个表已被重命名，但在该场景下，实际被重命名的是一个同义词。

## 删除同义词

如果您确定不再需要某个同义词，则可以将其删除。未使用的同义词可能会对需要增强或调试现有应用程序的其他人员造成困惑。使用 `DROP SYNONYM` 语句来删除私有同义词：

```sql
SQL> drop synonym inv;
```

如果它是一个公共同义词，则在删除时需要指定 `PUBLIC`：

```sql
SQL> drop public synonym inv_pub;
```

如果成功，您应该会看到此消息：

```
Synonym dropped.
```

## 管理序列

序列是一种数据库对象，用户可以访问它以选择唯一的整数。序列通常用于生成整数来填充主键和外键列。您可以通过 `SELECT`、`INSERT` 或 `UPDATE` 语句访问序列来使其递增。Oracle 保证在选择时序列号是唯一的；没有任何两个用户会话可以选择相同的序列号。无法保证序列生成的数字中不会偶尔出现间隔。通常，一定数量的序列值会缓存在内存中，如果发生实例故障（断电、异常关闭），内存中任何未使用的值都会丢失。即使您不缓存序列，也无法阻止用户在事务中获取一个序列值然后回滚该事务（事务被回滚，但序列值不会回滚）。对于大多数应用程序来说，拥有一个基本无间隔的唯一整数生成器是可以接受的。只需意识到间隔是可能存在的。

### 创建序列

对于许多应用程序，创建一个序列可以像这样简单：

```sql
SQL> create sequence inv_seq;
```

默认情况下，起始数字是 1，增量是 1，内存中缓存的默认序列数量是 20，最大值是 10²⁷。您可以通过此查询验证默认值：

```sql
SQL> select sequence_name, min_value, increment_by, cache_size, max_value
from user_sequences;
```

在创建序列时，您有很大的自由度来改变其各个方面。例如，以下命令创建一个起始值为 1,000、最大值为 1,000,000 的序列：

```sql
SQL> create sequence inv_seq2 start with 10000 maxvalue 1000000;
```

表 9-1 列出了创建序列时可用的各种选项。

**表 9-1 序列创建选项**

| 选项 | 描述 |
| --- | --- |
| `INCREMENT BY` | 指定序列号之间的间隔。 |
| `START WITH` | 指定生成的第一个序列号。 |
| `MAXVALUE` | 指定序列的最大值。 |
| `NOMAXVALUE` | 将序列的最大值设置为一个非常大的数字 (10²⁸ -1)。 |
| `MINVALUE` | 指定序列的最小值。 |
| `NOMINVALUE` | 对于升序序列，将最小值设置为 1；对于降序序列，将最小值设置为 –(10²⁸–1)。 |
| `CYCLE` | 指定当序列达到最大值或最小值时，应开始从最小值（对于升序序列）或最大值（对于降序序列）重新生成数字。 |
| `NOCYCLE` | 指示序列在达到最大值或最小值后停止生成数字。 |
| `CACHE` | 指定要预分配并保留在内存中的序列号数量。如果未指定 `CACHE` 和 `NOCACHE`，默认为 `CACHE 20`。 |
| `NOCACHE` | 指定序列号不应被缓存。 |
| `ORDER` | 保证数字按请求顺序生成。 |
| `NOORDER` | 如果不需要保证序列号按请求顺序生成，则使用此选项。通常这是可接受的，并且是默认设置。 |
| `SCALE` | 通过使用数字偏移量（可消除所有重复项）来启用序列可伸缩性。建议在使用 `SCALE` 时不要使用 `ORDER`。 |
| `EXTEND` | 默认值为 6，是可伸缩偏移量的值。 |
| `NOEXTEND` | `SCALE` 子句的默认设置。序列的宽度只能与序列中数字的最大位数相同。 |
| `NOSCALE` | 禁用序列可伸缩性。 |

### 使用序列伪列

创建序列后，您可以使用两个伪列来访问序列的值：

*   `NEXTVAL`
*   `CURRVAL`

您可以在任何 `SELECT`、`INSERT` 或 `UPDATE` 语句中引用这些伪列。要从 `INV_SEQ` 序列中检出一个值，请访问 `NEXTVAL` 值，如下所示：

```sql
SQL> select inv_seq.nextval from dual;
```

现在，此会话已检索到一个序列号，您可以通过多次访问 `CURRVAL` 值来重复使用它：

```sql
SQL> select inv_seq.currval from dual;
```

以下示例使用一个序列来填充父表的主键值，然后使用同一个序列来填充子表中相应的外键值。可以直接在 `INSERT` 语句中访问序列。首次访问序列时，请使用 `NEXTVAL` 伪列。

```sql
SQL> insert into inv(inv_id, inv_desc) values (inv_seq.nextval, 'Book');
```

如果您想重用同一个序列值，可以通过 `CURRVAL` 伪列来引用它。接下来，向子表中插入一条记录，该记录的外键列使用与其父表主键值相同的值：

```sql
SQL> insert into inv_lines
(inv_line_id,inv_id,inv_item_desc)
values
(1, inv_seq.currval, 'Tome1');
--
SQL> insert into inv_lines
(inv_line_id,inv_id,inv_item_desc)
values
(2, inv_seq.currval, 'Tome2');
```

### 自动递增列

**提示：** 从 Oracle Database 12c 开始，您可以创建具有标识列的表，这些列会自动填充序列值。详情请参见第 7 章。

如果您无法使用标识列，则可以通过使用触发器来模拟这种自动递增功能。例如，假设您创建了一个表和一个序列，如下所示：

```sql
SQL> create table inv(inv_id number, inv_desc varchar2(30));
SQL> create sequence inv_seq;
```

接下来，在 `INV` 表上创建一个触发器，该触发器会自动从序列中为 `INV_ID` 列填充值：

```sql
SQL> create or replace trigger inv_bu_tr
before insert on inv
for each row
begin
select inv_seq.nextval into :new.inv_id from dual;
end;
/
```

现在，向 `INV` 表中插入几条记录：

```sql
SQL> insert into inv (inv_desc) values( 'Book');
SQL> insert into inv (inv_desc) values( 'Pen');
```

从表中查询以验证 `INV_ID` 列确实是由序列自动填充的：

```sql
SQL> select * from inv;
INV_ID INV_DESC
---------- ------------------------------
1 Book
2 Pen
```

我通常不喜欢使用这种技术。是的，它使开发人员的工作更容易，因为他们不必担心填充键列。然而，DBA 需要生成更多代码来维护那些需要自动填充的列。因为我就是 DBA，并且我喜欢保持我所维护的数据库代码尽可能简单，所以我通常会告诉开发人员，我们不使用这种自动递增列的方法，而是使用在 DML 语句中直接调用序列的技术（如前一节“使用序列伪列”所示）。


## 可伸缩序列

一项新功能是可伸缩序列，它允许为序列添加唯一的数字偏移量前缀。与第 8 章讨论的反向键索引类似，这有助于避免热点问题，并确保每个服务器使用相同的偏移量，使数字集中在一起。

可伸缩序列使用 `SCALE`、`EXTEND`、`NOEXTEND` 和 `NOSCALE` 选项。

```
SQL> create sequence inv_seq scale;
```

关于 `scale` 选项的信息存储在序列字典列 `SCALE_FLAG` 和 `EXTEND_FLAG` 中。

```
SQL> select sequence_name, scale_flag, extend_flag from dba_sequences
where sequence_name='INV_SEQ';
SEQUENCE_NAME                  SCALE_FLAG   EXTEND_FLAG

INV_SEQ                        Y            N
SQL> select inv_seq.nextval from dual;
NEXTVAL
```

前三个数字值基于实例，接下来的三个基于会话，最后一个数字是序列的下一个值。

### 无间隙序列

人们有时会过分担心确保在向表中插入行时不会丢失任何一个序列值。在少数情况下，我见过应用程序因为序列值中的间隙而失败。我对这些问题有两个想法：

*   如果你担心间隙，那么你就没有正确思考你正在解决的问题。
*   如果你的应用程序因为间隙而失败，那么你的做法是错误的。

我的话很直白，我知道，但几乎没有（如果有的话）应用程序需要无间隙序列。如果你真的需要无间隙序列，那么使用 Oracle 序列对象就是错误的方法。你必须自己实现一个序列生成器。你将不得不经历痛苦的折腾来确保不存在间隙。这些折腾会损害你代码的性能。而且，最终你可能会失败。

### 实现生成唯一值的多个序列

我曾遇到一个开发者询问是否可以为应用程序创建多个序列，并保证每个序列生成的数字在所有序列中都是唯一的。如果你有这种类型的需求，可以通过几种不同的方式处理：

*   如果你心情不好，可以告诉开发者这是不可能的，并且标准是每个应用程序使用一个序列（这通常是我采取的方法）。
*   将序列设置为从不同的点开始并以不同的步长递增。
*   使用序列号范围。

如果你心情好，你可以通过指定奇数或偶数的起始数字，然后以 2 递增序列，来设置少量有限数量的序列，使其始终生成唯一值。例如，你可以设置两个奇数和两个偶数序列生成器；例如，

```
SQL> create sequence inv_seq_odd start with 1 increment by 2;
SQL> create sequence inv_seq_even start with 2 increment by 2;
SQL> create sequence inv_seq_odd_dwn start with -1 increment by -2;
SQL> create sequence inv_seq_even_dwn start with -2 increment by -2;
```

这四个序列生成的数字应该*永远不会*相交。然而，这种方法受限于只能使用四个序列。

如果你需要超过四个唯一序列，可以使用数字范围；例如，

```
SQL> create sequence inv_seq_low start with 1 increment by 1 maxvalue 10000000;
SQL> create sequence inv_seq_ml  start with 10000001 increment by 1 maxvalue 20000000;
SQL> create sequence inv_seq_mh  start with 20000001 increment by 1 maxvalue 30000000;
SQL> create sequence inv_seq_high start with 30000001 increment by 1 maxvalue 40000000;
```

使用这种技术，你可以设置许多不同的数字范围供每个序列使用。缺点是每个序列可以生成的唯一值数量是有限的。

### 创建单个序列还是多个序列

假设你有一个包含 20 个表的应用程序。一个常见的问题是，应该使用 20 个不同的序列来填充每个表的主键和外键列，还是只使用 1 个序列。

我建议只使用 1 个序列；1 个序列比多个序列更容易管理，并且意味着需要管理的 DDL 代码更少，出现问题时需要排查的地方也更少。

有时，开发者会提出诸如以下问题：

*   只使用 1 个序列存在性能问题。
*   序列号变得太高。

如果你缓存序列值，通常访问序列不会存在性能问题。序列的最大值是 `10²⁸–1`，所以如果序列每次递增 1，你将永远达不到最大值（至少，在这一生中不会）。

然而，在为父表和子表生成代理键的场景中，使用多个序列有时更方便。在这些情况下，可能有必要为每个应用程序使用多个序列。当你使用这种方法时，必须记住在添加表时添加序列，并在删除表时可能删除序列。这不是什么大事，但这意味着 DBA 需要多一点维护工作，并且开发者必须确保为每个表使用正确的序列。

### 查看序列元数据

如果你有 `DBA` 权限，可以查询 `DBA_SEQUENCES` 视图以显示数据库中所有序列的信息。要查看你模式拥有的序列，请查询 `USER_SEQUENCES` 视图：

```
SQL> select sequence_name, min_value, max_value, increment_by
from user_sequences;
```

要查看重新创建序列所需的 DDL 代码，请访问 `DBMS_METADATA` 视图。如果你使用 SQL*Plus 来执行 `DBMS_METADATA`，首先要确保设置了 `LONG` 变量：

```
SQL> set long 5000
```

此示例提取 `INV_SEQ` 的 DDL：

```
SQL> select dbms_metadata.get_ddl('SEQUENCE','INV_SEQ') from dual;
```

如果你想显示当前连接用户的所有序列的 DDL，请运行此 SQL：

```
SQL> select dbms_metadata.get_ddl('SEQUENCE',sequence_name) from user_sequences;
```

你还可以通过提供 `SCHEMA` 参数来生成特定用户拥有序列的 DDL：

```
SQL> select
dbms_metadata.get_ddl(object_type=>'SEQUENCE', name=>'INV_SEQ', schema=>'INV_APP')
from dual;
```

### 重命名序列

偶尔，你可能需要重命名一个序列。例如，序列可能以错误的名称创建，或者你可能想在从数据库中删除序列之前先重命名它。使用 `RENAME` 语句来完成此操作。此示例将 `INV_SEQ` 重命名为 `INV_SEQ_OLD`：

```
SQL> rename inv_seq to inv_seq_old;
```

你应该会看到以下消息：

```
Table renamed.
```

在这种情况下，即使消息显示“Table renamed”，实际被重命名的是序列。

### 删除序列

通常，你希望删除序列是因为它未被使用，或者你希望用新的起始数字重新创建它。要删除序列，请使用 `DROP SEQUENCE` 语句：

```
SQL> drop sequence inv_seq;
```

当对象被删除时，与该对象相关的所有权限也会被删除。因此，如果你需要重新创建序列，请记住重新授予其他可能需要使用该序列的用户的 `SELECT` 权限。

> **提示**
> 参见下一节“重置序列”，了解删除和重新创建序列的替代方法。

## 重置序列

您可能偶尔需要更改序列的当前值。例如，您可能工作在一个测试环境中，开发人员周期性地希望将数据库重置到之前的状态。一个典型的场景是，开发人员拥有用于截断表并用测试数据重新填充的脚本，并且作为该操作的一部分，希望将序列重置回像 1 这样的值。

Oracle 的文档指出，“要从不同的数字重新启动序列，必须删除并重新创建它。” 这并不完全准确。在大多数情况下，您应该避免删除序列，因为您必须为当前对该序列具有选择权限的用户重新授予对象权限。这可能导致您的应用程序在等待这些权限时出现临时停机。

以下技术演示了如何使用 `ALTER SEQUENCE` 语句将当前值设置为更高或更低的值。基本步骤如下：

1.  将 `INCREMENT BY` 子句更改为一个很大的数字。
2.  从序列中选择一次，以应用这个很大的正数或负数增量。
3.  将 `INCREMENT BY` 设置回其原始值（通常为 1）。

此示例将序列的下一个值设置为比当前值高 1000：

```
SQL> alter sequence myseq increment by 1000;
SQL> select myseq.nextval from dual;
SQL> alter sequence myseq increment by 1;
```

验证序列是否已设置为您期望的值：

```
SQL> select myseq.nextval from dual;
```

您也可以使用此技术将序列设置为比当前值小得多的数字。区别在于 `INCREMENT BY` 设置是一个很大的负数。例如，此示例将序列减少 1000：

```
SQL> alter sequence myseq increment by -1000;
SQL> select myseq.nextval from dual;
SQL> alter sequence myseq increment by 1;
```

验证序列是否已设置为您期望的值：

```
SQL> select myseq.nextval from dual;
```

此外，您可以通过 SQL 脚本自动执行将序列重置回某个值的任务。该技术在接下来的几行 SQL 代码中展示。该代码将提示您输入序列名称和希望序列重置到的值：

```
SQL> UNDEFINE seq_name
SQL> UNDEFINE reset_to
SQL> PROMPT "sequence name" ACCEPT '&&seq_name'
SQL> PROMPT "reset to value" ACCEPT &&reset_to
SQL> COL seq_id NEW_VALUE hold_seq_id
SQL> COL min_id NEW_VALUE hold_min_id
--
SQL> SELECT &&reset_to - &&seq_name..nextval - 1 seq_id
FROM dual;
--
SQL> SELECT &&hold_seq_id - 1 min_id FROM dual;
--
SQL> ALTER SEQUENCE &&seq_name INCREMENT BY &hold_seq_id MINVALUE &hold_min_id;
--
SQL> SELECT &&seq_name..nextval FROM dual;
--
SQL> ALTER SEQUENCE &&seq_name INCREMENT BY 1;
```

为确保序列已设置为您想要的值，请从中选择 `NEXTVAL`：

```
SQL> select &&seq_name..nextval from dual;
```

当您将应用程序在各种开发、测试和生产环境之间迁移时，这种方法非常有用。它允许您重置序列，而无需重新授予对象权限。

## 总结

视图、同义词和序列在 Oracle 数据库应用程序中被广泛使用。这些对象（连同表和索引）为创建复杂的应用程序提供了技术支持。

视图提供了一种创建和存储复杂多表连接查询的方法，然后这些查询可以被数据库用户和应用程序使用。视图可用于更新底层基表，也可以创建为只读以满足报表需求。

同义词（以及适当的权限）提供了一种机制，可以透明地允许用户访问属于另一个模式的对象。访问同义词的用户只需要知道同义词名称，而不管底层对象类型和所有者如何。这使得应用程序设计者可以无缝地将对象所有者与访问对象的用户分开。

序列生成唯一整数，这些整数通常被应用程序用来填充主键和外键列。Oracle 保证当访问序列时，它总是向选择的用户返回一个唯一值。

在安装 Oracle 二进制文件并创建数据库和表空间后，您通常会创建一个应用程序，该应用程序由所有者用户以及相应的表、约束、索引、视图、同义词和序列组成。有关这些对象的元数据存储在内部数据字典中。数据字典被广泛用于监控、故障排除和诊断问题。您必须非常熟练地从数据字典中检索信息。检索和分析数据字典信息是下一章的主题。


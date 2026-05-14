# 11. 大型对象

组织经常处理需要存储和供业务用户查看的大文件。通常，LOB 是一种适合存储大型非结构化数据的数据类型，例如文本、日志、图像、视频、声音和空间数据。Oracle 支持以下类型的 LOB：

*   字符大对象 (`CLOB`)
*   国家字符大对象 (`NCLOB`)
*   二进制大对象 (`BLOB`)
*   二进制文件 (`BFILE`)


## Oracle LOB 数据类型：介绍与选择

在 Oracle 8 之前，`LONG` 和 `LONG RAW` 数据类型是您在列中存储大量数据的唯一选择。您不应再使用这些数据类型。我之所以还提及 `LONG` 和 `LONG RAW`，是因为许多遗留应用程序（例如 Oracle 的数据字典）仍在使用它们。除此之外，您应该使用 `CLOB` 代替 `LONG`，使用 `BLOB` 代替 `LONG RAW`。另外，不要将 `RAW` 数据类型与 `LONG RAW` 混淆。`RAW` 数据类型存储少量二进制数据。而 `LONG RAW` 数据类型已被弃用超过十年。
另一个注意事项：不要不必要地使用 LOB 数据类型。例如，对于字符数据，如果您的应用程序需要少于 32,000 个单字节字符，请使用 `VARCHAR2` 数据类型（而不是 `CLOB`）。对于二进制数据，如果您处理的是少于 32,000 字节的二进制数据，请使用 `RAW` 数据类型（而不是 `BLOB`）。如果您仍不确定您的应用程序需要哪种数据类型，请参阅第 7 章了解 Oracle 数据类型的适当用法描述。

在深入探讨 LOB 的实现细节之前，先回顾每种 LOB 数据类型及其适当用途是审慎的做法。之后，将提供创建和使用 LOB 以及您应理解的相关功能的示例。

### 描述 LOB 类型

从早期版本的 Oracle 开始，通过 `CLOB`、`NCLOB`、`BLOB` 和 `BFILE` 数据类型，在数据库中存储大文件的能力得到了极大提升。这些额外的 LOB 数据类型让您可以存储更多的数据，并具有更强大的功能。表 11-1 总结了可用的 Oracle LOB 类型及其描述。

表 11-1：Oracle 大对象数据类型

| 数据类型 | 描述 | 最大大小 |
| --- | --- | --- |
| `CLOB` | 字符大对象，用于存储字符文档，如大型文本文件、日志文件、XML 文件等 | (4GB–1) * 块大小 |
| `NCLOB` | 国家字符大对象；以国家字符集格式存储数据；支持宽度可变的字符 | (4GB–1) * 块大小 |
| `BLOB` | 二进制大对象，用于存储非结构化位流数据（图像、视频等） | (4GB–1) * 块大小 |
| `BFILE` | 存储在数据库外部文件系统上的二进制文件大对象；只读 | 2⁶⁴–1 字节（操作系统可能强加小于此值的大小限制） |

#### `CLOB` 与 `NCLOB`

`CLOB` 用于存储字符数据，如 JSON、XML、文本和日志文件。`NCLOB` 与 `CLOB` 的处理方式相同，但可以包含数据库多字节国家字符集中的字符。

#### `BLOB`

`BLOB` 不是人可读的。`BLOB` 的典型用途包括电子表格、文字处理文档、图像以及音频和视频数据。

#### 内部 LOB（Internal LOBs）

`CLOB`、`NCLOB` 和 `BLOB` 被称为内部 LOB。这是因为这些数据类型存储在 Oracle 数据库内的数据文件中。内部 LOB 参与事务，并受 Oracle 数据库安全以及其备份和恢复功能的保护。

#### 外部 LOB（External LOBs）与 `BFILE`

`BFILE` 被称为外部 LOB。`BFILE` 列存储指向操作系统上数据库外部文件的指针。您可以将 `BFILE` 视为一种为操作系统文件系统上数据库外部的大型二进制文件提供只读访问的机制。

有时，会出现这样的问题：您应该使用 `BLOB` 还是 `BFILE`？`BLOB` 参与数据库事务，可以由 Oracle 进行备份、恢复和恢复。`BFILE` 不参与数据库事务，是只读的，并且不受任何 Oracle 安全、备份和恢复、复制或灾难恢复机制的保护。`BFILE` 更适用于那些在应用程序运行时是只读且不会改变的大型二进制文件。例如，您可能有大型二进制视频文件被数据库应用程序引用。在这种情况下，业务决定您不需要在应用程序实际只需要一个指向磁盘上大文件位置的指针（存储在数据库中）时，去创建和维护一个 500TB 的数据库。

### 说明 LOB 定位器、索引和块

内部 LOB（`CLOB`、`NCLOB`、`BLOB`）将数据存储在称为**数据块**的片段中。数据块是 LOB 分配的最小单位，由一个或多个数据库块组成。**LOB 定位器**存储在包含 LOB 列的行中。LOB 定位器指向一个 **LOB 索引**。LOB 索引存储有关 LOB 数据块位置的信息。当查询表时，数据库使用 LOB 定位器和相关的 LOB 索引来定位相应的 LOB 数据块。图 11-1 显示了表、行、LOB 定位器以及 LOB 定位器关联的索引和数据块之间的关系。

![`../images/214899_3_En_11_Chapter/214899_3_En_11_Fig1_HTML.png`](img/214899_3_En_11_Fig1_HTML.png)

图 11-1：表、行、LOB 定位器、LOB 索引和 LOB 段的关系

`BFILE` 的 LOB 定位器存储操作系统上的目录路径和文件名。图 11-2 显示了一个引用操作系统文件的 `BFILE` LOB 定位器。

![`../images/214899_3_En_11_Chapter/214899_3_En_11_Fig2_HTML.png`](img/214899_3_En_11_Fig2_HTML.png)

图 11-2：`BFILE` LOB 定位器包含在操作系统上定位文件的信息

> **注意**
> `DBMS_LOB` 包通过 LOB 定位器对 LOB 执行操作。

### 区分 BasicFiles 和 SecureFiles

对 LOB 做出了几项重大改进。Oracle 现在区分两种不同的底层 LOB 架构：

- BasicFiles
- SecureFiles

以下部分将讨论这两种 LOB 架构。

#### BasicFiles

BasicFiles 是 Oracle 给予 Oracle Database 11g 之前可用的 LOB 架构的名称。了解 BasicFiles LOB 仍然很重要，因为许多使用 Oracle 版本的商店不支持 SecureFiles。请注意，在 Oracle Database 11g 中，默认的 LOB 类型仍然是 BasicFiles。然而，在后续版本中，默认的 LOB 类型现在是 SecureFiles，并且应被用作存储 LOB 的方式。

#### SecureFiles

SecureFiles 是推荐使用的 LOB 架构选项。它包含以下增强功能（相对于 BasicFiles LOB）：

- 加密（需要 Oracle Advanced Security 选项）
- 压缩（需要 Oracle Advanced Compression 选项）


## Oracle SecureFiles 详解

### 1. SecureFiles 特性

SecureFiles 加密允许你透明地加密 LOB 数据（就像其他数据类型一样）。压缩功能可以显著节省空间。去重功能（需要 Oracle Advanced Compression 选项）会消除重复的 LOB，否则这些 LOB 会被存储多次。

### 2. 使用前的规划

在使用 SecureFiles 之前，需要做一些简单的规划。具体来说，使用 SecureFiles 需要满足以下条件：

*   SecureFiles LOB 必须存储在使用 **自动段空间管理（ASSM）** 的表空间中。
*   初始化参数 `DB_SECUREFILE` 控制是否可以使用 SecureFiles 文件，并定义了数据库默认的 LOB 架构。

### 3. 配置 ASSM 表空间

SecureFiles LOB 必须在使用 ASSM 的表空间中创建。要创建启用 ASSM 的表空间，需指定 `SEGMENT SPACE MANAGEMENT AUTO` 子句；例如：

```
SQL> create tablespace lob_data
datafile '/u01/dbfile/o18c/lob_data01.dbf'
size 1000m
extent management local
uniform size 1m
segment space management auto;
```

如果你有现有表空间，可以通过查询 `DBA_TABLESPACES` 视图来验证是否使用了 ASSM。对于任何你想用于 SecureFiles 的表空间，`SEGMENT_SPACE_MANAGEMENT` 列的值应为 `AUTO`：

```
select tablespace_name, segment_space_management
from dba_tablespaces;
```

### 4. 配置 DB_SECUREFILE 参数

此外，SecureFiles 的使用受数据库参数 `DB_SECUREFILE` 管控。你可以使用 `ALTER SYSTEM` 或 `ALTER SESSION` 来修改 `DB_SECUREFILE` 的值。表 11-2 描述了 `DB_SECUREFILE` 的有效值。

**表 11-2 `DB_SECUREFILE` 设置描述**

| `DB_SECUREFILE` 设置 | 描述 |
| :--- | :--- |
| `NEVER` | 无论是否指定了 `SECUREFILE` 选项，都将 LOB 创建为 BasicFiles 类型。 |
| `PERMITTED` | 允许创建 SecureFiles LOB。 |
| `PREFERRED` | 默认值；指定所有 LOB 都创建为 SecureFiles 类型，除非另有说明。 |
| `ALWAYS` | 将 LOB 创建为 SecureFiles 类型，除非底层表空间未使用 ASSM。 |
| `IGNORE` | 忽略 SecureFiles 选项以及任何 SecureFiles 设置。 |

### 5. 创建 LOB 列

#### 5.1 创建包含 LOB 列的表

默认的底层 LOB 架构是 SecureFiles。建议将 LOB 创建为 SecureFiles。如前所述，SecureFiles 允许你使用压缩和加密等功能。

#### 5.2 创建 BasicFiles LOB 列

要创建 LOB 列，必须指定 LOB 数据类型。最好显式指定 `STORE AS BASICFILE` 子句，以避免混淆使用的是哪种 LOB 架构。以下是一个示例：

```
SQL> create table patchmain(
patch_id   number
,patch_desc clob)
tablespace users
lob(patch_desc) store as basicfile;
```

### 6. 技术要点

创建包含 LOB 列的表时，必须了解一些技术基础。请回顾以下列表，并确保理解每一点：

*   在 Oracle Database 12c 之前，默认创建的 LOB 是 BasicFiles 类型。
*   Oracle 为每个 LOB 列创建一个 LOB 段和一个 LOB 索引。
*   LOB 段的名称格式为：`SYS_LOB<string>`。
*   LOB 索引的名称格式为：`SYS_IL<string>`。
*   `<string>` 对于每个 LOB 段及其关联的索引是相同的。
*   LOB 段和索引在与表相同的表空间中创建，除非你指定了不同的表空间。
*   LOB 段和 LOB 索引在向表插入记录之前不会创建（即所谓的延迟段创建功能）。这意味着在向表插入行之前，`DBA/ALL/USER_SEGMENTS` 和 `DBA/ALL/USER_EXTENTS` 视图中没有任何信息。

Oracle 为每个 LOB 列创建一个 LOB 段和一个 LOB 索引。LOB 段存储数据。LOB 索引跟踪 LOB 数据块的物理存储位置以及访问它们的顺序。



您可以查询 `DBA/ALL/USER_LOBS` 视图来显示 LOB 段和 LOB 索引名称：

```sql
SQL> select table_name, segment_name, index_name, securefile, in_row
from user_lobs;
```

以下是此示例的输出：

```sql
TABLE_NAME   SEGMENT_NAME              INDEX_NAME                SEC IN_
------------ ------------------------- ------------------------- --- ---
PATCHMAIN    SYS_LOB0000022332C00002$$ SYS_IL0000022332C00002$$  NO  YES
```

您也可以查询 `DBA/USER/ALL_SEGMENTS` 来查看关于 LOB 段的信息。如前所述，直到您向表中插入一行数据后，才会创建初始段（这是延迟段创建的特性）。这可能会令人困惑，因为您可能期望在创建表后立即能在 `DBA/ALL/USER_SEGMENTS` 中看到对应的行：

```sql
SQL> select segment_name, segment_type, segment_subtype, bytes/1024/1024 meg_bytes
from user_segments
where segment_name IN ('&&table_just_created',
'&&lob_segment_just_created',
'&&lob_index_just_created');
```

前面的查询会提示输入段名。输出显示没有行：

```sql
no rows selected
```

接下来，向包含 LOB 列的表中插入一条记录：

```sql
SQL> insert into patchmain values(1,'clob text');
```

重新对 `USER_SEGMENTS` 运行查询，显示已创建三个段——一个用于表，一个用于 LOB 段，一个用于 LOB 索引：

```sql
SEGMENT_NAME              SEGMENT_TYPE       SEGMENT_SU  MEG_BYTES
------------------------- ------------------ ---------- ----------
PATCHMAIN                 TABLE              ASSM            .0625
SYS_IL0000022332C00002$$  LOBINDEX           ASSM            .0625
SYS_LOB0000022332C00002$$ LOBSEGMENT         ASSM            .0625
```

## 在特定表空间中实现 LOB

默认情况下，LOB 段与其所在的表存储在同一个表空间中。您可以使用 `CREATE TABLE` 语句的 `LOB...STORE AS` 子句为 LOB 段指定一个独立的表空间。下一个建表脚本在某个表空间中创建表，并为 `CLOB` 和 `BLOB` 列创建独立的表空间：

```sql
SQL> create table patchmain
(patch_id   number
,patch_desc clob
,patch      blob
) tablespace users
lob (patch_desc) store as (tablespace lob_data)
,lob (patch)      store as (tablespace lob_data);
```

以下查询验证了该表关联的表空间：

```sql
SQL> select table_name, tablespace_name, 'N/A' column_name
from user_tables
where table_name='PATCHMAIN'
union
select table_name, tablespace_name, column_name
from user_lobs
where table_name='PATCHMAIN';
```

输出如下：

```sql
TABLE_NAME           TABLESPACE_NAME      COLUMN_NAME
-------------------- -------------------- --------------------
PATCHMAIN            LOB_DATA             PATCH
PATCHMAIN            LOB_DATA             PATCH_DESC
PATCHMAIN            USERS                N/A
```

如果您认为 LOB 段需要不同的存储特性（例如大小和增长模式），那么我建议您将 LOB 创建在与表数据分离的表空间中。这允许您将 LOB 列的存储与常规表数据存储分开管理。

## 创建 SecureFiles LOB 列

如前所述，默认的 LOB 架构是 SecureFiles。尽管如此，我建议您明确说明要实现哪种 LOB 架构，以避免任何混淆。如前所述，包含 SecureFile LOB 的表空间必须是 ASSM 管理的。以下是一个创建 SecureFiles LOB 的示例：

```sql
SQL> create table patchmain(
patch_id   number
,patch_desc clob)
lob(patch_desc) store as securefile (tablespace lob_data);
```

在查看关于 LOB 列的数据字典详细信息之前，请向表中插入一条记录以确保段信息可用（这是由于 Oracle Database 11g Release 2 及更高版本中的延迟段分配特性）；例如，

```sql
SQL> insert into patchmain values(1,'clob text');
```

您现在可以通过查询 `USER_SEGMENTS` 视图来验证 LOB 的架构：

```sql
SQL> select segment_name, segment_type, segment_subtype
from user_segments;
```

以下是一些示例输出，表明 LOB 段是 SecureFiles 类型：

```sql
SEGMENT_NAME              SEGMENT_TYPE       SEGMENT_SU
------------------------- ------------------ ----------
PATCHMAIN                 TABLE              ASSM
SYS_IL0000022340C00002$$  LOBINDEX           ASSM
SYS_LOB0000022340C00002$$ LOBSEGMENT         SECUREFILE
```

您也可以查询 `USER_LOBS` 视图来验证 SecureFiles LOB 架构：

```sql
SQL> select table_name, segment_name, index_name, securefile, in_row
from user_lobs;
```

输出如下：

```sql
TABLE_NAME   SEGMENT_NAME              INDEX_NAME                SEC IN_
------------ ------------------------- ------------------------- --- ---
PATCHMAIN    SYS_LOB0000022340C00002$$ SYS_IL0000022340C00002$$  YES YES
```

注意：使用 SecureFiles 架构后，您不再需要指定以下选项：`CHUNK`、`PCTVERSION`、`FREEPOOLS`、`FREELIST` 和 `FREELIST GROUPS`。

## 实现分区 LOB

您可以创建带有 LOB 列的分区表。这样做可以让您将 LOB 分布在多个表空间中。这种分区有助于平衡 I/O、维护以及备份和恢复操作。

您可以通过 `RANGE`、`LIST` 或 `HASH` 对 LOB 进行分区。下一个示例创建了一个 `LIST` 分区表，其中 LOB 列数据存储在与表数据分离的表空间中：

```sql
SQL> CREATE TABLE patchmain(
patch_id   NUMBER
,region     VARCHAR2(16)
,patch_desc CLOB)
LOB(patch_desc) STORE AS (TABLESPACE patch1)
PARTITION BY LIST (REGION) (
PARTITION p1 VALUES ('EAST')
LOB(patch_desc) STORE AS SECUREFILE
(TABLESPACE patch1 COMPRESS HIGH)
TABLESPACE inv_data1
,
PARTITION p2 VALUES ('WEST')
LOB(patch_desc) STORE AS SECUREFILE
(TABLESPACE patch2 DEDUPLICATE NOCOMPRESS)
TABLESPACE inv_data2
,
PARTITION p3 VALUES (DEFAULT)
LOB(patch_desc) STORE AS SECUREFILE
(TABLESPACE patch3 COMPRESS LOW)
TABLESPACE inv_data3
);
```

请注意，每个 LOB 分区都是使用其自身的存储选项创建的（有关 SecureFiles 功能的详细信息，请参阅本章后面的“实现 SecureFiles 高级特性”部分）。您可以查看 LOB 分区的详细信息，如下所示：

```sql
SQL> select table_name, column_name, partition_name, tablespace_name
,compression, deduplication
from user_lob_partitions;
```

以下是一些示例输出：

```sql
TABLE_NAME   COLUMN_NAME     PARTITION_ TABLESPACE_NAME COMPRE DEDUPLICATION
------------ --------------- ---------- --------------- ------ ------------
PATCHMAIN    PATCH_DESC      P1         PATCH1          HIGH   NO
PATCHMAIN    PATCH_DESC      P2         PATCH2          NO     LOB
PATCHMAIN    PATCH_DESC      P3         PATCH3          LOW    NO
```

提示：您也可以查询 `DBA/ALL_USER_PART_LOBS` 来获取关于分区 LOB 的信息。

在分区 LOB 列创建后，您可以更改其存储特性。为此，请使用 `ALTER TABLE ... MODIFY PARTITION` 语句。此示例修改 LOB 分区以使用高压缩度：

```sql
SQL> alter table patchmain modify partition p2
lob (patch_desc) (compress high);
```

下一个示例修改分区 LOB 以不保留重复值（通过 `DEDUPLICATE` 子句）：

```sql
SQL> alter table patchmain modify partition p3
lob (patch_desc) (deduplicate lob);
```

注意：本章讨论的分区和高级压缩是额外付费选项，仅在 Oracle 企业版中可用。

## 维护 LOB 列

以下部分描述了一些在 LOB 列上执行或涉及 LOB 列的常见维护任务，包括在表空间之间移动列以及向表中添加新的 LOB 列。


### 移动 LOB 列
如前所述，如果你创建了一个带有 LOB 列的表但没有指定表空间，那么默认情况下，该 LOB 会创建在与其表相同的表空间中。这种情况有时出现在数据库管理员（DBA）没有提前做好规划的环境中；只有在 LOB 列消耗了大量磁盘空间后，DBA 才会疑惑表为什么变得如此之大。

你可以使用 `ALTER TABLE...MOVE...STORE AS` 语句将 LOB 列移动到与表不同的表空间。基本语法如下：
```sql
SQL> alter table <table_name> move lob(<lob_column>) store as (tablespace <new_tablespace>);
```
下一个示例将 LOB 列移动到 `LOB_DATA` 表空间：
```sql
SQL> alter table patchmain
move lob(patch_desc)
store as securefile (tablespace lob_data);
```
你可以通过查询 `USER_LOBS` 来验证 LOB 是否已移动：
```sql
SQL> select table_name, column_name, tablespace_name from user_lobs;
```
总而言之，如果 LOB 列填充了大量数据，你几乎总是希望将 LOB 存储在与表中其余数据不同的表空间中。在这些情况下，LOB 数据有不同的增长和存储需求，最好在其独立的表空间中维护。

### 添加 LOB 列
如果你有一个现有表，想为其添加一个 LOB 列，请使用 `ALTER TABLE...ADD` 语句。以下语句向表中添加了 `INV_IMAGE` 列：
```sql
SQL> alter table patchmain add(inv_image blob);
```
此语句适用于快速向开发环境添加 LOB 列。对于其他任何情况，你都应指定存储特性。例如，以下命令指定在 `LOB_DATA` 表空间中创建一个 SecureFiles LOB：
```sql
SQL> alter table patchmain add(inv_image blob)
lob(inv_image) store as securefile(tablespace lob_data);
```
### 删除 LOB 列
你可能会遇到业务需求变化、不再需要某个列的情况。在删除列之前，考虑先重命名它，以便更好地识别是否有任何应用程序或用户仍在访问它：
```sql
SQL> alter table patchmain rename column patch_desc to patch_desc_old;
```
在确定没有人使用该列后，使用 `ALTER TABLE...DROP` 语句将其删除：
```sql
SQL> alter table patchmain drop(patch_desc_old);
```
你也可以通过删除并重新创建表（不包含 LOB 列）来移除 LOB 列。当然，这也会永久删除所有数据。

另外请记住，如果你的回收站已启用，并且删除表时没有使用 `PURGE` 子句，那么已删除的表仍然会占用空间。如果你想释放与表关联的空间，请使用 `PURGE` 子句，或在删除表后清空回收站。

### 缓存 LOB
默认情况下，在读写 LOB 列时，Oracle 不会将 LOB 缓存在内存中。你可以通过设置与缓存相关的存储选项来更改默认行为。此示例指定 Oracle 应将 LOB 列缓存在内存中：
```sql
SQL> create table patchmain(
patch_id number
,patch_desc clob)
lob(patch_desc) store as (tablespace lob_data cache);
```
你可以使用此查询验证 LOB 缓存设置：
```sql
SQL> select table_name, column_name, cache from user_lobs;
```
以下是一些示例输出：
```
TABLE_NAME           COLUMN_NAME          CACHE
-------------------- -------------------- ----------
PATCHMAIN            PATCH_DESC           YES
```
表 11-3 描述了与 LOB 相关的内存缓存设置。如果你的 LOB 频繁读写，请考虑使用 `CACHE` 选项。如果你的 LOB 列读取频繁但很少写入，那么 `CACHE READS` 设置更合适。如果 LOB 列很少被读写，那么 `NOCACHE` 设置是合适的。

#### 表 11-3：关于 LOB 列的缓存描述
| 缓存设置 | 含义 |
| :--- | :--- |
| **CACHE** | 指定访问时将 LOB 数据读入缓冲区缓存。 |
| **NOCACHE** | 指定访问时不将 LOB 数据读入缓冲区缓存。 |
| **CACHE READS** | 指定 LOB 数据仅在读取操作期间放入缓冲区缓存，而在写入操作期间不放入。 |


## 在行内和行外存储 LOB

### 缓存 LOB 数据

`CACHE` |
 Oracle 应将 LOB 数据置于缓冲区缓存中以便更快访问。

`CACHE READS` |
 Oracle 应在读取时将 LOB 数据置于缓冲区缓存中，但写入时不这样做。

`NOCACHE` |
 LOB 数据不应置于缓冲区缓存中。这是 SecureFiles 和 BasicFiles LOB 的默认设置。

### 存储行为

默认情况下，LOB 列最多约 4,000 个字符会与表行一起**行内存储**。如果 LOB 超过 4,000 个字符，Oracle 会自动将其存储在行数据之外。将 LOB 存储在行内的主要优点是，小型 LOB（少于 4,000 个字符）需要更少的 I/O，因为 Oracle 无需到行外搜索 LOB 数据。

然而，将 LOB 数据存储在行内并不总是可取的。其缺点是表行的大小可能变长。这可能会影响全表扫描、范围扫描以及对 LOB 列以外的列进行更新的性能。在这些情况下，您可能希望禁用行内存储。

### 控制行内存储

例如，您可以使用 `DISABLE STORAGE IN ROW` 子句明确指示 Oracle 将 LOB 存储在行外：

```sql
SQL> create table patchmain(
patch_id number
,patch_desc clob
,log_file   blob)
lob(patch_desc, log_file)
store as (
tablespace lob_data
disable storage in row);
```

如果您希望将最多 4,000 个字符的 LOB 存储在表行中，请在创建表时使用 `ENABLE STORAGE IN ROW` 子句：

```sql
SQL> create table patchmain(
patch_id number
,patch_desc clob
,log_file   blob)
lob(patch_desc, log_file)
store as (
tablespace lob_data
enable storage in row);
```

注意：LOB 定位器始终与行一起行内存储。

表创建后，您无法修改 LOB 的行内存储设置。更改行内存储的唯一方法是移动 LOB 列或删除并重新创建表。此示例通过移动 LOB 列来更改行内存储：

```sql
SQL> alter table patchmain
move lob(patch_desc)
store as (enable storage in row);
```

### 验证存储

您可以通过 `USER_LOBS` 的 `IN_ROW` 列来验证行内存储：

```sql
SQL> select table_name, column_name, tablespace_name, in_row
from user_lobs;
```

值为 `YES` 表示 LOB 存储在行内：

```sql
TABLE_NAME      COLUMN_NAME     TABLESPACE_NAME IN_ROW
--------------- --------------- --------------- ------
PATCHMAIN       LOG_FILE        LOB_DATA        YES
PATCHMAIN       PATCH_DESC      LOB_DATA        YES
```

## 实现 SecureFiles 高级功能

如前所述，SecureFiles LOB 架构允许您压缩 LOB 列、消除重复项以及透明地加密 LOB 数据。这些功能提供了 LOB 的高性能和易管理性。接下来的几节将介绍 SecureFiles 特有的功能。

### 压缩 LOB

如果您使用 SecureFiles LOB，则可以指定压缩级别。好处是 LOB 在数据库中占用的空间要少得多。缺点是读取和写入 LOB 可能需要更长时间。压缩值的描述请参见表 11-4。

表 11-4：SecureFiles LOB 可用的压缩级别

`压缩类型` | `描述`
--- | ---
`HIGH` | 最高压缩级别；读写 LOB 时延迟较高
`MEDIUM` | 中等压缩级别；如果指定了压缩但未指定级别，则为默认值
`LOW` | 最低压缩级别；读写 LOB 时延迟最低

此示例创建了一个具有低压缩级别的 `CLOB` 列：

```sql
SQL> CREATE TABLE patchmain(
patch_id   NUMBER
,patch_desc CLOB)
LOB(patch_desc) STORE AS SECUREFILE
(COMPRESS LOW)
TABLESPACE lob_data;
```

如果 LOB 已创建为 SecureFiles 类型，您可以更改其压缩级别。例如，此命令将压缩更改为 `HIGH`：

```sql
SQL> alter table patchmain modify lob(patch_desc) (compress high);
```



如果你创建了一个启用压缩的 LOB，但后来决定不再使用该功能，可以通过 `NOCOMPRESS` 子句修改 LOB 以取消压缩：

```
SQL> alter table patchmain modify lob(patch_desc) (nocompress);
```

提示
尽量通过 `CREATE TABLE` 语句来启用压缩、去重和加密。如果你使用 `ALTER TABLE` 语句，在修改 LOB 期间表会被锁定。

### 去重 LOB
如果你的应用程序中存在与两行或多行关联的相同 LOB，应考虑使用安全文件的去重功能。启用后，此功能会指示 Oracle 在将新 LOB 插入表中时检查该 LOB 是否已存在于另一行（针对相同的 LOB 列）。如果 LOB 已存在，则 Oracle 会存储指向现有相同 LOB 的指针。这可能意味着为你的应用程序节省大量空间。

注意
去重功能需要数据库企业版许可 Oracle 高级压缩选件。

以下示例创建了一个 LOB 列，使用了去重功能：

```
SQL> CREATE TABLE patchmain(
patch_id   NUMBER
,patch_desc CLOB)
LOB(patch_desc) STORE AS SECUREFILE
(DEDUPLICATE)
TABLESPACE lob_data;
```

要验证去重功能是否生效，请运行此查询：

```
SQL> select table_name, column_name, deduplication
from user_lobs;
```

以下是示例输出：

```
TABLE_NAME      COLUMN_NAME     DEDUPLICATION
--------------- --------------- ---------------
PATCHMAIN       PATCH_DESC      LOB
```

如果现有表包含安全文件 LOB，则可以修改该列以启用去重：

```
SQL> alter table patchmain
modify lob(patch_desc) (deduplicate);
```

这是另一个修改分区 LOB 以启用去重的示例：

```
SQL> alter table patchmain modify partition p2
lob (patch_desc) (deduplicate lob);
```

如果你决定不再启用去重，请使用 `KEEP_DUPLICATES` 子句：

```
SQL> alter table patchmain
modify lob(patch_desc) (keep_duplicates);
```

### 加密 LOB
你可以透明地加密安全文件 LOB 列（就像加密任何其他列一样）。在使用加密功能之前，你必须设置一个加密钱包。我在本节末尾附带了一个侧边栏，详细说明了如何设置钱包。

注意
安全文件加密功能需要数据库企业版许可 Oracle 高级安全选件。

`ENCRYPT` 子句启用安全文件加密，使用 Oracle 透明数据加密。传统的 LOB（不使用安全文件时）可以通过 `DBMS_CRYPTO` 实用工具进行加密。以下示例为 `PATCH_DESC LOB` 列启用加密：

```
SQL> CREATE TABLE patchmain(
patch_id number
,patch_desc clob)
LOB(patch_desc) STORE AS SECUREFILE (encrypt)
tablespace lob_data;
```

当你描述该表时，LOB 列现在显示加密已生效：

```
SQL> desc patchmain;
Name                                      Null?    Type
----------------------------------------- -------- ------------------------
PATCH_ID                                           NUMBER
PATCH_DESC                                         CLOB ENCRYPT
```

这是一个稍有不同的示例，指定了与 LOB 列内联的 `ENCRYPT` 关键字：

```
SQL> CREATE TABLE patchmain(
patch_id   number
,patch_desc clob encrypt)
LOB (patch_desc) STORE AS SECUREFILE;
```

你可以通过查询 `DBA_ENCRYPTED_COLUMNS` 视图来验证加密详细信息：

```
SQL> select table_name, column_name, encryption_alg
from dba_encrypted_columns;
```

以下是此示例的输出：

```
TABLE_NAME           COLUMN_NAME          ENCRYPTION_ALG
-------------------- -------------------- --------------------
PATCHMAIN            PATCH_DESC           AES 192 bits key
```

如果你已经创建了表，可以修改列以启用加密：

```
SQL> alter table patchmain modify
(patch_desc clob encrypt);
```



## Oracle SecureFiles LOB 配置与使用指南

### 指定加密算法
你还可以指定加密算法；例如：

```sql
SQL> alter table patchmain modify
(patch_desc clob encrypt using '3DES168');
```

你可以通过 `DECRYPT` 子句禁用 SecureFiles LOB 列的加密：

```sql
SQL> alter table patchmain modify
(patch_desc clob decrypt);
```

### 启用 Oracle Wallet
Oracle Wallet 是 Oracle 用于实现加密的机制。Wallet 是一个包含加密密钥的操作系统文件。启用 Wallet 需遵循以下步骤：

1.  修改 `SQLNET.ORA` 文件以包含 Wallet 的位置：

    ```
    ENCRYPTION_WALLET_LOCATION=
    (SOURCE=(METHOD=FILE) (METHOD_DATA=
    (DIRECTORY=/ora01/app/oracle/product/18.1.0.0/db_1/network/admin)))
    ```

2.  使用 `ALTER SYSTEM` 命令创建 Wallet 文件 (`ewallet.p18`)：

    ```sql
    SQL> alter system set encryption key identified by foo;
    ```

3.  启用加密：

    ```sql
    SQL> alter system set encryption wallet open identified by foo;
    ```

有关实现加密的完整详情，请参阅《Oracle Advanced Security Administrator’s Guide》，可从 Oracle 网站的 Technology Network 区域（`http://otn.oracle.com`）免费下载。

### 将 BasicFiles 迁移至 SecureFiles
你可以通过以下方法之一将 BasicFiles LOB 数据迁移至 SecureFiles：

*   创建新表，从旧表加载数据，然后重命名表
*   移动表
*   在线重定义表

以下部分将描述每种技术。

#### 创建新表
以下是一个创建新表并从旧表加载数据的简单示例。在此示例中，`PATCHMAIN_NEW` 是正在创建的、使用 SecureFiles LOB 的新表。

```sql
SQL> create table patchmain_new(
patch_id number
,patch_desc clob)
lob(patch_desc) store as securefile (tablespace lob_data);
```

接下来，用旧表的数据加载新创建的表：

```sql
SQL> insert into patchmain_new select * from patchmain;
```

现在，重命名表：

```sql
SQL> rename patchmain to patchmain_old;
SQL> rename patchmain_new to patchmain;
```

使用此技术时，请确保所有指向旧表的授权都重新授予新表。

#### 将表移动至 SecureFiles 架构
你也可以使用 `ALTER TABLE...MOVE` 语句将 LOB 的存储重新定义为 SecureFiles 类型；例如：

```sql
SQL> alter table patchmain
move lob(patch_desc)
store as securefile (tablespace lob_data);
```

你可以通过此查询验证该列现在是否为 SecureFiles 类型：

```sql
SQL> select table_name, column_name, securefile from user_lobs;
```

`SECUREFILE` 列现在的值为 `YES`：

```
TABLE_NAME      COLUMN_NAME     SEC
--------------- --------------- ---
PATCHMAIN       PATCH_DESC      YES
```

#### 使用在线重定义进行迁移
你也可以通过 `DBMS_REDEFINITION` 包在线重定义表。执行在线重定义请遵循以下步骤：

1.  确保表有主键。如果表没有主键，则创建一个：

    ```sql
    SQL> alter table patchmain
    add constraint patchmain_pk
    primary key (patch_id);
    ```

2.  创建一个将 LOB 列定义为 SecureFiles 类型的新表：

    ```sql
    SQL> create table patchmain_new(
    patch_id number
    ,patch_desc clob)
    lob(patch_desc)
    store as securefile (tablespace lob_data);
    ```

3.  映射列，并将数据从原始表复制到新表（如果行数很多，这可能需要很长时间）：

    ```sql
    SQL> declare
    l_col_map varchar2(2000);
    begin
    l_col_map := 'patch_id patch_id, patch_desc patch_desc';
    dbms_redefinition.start_redef_table(
    'MV_MAINT','PATCHMAIN','PATCHMAIN_NEW',l_col_map
    );
    end;
    /
    ```

4.  克隆被重定义表的依赖对象（授权、触发器、约束等）：

    ```sql
    SQL> set serverout on size 1000000
    SQL> declare
    l_err_cnt integer :=0;
    begin
    dbms_redefinition.copy_table_dependents(
    'MV_MAINT','PATCHMAIN','PATCHMAIN_NEW',1,TRUE, TRUE, TRUE, FALSE, l_err_cnt
    );
    dbms_output.put_line('Num Errors: ' || l_err_cnt);
    end;
    /
    ```

5.  完成重定义：

    ```sql
    SQL> begin
    dbms_redefinition.finish_redef_table('MV_MAINT','PATCHMAIN','PATCHMAIN_NEW');
    end;
    /
    ```

你可以通过此查询确认表已被重定义：

```sql
SQL> select table_name, column_name, securefile from user_lobs;
```

以下是此示例的输出：

```
TABLE_NAME           COLUMN_NAME          SECUREFILE
-------------------- -------------------- --------------------
PATCHMAIN_NEW        PATCH_DESC           NO
PATCHMAIN            PATCH_DESC           YES
```

### 查看 LOB 元数据
你可以使用任何 `DBA/ALL/USER_LOBS` 视图来显示数据库中有关 LOB 的信息：

```sql
SQL> select table_name, column_name, index_name, tablespace_name
from all_lobs
order by table_name;
```

同时请记住，LOB 段有一个对应的索引段。

```sql
SQL> select segment_name, segment_type, tablespace_name
from user_segments
where segment_name like 'SYS_LOB%'
or    segment_name like 'SYS_IL%';
```

通过这种方式，你可以在 `DBA/ALL/USER_SEGMENTS` 视图中查询 LOB 的段和索引信息。

### 加载 LOB 数据
加载 LOB 数据通常不是 DBA 的工作，但你应该熟悉用于填充 LOB 列的技术。开发人员可能会向你寻求故障排除、性能或空间相关问题的帮助。

#### 加载 CLOB
首先，创建一个指向存储 `CLOB` 文件的操作系统目录的 Oracle 数据库目录对象。加载 `CLOB` 时会使用此目录对象。在此示例中，Oracle 目录对象名为 `LOAD_LOB`，操作系统目录为 `/orahome/oracle/lob`：

```sql
SQL> create or replace directory load_lob as '/orahome/oracle/lob';
```

供参考，下面列出了用于创建加载 `CLOB` 文件的表的 DDL：

```sql
SQL> create table patchmain(
patch_id number primary key
,patch_desc clob
,patch_file blob)
lob(patch_desc, patch_file)
store as securefile (compress low) tablespace lob_data;
```

此示例还使用了一个名为 `PATCH_SEQ` 的序列。以下是序列创建脚本：

```sql
SQL> create sequence patch_seq;
```

下面的代码使用 `DBMS_LOB` 包将文本文件 (`patch.txt`) 加载到 `CLOB` 列中。在此示例中，表名是 `PATCHMAIN`，`CLOB` 列是 `PATCH_DESC`：

```sql
SQL> declare
src_clb bfile; -- point to source CLOB on file system
dst_clb clob;  -- destination CLOB in table
src_doc_name varchar2(300) := 'patch.txt';
src_offset integer := 1; -- where to start in the source CLOB
dst_offset integer := 1;  -- where to start in the target CLOB
lang_ctx integer := dbms_lob.default_lang_ctx;
warning_msg number; -- returns warning value if bad chars
begin
src_clb := bfilename('LOAD_LOB',src_doc_name); -- assign pointer to file
--
insert into patchmain(patch_id, patch_desc) -- create LOB placeholder
values(patch_seq.nextval, empty_clob())
returning patch_desc into dst_clb;
--
dbms_lob.open(src_clb, dbms_lob.lob_readonly); -- open file
--
-- load the file into the LOB
dbms_lob.loadclobfromfile(
dest_lob => dst_clb,
src_bfile => src_clb,
amount => dbms_lob.lobmaxsize,
dest_offset => dst_offset,
src_offset => src_offset,
bfile_csid => dbms_lob.default_csid,
lang_context => lang_ctx,
warning => warning_msg
);
dbms_lob.close(src_clb); -- close file
--
dbms_output.put_line('Wrote CLOB: ' || src_doc_name);
end;
/
```

你可以将此代码放在一个文件中并从 SQL 命令提示符执行。在此示例中，包含代码的文件名为 `clob.sql`：

```sql
SQL> set serverout on size 1000000
SQL> @clob.sql
```

以下是预期的输出：

```
Wrote CLOB: patch.txt
PL/SQL procedure successfully completed.
```



#### 加载 BLOB

加载 `BLOB` 与加载 `CLOB` 类似。
此示例使用了上一个示例（该示例加载了一个 `CLOB`）中的目录对象、表和序列。加载 `BLOB` 比加载 `CLOB` 更简单，因为你无需指定字符集信息。

此示例将名为 `patch.zip` 的文件加载到 `PATCH_FILE BLOB` 列中：

```sql
SQL> declare
src_blb bfile; -- 指向文件系统上的源 BLOB
dst_blb blob;  -- 表中的目标 BLOB
src_doc_name varchar2(300) := 'patch.zip';
src_offset integer := 1; -- 在源 BLOB 中的起始位置
dst_offset integer := 1;  -- 在目标 BLOB 中的起始位置
begin
src_blb := bfilename('LOAD_LOB',src_doc_name); -- 将指针分配给文件
--
insert into patchmain(patch_id, patch_file)
values(patch_seq.nextval, empty_blob())
returning patch_file into dst_blb; -- 首先创建 LOB 占位列
dbms_lob.open(src_blb, dbms_lob.lob_readonly);
--
dbms_lob.loadblobfromfile(
dest_lob => dst_blb,
src_bfile => src_blb,
amount => dbms_lob.lobmaxsize,
dest_offset => dst_offset,
src_offset => src_offset
);
dbms_lob.close(src_blb);
dbms_output.put_line('Wrote BLOB: ' || src_doc_name);
end;
/
```

你可以将此代码放在一个文件中，并从 SQL 命令提示符运行它。这里，包含代码的文件名为 `blob.sql`：

```sql
SQL> set serverout on size 1000000
SQL> @blob.sql
```

预期输出如下：

```
Wrote BLOB: patch.zip
PL/SQL procedure successfully completed.
```

## 度量 LOB 消耗的空间

如前所述，LOB 由行内 LOB 定位器、LOB 索引和由一个或多个块组成的 LOB 段构成。LOB 索引所使用的空间通常与 LOB 段所使用的空间相比可以忽略不计。你可以通过查询 `DBA/ALL/USER_SEGMENTS` 的 `BYTES` 列来查看一个段消耗的空间（就像数据库中的任何其他段一样）。以下是一个示例查询：

```sql
SQL> select segment_name, segment_type, segment_subtype,
bytes/1024/1024 meg_bytes
from user_segments;
```

你可以通过连接到 `USER_LOBS` 视图来修改查询，仅报告 LOB：

```sql
SQL> select a.table_name, a.column_name, a.segment_name, a.index_name
,b.bytes/1024/1024 meg_bytes
from user_lobs a, user_segments b
where a.segment_name = b.segment_name;
```

你也可以使用 `DBMS_SPACE.SPACE_USAGE` 包和过程来报告 LOB 正在使用的块。此包仅适用于在 ASSM 管理的表空间中创建的对象。`SPACE_USAGE` 过程有两种不同的形式：一种用于报告 BasicFiles LOB，另一种用于报告 SecureFiles LOB。

### BasicFiles 已用空间

以下是如何为 BasicFiles LOB 调用 `DBMS_SPACE.SPACE_USAGE` 的示例：

```sql
SQL> declare
p_fs1_bytes number;
p_fs2_bytes number;
p_fs3_bytes number;
p_fs4_bytes number;
p_fs1_blocks number;
p_fs2_blocks number;
p_fs3_blocks number;
p_fs4_blocks number;
p_full_bytes number;
p_full_blocks number;
p_unformatted_bytes number;
p_unformatted_blocks number;
begin
dbms_space.space_usage(
segment_owner      => user,
segment_name       => 'SYS_LOB0000024082C00002$$',
segment_type       => 'LOB',
fs1_bytes          => p_fs1_bytes,
fs1_blocks         => p_fs1_blocks,
fs2_bytes          => p_fs2_bytes,
fs2_blocks         => p_fs2_blocks,
fs3_bytes          => p_fs3_bytes,
fs3_blocks         => p_fs3_blocks,
fs4_bytes          => p_fs4_bytes,
fs4_blocks         => p_fs4_blocks,
full_bytes         => p_full_bytes,
full_blocks        => p_full_blocks,
unformatted_blocks => p_unformatted_blocks,
unformatted_bytes  => p_unformatted_bytes
);
dbms_output.put_line('Full bytes  = '||p_full_bytes);
dbms_output.put_line('Full blocks = '||p_full_blocks);
dbms_output.put_line('UF bytes    = '||p_unformatted_bytes);
dbms_output.put_line('UF blocks   = '||p_unformatted_blocks);
end;
/
```

在此 PL/SQL 中，你需要修改代码，以便报告你环境中 LOB 段的情况。

### SecureFiles 已用空间

以下是如何为 SecureFiles LOB 调用 `DBMS_SPACE.SPACE_USAGE` 的示例：

```sql
SQL> DECLARE
l_segment_owner         varchar2(40);
l_table_name            varchar2(40);
l_segment_name          varchar2(40);
l_segment_size_blocks   number;
l_segment_size_bytes    number;
l_used_blocks           number;
l_used_bytes            number;
l_expired_blocks        number;
l_expired_bytes         number;
l_unexpired_blocks      number;
l_unexpired_bytes       number;
--
CURSOR c1 IS
SELECT owner, table_name, segment_name
FROM dba_lobs
WHERE table_name = 'PATCHMAIN';
BEGIN
FOR r1 IN c1 LOOP
l_segment_owner := r1.owner;
l_table_name := r1.table_name;
l_segment_name := r1.segment_name;
--
dbms_output.put_line('-----------------------------');
dbms_output.put_line('Table Name         : ' || l_table_name);
dbms_output.put_line('Segment Name       : ' || l_segment_name);
--
dbms_space.space_usage(
segment_owner           => l_segment_owner,
segment_name            => l_segment_name,
segment_type            => 'LOB',
partition_name          => NULL,
segment_size_blocks     => l_segment_size_blocks,
segment_size_bytes      => l_segment_size_bytes,
used_blocks             => l_used_blocks,
used_bytes              => l_used_bytes,
expired_blocks          => l_expired_blocks,
expired_bytes           => l_expired_bytes,
unexpired_blocks        => l_unexpired_blocks,
unexpired_bytes         => l_unexpired_bytes
);
--
dbms_output.put_line('segment_size_blocks: '||  l_segment_size_blocks);
dbms_output.put_line('segment_size_bytes : '||  l_segment_size_bytes);
dbms_output.put_line('used_blocks        : '||  l_used_blocks);
dbms_output.put_line('used_bytes         : '||  l_used_bytes);
dbms_output.put_line('expired_blocks     : '||  l_expired_blocks);
dbms_output.put_line('expired_bytes      : '||  l_expired_bytes);
dbms_output.put_line('unexpired_blocks   : '||  l_unexpired_blocks);
dbms_output.put_line('unexpired_bytes    : '||  l_unexpired_bytes);
END LOOP;
END;
/
```

同样，在此 PL/SQL 中，你需要修改代码，以便报告你环境中包含 LOB 段的表的情况。

## 读取 BFILE

如前所述，`BFILE` 数据类型只是一个表中的列，用于存储指向操作系统文件的指针。`BFILE` 为你提供对磁盘上二进制文件的只读访问权限。要访问 `BFILE`，你必须首先创建一个目录对象。这是一个存储操作系统目录位置的数据库对象。目录对象使 Oracle 能够感知磁盘上 `BFILE` 的位置。

此示例首先创建一个目录对象，创建一个包含 `BFILE` 列的表，然后使用 `DBMS_LOB` 包访问一个二进制文件：

```sql
SQL> create or replace directory load_lob as '/orahome/oracle/lob';
```

接下来，创建一个包含 `BFILE` 数据类型的表：

```sql
SQL> create table patchmain
(patch_id   number
,patch_file bfile);
```

对于此示例，一个名为 `patch.zip` 的文件位于上述目录中。你通过使用目录对象和文件名向表中插入一条记录，使 Oracle 知晓该二进制文件：

```sql
SQL> insert into patchmain values(1, bfilename('LOAD_LOB','patch.zip'));
```

现在，你可以通过 `DBMS_LOB` 包访问 `BFILE`。例如，如果你想验证文件是否存在或显示 LOB 的长度，可以按如下方式操作：

```sql
SQL> select dbms_lob.fileexists(bfilename('LOAD_LOB','patch.zip')) from dual;
SQL> select dbms_lob.getlength(patch_file) from patchmain;
```

通过这种方式，二进制文件的行为类似于 `BLOB`。最大的区别在于，该二进制文件并不存储在数据库内部。

**提示**
请参阅 *Oracle Database PL/SQL Packages and Types Reference* 指南，获取有关使用 `DBMS_LOB` 包的完整详细信息。
该指南可在 [`http://otn.oracle.com`](http://otn.oracle.com) 上获取。



## 概述

Oracle 允许你通过各种 LOB（大对象）数据类型在数据库中存储大型对象。LOB 便于存储、管理和检索视频片段、图像、电影、文字处理文档、大型文本文件等。Oracle 可以将这些文件存储在数据库中，从而提供备份恢复和安全保护（就像对任何其他数据类型所做的那样）。`BLOB` 用于存储二进制文件，例如图像（JPEG、MPEG）、电影文件、声音文件等。如果将文件存储在数据库中不可行，则可以使用 `BFILE` LOB。

### LOB 架构

Oracle 为 LOB 提供了两种底层架构：BasicFiles 和 SecureFiles。BasicFiles 是自 Oracle 版本 8 以来就可用的 LOB 架构。SecureFiles 功能在 Oracle 数据库 11g 中引入，现在是 LOB 的默认值。SecureFiles 具有许多高级选项，例如压缩、重复数据删除和加密（这些特定功能需要 Oracle 额外许可）。

LOB 提供了一种管理大型文件的方法。Oracle 还有另一个特性——分区，它允许你管理非常大的表和索引。分区将在下一章详细介绍。


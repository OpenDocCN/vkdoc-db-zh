# 解决方案

使用适当的 `DBMS_STATS.SET_*_PREFS` 过程来更改控制统计信息收集的参数的默认值。使用以下过程来更改在统计信息收集各个级别使用的参数的默认值：

*   `SET_TABLE_PREFS`：允许您为特定表指定 `DBMS_STATS.GATHER_*_STATS` 过程使用的默认参数
*   `SET_SCHEMA_PREFS`：允许您更改用于特定模式中所有对象的 `DBMS_STATS.GATHER_*_STATS` 过程的默认参数
*   `SET_DATABASE_PREFS`：允许您更改用于整个数据库（包括所有用户模式和系统模式如 SYS 和 SYSTEM）的 `DBMS_STATS.GATHER_*_STATS` 过程的默认参数
*   `SET_GLOBAL_PREFS`：设置全局统计信息首选项；此过程允许您更改数据库中任何尚未在表级别设置首选项的对象的默认统计信息收集参数。如果您未设置表级首选项或未在 `DBMS_STATS.GATHER_*_STATS` 过程中显式设置任何参数，则参数将默认为其全局设置。

以下示例展示了如何通过调用 `SET_DATABASE_PREFS` 过程在数据库级别设置默认首选项：

```sql
SQL>  execute dbms_stats.set_database_prefs('ESTIMATE_PERCENT','20');
```

一旦您在数据库级别设置了某个参数的首选项，它将应用于数据库中的所有表。请注意，`SET_*_PREFS` 过程接受三个参数：

*   `pname` 指的是首选项的名称，例如前面示例中使用的 `ESTIMATE_PERCENT` 首选项。
*   `pvalue` 允许您为首选项指定一个值。如果您为 `pvalue` 参数指定 `NULL` 作为值，则首选项的值将被设置为 Oracle 默认值。
*   `add_sys` 是一个可选参数，如果设置为 `TRUE`，则还将包括所有 Oracle 拥有的表在统计信息收集过程中。

#### 工作原理

所有 `DBMS_STATS.GATHER_*_STATS` 过程均使用参数的默认值。有两种方式可以为 `DBMS_STATS` 包中过程（例如 `GATHER_TABLE_STATS` 过程）的各个参数指定值。您可以在执行 `GATHER_TABLE_STATS` 等过程来收集统计信息时指定偏好值。或者，您也可以使用适当的 `DBMS_STATS.SET_*_PREFS` 过程，在表、方案、数据库或全局级别更改偏好的默认值，如“解决方案”部分所示。如果您未为任何统计信息收集参数指定值，数据库将使用该参数的默认值。

您可以通过执行 `DBMS_STATS.GET_PREFS` 过程来查找任何偏好的默认值。以下示例展示了如何查找 `STALE_PERCENT` 参数的当前设置值：

```sql
SQL> select dbms_stats.get_prefs ('STALE_PERCENT','SH') stale_percent from dual;

STALE_PERCENT
--------------------------------------------------------------------------------

10

SQL>
```

您在表、方案或数据库级别收集统计信息时，需要指定类似的偏好。以下是您可以指定的各种偏好的描述，用于控制数据库收集统计信息的方式。

##### CASCADE

此参数指定数据库是否应随表统计信息一起收集索引统计信息。默认情况下，数据库会自动为表的所有索引收集统计信息（`cascade=true`）。

##### DEGREE

此参数指定数据库在收集统计信息时必须使用的并行度。Oracle 建议使用常量 `DBMS_STATS.AUTO_DEGREE` 的默认设置。Oracle 会根据对象和与并行度相关的初始化参数选择正确的并行度。当您使用默认的 `DBMS_STATS.AUTO_DEGREE` 设置时，Oracle 会根据对象的大小确定并行度。如果对象足够小，Oracle 会以串行方式收集统计信息；如果对象很大，Oracle 会根据 CPU 数量使用默认的并行度。请注意，默认并行度是 `NULL`，这意味着仅当您使用 `DEGREE` 子句在表级别设置了并行度，数据库才会使用并行方式收集统计信息。

##### ESTIMATE_PERCENT

此参数指定数据库用于估算统计信息的行百分比。对于大表，估算是唯一能在合理时间内完成统计信息收集过程的方式——统计信息收集并非简单过程——它对资源密集型，且对大表消耗大量时间。您可以为 `estimate_percent` 参数设置 0 到 100 之间的值。一个经验法则是，表的数据越均匀，所需的样本量就越小。另一方面，如果表的数据高度倾斜，则应使用更大的样本量来捕获数据分布的差异。当然，将此参数设置为 100 意味着数据库不进行估算——它将收集表中每一行的统计信息。通常，DBA 会将 `estimate_percent` 参数设置得过高，因为他们曾在设置小样本量时对某个表有过糟糕的体验。如果您认为数据分布均匀，即使 1% 或 2% 的样本也能为您提供非常准确的统计信息，并节省大量时间和处理开销。

![images](img/square.jpg) **提示** 在 Oracle Database 11g 中，通过将 `estimate_percent` 参数设置为 `DBMS_STATS.AUTO_SAMPLE_SIZE` 计算的关键统计信息 NDV（不同值的数量），在统计学上与通过 100% 完整统计信息计算的 NDV 计数是相同的。最佳实践是从 `AUTO_SAMPLE_SIZE` 开始，仅在必要时才设置您自己的样本大小。

为 `estimate_percent` 参数选择最佳大小并不容易——如果设置得太高，收集统计信息会花费很长时间。如果设置得太低，您确实可以快速收集统计信息，但这些统计信息很可能不准确。默认情况下，数据库使用常量 `DBMS_STATS.AUTO_SAMPLE_SIZE` 来确定最佳样本大小。您可以通过以下方式为 `estimate_percent` 参数指定 `AUTO` 值：

```sql
SQL> exec dbms_stats.gather_table_stats(NULL, 'MASSIVE_TABLE', estimate_percent=>
dbms_stats.auto_sample_size)
```

当您为 `estimate_percent` 参数设置 `AUTO` 值时，数据库不仅会自动确定采样大小，还会随着数据分布的变化调整样本大小。NDV 是评估不同样本大小收集的统计信息准确性的一个好标准。列的 NDV 定义如下：

`准确率 = 1 - (估算的 NDV - 实际的 NDV) / 实际的 NDV`

准确率范围在 0% 到 100% 之间。100% 的样本大小总会给您 100% 的准确率——重要的是，在 Oracle 11g 中，自动采样提供的准确率非常接近 100%，并且只占用为大表收集完整统计信息所需时间的一小部分。

### METHOD_OPT

`METHOD_OPT` 参数可以指定两件事：数据库将为其收集统计信息的列，以及数据库将在其上创建直方图的列。您还可以指定直方图中的桶数。您可以为此参数指定以下选项之一：

```
FOR ALL [INDEXED  |  HIDDEN]  COLUMNS  [size_clause]
```
```
FOR COLUMNS [size_clause]  column  [size_clause]  [,COLUMN  [size_clause]…]
```

`FOR ALL` 选项允许您指定数据库必须为所有列收集统计信息，或者仅为索引列收集。如果指定了 `INDEXED COLUMNS` 选项，数据库将只为那些有索引的列收集统计信息。请谨慎使用此选项，因为数据库不会收集表列的统计信息，而是使用基本的默认统计信息。在数据仓库环境中使用 `FOR ALL INDEXED COLUMNS` 可能尤其有问题，因为该环境中索引使用不多。

`FOR COLUMNS` 选项允许您指定数据库必须在其中一列或多列上收集统计信息，而不是默认的在所有列上收集。在此上下文中指定 `column` 子句的方法如下：

```
column:= column_name  |  extension name |  extension
```

`column_name` 子句指的是列的名称，`extension` 可以是列组（格式为 `column_name, column_name [,…]`）或表达式。

`FOR ALL` 和 `FOR COLUMNS` 选项的关键子句是 `size_clause`。`size_clause` 确定数据库是否应为某列收集直方图，以及在什么条件下收集。一个选项是提供一个整数值，指示您想要的直方图桶数——范围在 1 到 254 之间——例如：

```
SQL> exec dbms_stats.gather_table_stats('HR','EMPLOYEES',method_opt=> 'for columns size 254 job_id')
PL/SQL procedure successfully completed.
SQL>
```

执行此过程时，数据库会收集直方图，并且将有 254 个直方图桶。

![images](img/square.jpg) `注意` `integer` 子句的值为 1（例如，`'FOR ALL COLUMNS SIZE 1'`）实际上不会在列上创建任何直方图，因为所有数据都被放入一个桶中。此外，如果表上已经有直方图，将 `integer` 子句的值设置为 1 将删除这些直方图。

> `size_clause` 的另一个选项是指定以下三个值之一：
> 
> `REPEAT`：指定数据库必须仅在那些已为其收集了直方图的列上收集直方图；将值设置为 `repeat` 而不是整数值 1 可确保保留任何有用的直方图。
> 
> `AUTO`：让数据库根据每列的数据分布（是均匀还是倾斜）和实际的列使用情况统计信息来决定应在其上收集直方图的列。
> 
> `SKEWONLY`：让数据库根据每列的数据分布来决定应在其上收集直方图的列。
> 
> 以下示例指定了 `SKEWONLY`：

```
SQL> exec dbms_stats.gather_table_stats('HR','EMPLOYEES',method_opt=> 'for all columns size skewonly')
PL/SQL procedure successfully completed.
SQL>
```

当您指定 `SKEWONLY` 时，数据库将查看每列的数据分布，以确定数据是否倾斜到足以证明创建直方图是合理的。

`METHOD_OPT` 参数的默认值是 `FOR ALL COLUMNS SIZE AUTO`。也就是说，数据库将为表中的所有列收集统计信息，并自动选择应为其创建直方图的列。

### NO_INVALIDATE

您可以为这个参数设置三个不同的值。值 `TRUE` 表示数据库不会使其正在收集统计信息的表的依赖游标失效。值 `FALSE` 表示数据库立即使依赖游标失效。最后，您可以将此参数设置为值 `DBMS_STATS.AUTO_INVALIDATE`，让 Oracle 决定使游标失效——这也是 `NO_INVALIDATE` 参数的默认值。

### GRANULARITY

此参数确定数据库如何处理分区表的统计信息收集。您可以为 `GRANULARITY` 参数指定以下各种选项：

> `ALL`：收集子分区、分区和全局级别的统计信息；此设置提供了非常准确的表统计信息集，但资源消耗极大，完成时间比使用其他选项的统计信息收集作业长得多。
> 
> `GLOBAL`：仅收集表的全局统计信息。
> 
> `PARTITION`：仅收集分区级别的统计信息——分区级别的统计信息在表级别汇总，在表级别可能不太准确。
> 
> `GLOBAL AND PARTITION`：收集全局和分区级别的统计信息，但不收集子分区级别的统计信息。
> 
> `SUBPARTITION`：仅收集子分区统计信息。
> 
> `AUTO`：这是 `GRANULARITY` 参数的默认值，根据分区类型确定粒度。

请注意，`ALL` 设置除了消耗大量资源外，完成时间可能也很长。对于复合分区表，收集子分区级别的统计信息通常并不真正必要。在大多数情况下，默认的 `AUTO` 设置效果很好。`ALL` 设置绝对是过度的，在大多数（可能所有）情况下都没有必要。

### PUBLISH

默认情况下，数据库在完成统计信息收集过程后立即发布所有统计信息。您可以通过将 `PUBLISH` 参数设置为 `FALSE` 来指定数据库将新收集的统计信息保留为待处理统计信息。

### INCREMENTAL

`INCREMENTAL` 首选项确定数据库是否无需执行全表扫描即可维护分区表的统计信息。此参数的默认值为 `FALSE`，意味着数据库执行全表扫描来维护全局统计信息。配方 13-19 详细讨论了增量统计信息收集。

### STALE_PERCENT

`STALE_PERCENT` 首选项确定表中必须更改的行的比例，然后数据库才认为该表的统计信息“过时”并开始收集新的统计信息。默认情况下，`STALE_PERCENT` 参数设置为 10%。不要对完全没有更改或更改很少的表收集统计信息，因为您将收集不必要的统计信息。

### AUTOSTATS_TARGET

此首选项仅对自动统计信息收集有效，在使用 `SET_GLOBAL_STATS` 过程设置全局统计信息首选项时指定。您可以为此首选项设置以下值：

> `ALL`：为数据库中的所有对象收集统计信息。
> 
> `ORACLE`：为所有 Oracle 拥有的对象收集统计信息。
> 
> `AUTO`：数据库决定应为其收集统计信息的对象。
> 
> `AUTOSTATS_TARGET` 参数的默认值是 `AUTO`。请注意，目前 `ALL` 和 `AUTO`（默认）设置的工作方式相同。Oracle 建议您将 `AUTOSTATS_TARGET` 首选项设置为值 `ORACLE`，以确保数据库收集最新的字典统计信息（针对 SYS 和 SYSTEM 拥有的对象）。

我们在本配方中融入了多个 Oracle 统计信息收集的最佳实践。除非有充分的理由不这样做，否则尽量坚持使用首选项的默认设置。请记住，如果您要创建新表，可以先加载数据并仅针对该表收集统计信息。然后再创建索引，因为数据库在索引创建时会自动计算索引的统计信息。

## 13-4. 手动生成统计信息

#### 问题

你需要判断是应该让数据库自动收集优化器统计信息，还是必须手动收集这些统计信息。

#### 解决方案

在大多数情况下，自动统计信息收集任务足以收集优化器统计信息。事实上，许多生产环境的数据库都如**配方 13-2**所示，实现了统计信息自动收集，并且从未使用手动统计信息收集流程。然而，在某些情况下，手动收集统计信息可能是必要的。以下是两种必须手动收集统计信息的场景。

##### 易变表

如果你的数据库中包含在一天内经历大量删除（甚至截断）操作的**易变表**，那么在夜间维护窗口运行的自动统计信息收集作业可能就不够充分了。对于表数据在一天中持续波动的情况，你可以采用以下几种策略：

*   在表的行数处于其“典型”状态时收集统计信息。完成后，锁定这些统计信息，以防止自动统计信息收集作业在夜间维护窗口为该表收集新的统计信息。
*   另一种选择是将表的统计信息设为 null。

你可以先删除统计信息，然后立即锁定表的统计信息，从而将表的统计信息设为 null，如下例所示：

```
SQL> execute dbms_stats.delete_table_stats('OE','ORDERS');

PL/SQL procedure successfully completed.

SQL> execute dbms_stats.lock_table_stats('OE','ORDERS');

PL/SQL procedure successfully completed.

SQL>
```

##### 批量加载表

对于你进行批量数据加载的表，必须在加载数据后立即收集统计信息。如果你没有在批量加载数据后立即收集统计信息，数据库仍可以使用动态采样来估算统计信息，但这些统计信息不如你手动收集的统计信息全面。

#### 工作原理

在 Oracle Database 10g 版本引入自动优化器统计信息收集功能之前，每位 DBA 都会使用推荐的 `DBMS_STATS` 包（甚至更早的 `analyze table` 命令）来编写脚本收集统计信息。有了自动统计信息收集，DBA 就不必再通过调度 `DBMS_STATS` 作业来收集优化器统计信息了。然而，你仍然可能会遇到自动统计信息收集不合适的情况。“解决方案”部分描述了两种这样的情况，以及如何通过手动收集统计信息来处理它们。

你可以通过使用相应的 `DBMS_STATS.GATHER_*_STATS` 过程，在表、模式或数据库级别手动收集统计信息。

当你手动收集优化器统计信息时，建议坚持使用控制数据库如何收集统计信息的各个参数的默认设置。通常，当恢复为默认设置时，统计信息收集作业的性能（速度）和统计信息本身的质量都会得到改善。例如，许多 DBA 通过 `estimate_percent` 参数设置了过高的采样比例，而不是让数据库基于 `DBMS_STATS.AUTO_SAMPLE_SIZE` 常量使用适当的采样比例。

### 13-5\. 锁定统计信息

#### 问题

你想锁定表或模式的统计信息，以冻结这些统计信息。

#### 解决方案

你可以通过执行相应的 `DBMS_STATS.LOCK_*` 过程来锁定表或模式的统计信息。例如，你可以使用 `DBMS_STATS` 包中的 `LOCK_TABLE_STATS` 过程来锁定表的统计信息：

```
SQL> execute dbms_stats.lock_table_stats(ownname=>'SH',tabname=>'SALES');

PL/SQL procedure successfully completed.
SQL>
```

你可以通过执行以下过程来解锁表的统计信息：

```
SQL> execute dbms_stats.unlock_table_stats(ownname=>'SH',tabname=>'SALES');

PL/SQL procedure successfully completed.

SQL>
```

你可以使用 `DBMS_STATS.LOCK_SCHEMA_STATS` 过程来锁定模式的统计信息，如下所示：

```
SQL> execute dbms_stats.lock_schema_stats('SH');

PL/SQL procedure successfully completed.
SQL>
```

使用 `UNLOCK_SCHEMA_STATS` 过程解锁统计信息：

```
SQL> execute dbms_stats.unlock_schema_stats('SH');

PL/SQL procedure successfully completed.

SQL>
```

#### 工作原理

你可能希望锁定表的统计信息以冻结当前的统计信息集。你也可以先删除现有统计信息然后再锁定——在这种情况下，你是强制数据库使用动态采样来估算表的统计信息。删除表的统计信息然后锁定，在效果上等同于将表的统计信息设为 null。你可以选择在使用 `GATHER_TABLE__STATS` 过程时设置 `force` 参数来覆盖表对其统计信息的锁定。

![images](img/square.jpg) **注意** 锁定表也会锁定所有依赖于该表的统计信息，例如索引、直方图和列统计信息。

### 13-6\. 处理缺失统计信息

#### 问题

数据库中的某些表缺失统计信息，因为这些表是在夜间批处理窗口之外加载数据的。你无法在白天数据库处理其他工作负载时收集这些表的统计信息。

#### 解决方案

Oracle 使用动态采样来弥补缺失的统计信息。当你启用动态采样时，数据库将扫描表中数据块的随机样本。你可以通过设置 `optimizer_dynamic_sampling` 初始化参数在数据库中启用/禁用动态采样。默认情况下动态采样是启用的，如下所示：

```
SQL> sho parameter dynamic

NAME                                  TYPE        VALUE
------------------------------------ ----------- -----
optimizer_dynamic_sampling           integer     2
SQL>
```

动态采样的默认级别是 2——将其设置为 0 将禁用动态采样。你可以通过设置不同的采样级别来修改默认值，如下所示：

```
SQL> alter system set optimizer_dynamic_sampling=4 scope=both;
System altered.

SQL>
```

#### 工作原理

理想情况下，您应该使用 `DBMS_STATS` 包（手动或通过自动统计信息收集）来收集优化器统计信息。在您没有机会为新创建或新加载的表收集统计信息的情况下，该表将不会有任何统计信息，直到数据库通过其自动统计信息收集作业自动生成统计信息，或者您安排了手动统计信息收集作业。即使您不收集任何统计信息，数据库也会使用一些基本统计信息（如表和索引块数）来估计查询中谓词的选择性。动态采样更进一步，通过在编译时动态收集额外的统计信息来增强这些基本统计信息。当处理涉及无统计信息表的频繁执行查询时，动态采样特别有帮助。

![images](img/square.jpg) **注意** Oracle 不对外部表执行动态采样。

动态采样是有成本的，因为数据库在查询编译期间使用资源来收集统计信息。如果您不频繁执行这些查询，数据库每次执行涉及必须动态收集统计信息的表的查询时都会产生开销。为了使动态采样真正发挥作用，它必须帮助频繁执行的查询。

理解动态采样级别（范围从 0 到 10）的含义很重要。请注意，各采样级别使用的样本大小以数据块为单位，而不是行数。以下是对各种动态采样级别的简要描述。

> *级别 1*: 对没有统计信息的所有表使用动态采样，前提是查询中至少有一个没有统计信息的非分区表；该表还必须没有任何索引，并且必须大于 32 个块（这是此级别的样本大小）。
>
> *级别 2*: 如果查询中至少有一个表没有统计信息，则使用动态采样；样本大小为 64 个块。
>
> *级别 3*: 如果查询满足级别 2 的标准，并且 `WHERE` 子句谓词中有一个或多个表达式，则使用动态采样；样本大小为 64 个块。
>
> *级别 4*: 如果查询满足所有级别 3 的标准，并且使用复杂谓词（例如多个谓词之间的 `OR/AND` 运算符），则使用动态采样；样本大小为 64 个块。
>
> *级别 5–9*: 如果语句满足级别 4 的标准，则使用动态采样；这些级别的区别仅在于样本大小，范围从 128 到 4086 个块。
>
> *级别 10*: 最全面的级别——它对所有语句都使用动态采样，并且使用的样本并非真正的样本，因为它会检查所有数据块以获取统计信息。

动态采样是对 `DBMS_STATS` 包过程收集的统计信息的补充。Oracle 通常不期望您使用此功能，因为在生成执行计划期间收集优化器统计信息有额外开销。动态采样确实有助于获得更好的基数估计，但由于涉及的开销，它更适用于数据仓库或决策支持系统中的长运行查询，而不是 OLTP 数据库中的查询。您还必须记住，通过动态采样收集的统计信息与通过 `DBMS_STATS` 过程收集的统计信息绝不相同。动态采样仅收集基本的统计信息，例如数据块数量和列的最高/最低值。如果您必须设置动态采样级别，请使用 `alter session` 语句在会话级别进行设置，而不是在数据库范围内设置。

### 13-7. 导出统计信息

#### 问题

您想要将一组统计信息从您的生产数据库导出到一个小得多的测试数据库。

#### 解决方案

您可以使用 `DBMS_STATS.EXPORT_*_STATS` 过程将优化器统计信息从一个数据库导出到另一个数据库。这些过程允许您从源表、方案或数据库导出优化器统计信息。完成统计信息导出后，您必须执行一个 `DBMS_STATS.IMPORT_*_STATS` 过程，将统计信息导入不同的数据库。您可以在表、方案或数据库级别导出统计信息。以下是一个说明如何从表导出统计信息的示例。

1.  创建一个表来保存导出的统计信息：
    ```sql
    SQL> execute dbms_stats.create_stat_table(ownname=>'SH',stattab=>'mytab',
                                             tblspace=>'USERS')

    PL/SQL procedure successfully completed.

    SQL>
    ```

2.  使用 `DBMS_STATS.EXPORT_*STATS` 过程将表 `SH.SALES` 的统计信息从数据字典导出到 `mytab` 表。
    ```sql
    SQL> exec dbms_stats.export_table_stats(ownname=> 'SH',tabname=>'SALES',stattab=>
    'mytab')

    PL/SQL procedure successfully completed.

    SQL>
    ```

3.  在另一个数据库中，使用 `DBMS_STATS.IMPORT_*STATS` 过程导入统计信息。
    ```sql
    SQL> exec dbms_stats.import_table_stats(ownname=>'SH',tabname=>'SALES',stattab=>
    'MyTab',no_invalidate=>true);

    PL/SQL procedure successfully completed.

    SQL>
    ```

#### 工作原理

`EXPORT_TABLE_STATS` 过程从数据字典导出表的当前统计信息，并将它们存储在您创建的表中。请注意，此过程不会生成新的统计信息，数据库将继续使用该表（在我们的示例中是 `SH.SALES`）的当前统计信息。默认情况下，`cascade` 选项为 `true`，这意味着该过程将同时导出 `SH.SALES` 表中所有索引的统计信息以及列统计信息。

只有在将导出的统计信息导入到数据字典（在相同或不同的数据库中）后，才能使优化器使用这些统计信息。`IMPORT_TABLE_STATS` 过程将您之前导出的统计信息导入到数据字典中。将 `no_invalidate` 参数设置为 `true`（默认为 `false`）可确保任何依赖的游标不会失效。默认情况下，当统计信息被锁定时，您无法导入表的统计信息。您可以通过将 `force` 参数设置为 `true` 来覆盖此属性。如果您要将统计信息导入到与导出统计信息不同的数据库，则必须将存储统计信息的表导出到目标数据库。然后，您必须在执行 `IMPORT_TABLE_STATS` 过程之前将该表导入到目标数据库中。

导出和导入统计信息是在测试系统中向优化器呈现与生产系统中相同的统计信息以确保执行计划一致的理想方法。当您想要保留一组已知良好的统计信息的时间超过“恢复统计信息”功能（在配方 13-8 中解释）所允许的时间时，这也是一个很好的策略。导出和导入统计信息的功能使您可以在决定哪组参数最适合您的数据库之前，测试不同的统计信息集。

在此配方中，我们向您展示了如何导出和导入表级统计信息。`DBMS_STATS` 包还包含在列、索引、方案和数据库级别导出和导入统计信息的过程。此外，还有用于导出和导入字典统计信息、固定对象统计信息和系统统计信息的过程。

### 13-8. 恢复统计信息的先前版本

#### 问题

在收集新的统计信息后，某些查询的性能突然下降。您想看看是否可以使用一组您已知运行良好的旧统计信息。



#### 解决方案

使用 `DBMS_STATS.RESTORE_STATS` 过程可以将优化器统计信息集恢复到旧版本。在恢复旧统计信息之前，请先检查可以回溯多久来恢复统计信息：
```sql
SQL> select dbms_stats.get_stats_history_availability from dual;

GET_STATS_HISTORY_AVAILABILITY
----------------------------------------------------------------------
19-APR-11 07.49.26.718000000 AM -04:00

SQL>
```
此查询的输出显示，您可以将统计信息恢复到比所示时间戳（19-APR-11 07.49.26.718000000 AM -04:00）更近的时间点。

执行 `DBMS_STATS` 包中的 `RESTORE_*_STATS` 过程可以将统计信息恢复到更早的时期。以下示例展示了如何在模式级别恢复统计信息。
```sql
SQL> exec dbms_stats.restore_schema_stats(ownname=>'SH',as_of_timestamp=>'19-MAY-11 01.30.31.323000 PM -04:00',no_invalidate=>false)

PL/SQL procedure successfully completed.

SQL>
```

#### 工作原理

当数据库收集新的统计信息时，它不会丢弃前一组统计信息，而是保留旧版本一定天数。您可以通过执行 `DBMS_STATS.RESTORE_*_STATS` 过程来恢复旧统计信息，这些过程会用您指定时间的统计信息替换当前统计信息。当您希望优化器使用与访问旧统计信息集时相同的执行计划时，请恢复统计信息。默认情况下，数据库会管理历史统计信息的保留和清除。以下是找出数据库默认保留多少天统计信息的方法。
```sql
SQL> select dbms_stats.get_stats_history_retention from dual;

GET_STATS_HISTORY_RETENTION
---------------------------
                          31
SQL>
```
数据库会自动清除超过 31 天前收集的统计信息（前提是存在更新的统计信息！）。您可以通过执行 `DBMS_STATS.PURGE_STATS` 过程手动清除所有旧版本的统计信息。您可以通过执行以下命令来更改数据库保留统计信息的天数：
```sql
SQL> exec dbms_stats.alter_stats_history_retention(retention=>60);

PL/SQL procedure successfully completed.

SQL>
```
该命令指示数据库将历史统计信息保存 60 天。

在“解决方案”部分展示的示例中，我们展示了如何为模式恢复统计信息。您可以类似地使用 `RESTORE_DATABASE_STATS` 过程为数据库恢复统计信息，或使用 `RESTORE_TABLE_STATS` 过程为表恢复统计信息。您还可以使用 `RESTORE_DICTIONARY_STATS` 过程恢复字典统计信息，使用 `RESTORE_SYSTEM_STATS` 过程恢复系统统计信息。

### 13-9. 收集系统统计信息

#### 问题

您知道优化器在选择执行计划时会使用系统的 I/O 和 CPU 特性。您希望确保优化器正在使用准确的系统统计信息。

#### 解决方案

您可以收集两种类型的系统统计信息来捕获系统的 I/O 和 CPU 特性。您可以收集*工作负载统计信息*或*非工作负载统计信息*，以使优化器能够更准确地估计真实的 I/O 和 CPU 成本，这对于查询优化至关重要。

当数据库收集非工作负载统计信息时，它会模拟工作负载。以下是使用 `DBMS_STATS.GATHER_SYSTEM_STATS` 过程收集非工作负载统计信息的方法：
```sql
SQL> execute dbms_stats.gather_system_stats()

PL/SQL procedure successfully completed.

SQL>
```
您还可以在数据库处理典型工作负载时收集系统统计信息。这些系统统计信息称为工作负载统计信息，更能代表实际的系统 I/O 和 CPU 特性，并向优化器提供更准确的系统硬件状况。您可以通过执行带有 “start” 和 “stop” 选项的 `DBMS_STATS.GATHER_SYSTEM_STATS` 过程来收集工作负载系统统计信息：
```sql
SQL> execute dbms_stats.gather_system_stats('start')

PL/SQL procedure successfully completed.
SQL>
```
您可以在工作负载窗口开始前执行上述命令。工作负载窗口结束后，执行以下命令停止系统统计信息收集：
```sql
SQL> execute dbms_stats.gather_system_stats('stop')

PL/SQL procedure successfully completed.

SQL>
```
您也可以使用 `interval` 参数执行 `GATHER_SYSTEM_STATS` 过程，以指示数据库在您指定的时间段内收集工作负载系统统计信息，并在该时间段结束时自动停止统计信息收集过程。以下是一个示例：
```sql
SQL> execute dbms_stats.gather_system_stats('interval',90);
PL/SQL procedure successfully completed.
SQL>
```
上述命令收集 90 分钟的工作负载统计信息。

一旦收集了非工作负载或工作负载系统统计信息，您可以在 `sys.aux_stats$` 视图中查看为各种系统统计信息捕获的值，如下一节所示。

![images](img/square.jpg) **提示** Oracle 强烈建议收集系统统计信息，以便为优化器提供更准确的 CPU 和 I/O 成本估算。




## 工作原理

准确的系统统计信息对于优化器评估可执行计划至关重要。正是通过估算各种系统性能特性（如 I/O 速度和 CPU 速度），优化器才能计算出全表扫描与索引读取等操作的成本。

你可以通过收集系统统计信息，向优化器传递最多九个优化器系统统计信息。数据库在无负载模拟统计信息收集过程中收集前三个统计信息。在负载模式系统统计信息收集过程中，它会收集所有九个系统统计信息。以下是这九个系统统计信息的摘要：

> `cpuspeedNW`： 显示无负载 CPU 速度，以平均每秒 CPU 周期数为单位
>
> `ioseektim`： 寻道时间、延迟时间和操作系统开销时间的总和
>
> `iotfrspeed`： 代表 I/O 传输速度，告诉优化器数据库在单次读取请求中读取数据的速度
>
> `cpuspeed`： 代表负载统计信息收集期间的 CPU 速度
>
> `maxthr`： 最大 I/O 吞吐量
>
> `slavethr`： 平行从属 I/O 平均吞吐量
>
> `sreadtim`： 单块读取时间统计信息显示随机单块读取的平均时间。
>
> `mreadtim`： 多块读取时间统计信息显示顺序多块读取的平均时间（以秒为单位）。
>
> `mbrc`： 多块读取计数统计信息显示以块为单位的平均多块读取计数。

当你收集无负载系统统计信息时，数据库只捕获 `cpuspeedNW`、`ioseektim` 和 `iotfrspeed` 系统统计信息。以下是一个查询，显示 Oracle 11g 数据库（Windows 系统上）中的默认系统统计信息。

```
SQL> select pname, pval1 from sys.aux_stats$ where sname = 'SYSSTATS_MAIN';

PNAME                               PVAL1
------------------------------ ----------
CPUSPEED
CPUSPEEDNW                      1183.90219
IOSEEKTIM                              10
IOTFRSPEED                           4096
MAXTHR
MBRC
MREADTIM
SLAVETHR
SREADTIM

9 rows selected.

SQL>
```

数据库默认使用无负载系统统计信息，当你启动实例时，三个无负载统计信息—I/O 传输速度（`IOTFRSPEED`）、I/O 寻道时间（`IOSEEKTIM`）和 CPU 速度（`CPUSPEEDNW`）—被初始化为默认值。一旦你如“解决方案”部分所示收集了无负载统计信息，这三个无负载系统统计信息中的部分或全部可能会改变。在我们的案例中，收集无负载统计信息后，`CPUSPEEDNW` 的值变为 2039.06，`IOSEEKTIM` 统计信息的值变为 14.756。然而，`IOTFRSPEED` 统计信息的值保持 4096 不变。

如果你注意到即使手动收集了几次统计信息，`sys.aux_stats$` 视图仍然显示无负载统计信息的默认值，你可以使用 `DBMS_STATS.SET_SYSTEM_STATS` 过程手动将统计信息值设置为你 I/O 或 CPU 系统的已知规格。你可以使用此过程为九个系统统计信息中的任何一个设置值。

当你在负载模式下收集系统统计信息时，你会看到剩余六个系统统计信息中部分或全部的值。在我们的示例中，这些是通过运行负载模式下的 `GATHER_SYSTEM_STATS` 过程收集的系统统计信息。

```
SQL>  select pname, pval1 from sys.aux_stats$ where sname = 'SYSSTATS_MAIN';

PNAME                               PVAL1
------------------------------ ----------
CPUSPEED                             2040
CPUSPEEDNW                       2039.06
IOSEEKTIM                         14.756
IOTFRSPEED                           4096
MAXTHR
MBRC                                   7
MREADTIM                        46605.947
SLAVETHR
SREADTIM                        51471.538

9 rows selected.

SQL>
```

如果在负载统计信息收集期间数据库执行了任何全表扫描，Oracle 会使用 `mbrc` 和 `mreadtim` 统计信息的值来估算全表扫描的成本。如果没有这两个统计信息，数据库会使用 `db_file_multiblock_read_count` 参数的值来估算全表扫描的成本。

你可以通过执行 `DELETE_SYSTEM_STATS` 过程来删除所有系统统计信息：

```
SQL> execute dbms_stats.delete_system_stats()

PL/SQL procedure successfully completed.

SQL>
```

根据 Oracle 的说法，收集负载统计信息不会给你的系统带来额外的开销。但是，请确保指定特定的时间间隔或在短暂收集后停止统计信息收集，以避免潜在的开销。

### 13-10\. 验证新统计信息

#### 问题

你正在收集新的统计信息，但在验证它们之前，不希望数据库自动使用这些统计信息。

#### 解决方案

在这个例子中，我们将展示如何阻止数据库自动发布表的新统计信息。以下是必须遵循的步骤。

1.  执行以下语句，阻止数据库自动发布为 `SH.SALES` 表收集的新统计信息。

    ```
    SQL> execute dbms_stats.set_table_prefs('SH','SALES','PUBLISH','false');

    PL/SQL procedure successfully completed.

    SQL>
    ```

    该语句将 `SH.SALES` 表的 `PUBLISH` 参数首选项设置为 `false`（默认=`true`）。从现在开始，数据库不会自动发布为 `SH.SALES` 表收集的统计信息。相反，它会将这些统计信息暂时搁置，等待你的批准。这些统计信息被称为待定统计信息，因为数据库尚未将其提供给优化器。

2.  为 `SH.SALES` 表收集新的统计信息：
    ```
    SQL> exec dbms_stats.gather_table_stats('sh','sales');

    PL/SQL procedure successfully completed.
    SQL>
    ```

3.  告诉优化器使用新收集的待定统计信息，以便你可以用这些统计信息测试你的查询：
    ```
    SQL>  alter session set optimizer_use_pending_statistics=true;

    Session altered.

    SQL>
    ```

4.  通过针对 `SH.SALES` 表运行工作负载并检查性能和执行计划来进行测试。
5.  如果你对新（待定）统计信息集感到满意，通过执行以下语句使其公开：
    ```
    SQL> execute dbms_stats.publish_pending_stats('SH','SALES');

    PL/SQL procedure successfully completed.

    SQL>
    ```

6.  如果你想删除新统计信息，请执行以下命令：
    ```
    SQL> exec dbms_stats.delete_pending_stats('SH','SALES');

    PL/SQL procedure successfully completed.

    SQL>
    ```




### 13-10. 控制统计信息的发布

#### 工作原理

默认情况下，数据库会立即开始使用收集到的所有统计信息。但是，您可以指定数据库不自动使用新收集的统计信息，直到您确认这些统计信息将改进或至少不会降低当前的执行计划。您可以通过将新统计信息保持在*待处理*状态来实现这一点。使统计信息对优化器可用，以便其在计算执行计划时能够使用它们，这被称为*发布*统计信息。数据库将已发布的统计信息存储在其数据字典中。如果您不确定一组新统计信息的效果，可以防止数据库自动发布统计信息，直到您先完成测试。当您将统计信息保持在待处理状态时，数据库不会将其存储在数据字典中——而是将其存储在一个私有区域中，并且只有在您将`optimizer_use_pending_statistics`参数设置为`true`时，才会使这些统计信息对优化器可用。

在指定数据库必须将新收集的统计信息保持在待处理状态后，您可以选择发布新统计信息或将其删除。使用`publish_pending_stats`过程发布统计信息，使用`delete_pending_stats`过程删除统计信息。如果您删除了某个对象的待处理统计信息，数据库将使用该对象的现有统计信息。

在此示例中，我们展示了如何更改表级别的统计信息`PUBLISH`设置。您也可以在模式级别执行此操作，但不能在数据库级别执行。如果在模式级别操作，则需要运行以下语句（模式名称为 SH）。

```sql
SQL> execute dbms_stats.set_schema_prefs('SH','PUBLISH','false');
SQL> execute dbms_stats.publish_pending_stats(null,null);
SQL> execute dbms_stats.delete_pending_stats('SH');
```

### 13-11. 强制优化器使用索引

#### 问题

您知道在某个列上使用特定索引会加快查询速度，但优化器在其执行计划中并未使用该索引。您希望强制优化器使用该索引。

#### 解决方案

当优化器没有使用索引时，您可以通过调整`optimizer_index_cost_adj`初始化参数来强制其使用。您可以在系统或会话级别设置此参数。以下示例展示了如何在会话级别设置此参数：

```sql
SQL> alter session set optimizer_index_cost_adj=50;

Session altered.
SQL>
```

`optimizer_index_cost_adj`参数的默认值为 100，您可以将该参数设置为介于 0 到 10000 之间的值。该参数的值越低，优化器使用索引的可能性就越大。

#### 工作原理

`optimizer_index_cost_adj`参数允许您调整索引访问的成本。优化器对此参数使用默认值 100，这意味着它基于正常的成本模型来评估索引访问路径。根据优化器对执行索引读取成本的估计，它决定是否使用索引。这通常工作良好。然而，在某些情况下，即使使用索引能带来更好的执行计划，优化器也不会使用它，因为优化器对索引访问路径成本的估计可能不准确。由于优化器对`optimizer_index_cost_adj`参数使用默认值 100，您可以通过将此参数设置为较小的值，使索引成本在优化器看来更低。任何小于 100 的值都会使索引使用（就索引读取的成本而言）在优化器看来更便宜。通常，当您这样做时，优化器就会开始使用您希望它使用的索引。在我们的示例中，我们将`optimizer_index_cost_adj`参数设置为 50，使索引访问路径的成本看起来是其正常成本（100）的一半。您设置此参数的值越低，索引成本访问路径在优化器看来就越便宜，它就越倾向于选择索引访问路径而非全表扫描。

我们建议您仅针对特定查询在会话级别设置`optimizer_index_cost_adj`参数，因为如果在数据库级别设置，它有可能改变许多查询的执行计划。默认情况下，如果您设置了`ALL_ROWS`优化器目标，优化器会内置偏好全表扫描。通过将`optimizer_index_cost_adj`参数设置为小于 100 的值，您就是在诱导优化器偏好索引扫描而非全表扫描。可以放心地使用`optimizer_index_cost_adj`参数，尤其是在 OLTP 环境中，您可以尝试将该参数设置为较低的值（例如 5 或 10），以强制优化器使用索引。

默认情况下，优化器假设与全表扫描相关的多块读取 I/O 成本和与索引读取相关的单块读取成本是相同的。然而，单块读取的成本可能低于多块读取。`optimizer_index_cost_adj`参数允许您更准确地调整与索引读取相关的单块读取成本，以反映索引读取相对于全表扫描的真实成本。默认值 100 意味着单块读取是多块读取的 100%——因此它告诉优化器将索引读取的成本视为与多块 I/O 全表扫描的成本相同。当您将参数设置为 50 时，您是在告诉优化器单块 I/O（索引读取）的成本仅为多块 I/O 成本的一半。这会使优化器选择索引读取而非全表扫描。

请注意，准确的系统统计信息（`mbrc`、`mreadtim`、`sreadtim`等）会影响索引与全表扫描的使用。理想情况下，您应该收集工作负载系统统计信息，而不要调整`optimizer_index_cost_adj`参数。您也可以计算单块读取和多块读取的相对成本，并基于这些计算来设置`optimizer_index_cost_adj`参数值。然而，最佳策略是简单地在会话级别针对特定语句使用该参数，而不是在数据库级别使用。只需尝试该参数的不同级别，直到优化器开始使用索引。

您也可以使用更“科学”的方法来确定`optimizer_index_cost_adj`参数的正确设置，即将其设置为反映单块和多块读取“真实”差异的值。您可以简单地比较`db file sequential read`等待事件（代表单块 I/O）和`db file scattered read`等待事件（代表多块 I/O）的平均等待时间，从而得出`optimizer_index_cost_adj`参数的近似值。发出以下查询以查看两个等待事件的平均等待时间。

```sql
SQL> select event, average_wait from v$system_event
     where event like 'db file s%read';
EVENT                                                     AVERAGE_WAIT
---------------------------------------------------------------- ------------
db file sequential read                                              .91
db file scattered read                                              1.41

SQL>
```

根据此查询的输出，单块顺序读取大约花费多块（分散）读取所需时间的 75%。这表明`optimizer_cost_index_adj`参数应设置为 75 左右。然而，如前所述，不建议在数据库级别设置该参数——相反，应谨慎地在您希望强制使用索引的特定语句中使用此参数。

### 13-12. 启用查询优化器功能



### 问题

你升级了数据库，但你希望确保查询计划不会因新版本中的新优化器特性而发生变化。

### 解决方案

默认情况下，数据库会启用当前数据库版本的所有查询优化器特性。你可以通过设置 `optimizer_features_enable` 初始化参数来控制数据库中启用的优化器特性集。例如，如果你运行的是 Oracle 数据库 11g 第 2 版，优化器特性将设置为 11.2 版本，如下所示：

```sql
SQL> show parameter optimizer_features_enable

NAME                                 TYPE        VALUE
------------------------------------ ----------- ----------
optimizer_features_enable            string      11.2.0.1
SQL>
```

你可以通过将 `optimizer_features_enable` 参数设置为与其默认值（与数据库版本相同）不同的值，将数据库的优化器特性设置为更早的版本。例如，在 11.x 版本中，你可以这样做：

```sql
SQL> alter system set optimizer_features_enable='10.2.0.5';

System altered.
SQL>
```

现在你可以检查该参数的当前值：

```sql
SQL> show  parameter optimizer_features_enable

NAME                                 TYPE        VALUE
------------------------------------ ----------- --------------
optimizer_features_enable            string      10.2.0.5
SQL>
```

你可以将 `optimizer_features_enable` 参数设置为任何过去的主要版本或点版本，一直可以追溯到 Oracle 数据库 8.0 版本。

### 工作原理

将 `optimizer_features_enable` 参数设置为前一个数据库版本的值，可以确保在升级数据库时，优化器的行为与升级前完全一致。这是数据库管理员常用的一种策略，以确保查询计划在升级后不会突然恶化。一旦你对新的优化器特性有了更深入的了解，你就可以将 `optimizer_features_enable` 参数的值设置为升级后数据库版本相同的值。

当然，当你将 `optimizer_features_enable` 参数设置为低于当前版本的值时，你将无法利用任何新的优化器特性——但你也不会因执行计划的突然变化而感到意外。优化器特性在不同版本之间不会发生剧烈变化，但这完全取决于数据库版本。例如，在 11.1.0.6 版本中有六个主要的新优化器特性，在 10.2.0.2 版本中是没有的。这些特性包括增强的绑定变量窥探功能以及使用扩展统计信息来估算选择性的能力。不同的应用程序在引入新的优化器特性后会有不同的表现——这正是升级期间保留当前优化器特性集为你提供安全网的地方。你有机会在生产数据库中启用新特性之前，对它们进行全面测试并理解其影响。

“解决方案”部分中的示例展示了如何为整个数据库设置优化器特性级别。然而，你也可以仅在会话级别启用它（`alter session …`），以便在升级后测试执行计划的回退情况。你还可以通过提示（hint）指定版本号，从而使用特定版本的优化器特性来测试查询，如下所示（在一个 11.2 版本的数据库中）：

```sql
SQL>  select /*+ optimizer_features_enable ('11.1.0.6')  */ sum(sales) from sales
      order by product_id;
```

这个 `SELECT` 语句是在 11.2 版本数据库中执行的，但使用了 11.1 版本的优化器特性。

## 13-13. 防止数据库创建直方图

### 问题

你认为特定列上存在的直方图导致了次优的执行计划。你希望数据库不要在该列上使用任何直方图。

### 解决方案

如果你想阻止 Oracle 使用它在列上自动收集的直方图，你需要做两件事：

1.  通过执行 `DELETE_COLUMN_STATS` 过程来删除直方图：
    ```sql
    SQL> begin
      2  dbms_stats.delete_column_stats(ownname=>'SH',tabname=>'SALES',
      3  colname=>'PROD_ID',col_stat_type=>'HISTOGRAM');
      4  end;
      5  /

    PL/SQL procedure successfully completed.

    SQL>
    ```
2.  删除直方图后，通过执行以下 `SET_TABLE_PREFS` 过程，告诉 Oracle 不要在 `PROD_ID` 列上创建直方图：
    ```sql
    SQL> begin
      2  dbms_stats.set_table_prefs('SH','SALES','METHOD_OPT','FOR  ALL COLUMNS SIZE
         AUTO,
         FOR COLUMNS  SIZE 1 PROD_ID');
      3  end;
      4  /

    PL/SQL procedure successfully completed.

    SQL>
    ```

#### 工作原理

出于各种原因，数据库管理员有时会希望阻止优化器在列上使用直方图。如果列上已经存在直方图，你必须先将其移除，然后使用 `dbms_stats.set_table_prefs` 过程来阻止数据库在该列上创建直方图。在 Oracle 数据库 10g 版本中，你需要先删除直方图，冻结统计信息（使用 `lock_table_stats` 过程），然后手动收集表的统计信息，指定数据库不得为你删除了直方图的列收集统计信息。因为你锁定了统计信息，所以在执行 `dbms_stats.gather_table_stats` 过程以手动收集表的统计信息时，还必须指定 `force=true` 选项。正如你所见，11g 版本中的 `dbms_stats.set_table_prefs` 过程使事情变得简单多了。

在“解决方案”部分所示的命令中，`FOR ALL COLUMNS SIZE AUTO` 选项告诉数据库在 Oracle 认为存在数据倾斜的任何列上创建直方图。然而，`FOR COLUMNS SIZE 1 PROD_ID` 则告诉数据库不要在 `SH.SALES` 表的 `PROD_ID` 列上创建直方图。`SIZE` 列接受 1-254 的值，你指定的整数表示直方图中的桶数。告诉数据库只使用一个桶（`N`=1）意味着所有数据都将位于一个桶中——即，数据库不会在该列上创建直方图。

## 13-14. 在未使用绑定变量时提高性能

### 问题

由于各种原因，你的开发人员没有在代码中指定绑定变量。你注意到由于未使用绑定变量，导致了严重的闩锁争用和糟糕的响应时间。你想在无法更改现有代码的情况下，改善数据库在这种情况下的性能。

### 解决方案

如果你的应用程序不使用绑定变量，数据库中昂贵的硬解析将会增加。为避免这种情况，你需要将 `cursor_sharing` 初始化参数设置为非默认值。此参数的默认值为 `EXACT`。你可以将 `cursor_sharing` 参数设置为 `FORCE` 或 `SIMILAR`，以确定哪些 SQL 语句可以共享相同的游标。

以下是如何将 `cursor_sharing` 参数设置为 `force` 或 `similar`。

```sql
SQL> alter system set cursor_sharing=force;
SQL> alter system set cursor_sharing=similar;
```

将 `cursor_sharing` 参数设置为非默认值会产生若干影响，下一节将对此进行解释。



### 工作原理

编写 SQL 代码的最佳实践是使用绑定变量，这样 SQL 语句才是可共享的。在解析阶段，优化器会将当前 SQL 语句的文本与存储在共享池中已有语句的文本进行比较。数据库仅当一条 SQL 语句在所有方面——包括每个字符、空格甚至大小写——都与另一条语句完全匹配时，才认为它们是 `完全相同` 的。当你将 `cursor_sharing` 参数保留为其默认值 `EXACT` 时，Oracle 在重新执行使用绑定变量的 SQL 语句时会重用共享 SQL 区域。无需对新语句进行硬解析，因为共享池中已经存在一个已解析的版本。这条新语句可以使用一个现有的游标（因此称为共享游标），而无需创建自己的父游标。

如果代码没有使用绑定变量，但数据库正在解析的新 SQL 语句与共享池中一条先前解析过的语句在所有方面都相同，则该语句被视为与之前的语句 `相似`。

默认情况下，数据库只在 SQL 语句完全相同时共享游标，而在相似时不共享。如果应用程序使用字面值而非绑定变量，数据库将执行大量的硬解析，在繁忙的系统中，这可能会给共享池和游标缓存带来巨大压力。你可以通过将 `cursor_sharing` 参数设置为 `FORCE` 或 `SIMILAR`，使得当新语句与已有解析语句相似（但不完全相同时），数据库也能共享游标。将 `cursor_sharing` 参数设置为 `FORCE` 或 `SIMILAR` 允许数据库用系统生成的绑定变量替换字面值。这样做的目标是减少共享池中的父游标数量。即使应用程序不使用绑定变量也能共享游标，可以通过减少游标缓存（位于共享池中）中的父游标数量来缓解共享池的压力。若将 `cursor_sharing` 参数保留为其默认值，当数据库解析的语句与共享池中已有语句不完全相同时，它将执行硬解析。然而，如果你将该参数设置为 `FORCE` 或 `SIMILAR`，那么当在共享池中找到一个相似语句时，数据库将执行代价低得多的软解析。

#### 何时将 CURSOR_SHARING 设置为非默认值

理想情况下，你应该将 `cursor_sharing` 参数保留为其默认值 `EXACT`。但是，如果你的响应时间因大量的库缓存未命中而受影响，并且 SQL 语句没有使用绑定变量，那么可以考虑将 `cursor_sharing` 参数设置为 `FORCE` 或 `SIMILAR`。如果应用程序不使用绑定变量，你会束手无策——修复需要很长时间，与此同时，你将面对一个运行缓慢的数据库。在这种情况下，请立即更改 `cursor_sharing` 参数的默认设置。将 `cursor_sharing` 参数设置为非默认值实际上没有问题，只有一些微小的缺点，例如不支持星型转换等。

![images](img/square.jpg) 提示 Oracle 建议在 OLTP 环境中，将 `CURSOR_SHARING` 参数设置为 `FORCE`。

Oracle 建议，如果可能，应将 `cursor_sharing` 参数保留为其默认值 `EXACT`，并通过在代码中使用绑定变量而非字面值来实现 SQL 的可共享性。如果你确实因为共享池压力和门争用而决定将默认设置更改为 `SIMILAR` 或 `FORCE`，请注意这样做会带来一些性能影响。如果你将 `cursor_sharing` 参数设置为 `FORCE`，数据库将使用系统生成的绑定值，对语句的每次执行使用相同的执行计划，并为每个不同的 SQL 语句使用一个父游标和一个子游标。如果你将该参数设置为 `SIMILAR`，并且数据库为其生成绑定值的列上没有直方图，那么其行为与 `FORCE` 设置相同。另一方面，如果有直方图，`SIMILAR` 设置将导致基于绑定变量值的不同执行计划，因此每次执行都会有不同的子游标。在没有直方图的情况下，在 `SIMILAR` 设置下，数据库的行为就像没有使用绑定变量一样。

尽管 Oracle 历来建议将 `cursor_sharing` 参数设置为 `SIMILAR`，但在 Oracle Database 11g 中，除非你处于 DSS 环境，否则 Oracle 建议你将 `cursor_sharing` 参数设置为 `FORCE`，因为与将其设置为 `SIMILAR` 值相比，这能限制子游标的增长。




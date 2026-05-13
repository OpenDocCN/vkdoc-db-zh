# 9. 使用索引和视图优化脚本

处理大量数据时，优化 SQL 查询可以显著提高性能。这种优化可以通过索引和视图实现，两者都是强大的工具。索引通过允许数据库引擎更高效地定位行来加速数据检索，减少了扫描整张表的需要。而视图则通过存储可重用的 SQL 逻辑来简化复杂查询，提高了可读性和可维护性。此外，物化视图通过存储预计算的结果，减少了重新计算数据的需要。索引和视图共同帮助最大限度地减少查询执行时间、降低系统资源消耗并提高应用程序的整体响应速度，使其成为维护高效且可扩展的数据库系统的关键。本章将探讨如何利用这些功能来优化数据分析过程。

## 简介

索引是提高数据检索速度的数据库结构。它们的工作原理类似于书籍索引，允许数据库快速定位行而无需扫描整个表。索引是优化 SQL 查询的关键工具，它们提供了几个关键优势，使其对数据分析师和数据库管理员至关重要。更快的查询执行是使用索引的主要原因之一。通过允许数据库快速定位相关行而无需扫描整个表，索引显著加快了 `SELECT`、`WHERE`、`JOIN` 和 `ORDER BY` 等操作的速度。这对于复杂查询或涉及大型数据集的查询尤其有帮助，因为它减少了检索结果所需的时间。另一个优势是高效的数据检索。数据库可以利用索引直接访问所需的行，而不是执行可能消耗大量资源且速度缓慢的全表扫描，这使得查询更加高效。这种效率在处理包含数百万行的大型表时尤其有价值。使用索引与扫描整个表之间的性能差异可能非常显著。通过使用索引，数据分析师可以确保查询即使在处理海量数据时也能平稳快速地运行。

视图是显示 SQL 查询结果的虚拟表。虽然它本身不存储数据，但它简化了复杂查询并抽象了底层数据结构。视图是 SQL 中简化和优化数据分析工作流程的重要工具。视图的主要优势之一是能够简化复杂查询。通过将复杂的查询分解为可重用的组件，视图使编写、阅读和维护 SQL 代码变得更加容易。这节省了时间并减少了错误。此外，视图提供了数据抽象，允许分析师向最终用户隐藏底层数据库模式的复杂性。通过这种抽象，用户可以在不了解数据库结构的情况下与数据交互。另一个关键优势是安全性。视图可以限制对特定列或行的访问，确保敏感信息仅对授权用户可访问。最后，视图通过为常用查询提供标准化接口来促进一致性。这种一致性确保所有用户使用相同的数据定义进行工作，减少了差异并提高了分析的可靠性。

```
if (condVar > someVal) {console.log("xxx")}
```



## 理解索引

SQL 中的 `索引` 是一种数据结构，用于提升表的检索速度。它的工作方式类似于书籍的索引——帮助你快速找到特定信息，而无需扫描全部内容。SQL 索引能带来性能收益，包括更快的搜索查询、改进的排序与过滤，以及高效的表间连接。然而，它们也会消耗额外的磁盘空间，并且由于需要更新索引，可能会减慢 `INSERT`、`UPDATE` 和 `DELETE` 操作。

在数据分析中，虽然索引能提升查询性能，但它们也存在需要权衡的地方，可能影响数据存储和写入操作。它们会占用磁盘空间，并减慢 `INSERT`、`UPDATE` 和 `DELETE` 操作，这些操作在数据分析过程中相对于 `SELECT` 使用得较少。

索引会消耗额外的磁盘空间，因为它们存储了引用表行的独立数据结构。特别是对于大型数据集和多个索引，这会显著增加存储成本并减慢数据库备份速度。此外，索引会影响 `INSERT`、`UPDATE` 和 `DELETE` 操作。每次修改一行时，所有关联的索引也必须更新，这增加了开销并减慢了写入操作。例如，插入大量行需要更新所有索引，这会显著增加插入时间。因此，必须取得平衡。在读取频繁但写入较少的数据分析中，索引非常有益。然而，对于写入频繁的表，过多的索引会降低性能。一些优化措施，如仅索引特定数据子集的部分索引，以及在批量导入期间临时禁用索引，可以缓解这些性能缺点。

> 注意
>
> 有些数据库系统以不同方式处理索引。例如，SQL Server 中的聚集索引和 Oracle 中的索引组织表将实际表数据存储在索引本身之中。在这些情况下，索引和表数据之间的区别变得模糊，存储或性能影响可能略有不同。

### 创建索引的基本语法

创建索引的基本语法如下：

```
CREATE INDEX index_name ON table_name (column_name);
```

这里，`CREATE INDEX` 是创建索引的命令。`index_name` 是索引的名称。最好使用描述性的名称。`ON table_name` 指定了创建索引的表，`column_name` 是索引所基于的一个或多个列。

### PostgreSQL 中的索引类型

PostgreSQL 中的索引类型分为七种——B 树索引、唯一索引、哈希索引、广义倒排索引、广义搜索树（GiST）索引、部分索引和复合索引。本节将详细讨论每一种。

#### B 树索引

`B 树索引` 是 PostgreSQL 中最常见和默认的索引类型。它对于使用 `=` 运算符的相等比较以及涉及 `<`、`>`、`<=` 和 `>=` 等运算符的范围查询特别有效。此外，B 树索引能显著加快使用 `ORDER BY` 子句执行的排序操作。通过将数据组织成平衡树结构，它们允许从根节点快速遍历到叶节点，即使数据集增长也能确保一致的性能。这使得 B 树索引成为需要频繁搜索和有序数据检索的大型表的理想选择。

```
CREATE INDEX idx_salary ON employees (salary);
```

#### 唯一索引

`唯一索引` 确保特定列或列组合中的所有值都是唯一的，防止表中出现重复条目。PostgreSQL 在定义带有 `PRIMARY KEY` 或 `UNIQUE` 约束的列时会自动创建唯一索引。例如，主键列使用索引来维护数据完整性，确保每个值都是唯一的且非 `NULL`。同样，唯一约束强制值的唯一性，同时允许 `NULL` 值。唯一索引对于维护数据准确性、支持高效查找和优化查询性能至关重要，尤其是在验证用户输入或维护引用完整性时。

```
CREATE UNIQUE INDEX idx_unique_name ON employees (name);
```

#### 哈希索引

PostgreSQL 中的 `哈希索引` 针对使用 `=` 运算符的简单相等比较进行了优化。与支持相等和范围查询的 B 树索引不同，哈希索引专为精确匹配设计，因此对此类查询更快。这种效率源于哈希算法，它将数据值映射到固定位置，实现快速查找。然而，哈希索引不适用于范围查询或排序，因为它们不维护数据顺序。虽然它们在精确匹配方面提供更快的性能，但使用时需要更加谨慎，因为它们不支持 `<`、`>` 或 `ORDER BY` 等高级查询类型。

```
CREATE INDEX idx_hash_department ON employees USING HASH (department);
```

#### 广义倒排索引（GIN）

`广义倒排索引`（GIN）设计用于索引复杂数据结构，如 PostgreSQL 中的数组、JSON 字段和全文搜索。与传统的 B 树索引不同，GIN 通过将每个不同元素映射到相应的行条目，高效处理存储在单个列中的多个值。

```
CREATE INDEX idx_gin_name ON employees USING GIN (to_tsvector('english', name));
```

此索引能够快速查找姓名列中的单词，显著提高了大型数据集中的搜索性能。

#### 广义搜索树（GiST）索引

`广义搜索树`（GiST）索引是一种灵活、平衡树的索引机制，用于复杂数据类型，如几何形状、网络地址和全文搜索。与 B 树不同，GiST 索引允许自定义搜索策略，使其成为空间查询和全文索引所必需的。

```
CREATE INDEX idx_gist_salary ON employees USING GiST (salary);
```

这使得 PostgreSQL 能够快速定位特定薪资范围内的员工，提高了基于范围的查询性能。此外，GiST 索引支持最近邻搜索，这使得它们在基于距离的查询常见的地理空间应用中很有价值。

#### 部分索引

`部分索引` 是一种优化的索引，只存储表行的子集，减少存储开销并提高查询性能。PostgreSQL 通过仅索引满足预定义条件的记录，避免维护不必要的索引条目，从而加快对经常在索引条件上进行筛选的查询的查找速度。

```
CREATE INDEX idx_high_salary ON employees (salary) WHERE salary > 50000;
```

此索引对于频繁检索收入超过 50,000 美元的员工的查询非常有用，使其比扫描整个表快得多。部分索引在大型数据集中特别有效，因为在这些情况下，完整索引将是浪费且不必要的。



### 复合索引

一个 `复合索引` 会同时索引多个列，非常适合那些按多个属性进行过滤或排序的查询。与其为每一列创建单独的索引，PostgreSQL 使用复合索引来加速那些在过滤条件中涉及索引列组合的查询。

```sql
CREATE INDEX idx_name_department ON employees (name, department);
```

此索引在按特定部门搜索员工时能提升性能，因为它允许 PostgreSQL 同时基于这两个列高效地导航数据。复合索引在多列过滤、排序和 `JOIN` 操作中尤其有用。

> **注意**
> 对于数据分析任务，最适用的 PostgreSQL 索引是 B-tree、GIN 和 GiST，因为它们在处理大型数据集、复杂查询和特定数据类型方面效率很高。

1.  **B-tree 索引：** 适用于等值比较 (`=`)、范围查询 (`<`, `>`, `<=`, `>=`) 和排序 (`ORDER BY`)。其平衡结构确保了性能稳定，对于过滤和聚合大型数据集至关重要。
2.  **GIN 索引：** 高效索引复合数据类型，如数组、JSON 和全文搜索。它对于涉及半结构化数据和在嵌套结构中快速查找的高级数据分析任务特别有用。
3.  **GiST 索引：** 最适合多维数据和空间分析。它支持几何数据类型、文本搜索和相似性搜索，对于地理信息系统 (GIS) 和基于相似性的数据分析很有价值。
4.  **BRIN 索引：** 针对非常大的数据集进行了优化，尤其是那些具有自然顺序的数据，如时间序列数据。通过为每个数据块存储最少的信息，BRIN 减少了存储空间并加速了扫描大范围连续数据的查询。

其他索引类型，如 Hash，更为专用，在一般数据分析场景中较少使用。唯一索引对数据完整性至关重要，但不会直接影响分析性能。

### 删除索引

如果不再需要某个索引，可以使用 `DROP INDEX` 命令将其移除：

```sql
DROP INDEX idx_department;
```

### 使用 EXPLAIN 检查索引使用情况

当查询 PostgreSQL 时，可以使用 `EXPLAIN` 关键字来查看是否使用了索引：

```sql
EXPLAIN SELECT * FROM employees WHERE department = 'IT';
```

查找输出中的 `Index Scan`，这表明正在使用索引。如果看到 `Seq Scan`，则表示 PostgreSQL 正在扫描整张表（速度较慢）。

## 何时使用及何时避免使用索引

数据库索引是优化查询性能的强大工具，但其使用应具有策略性。索引在处理大型、频繁查询的表时非常有益，尤其是在特定列上进行搜索、过滤、排序或连接时。然而，过度使用索引可能适得其反。因为索引会消耗额外的磁盘空间并减慢 `INSERT`、`UPDATE` 和 `DELETE` 操作，所以避免过度索引至关重要。平衡的方法是关键，需要根据查询模式和数据修改频率谨慎选择要索引的列。关于何时使用以及何时避免使用索引的概述说明见表 9-1。

**表 9-1：何时使用及何时避免使用索引**

| 何时使用索引 | 避免过度索引的原因 |
| --- | --- |
| 表数据量大且频繁被查询 | 索引消耗磁盘空间（过度索引会增加存储成本） |
| 特定列经常被搜索、过滤或排序 | 索引会减慢 `INSERT`、`UPDATE` 和 `DELETE` 操作 |
| 查询涉及大表之间的连接 | 维护多个索引会在数据修改时增加开销 |
| 查询频繁使用 `WHERE`、`ORDER BY` 或 `GROUP BY` | 每次修改数据时索引都需要更新，会减慢写入速度 |
| 对大文本字段执行全文搜索（推荐使用 GIN） | 过多索引可能会使查询规划器混淆，导致执行计划次优 |
| 表有外键或需要强制唯一性 | 索引对于小表或低选择性列可能无法提供好处 |
| 查询按 JSON 字段或数组元素过滤（推荐使用 GIN 或 GiST） | 如果查询很少使用索引列，索引可能会降低性能 |
| 时间序列或顺序数据需要高效的范围查询（推荐使用 BRIN） | 部分索引或复合索引在某些场景下可能更高效 |

## 索引在数据分析任务中的作用

索引在数据库中充当数据检索的指南，有助于加速数据分析。这使得数据库可以跳过全表扫描并快速定位特定数据点。特别是在处理大型数据集时，穷举搜索会慢得无法承受。通过优化过滤和排序操作，索引使分析人员能够根据特定标准快速缩小数据范围或进行有意义的排列。这对于识别数据中的趋势、异常值或模式等任务至关重要。

索引还通过促进多个表之间的连接来提升数据仓库和分析性能。索引有助于更快地进行连接，从而获得来自不同来源数据的全景视图。索引不仅支持范围查询，还能加速总和、平均值和计数的计算，这对于汇总和理解数据分布至关重要。索引使数据分析师能够更快地进行复杂调查、提取有价值的见解并做出数据驱动的决策。尽管在存储空间和写入性能方面存在权衡，但对于涉及大量读写操作的工作负载而言，其益处远大于缺点。

## 使用 EXPLAIN 审查查询执行

`EXPLAIN` 有助于分析 PostgreSQL 计划如何执行查询，这在优化大型数据集上的分析查询时至关重要。

```sql
EXPLAIN query_statement;
```

这里，`query_statement` 是待分析的 SQL 查询，它提供了数据库引擎打算如何执行该 SQL 语句的见解。简而言之，PostgreSQL 分析并解析由 `EXPLAIN query_statement` 提供的 SQL 语句。它不会实际执行查询并返回结果，而是生成一个执行计划。该计划概述了执行查询的步骤。作为输出，此计划显示了估计的成本和操作顺序。

> **注意**
> 关于扫描类型，`EXPLAIN` 可以揭示数据库是执行顺序扫描（读取表的每一行），还是使用索引通过索引扫描进行高效检索、仅使用索引数据的索引 only 扫描，或使用位图索引的位图扫描。当查询涉及 `JOIN` 时，`EXPLAIN` 会详细说明所使用的 `JOIN` 方法，例如迭代匹配的 `NestedLoop`、合并已排序行的 `MERGE JOIN`，或使用哈希表探测的 `Hash Join`。此外，输出清晰地显示了应用于数据的过滤条件，因此您可以理解数据库是如何缩小结果范围的。



## 使用 `EXPLAIN ANALYZE` 进行性能测量

`EXPLAIN ANALYZE` 会执行查询并提供实际的运行时统计数据。在添加索引后测试性能改进时，最好使用此工具。

```
EXPLAIN ANALYZE query_statement;
```

此处，`query_statement` 是需要用实时指标进行分析的 SQL 查询。`EXPLAIN ANALYZE` 是 PostgreSQL 中用于数据库性能分析的工具。它超越了简单 `EXPLAIN` 提供的估算成本，通过实际执行查询并收集实时指标来进行分析。表 9-2 指出了 `EXPLAIN` 和 `EXPLAIN ANALYZE` 的关键区别。

表 9-2：`EXPLAIN` 与 `EXPLAIN ANALYZE` 的关键区别

| 特性 | `EXPLAIN` | `EXPLAIN ANALYZE` |
| --- | --- | --- |
| 实际执行 | 生成执行计划（不执行） | 执行查询；收集运行时统计信息 |
| 实时指标 | 提供估算成本 | 提供实际执行时间、行数 |

除了估算成本和提供计划结构外，`EXPLAIN ANALYZE` 还提供运行时指标以增强分析。在 PostgreSQL 中，规划器的成本估算表示为两个数字：启动成本和总成本。这些是基于估算的 I/O、CPU 使用率和行数的无单位值，用于比较不同的执行计划，而不是反映实际时间。`EXPLAIN ANALYZE` 揭示了每个节点的实际执行时间（以毫秒为单位），这直接突出了性能瓶颈。实际行数允许精确比较规划器的估算结果，揭示其准确性。在嵌套循环连接中，循环次数指定了内层循环执行的频率。此外，根据具体查询和 PostgreSQL 版本，可能还包括其他运行时统计信息。这可以提供查询实际执行情况的概览。

## 第一个故事：高尔夫表现数据分析

玛蒂娜是一家高尔夫俱乐部的数据分析师，负责评估球员表现和优化赛事运营。她管理着一个跟踪球员成绩、球场详情和比赛结果的数据库。随着更多比赛的举行，表 9-3、9-4 和 9-5 所示的数据表不断增长，她希望确保查询能高效运行。借助索引和 `EXPLAIN ANALYZE` 等性能分析工具，玛蒂娜旨在加快数据检索速度，从而做出更好的决策。

表 9-5：成绩表

| score_id | player_id | tournament_id | round_number | Strokes |
| --- | --- | --- | --- | --- |
| 1 | 1 | 1 | 1 | 70 |
| 2 | 1 | 1 | 2 | 72 |
| 3 | 2 | 1 | 1 | 68 |
| 4 | 2 | 1 | 2 | 70 |
| 5 | 3 | 2 | 1 | 71 |
| 6 | 3 | 2 | 2 | 69 |
| 7 | 4 | 2 | 1 | 70 |
| 8 | 4 | 2 | 2 | 68 |
| 9 | 5 | 3 | 1 | 73 |
| 10 | 5 | 3 | 2 | 72 |

表 9-4：赛事表

| tournament_id | Name | Location | Year |
| --- | --- | --- | --- |
| 1 | Masters Tournament | Augusta National Golf Club | 2023 |
| 2 | US Open | Pebble Beach | 2023 |
| 3 | The Open Championship | St Andrews | 2023 |
| 4 | PGA Championship | Oak Hill | 2023 |
| 5 | Ryder Cup | Marco Simone | 2023 |

表 9-3：球员表

| player_id | Name | Age | Nationality |
| --- | --- | --- | --- |
| 1 | Tiger Woods | 48 | USA |
| 2 | Rory McIlroy | 34 | Northern Ireland |
| 3 | Jon Rahm | 29 | Spain |
| 4 | Brooks Koepka | 33 | USA |
| 5 | Dustin Johnson | 40 | USA |
| 6 | Collin Morikawa | 27 | USA |
| 7 | Justin Thomas | 31 | USA |
| 8 | Phil Mickelson | 53 | USA |
| 9 | Jordan Spieth | 31 | USA |
| 10 | Viktor Hovland | 26 | Norway |

玛蒂娜在经常用于筛选和连接的列上创建索引，以提高查询性能。

```
CREATE INDEX idx_player_id ON scores(player_id);
CREATE INDEX idx_tournament_id ON scores(tournament_id);
CREATE INDEX idx_name ON players(name);
```

玛蒂娜选择用于索引的列经常用于筛选、连接和排序。这些是索引提高数据分析效率的主要用例。例如，`scores(player_id)` 列上的索引可以加速按球员 ID 筛选或连接的查询，这在检索特定球员成绩时很常见。由于 `player_id` 是 `Scores` 表中的外键，对其进行索引可以提高与 `Players` 表的连接性能。要按赛事筛选成绩，`scores(tournament_id)` 列对于分析特定赛事的球员表现至关重要。对 `tournament_id` 进行索引可以加速对来自 `Tournaments` 表的成绩进行分组、筛选或连接的查询。对于 `players(name)` 列，`name` 上的索引支持按姓名直接查找球员，这在分析单个球员表现时很常见。尽管姓名并非唯一，但对此列进行索引仍然能提高搜索效率。

选择这些索引是为了在频繁的数据分析任务中平衡性能提升，同时最小化因过度索引带来的开销。

为了回答以下问题，玛蒂娜打算使用这些数据和 SQL 查询：

*   谁在大师赛的任何一轮中打出了低于 70 杆的成绩？
*   计算每位球员在美国公开赛中的平均杆数。
*   找出在英国公开赛中总杆数最低的前三名球员。
*   检索泰格·伍兹的所有成绩以进行表现分析。

玛蒂娜编写了以下查询来寻找在大师赛任何一轮中打出低于 70 杆成绩的球员。玛蒂娜使用 `EXPLAIN` 来显示索引是否用于 `JOIN` 和筛选。为了获得更好的性能，索引扫描应取代顺序扫描。此查询受益于 `player_id` 和 `tournament_id` 上的索引，因为它同时对两者进行了筛选。



### 使用 EXPLAIN 分析查询性能

```
EXPLAIN SELECT p.name, s.strokes
FROM scores s
JOIN players p ON s.player_id = p.player_id
JOIN tournaments t ON s.tournament_id = t.tournament_id
WHERE t.name = 'Masters Tournament' AND s.strokes < 70;
```

该查询检索大师赛中杆数低于 70 的球员姓名和成绩。它连接了三个表：`Scores`、`Players` 和 `Tournaments`。`JOIN` 条件将 `scores.player_id` 与 `players.player_id` 以及 `scores.tournament_id` 与 `tournaments.tournament_id` 关联起来。`WHERE` 子句将结果过滤为仅包含杆数小于 70 且锦标赛名称为 `Masters Tournament` 的记录。

由于使用了 `EXPLAIN` 命令，执行此查询将显示如下结果：

```
QUERY PLAN

Nested Loop  (cost=13.92..47.53 rows=4 width=122)
->  Hash Join  (cost=13.78..46.54 rows=4 width=8)
Hash Cond: (s.tournament_id = t.tournament_id)
->  Seq Scan on scores s  (cost=0.00..31.25 rows=567 width=12)
Filter: (strokes < 70)
->  Hash  (cost=13.75..13.75 rows=2 width=4)
->  Seq Scan on tournaments t  (cost=0.00..13.75 rows=2 width=4)
Filter: ((name)::text = 'Masters Tournament'::text)
->  Index Scan using players_pkey on players p  (cost=0.15..0.25 rows=1 width=122)
Index Cond: (player_id = s.player_id)
(10 rows)
```

这里，`EXPLAIN` 的输出展示了 PostgreSQL 如何执行查询。它从对 `scores` 表的顺序扫描开始，过滤杆数 `< 70` 的行，成本为 31.25。然后，对 `tournaments` 表进行哈希扫描，过滤出 “`Masters Tournament`”，成本为 13.75。一次哈希连接将 `scores` 和 `tournaments` 连接起来，成本为 46.54。最后，一个嵌套循环通过使用 `players_pkey` 的索引扫描将这个结果与 `Players` 表连接，成本为 0.25。对 `players.player_id` 的索引提高了性能，但对 `scores` 和 `tournaments` 的顺序扫描表明还有更多优化策略。

### 使用 EXPLAIN ANALYZE 分析聚合查询

为了计算每位球员在美国公开赛中的平均杆数，Martina 编写了以下查询：

```
EXPLAIN ANALYZE SELECT p.name, AVG(s.strokes) AS avg_strokes
FROM scores s
JOIN players p ON s.player_id = p.player_id
JOIN tournaments t ON s.tournament_id = t.tournament_id
WHERE t.name = 'US Open'
GROUP BY p.name;
```

该查询计算参加了美国公开赛的球员的平均杆数。它使用 `player_id` 和 `tournament_id` 连接 `scores`、`players` 和 `tournaments` 表。`WHERE` 子句按美国公开赛筛选成绩，`GROUP BY` 子句按球员姓名对结果进行分组。`AVG()` 函数计算每位球员的平均杆数。此查询在数据分析和商业智能中很常见，用于评估球员在特定锦标赛中的表现。适当的索引提高了连接效率并减少了查询执行时间，尤其是在处理包含多个锦标赛和球员的大型数据集时。

```
QUERY PLAN

GroupAggregate  (cost=47.62..47.84 rows=11 width=150) (actual time=0.137..0.156 rows=2 loops=1)
Group Key: p.name
->  Sort  (cost=47.62..47.65 rows=11 width=122) (actual time=0.124..0.138 rows=4 loops=1)
Sort Key: p.name
Sort Method: quicksort  Memory: 25kB
->  Nested Loop  (cost=13.92..47.43 rows=11 width=122) (actual time=0.057..0.095 rows=4 loops=1)
->  Hash Join  (cost=13.78..45.30 rows=11 width=8) (actual time=0.041..0.061 rows=4 loops=1)
Hash Cond: (s.tournament_id = t.tournament_id)
->  Seq Scan on scores s  (cost=0.00..27.00 rows=1700 width=12) (actual time=0.004..0.013 rows=10 loops=1)
->  Hash  (cost=13.75..13.75 rows=2 width=4) (actual time=0.013..0.015 rows=1 loops=1)
Buckets: 1024  Batches: 1  Memory Usage: 9kB
->  Seq Scan on tournaments t  (cost=0.00..13.75 rows=2 width=4) (actual time=0.004..0.007 rows=1 loops=1)
Filter: ((name)::text = 'US Open'::text)
Rows Removed by Filter: 4
->  Index Scan using players_pkey on players p  (cost=0.15..0.19 rows=1 width=122) (actual time=0.004..0.004 rows=1 loops=4)
Index Cond: (player_id = s.player_id)
Planning Time: 0.179 ms
Execution Time: 0.227 ms
(18 rows)
```

`EXPLAIN ANALYZE` 的输出显示了查询的执行计划及实际执行时间。该过程从对 `Scores` 表的顺序扫描开始，用时 0.013 毫秒检索出十行数据。对 `Tournaments` 表的顺序扫描过滤出 “US Open”，用时 0.007 毫秒检索出一行。一次哈希连接在 0.061 毫秒内连接了 `scores` 和 `tournaments`，随后对每个匹配的行使用 `players_pkey` 进行索引扫描，这非常高效（成本：0.19）。结果使用快速排序进行排序（内存：25kB），然后 `GroupAggregate` 计算平均杆数。总执行时间为 0.227 毫秒，考虑到查询的简单性、处理的行数少以及 I/O 最小（尤其是与更复杂的查询或未优化的连接相比），这被认为是高效的。

### 查找总成绩最低的球员

为了找到公开锦标赛中总成绩最低的前三名球员，以下查询计算参加了 “The Open Championship” 锦标赛的球员的总杆数。它使用 `player_id` 和 `tournament_id` 连接 `Scores`、`Players` 和 `Tournaments` 表。`WHERE` 子句为 “The Open Championship” 筛选成绩，`GROUP BY` 子句按球员姓名对结果进行分组。`SUM()` 函数计算每位球员的总杆数。`ORDER BY total_strokes ASC` 将结果按升序排列，`LIMIT 3` 返回总杆数最少的三名球员。

```
EXPLAIN SELECT p.name, SUM(s.strokes) AS total_strokes
FROM scores s
JOIN players p ON s.player_id = p.player_id
JOIN tournaments t ON s.tournament_id = t.tournament_id
WHERE t.name = 'The Open Championship'
GROUP BY p.name
ORDER BY total_strokes ASC
LIMIT 3;
```

`EXPLAIN` 输出揭示了查询的执行计划。该过程从对 `Scores` 表的顺序扫描开始，检索 1700 行，然后对 `Tournaments` 表进行顺序扫描，过滤出 “The Open Championship”。一次哈希连接合并这些表。接着，使用 `players_pkey` 进行索引扫描以高效地检索球员姓名。结果被排序两次，首先按球员姓名使用 `Sort`，然后按总杆数使用另一个 `Sort`（成本=47.96）。`Limit` 步骤返回前三名球员。如图所示，查询计划表明由于适当的索引和排序技术，性能高效：

```
QUERY PLAN

Limit  (cost=47.96..47.96 rows=3 width=126)
->  Sort  (cost=47.96..47.98 rows=11 width=126)
Sort Key: (sum(s.strokes))
->  GroupAggregate  (cost=47.62..47.81 rows=11 width=126)
Group Key: p.name
->  Sort  (cost=47.62..47.65 rows=11 width=122)
Sort Key: p.name
->  Nested Loop  (cost=13.92..47.43 rows=11 width=122)
->  Hash Join  (cost=13.78..45.30 rows=11 width=8)
Hash Cond: (s.tournament_id = t.tournament_id)
->  Seq Scan on scores s  (cost=0.00..27.00 rows=1700 width=12)
->  Hash  (cost=13.75..13.75 rows=2 width=4)
->  Seq Scan on tournaments t  (cost=0.00..13.75 rows=2 width=4)
Filter: ((name)::text = 'The Open Championship'::text)
->  Index Scan using players_pkey on players p  (cost=0.15..0.19 rows=1 width=122)
Index Cond: (player_id = s.player_id)
(16 rows)
```

### 检索单个球员的所有成绩

为了检索 Tiger Woods 的所有成绩以进行表现分析，以下查询检索 Tiger Woods 的锦标赛名称、轮次数和杆数。它使用 `player_id` 和 `tournament_id` 连接 `Scores`、`Players` 和 `Tournaments` 表。`WHERE` 子句过滤 `Players` 表，仅包含姓名与 “Tiger Woods” 匹配的行。查询输出来自 `Scores` 表的相关成绩和轮次数，以及来自 `Tournaments` 表的锦标赛名称。此查询对于分析球员在不同锦标赛中的表现非常有用，帮助教练、分析师和球迷评估稳定性并识别跨多轮比赛的表现趋势。



```
EXPLAIN ANALYZE SELECT t.name, s.round_number, s.strokes
FROM scores s
JOIN players p ON s.player_id = p.player_id
JOIN tournaments t ON s.tournament_id = t.tournament_id
WHERE p.name = 'Tiger Woods';
```

`EXPLAIN ANALYZE` 的输出展示了查询的执行步骤。首先，对 `Scores` 表进行顺序扫描，检索出 1,700 行。接着，对 `Players` 表进行顺序扫描，通过过滤 9 行找到名为“Tiger Woods”的球员。然后，通过哈希连接合并两个表，匹配 `player_id` 值。随后，使用 `tournaments_pkey` 进行索引扫描，根据 `tournament_id` 检索赛事名称，该操作执行了两次。查询总执行时间为 0.107 毫秒，这得益于索引和有限的匹配行数，表明性能很快。

```
QUERY PLAN

Nested Loop  (cost=14.55..47.85 rows=10 width=126) (actual time=0.041..0.082 rows=2 loops=1)
->  Hash Join  (cost=14.40..45.92 rows=10 width=12) (actual time=0.027..0.059 rows=2 loops=1)
    Hash Cond: (s.player_id = p.player_id)
    ->  Seq Scan on scores s  (cost=0.00..27.00 rows=1700 width=16) (actual time=0.005..0.014 rows=10 loops=1)
    ->  Hash  (cost=14.38..14.38 rows=2 width=4) (actual time=0.016..0.018 rows=1 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Seq Scan on players p  (cost=0.00..14.38 rows=2 width=4) (actual time=0.004..0.007 rows=1 loops=1)
            Filter: ((name)::text = 'Tiger Woods'::text)
            Rows Removed by Filter: 9
->  Index Scan using tournaments_pkey on tournaments t  (cost=0.15..0.19 rows=1 width=122) (actual time=0.006..0.006 rows=1 loops=2)
    Index Cond: (tournament_id = s.tournament_id)
Planning Time: 0.101 ms
Execution Time: 0.107 ms
(13 rows)
```

对于按球员姓名筛选、连接大型表以及聚合分数等数据分析任务，使用索引能显著提升速度。使用 `EXPLAIN` 和 `EXPLAIN ANALYZE` 有助于确认 PostgreSQL 是否利用了索引来获得更好的性能。这种优化对于商业智能至关重要，能够基于球员表现和赛事结果实现更快的决策。

Martina 出于多种原因，希望在此时删除一个索引。首先，如果查询不再使用该索引，它会不必要地消耗存储空间，并在未来的 `INSERT`、`UPDATE` 和 `DELETE` 等写入操作中降低速度。其次，如果数据分布随时间发生变化，索引可能会变得低效，需要重新创建。最后，删除冗余索引可以提升数据库的整体性能，尤其是在为重叠目的维护多个索引的情况下。要在 PostgreSQL 中删除索引，可使用 `DROP INDEX` 语句。当 Martina 想要删除名为 `players_pkey` 的索引时，她可以按如下方式操作：

```
DROP INDEX players_pkey;
```

在这个故事中，重点在于如何优化查询执行，以及如何评估这种优化。如前所述，在处理大型数据表和海量数据时，索引具有积极效果，但这样的表很难在本书的篇幅内容纳！因此，本故事的目的仅限于深入洞察使用 `INDEX` 进行优化的过程，并使用 `EXPLAIN` 或 `EXPLAIN ANALYZE` 评估其有效性的过程。

## 理解 SQL 视图

在 PostgreSQL 中，*视图* 本质上是一个虚拟表，源自存储查询的结果。它本身不存储数据；而是提供了一个来自一个或多个表的底层数据的定制化、简化的视图。

PostgreSQL 中的 SQL 视图提供了一种强大的方式来简化数据库交互。视图的概念可以描述为一个作为存储查询结果而创建的虚拟表。视图的一个关键优势是它们通过封装复杂查询来简化它们，使其可以复用；你无需反复编写复杂的查询，而是可以像使用标准表一样使用视图。视图的使用通过限制对特定列或行的访问来增强数据安全性，允许用户只看到他们所需的信息。此外，视图提供了数据抽象，即使在表结构发生变更时也能保护应用程序免受其影响。视图还通过将复杂查询简化为更小、更易管理的组成部分来提高可读性。

### SQL 视图的基本语法

SQL 视图的基本语法如下：

```
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table1
WHERE condition;
```

这里，`CREATE VIEW` 创建了一个称为视图的虚拟表。它存储了一个 SQL 查询，该查询在调用时动态生成数据，而不存储数据本身。语法以 `CREATE VIEW` 开头，后跟所需的视图名称。`AS` 关键字引入了定义视图的查询。`SELECT` 语句指定了要包含的列，使用 `FROM` 指定了源表，并使用 `WHERE` 指定了可选的过滤条件。

### PostgreSQL 中的视图类型

在 PostgreSQL 中实现视图有两种方式——使用标准视图和物化视图。使用 `CREATE VIEW` 创建的标准视图充当虚拟表，呈现存储查询的结果而不实际存储数据。这使它们始终保持最新，能够反映底层表的任何更改。相比之下，使用 `CREATE MATERIALIZED VIEW` 创建的物化视图将查询结果存储为物理表。这创建了创建或刷新时数据的快照。

这种存储机制显著提升了频繁访问的复杂查询的性能，但需要手动或计划的数据刷新。在实时数据准确性和查询性能优化之间权衡，决定了在标准视图和物化视图之间的选择（参见表 9-6）。

表 9-6

标准视图与物化视图的关键区别

| 特性 | 标准视图 | 物化视图 |
| --- | --- | --- |
| 数据存储 | 虚拟；不存储物理数据 | 物理表；存储数据 |
| 数据时效性 | 始终最新 | 快照；需要手动刷新 |
| 性能 | 查询性能取决于底层表 | 提升复杂查询的性能 |
| 使用场景 | 简化、安全、抽象 | 性能优化、报表 |

### 视图在数据分析任务中的作用

视图可以在 SQL 数据分析中发挥关键作用，作为一层抽象，显著提升分析过程的效率。它们通过呈现专注的、预聚合和过滤后的表示来简化复杂的数据模型，降低查询复杂度并提高可读性。通过封装复杂的逻辑，视图充当可复用的分析构建块，确保一致性，并通过预计算的摘要和分割的数据子集来加速分析。除了加快数据处理速度，这还提高了准确性和生产力，让分析人员能够专注于提取洞见，而不是检索数据。此外，视图通过限制对敏感信息的访问来增强数据安全性，使其成为精简和安全分析的重要工具。



## 第二个故事：赛车数据分析

娜塔莉是 Speed Track Racing 的一名数据分析师，需要分析近期比赛的赛事表现数据。如表 9-7、9-8 和 9-9 所示，数据库包含三个主要表：`Drivers`（车手）、`Races`（比赛）和`Results`（结果）。通过创建 SQL 视图并利用这些表中的数据，娜塔莉希望优化重复性查询，提高分析效率。

**表 9-9**
**结果表**

| `result_id` | `driver_id` | `race_id` | `position` | `lap_time_sec` |
| --- | --- | --- | --- | --- |
| 1 | 1 | 101 | 1 | 90.23 |
| 2 | 2 | 101 | 2 | 90.45 |
| 3 | 3 | 101 | 3 | 91.12 |
| 4 | 4 | 102 | 1 | 87.32 |
| 5 | 5 | 102 | 2 | 87.78 |
| 6 | 6 | 103 | 1 | 89.01 |
| 7 | 7 | 103 | 2 | 89.23 |
| 8 | 8 | 104 | 1 | 88.45 |
| 9 | 9 | 104 | 3 | 89.67 |
| 10 | 10 | 105 | 1 | 92.89 |

**表 9-8**
**比赛表**

| `race_id` | `Location` | `Date` |
| --- | --- | --- |
| 101 | 银石赛道 | 2023-07-09 |
| 102 | 蒙扎赛道 | 2023-09-03 |
| 103 | 斯帕-弗朗科尔尚赛道 | 2023-07-30 |
| 104 | 铃鹿赛道 | 2023-09-24 |
| 105 | 蒙特卡洛赛道 | 2023-05-28 |
| 106 | 奥斯汀赛道 | 2023-10-22 |
| 107 | 新加坡赛道 | 2023-09-17 |
| 108 | 因特拉格斯赛道 | 2023-11-05 |
| 109 | 匈格罗宁赛道 | 2023-07-23 |
| 110 | 阿尔伯特公园赛道 | 2023-04-02 |

**表 9-7**
**车手表**

| `driver_id` | `Name` | `Team` |
| --- | --- | --- |
| 1 | 刘易斯·汉密尔顿 | 梅赛德斯 |
| 2 | 马克斯·维斯塔潘 | 红牛 |
| 3 | 夏尔·勒克莱尔 | 法拉利 |
| 4 | 塞尔吉奥·佩雷兹 | 红牛 |
| 5 | 卡洛斯·塞恩斯 | 法拉利 |
| 6 | 乔治·拉塞尔 | 梅赛德斯 |
| 7 | 兰多·诺里斯 | 迈凯伦 |
| 8 | 费尔南多·阿隆索 | 阿斯顿·马丁 |
| 9 | 皮埃尔·加斯利 | 阿尔派 |
| 10 | 埃斯特班·奥康 | 阿尔派 |

娜塔莉旨在基于从近期比赛中收集的赛事表现数据，回答以下问题。

*   在各场比赛中获得第一名的顶尖表现者是谁？
*   每位车手在所有比赛中的平均圈速是多少？
*   每支车队获得了多少次第一名？
*   每场比赛的获胜者是谁，以及比赛地点和日期？
*   谁在每场比赛中创造了最快单圈？

### 识别顶尖表现者

娜塔莉编写了以下查询，以识别在不同赛道上持续表现优异的车手。使用视图可以简化此任务，视图存储了连接`Drivers`、`Races`和`Results`表的逻辑。分析师无需每次重写复杂的`JOIN`语句，可以直接查询`top_performers`视图。这提高了可读性，减少了错误，并确保了一致性。

```sql
CREATE VIEW top_performers AS
SELECT d.name, r.location, res.position
FROM results res
JOIN drivers d ON res.driver_id = d.driver_id
JOIN races r ON res.race_id = r.race_id
WHERE res.position = 1;
```

此 SQL 查询创建了一个名为`top_performers`的视图，用于显示比赛获胜者。该查询选择了车手姓名、比赛地点和完赛名次，并连接了`Results`、`Drivers`和`Races`这三个表。它使用`driver_id`和`race_id`作为键来连接这些表，并筛选出仅包含第一名的记录（`position = 1`）。基于此筛选条件，视图将显示每位获胜车手的姓名、他们获胜的地点及其名次。表 9-10 说明了此查询中`SELECT`语句的执行结果。

**表 9-10**
**比赛获胜者**

| `Name` | `Location` | `Position` |
| --- | --- | --- |
| 刘易斯·汉密尔顿 | 银石赛道 | 1 |
| 塞尔吉奥·佩雷兹 | 蒙扎赛道 | 1 |
| 乔治·拉塞尔 | 斯帕-弗朗科尔尚赛道 | 1 |
| 费尔南多·阿隆索 | 铃鹿赛道 | 1 |
| 埃斯特班·奥康 | 蒙特卡洛赛道 | 1 |

### 计算平均圈速

娜塔莉编写了以下查询，以找出每位车手在所有比赛中的平均圈速，从而回答关于车手一致性的问题。这简化了报告流程，特别是在多个仪表板或报告中需要此指标时。视图还能确保只有一个真实数据源，保持所有分析中计算的一致性。

```sql
CREATE VIEW avg_lap_time AS
SELECT d.name, AVG(res.lap_time_sec) AS avg_time
FROM results res
JOIN drivers d ON res.driver_id = d.driver_id
GROUP BY d.name;
```

此 SQL 查询创建了一个名为`avg_lap_time`的视图，用于计算每位车手的平均圈速。它从`Drivers`表中选择车手姓名，并从`Results`表中计算以秒为单位的平均圈速。该查询使用`driver_id`作为公共键连接这两个表，从而将圈速与相应的车手匹配。结果按车手姓名分组，这意味着视图将显示每位车手的姓名及其在所有记录比赛中的平均圈速。使用此视图，可以快速比较车手在速度方面的整体表现。表 9-11 展示了车手姓名及其计算得出的以秒为单位的平均圈速。

**表 9-11**
**每位车手的平均圈速**

| `Name` | `avg_time` |
| --- | --- |
| 塞尔吉奥·佩雷兹 | 87.31999969 |
| 兰多·诺里斯 | 89.2300034 |
| 皮埃尔·加斯利 | 89.66999817 |
| 刘易斯·汉密尔顿 | 90.2300034 |
| 费尔南多·阿隆索 | 88.44999695 |
| 马克斯·维斯塔潘 | 90.44999695 |
| 埃斯特班·奥康 | 92.88999939 |
| 乔治·拉塞尔 | 89.01000214 |
| 夏尔·勒克莱尔 | 91.12000275 |
| 卡洛斯·塞恩斯 | 87.77999878 |

### 统计车队胜利次数

为了确定每支车队获得第一名的次数，娜塔莉编写了以下查询，通过汇总每支车队车手的获胜次数来评估车队表现。

```sql
CREATE VIEW team_wins AS
SELECT d.team, COUNT(*) AS wins
FROM results res
JOIN drivers d ON res.driver_id = d.driver_id
WHERE res.position = 1
GROUP BY d.team;
```

此 SQL 查询创建了一个名为`team_wins`的视图，用于追踪每支赛车队的比赛获胜次数。它连接了`Results`和`Drivers`表，将赛事表现与车队信息关联起来。该查询仅统计`position`等于`1`的第一名成绩，并按车队名称对结果进行分组。这意味着视图将显示每支车队在所有比赛中的总获胜次数。通过使用`COUNT(*)`，它统计了车队车手获得第一名的比赛场次，清晰地展示了车队在赛车比赛中的整体表现和成功程度。

在此，一个名为`team_wins`的视图通过创建一个预定义的、易于访问的赛事表现摘要，简化了数据分析。视图允许快速检索复杂信息——在此案例中是车队胜利——而无需反复编写复杂的 SQL 连接语句。视图还提供了一种便捷、性能高效的方式来追踪和分析车队在多场比赛中的成功。表 9-12 展示了前述查询的`SELECT`输出结果，即每支赛车队的比赛获胜次数。

**表 9-12**
**每支赛车队的比赛获胜次数**

| `Team` | `Wins` |
| --- | --- |
| 阿尔派 | 1 |
| 阿斯顿·马丁 | 1 |
| 梅赛德斯 | 2 |
| 红牛 | 1 |

### 检索包含详细信息的比赛获胜者

以下查询旨在查找每场比赛的获胜者及其比赛地点和日期。此查询在突出比赛获胜者的同时，将其与地点和日期关联，便于赛事追踪。

```sql
CREATE VIEW race_winners AS
SELECT d.name, r.location, r.date
FROM results res
JOIN drivers d ON res.driver_id = d.driver_id
JOIN races r ON res.race_id = r.race_id
WHERE res.position = 1;
```


## 为数据分析创建和管理 SQL 视图

这个 SQL 查询创建了一个名为 `race_winners` 的视图。它通过连接 `Results`、`Drivers` 和 `Races` 表，选取了获胜者姓名、比赛地点和日期。`JOIN` 子句将车手和比赛信息关联到比赛结果。`WHERE` 子句筛选出 `position` 等于 `1` 的结果，从而确定比赛获胜者。创建视图的结果是，未来的查询将得以简化。分析师无需反复编写复杂的 `JOIN` 语句，而是可以直接查询 `race_winners` 来获取一份简明的冠军名单。这提高了可读性和效率。表 9-13 展示了获胜者姓名、比赛地点和日期。

**表 9-13：获胜者姓名、比赛地点和日期**

| 姓名 | 地点 | 日期 |
| --- | --- | --- |
| Lewis Hamilton | Silverstone | 2023-07-09 |
| Sergio Perez | Monza | 2023-09-03 |
| George Russell | Spa-Francorch. | 2023-07-30 |
| Fernando Alonso | Suzuka | 2023-09-24 |
| Esteban Ocon | Monaco | 2023-05-28 |

此查询用于识别每场比赛中做出最快圈速的车手，并突出显示了无论比赛排名如何的出色单圈表现。

```
CREATE VIEW fastest_laps AS
SELECT d.name, r.location, MIN(res.lap_time_sec) AS fastest_time
FROM results res
JOIN drivers d ON res.driver_id = d.driver_id
JOIN races r ON res.race_id = r.race_id
GROUP BY r.location, d.name;
```

这个查询创建了一个名为 `fastest_laps` 的视图。其目的是确定每位车手在每个比赛地点取得的最快圈速。该查询连接了 `Results`、`Drivers` 和 `Races` 表，以将车手姓名和比赛地点与圈速相关联。`MIN(res.lap_time_sec)` 函数确定了每位车手在每个地点的最短圈速。`GROUP BY r.location, d.name` 子句对结果进行组织，确保为每位车手在每个比赛地点分别计算最短圈速。该视图简化了对最快圈速数据的访问和分析。

> **注意**
>
> 在 PostgreSQL 中，`MIN()` 函数是一个聚合函数，它返回指定值集中的最小值。它通常用于查找表中某列的最小值。一旦应用 `MIN()`，它会分析选定的列并返回最低值。例如，它可以用来识别最早的日期、最低的价格或最快的圈速。为了在特定组内查找最小值，它常与 `GROUP BY` 结合使用。

表 9-14 提供了每位车手在每个比赛地点取得的最快圈速。

**表 9-14：每位车手在每个比赛地点取得的最快圈速**

| 姓名 | 地点 | 最快圈速 |
| --- | --- | --- |
| Carlos Sainz | Monza | 87.78 |
| Sergio Perez | Monza | 87.32 |
| Pierre Gasly | Suzuka | 89.67 |
| Charles Leclerc | Silverstone | 91.12 |
| Max Verstappen | Silverstone | 90.45 |
| Fernando Alonso | Suzuka | 88.45 |
| Lando Norris | Spa-Francorch. | 89.23 |
| George Russell | Spa-Francorch. | 89.01 |
| Esteban Ocon | Monaco | 92.89 |
| Lewis Hamilton | Silverstone | 90.23 |

视图通过存储预定义的查询、减少代码重复和提高可读性来简化数据访问。通过优化复杂的连接和聚合，视图使分析更快速、更高效。有了这些视图，Nathalie 可以快速访问重要的比赛表现洞察，而无需反复编写复杂的 SQL 查询。

### 管理视图

如前所述，PostgreSQL 中的 `CREATE VIEW` 语句用于基于 SQL 语句的结果集定义一个虚拟表。`CREATE VIEW` 的语法很简单，它允许创建一个命名查询，该查询可以像表一样使用，从而简化复杂查询并增强数据安全性。

#### 更新和修改视图 (ALTER VIEW)

虽然 PostgreSQL 没有直接的 `ALTER VIEW` 语句来修改视图的查询定义，但可以使用 `CREATE OR REPLACE VIEW` 来修改视图。此命令允许你重新定义视图的查询，而无需删除并重新创建它，从而保留任何依赖对象。其语法与 `CREATE VIEW` 类似，但使用 `CREATE OR REPLACE VIEW` 可确保如果视图已存在，则其定义将被替换。保持一致性并避免中断至关重要。

PostgreSQL 提供了 `ALTER VIEW` 语句来修改视图属性，例如重命名视图或更改其列名和默认值。基本语法包括以下内容。

重命名视图：

```
ALTER VIEW view_name RENAME TO new_view_name;
```

更改视图中的列名：

```
ALTER VIEW view_name RENAME COLUMN old_column_name TO new_column_name;
```

设置或移除视图列的默认值：

```
ALTER VIEW view_name ALTER COLUMN column_name SET DEFAULT default_value;
ALTER VIEW view_name ALTER COLUMN column_name DROP DEFAULT;
```

因此，这些命令可以在不需要完全重新定义的情况下维护视图的一致性。这使得更新更高效，并最大限度地减少了对依赖对象的干扰。

#### 删除视图 (DROP VIEW)

`DROP VIEW` 语句用于从数据库中移除一个视图。语法很简单：

```
DROP VIEW view_name;
```

此命令会永久删除视图定义。每当一个视图被删除，任何依赖于它的查询或应用程序都将失败。为了增强灵活性，PostgreSQL 提供了许多可以使用的附加选项。

如果视图不存在，则避免报错：

```
DROP VIEW IF EXISTS view_name;
```

这可以防止在脚本和自动化过程中，当视图的存在性无法保证时发生错误。

一次删除多个视图：

```
DROP VIEW view_name1, view_name2, ...;
```

使用 `RESTRICT` 防止意外删除：

```
DROP VIEW view_name RESTRICT;
```

这确保了只有在没有其他对象依赖于该视图时才会将其删除。除了管理数据库完整性外，这些选项还允许受控地清理未使用的视图，防止意外的中断。

#### ALTER VIEW 和 DROP VIEW 在数据分析中的作用

在数据分析中，视图在构建和简化复杂查询方面起着至关重要的作用，使数据检索更高效。能够使用 `CREATE OR REPLACE VIEW` 更新视图，确保分析师可以在不中断工作流的情况下修改数据表示方式。这在数据模型演变、需要调整现有视图而不影响依赖的报告或应用程序的情况下尤其有益。另一方面，`DROP VIEW` 对于通过删除过时或不必要的视图来维护干净、优化的数据库至关重要。然而，`IF EXISTS` 选项在动态和自动化环境中特别有用，在这些环境中，分析师在删除视图时必须谨慎，以防止破坏依赖的查询。

### 视图在优化 SQL 查询中的作用

当视图被正确使用时，它们通过封装复杂逻辑来减少冗余，从而避免在多个脚本中重复冗长的 SQL 代码。这样，可维护性得到增强，不一致性得以减少。视图有效地平衡了存储、速度和查询复杂性。通过抽象复杂查询，视图提供了简化的接口，减少了在日常分析中对复杂 SQL 的需求。特别是物化视图通过预编译和存储结果提供了速度优势，尽管这是以存储为代价的。而标准视图虽然不物理存储数据，但它们简化了查询复杂性，从而产生更易于管理、更清晰的脚本。通过视图的战略性实施，数据访问得到优化，确保查询既高效又可读，从而优化了整体性能和可维护性。


### 在 PostgreSQL 中同时使用视图和索引

在 SQL 中，视图和索引是互补的，但服务于不同的目的。`视图` 是虚拟表，存储查询的结果定义，使复杂查询更易于阅读和重用。它们通过限制对特定列的访问来简化数据访问、保持一致性并提高安全性。`索引` 通过允许数据库更高效地定位行（而不是扫描整张表）来加速数据检索。对于处理大型数据集时，它们是性能优化的关键。

由于 `视图` 本身不存储数据，因此它们不会自动从 `索引` 中受益，除非它们是物化的或用于带索引的查询。本节的目的是探讨如何有效地同时使用两者。

## 第三个故事：在线零售数据分析师

Kris 是一家在线零售公司的数据分析师。该公司的销售团队经常向 Kris 索要关于产品表现的报告。然而，每次查询原始的 `sales` 表都很慢，尤其是随着数据库的增长。为了解决这个问题，Kris 决定使用 `视图` 来构建结构化查询，并使用 `索引` 来提升性能。

### 注意

毫无疑问，在真实场景中，数据集比表 9-15 中展示的样本要大得多且复杂得多。由于书中呈现故事的篇幅限制，该表仅限于十行。只要将相同的方法应用于大型数据集，这个限制不会影响查询效果。

### 表 9-15：销售表

| sale_id | product_id | amount | sale_date |
| --- | --- | --- | --- |
| 1 | 101 | 120.5 | 2024-03-01 |
| 2 | 102 | 85.75 | 2024-03-02 |
| 3 | 101 | 99.99 | 2024-03-02 |
| 4 | 103 | 150 | 2024-03-03 |
| 5 | 101 | 110 | 2024-03-04 |
| 6 | 104 | 200 | 2024-03-05 |
| 7 | 102 | 80.5 | 2024-03-06 |
| 8 | 103 | 175.25 | 2024-03-07 |
| 9 | 101 | 125 | 2024-03-08 |
| 10 | 105 | 300 | 2024-03-09 |

Kris 检查了 `销售` 表，该表跟踪客户购买记录，包括 `sale_id`、`product_id`、`amount`（销售金额）和 `sale_date`。

Kris 注意到大多数报告都通过 `sale_date` 过滤数据，这意味着数据库必须扫描整张表才能找到相关记录。为了改进这一点，Kris 在 `sale_date` 列上创建了一个索引：

```sql
CREATE INDEX idx_sale_date ON sales(sale_date);
```

现在，每当查询按 `sale_date` 过滤时，PostgreSQL 都会使用该索引，显著提升了性能。

接下来，Kris 使用一个 `视图` 来计算每个产品的总销售额，而不是手动聚合销售数据：

```sql
CREATE VIEW sales_summary AS
SELECT product_id, SUM(amount) AS total_sales
FROM sales
GROUP BY product_id;
```

有了这个 `视图`，她可以快速检索总销售额，而无需重写复杂的查询。

现在，为了消除每次查询都重新计算数据的需要，Kris 创建了一个物化 `视图`，用于存储计算结果：

```sql
CREATE MATERIALIZED VIEW sales_summary_mat AS
SELECT product_id, SUM(amount) AS total_sales
FROM sales
GROUP BY product_id;
```

为了加快查找速度，Kris 在 `product_id` 上添加了一个 `索引`：

```sql
CREATE INDEX idx_sales_summary ON sales_summary_mat(product_id);
```

每当销售团队需要报告时，Kris 不再需要重新处理整个数据集，而是直接查询物化 `视图`：

```sql
SELECT * FROM sales_summary_mat;
```

### 表 9-16：每个产品的总销售额

| product_id | total_sales |
| --- | --- |
| 104 | 200 |
| 105 | 300 |
| 103 | 325.25 |
| 101 | 455.49 |
| 102 | 166.25 |

由于每天都有新的销售记录产生，Kris 会定期刷新物化 `视图`：

```sql
REFRESH MATERIALIZED VIEW sales_summary_mat;
```

如前所述，物化 `视图` 物理地存储查询结果，而不是每次重新计算。`REFRESH MATERIALIZED VIEW sales_summary_mat;` 命令必须用于保持数据的最新状态，因为物化 `视图` 在基础表更改时不会自动更新。如果不运行此命令，物化 `视图` 将继续显示过时的数据。即使 `销售` 表中添加了新的销售记录，或对现有记录进行了更新或删除。这可能导致报告出现差异和错误的商业决策。通过刷新物化 `视图`，最新的销售数据可以反映在摘要中，使销售团队能够使用准确且最新的信息。虽然刷新物化 `视图` 可能比较耗费资源，但对于维护数据完整性至关重要，尤其是在报告和分析依赖于新鲜数据的环境中。

销售团队现在可以即时获得他们的报告，而 Kris 也不再需要手动运行缓慢且重复的查询了。

在 SQL 中同时使用 `视图` 和 `索引` 具有几个优点。`视图` 的优化是关键优势之一，因为它们通过结构化复杂的数据检索来简化查询，而 `索引` 则通过减少全表扫描的需要来增强查询执行。除了改进安全性和访问控制外，`视图` 可以限制对特定列的访问或连接多个表以仅呈现相关数据，而 `索引` 确保对这些数据的访问尽可能高效。此外，带有 `索引` 的物化 `视图` 提供了显著的性能提升，它们物理存储查询结果，使得 `索引` 可以直接应用于结果。这可以减少计算时间并提高查询效率，特别是对于频繁访问的数据。

## 总结

本章重点介绍了 SQL `索引` 和 `视图` 在优化数据分析方面的强大功能。在本章中，你学习了如何提高查询性能和数据管理能力。SQL `索引` 通过减少全表扫描的需要来实现更快的数据检索。`视图` 通过封装复杂逻辑来实现结构化且简化的查询执行。此外，同时使用两者可以显著提高数据库操作的效率、可维护性和安全性。

### 关键点

*   `索引` 通过减少全表扫描的需要来增强查询性能，从而能够从大型数据集中更快地检索数据。
*   `视图` 通过将逻辑封装成可重用的结构来简化复杂查询，使数据分析更高效、更易于维护。
*   物化 `视图` 物理存储查询结果，允许进行 `索引` 优化，以实现更快的报告和分析处理。
*   `索引` 和 `视图` 的结合提高了效率，确保频繁访问的数据既结构化又可快速检索。

### 核心要点

*   **优化的查询性能**：`索引` 通过最小化全表扫描来减少查询执行时间，提高数据检索效率。
*   **简化的数据访问**：`视图` 将复杂查询转换为可重用的结构，增强了可读性和可维护性。
*   **更快的分析处理**：物化 `视图` 存储预计算的结果，允许执行 `索引` 优化以更快地获得洞察。
*   **高效的数据管理**：结合 `视图` 和 `索引` 确保以最小的开销对频繁查询的数据进行结构化访问。

### 展望未来

下一章“分析炼金术：将数据转化为黄金”将探讨将原始数据转化为有意义洞察的强大技术。它将研究能够实现更深入数据探索的高级分析函数，以及用于数据比较的 `窗口函数`，这些函数允许你分析行之间的趋势和关系，同时介绍通过富有洞察力的结构化报告将原始数据转化为引人入胜故事的策略。



## 测试你的技能

安吉拉是一家大型电商公司的数据库管理员，负责优化数据检索和报表生成。公司的商业智能团队需要快速高效地访问销售数据，以回答关键的绩效问题，例如：

*   过去六个月中，各类别中最畅销的产品是什么？
*   不同地区和时间段之间的收入增长情况如何比较？
*   哪些客户频繁进行高价值购买？他们偏好哪些产品？
*   季节性趋势如何影响不同类别的产品需求？
*   订单的退货率是多少？这对总收入有何影响？

安吉拉意识到，在庞大的销售数据集上反复执行复杂查询会减慢报表和分析的速度。为了解决这个问题，她决定使用索引来加速对常用查询列（如 `order_date`、`customer_id` 和 `product_category`）的搜索。此外，她还通过预定义常用查询（如每个地区的总收入或最畅销产品）来创建视图，以简化分析师的数据检索工作。参见表 9-17、9-18 和 9-19。

### 表 9-19：客户表

| 客户 ID | 姓名 | 年龄 | 地区 | 总消费额 |
| --- | --- | --- | --- | --- |
| 501 | 爱丽丝 | 32 | 东部 | 375 |
| 502 | 鲍勃 | 45 | 西部 | 120 |
| 503 | 查理 | 28 | 南部 | 200 |
| 504 | 大卫 | 40 | 东部 | 75 |
| 505 | 伊娃 | 35 | 南部 | 160 |
| 506 | 弗兰克 | 50 | 西部 | 90 |
| 507 | 格蕾丝 | 27 | 北部 | 360 |
| 508 | 亨利 | 41 | 东部 | 600 |
| 509 | 艾琳 | 30 | 南部 | 25 |

### 表 9-18：产品表

| 产品 ID | 产品名称 | 类别 | 单价 | 库存数量 |
| --- | --- | --- | --- | --- |
| 2001 | 笔记本 | 文具 | 25 | 500 |
| 2002 | 笔记本电脑 | 电子产品 | 40 | 100 |
| 2003 | 智能手机 | 电子产品 | 120 | 50 |
| 2004 | 办公椅 | 家具 | 300 | 30 |
| 2005 | 耳机 | 电子产品 | 45 | 200 |

### 表 9-17：订单表

| 订单 ID | 客户 ID | 产品 ID | 订单日期 | 数量 | 总价 | 地区 |
| --- | --- | --- | --- | --- | --- | --- |
| 1001 | 501 | 2001 | 2024-01-05 | 2 | 50 | 东部 |
| 1002 | 502 | 2003 | 2024-02-12 | 1 | 120 | 西部 |
| 1003 | 503 | 2002 | 2024-02-15 | 5 | 200 | 南部 |
| 1004 | 504 | 2001 | 2024-03-01 | 3 | 75 | 东部 |
| 1005 | 501 | 2004 | 2024-03-10 | 1 | 300 | 北部 |
| 1006 | 505 | 2002 | 2024-03-18 | 4 | 160 | 南部 |
| 1007 | 506 | 2005 | 2024-04-02 | 2 | 90 | 西部 |
| 1008 | 507 | 2003 | 2024-04-08 | 3 | 360 | 北部 |
| 1009 | 508 | 2004 | 2024-04-15 | 2 | 600 | 东部 |
| 1010 | 509 | 2001 | 2024-04-20 | 1 | 25 | 南部 |


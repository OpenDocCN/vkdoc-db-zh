# 附录 B：术语表

本附录按字母顺序提供了与 SQL 和数据分析相关的常用术语表。该术语表包含超过 100 个术语，为任何使用 PostgreSQL 进行数据分析的人员提供了全面的参考。



## A

`ACID (Atomicity, Consistency, Isolation, Durability)`: 保证数据库事务被可靠处理的一组属性。

`Aggregate functions`: 对多行进行计算并返回单个值的函数。

`ANALYZE`: 收集表内容统计信息的命令，查询规划器用它来生成高效的执行计划。

`Array`: 一种数据类型，可以在单列中存储同一类型的多个值。

## B

`B-tree index`: PostgreSQL 中的默认索引类型；一种针对范围查询和等值条件优化的平衡树结构。

`Backup`: 数据库数据的副本，可在数据丢失时用于恢复数据库。

`BETWEEN`: 用于测试值是否落在指定范围内的运算符。

`BIGINT`: 能够存储在 -9,223,372,036,854,775,808 和 9,223,372,036,854,775,807 之间的整数的数据类型。

`BOOLEAN`: 可以存储 `true`、`false` 或 `NULL` 值的数据类型。

`BRIN (Block Range INdex)`: 适用于具有自然顺序的非常大的表的索引类型；存储块范围的摘要信息。

## C

`Cast`: 将值从一种数据类型转换为另一种数据类型的转换操作。

`CTE (Common Table Expression)`: 在查询中定义的命名临时结果集，可以被多次引用。

`COALESCE`: 返回列表中第一个非 `NULL` 值的函数。

`Collation`: 确定数据库中如何排序和比较文本数据的一组规则。

`Composite type`: 由多个字段组成的用户定义数据类型。

`Concurrency control`: 当多个事务同时执行时，确保数据一致性的机制。

`Connection pooling`: 维护数据库连接池以供重用的技术，用于提高性能。

`Constraints`: 在数据列上强制执行以维护数据完整性的规则。

`COPY`: 用于在 PostgreSQL 表和文件之间批量加载或卸载数据的命令。

`Correlation`: 指示两个变量一起波动程度的统计量度。

`Correlated subquery`: 引用外部查询列的子查询。

`CUBE`: `GROUP BY` 的扩展，生成多个分组集合以进行多维聚合。

`Cursor`: 用于遍历查询结果集的数据库对象。

## D

`Data type`: 指定对象可以保存的数据类型的属性。

`Database cluster`: 由单个 PostgreSQL 服务器实例管理的数据库集合。

`Deadlock`: 两个或多个事务互相等待对方释放锁的情况。

`DISTINCT`: 用于从结果集中消除重复行的子句。

`Domain`: 带有约束的用户定义数据类型。

## E

`ENUM`: 表示静态、有序值集的数据类型。

`EXPLAIN`: 显示语句执行计划但不实际执行该语句的命令。

`EXPLAIN ANALYZE`: 显示执行计划和实际运行时统计信息的命令。

`Extension`: 向 PostgreSQL 添加功能的模块，例如 PostGIS。

`Extract, Transform, Load (ETL)`: 将数据从源系统复制到数据库中的过程。

## F

`Foreign key`: 维护两个表之间参照完整性的约束。

`Full text search`: 允许在文本数据中搜索特定单词或短语的功能。

`Function`: 可重用的 SQL 代码块，执行特定任务并可以返回一个值。

## G

`Generalized Inverted Index (GIN)`: 用于处理在单列中存储多个值的情况的索引结构，常用于数组和全文搜索。

`Generalized Search Tree (GiST)`: 支持任意索引方案的索引结构，用于几何数据类型和全文搜索。

`GROUP BY`: 用于将具有相同值的行分组到汇总行中的子句。

## H

`Hash index`: 适用于等值比较但不适用于范围查询的索引结构。

`HAVING`: 用于过滤 `GROUP BY` 查询中的分组的子句。

`Histogram`: 表示列中值分布的数据结构。

`HyperLogLog`: 用于近似不同计数计算的算法。

## I

`Index`: 提高数据检索操作速度的数据结构。

`Index-only scan`: 所有需要的数据都可以从索引中检索而无需访问表的查询执行方式。

`INNER JOIN`: 当两个表中都有匹配时返回行。

`Isolation level`: 确定如何强制执行事务完整性的设置。

## J

`JOIN`: 基于相关列将来自两个或多个表的行组合起来的 SQL 操作。

`JSON/JSONB`: 存储 JSON（JavaScript 对象表示法）数据的数据类型，具有不同的性能特性。

## K

`Key`: 用于标识记录或在表之间建立关系的字段或字段组合。

## L

`LATERAL JOIN`: 允许 `FROM` 子句中的子查询引用 `FROM` 子句中前面项目的列。

`LEFT JOIN`: 返回左表中的所有行以及右表中的匹配行。

`LIMIT`: 限制查询返回行数的子句。

`LISTEN/NOTIFY`: PostgreSQL 中的异步通知系统。

`Locale`: 定义语言和文化偏好参数的一组参数。

`Lock`: 用于控制对数据的并发访问的机制。

## M

`MADlib`: 一个用于可扩展的数据库内分析的开源库。

`Materialized view`: 物理存储结果集并且必须显式刷新的视图。

`Multi-Version Concurrency Control (MVCC)`: PostgreSQL 使用的允许并发访问数据库的技术。

## N

`Natural JOIN`: 根据同名列自动连接表的 `JOIN`。

`Normalization`: 组织数据以减少冗余并提高数据完整性的过程。

`NULL`: 用于指示数据值不存在的特殊标记。

`NULLIF`: 如果两个指定表达式相等则返回 `NULL` 的函数。

## O

`Object Identifier (OID)`: PostgreSQL 内部使用的系统列。

`OFFSET`: 在返回结果之前跳过指定行数的子句。

`Online Analytical Processing (OLAP)`: 使用户能够从多个角度分析多维数据的数据处理。

`Online Transaction Processing (OLTP)`: 专注于面向事务的应用程序的数据处理。

`Operator`: 指定要执行的操作的符号。

`Optimizer`: 确定执行查询最有效方式的组件。

`ORDER BY`: 用于对结果集进行排序的子句。

## P

`Partition`: 将大型表划分为更小、更易于管理的部分。

`Percentile`: 低于给定百分比观测值的值。

`Prepared statement`: SQL 语句模板，可以使用不同的参数执行多次。

`Primary key`: 唯一标识表中每行的列或列组。

`Procedural language`: 用于编写存储过程和函数的语言（例如 PL/pgSQL）。

## Q

`Query`: 向数据库请求数据或信息的查询。

`Query plan`: PostgreSQL 用于访问数据的步骤序列。

`Query optimizer`: 确定执行查询最有效方式的组件。

## R

`RAISE`: 在 PL/pgSQL 中用于报告消息和引发错误的语句。

`Recursive query`: 引用自身的查询。

`Referential integrity`: 确保表之间的关系保持一致的属性。

`RETURNING`: 从受 `INSERT`、`UPDATE` 或 `DELETE` 影响的行中返回数据的子句。

`ROLLUP`: `GROUP BY` 的扩展，创建层次分组集合。

`Row-level security`: 限制用户可以访问哪些行的功能。




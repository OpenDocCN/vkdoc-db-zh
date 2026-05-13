# 优化过程

识别和计算查询各种执行计划的三个主要任务是：

*   确定要使用的并行度。
*   确定要应用哪些转换。
*   确定我称为 `final state optimization` 的内容。我创造了这个术语，因为“优化”这个词有歧义。一方面，优化可以指整个过程，包括确定并行度和必要的转换。另一方面，它可以指在转换和并行度确定后，识别和计算各种执行计划的过程。为了避免混淆，我将使用术语 `final state optimization` 来指代后一个、更狭窄的概念。

这三个任务并非相互独立。在不知道串行执行计划是什么样子之前，你无法真正计算出使用什么并行度。毕竟，一个表及其各个索引在数据字典中可能指定了不同的并行度。但除非知道全表扫描将在多大程度上被并行化，否则你通常无法判断使用索引还是全表扫描更好。幸运的是，你和我都不需要理解 CBO 如何理清这一切。我们只需要理解基础知识，让我们开始吧。

## 并行度

如果 CBO 完全不关心资源消耗，你可能想知道为什么它不大多数时候（即使不是所有时候）都使用并行查询。简短的回答是，默认情况下，CBO 被禁止考虑并行查询、并行 DML 和并行 DDL。要允许 CBO 考虑并行查询，你需要执行以下操作之一：

*   向查询添加 `PARALLEL` 优化器提示。
*   更改查询涉及的一个或多个表或索引，以指定非默认的并行度。
*   使用 `ALTER SESSION` 语句为会话指定并行度。
*   在系统上设置 `automatic degree of parallelism` (auto DOP)。



如果你执行上述任一操作，成本优化器将被允许考虑并行操作，但在所有情况下，最大并行度都会受到限制，以确保资源消耗不会过度失控。清单 2-1 提供了启用并行查询操作的示例。

清单 2-1. 启用并行查询的不同方法

```
CREATE TABLE t
AS
   SELECT ROWNUM c1 FROM all_objects;

CREATE INDEX i
   ON t (c1);

--
-- 使用优化器提示来允许成本优化器考虑并行操作
--
EXPLAIN PLAN
   FOR
      SELECT /*+ parallel(10) */
            * FROM t;

SELECT * FROM TABLE (DBMS_XPLAN.display);
--
-- 启用涉及表 T 的操作的并行操作
--
ALTER TABLE t PARALLEL (DEGREE 10);
--
-- 启用索引 I 的并行操作
--

ALTER INDEX i
   PARALLEL (DEGREE 10);

--
-- 尝试强制所有 SQL 语句的并行查询度为 10。
-- 请注意，FORCE 并不意味着并行性得到保证！
--
ALTER SESSION FORCE PARALLEL QUERY PARALLEL 10;
--
-- 为系统设置自动 DOP（以 SYS 用户登录，需要重启）
--

DELETE FROM resource_io_calibrate$;

INSERT INTO resource_io_calibrate$
     VALUES (CURRENT_TIMESTAMP
            ,CURRENT_TIMESTAMP
            ,0
            ,0
            ,200
            ,0
            ,0);

COMMIT;

ALTER SYSTEM SET parallel_degree_policy=auto;
```

让我们逐一看看这些示例。在创建了一个表及其关联索引后，下一条语句使用了 Oracle 数据库 11gR2 中引入的 `PARALLEL` 优化器提示的变体，为该语句启用并行度为 10 的并行查询。请注意，与 SQL 语言参考手册相反，此提示并不强制进行并行查询；它只是允许成本优化器考虑它。关键在于，某些操作，例如对非分区索引的索引范围扫描，是无法并行化的，并且在指定的并行度下，串行的索引范围扫描可能比并行的全表扫描更高效。

如果你真的想强制进行并行查询，可以尝试使用清单 2-1 中给出的 `ALTER SESSION` 语句。这里我也指定了要使用的并行度，并且由于同样的原因，平行性同样无法得到保证！`ALTER SESSION` 语句所做的全部事情，就是隐式地向会话中的所有语句添加一个 `PARALLEL` 提示。

**强制并行操作**

如果成本优化器选择了串行索引范围扫描，而你想强制进行并行全表扫描，那么即使使用了清单 2-1 中的 `ALTER SESSION` 语句，尽管其语句中包含了具有误导性的 `FORCE` 关键字，也可能不够。你可能还需要在代码中添加提示来强制进行全表扫描。

我在清单 2-1 的任何示例中都不必指定并行度。如果我没有指定，那么成本优化器将选择“理想的并行度”，即允许查询最快完成的并行度，前提是该度不超过初始化参数 `PARALLEL_DEGREE_LIMIT` 指定的值。

**并行度与成本**

在某些情况下，增加并行度会适得其反，原因多种多样。尽管理想的并行度反映了这一现实，但报告中显示的并行度高于理想并行度的计划的成本，将低于使用理想并行度的计划。

为语句启用并行查询的最终方法是使用 Oracle 数据库 11gR2 中引入的自动 DOP 功能。此功能基本上意味着，对于预计运行时间超过 10 秒的所有语句，都将使用理想的并行度。理论上，你应该使用 `DBMS_RESOURCE_MANAGER.CALIBRATE_IO` 例程来设置自动 DOP，以便成本优化器能够更准确地计算理想的并行度。实际上，成本优化器团队建议改用清单 2-1 中的代码，该代码为输入/输出子系统指定了每秒 200 MB 的吞吐量。所有其他指标，如磁盘数量（对于企业存储来说已无意义），则未指定。无论哪种方式，你都必须重启数据库实例。

我曾在一个拥有 64 个处理器和 64GB 系统全局区的大型系统上尝试使用非官方方法进行自动 DOP，效果非常好。我从使用文档方法的人那里听到了好坏参半的报告。有些人很满意，有些人则报告了问题。

**查询转换**

成本优化器在优化查询时的第二项主要任务是考虑可能应用于该语句的潜在转换。换句话说，成本优化器着眼于以更简单或更高效的方式重写语句。如今，成本优化器可以考虑很多可能的转换，我将在第 13 章中介绍其中的大部分。

查询转换有两种类型：

*   *基于成本的转换*：只有当转换后查询能获得的最优计划的成本低于未转换查询能获得的最优计划的成本时，成本优化器才应应用此转换。
*   *启发式转换*：此类转换是无条件应用的，或者基于一些简单的规则。

如今，绝大多数转换都是基于成本的，但有一两个不是。无论转换的类型如何，你总是可以通过优化器提示来强制或抑制它。清单 2-2 显示了一个示例查询，其中成本优化器不恰当地应用了启发式查询转换。

清单 2-2. 简单的视图合并查询转换

```
CREATE TABLE t1
AS
       SELECT 1 c1
         FROM DUAL
   CONNECT BY LEVEL <= 100;

CREATE TABLE t2
AS
       SELECT 1 c2
         FROM DUAL
   CONNECT BY LEVEL <= 100;

CREATE TABLE t3
AS
       SELECT 1 c3
         FROM DUAL
   CONNECT BY LEVEL <= 100;

CREATE TABLE t4
AS
       SELECT 1 c4
         FROM DUAL
   CONNECT BY LEVEL <= 100;

EXPLAIN PLAN
   FOR
      WITH t1t2
           AS (SELECT t1.c1, t2.c2
                 FROM t1, t2
                WHERE t1.c1 = t2.c2)
          ,t3t4
           AS (SELECT t3.c3, t4.c4
                 FROM t3, t4
                WHERE t3.c3 = t4.c4)
      SELECT COUNT (*)
        FROM t1t2 j1, t3t4 j2
       WHERE j1.c1 + j1.c2 = j2.c3 + j2.c4;

SELECT * FROM TABLE (DBMS_XPLAN.display (format => 'BASIC +COST'));

PAUSE

-- 上面的查询被转换成了下面的查询

EXPLAIN PLAN
   FOR
      SELECT COUNT (*)
        FROM t1
            ,t2
            ,t3
            ,t4
       WHERE     t1.c1 = t2.c2
             AND t3.c3 = t4.c4
             AND t1.c1 + t2.c2 = t3.c3 + t4.c4;

SELECT * FROM TABLE (DBMS_XPLAN.display (format => 'BASIC +COST'));
-- 生成的执行计划如下：

Plan hash value: 2241143226

| Id  | Operation               | Name | Cost (%CPU)|
```



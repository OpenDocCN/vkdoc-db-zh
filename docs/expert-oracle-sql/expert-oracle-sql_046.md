# 工作区

SQL 语句中的许多操作都需要一些内存才能运行，这些内存分配被称为工作区。

## 需要工作区的操作

以下是一些需要工作区的操作：

*   除嵌套循环（不需要工作区）外的所有类型的连接。
*   排序，除了其他用途外，用于实现 `ORDER BY` 子句以及一些聚合和分析函数。
*   尽管一些聚合和分析函数需要排序，但有些并不需要。在本书中已多次出现的 `SORT AGGREGATE` 操作仍然需要一个工作区，即使它不涉及排序！
*   `MODEL` 子句需要一个工作区来存放被操作的单元格。这将在本书后面讨论 `MODEL` 子句时进行介绍。

## 为工作区分配内存

为每个工作区分配多少内存，以及在内存不足时从临时表空间分配磁盘空间，是运行时引擎的职责。在计算操作成本时，CBO 会猜测运行时引擎将分配多少内存，但执行计划并不包含任何关于分配多少内存的指令。

一个常见的误解是，工作区内存是一次性大块分配的。事实并非如此。工作区内存是根据需要一点一点分配的，但可能到达某个点时，运行时引擎不得不宣布已经足够，操作必须使用临时表空间中的空间来完成。

为工作区分配多少内存的计算取决于初始化参数 `WORKAREA_SIZE_POLICY` 是设置为 `MANUAL` 还是 `AUTO`。

### 当 WORKAREA_SIZE_POLICY 设置为 AUTO 时的内存分配计算

如今我们几乎总是显式地将初始化参数 `PGA_AGGREGATE_TARGET` 设置为一个非零值，因此会进行自动工作区大小调整。在这种情况下，运行时引擎会尝试将总内存量控制在 `PGA_AGGREGATE_TARGET` 以下，但这并不能保证。`PGA_AGGREGATE_TARGET` 用于推导以下隐藏初始化参数的值：

*   `_smm_max_size`：串行执行的语句中，单个工作区的最大大小。该值以千字节为单位指定。
*   `_smm_px_max_size`：并行执行的查询中，所有匹配工作区的总最大大小。该值以千字节为单位指定。
*   `_pga_max_size`：一个进程内所有工作区可分配的最大内存量。该值以字节为单位指定。

计算这些隐藏值的内部规则似乎相当复杂，并且会因版本而异，因此我不打算在此记录它们。请记住：

*   当自动 `WORKAREA_SIZE_POLICY` 设置为 `AUTO` 时，任何工作区的最大大小最多为 1GB。
*   当 `WORKAREA_SIZE_POLICY` 设置为 `MANUAL` 时，分配给一个进程的所有工作区的总内存最多为 2GB。
*   增加语句的并行度，在达到某个点后，将不再对你的操作可用内存产生影响。

### 当 WORKAREA_SIZE_POLICY 设置为 MANUAL 时的内存分配计算



要获得超过 1GB 的单个**工作区**，或让串行执行的 SQL 语句在多个工作区中使用超过 2GB 的空间，唯一的方法是将 `WORKAREA_SIZE_POLICY` 设置为 `MANUAL`。这可以在会话级别完成。然后，您可以在会话级别设置以下参数：

*   `HASH_AREA_SIZE`：用于 `HASH JOIN` 的工作区的最大大小。以字节为单位指定，必须小于 2GB。
*   `SORT_AREA_SIZE`：用于所有其他目的的工作区的最大大小。这包括诸如 `HASH GROUP BY` 之类的操作，所以请不要混淆！`SORT_AREA_SIZE` 的值以字节为单位指定，可以设置为任何小于 2GB 的值。

`HASH_AREA_SIZE` 和 `SORT_AREA_SIZE` 参数仅在 `WORKAREA_SIZE_POLICY` 设置为 `MANUAL` 时使用。否则，它们将被忽略。

## Optimal, One-Pass, and Multipass Operations

如果您的排序、连接或其他操作完全在分配给工作区的内存中完成，则称为 **optimal** 操作。如果需要磁盘空间，则可能是数据被写入磁盘一次，然后又读回一次。在这种情况下，它被称为 **one-pass** 操作。有时，如果工作区严重不足，数据需要被读写多次。这被称为 **multipass** 操作。通常，尽管存在一些晦涩的异常情况，**optimal** 操作会比 **one-pass** 操作完成得更快，而 **multipass** 操作则比 **one-pass** 操作花费的时间长得多。正如你可能想象的那样，识别 **multipass** 操作非常有用，幸运的是，`V$SQL_PLAN_STATISTICS_ALL` 中有几个列可以帮上忙：

*   `LAST_EXECUTION`：该值为 `OPTIMAL`，或者指定了该语句上次执行所经历的精确 **pass** 次数。
*   `ESTIMATED_OPTIMAL_SIZE`：估计此工作区为了在内存中完全执行操作（**optimal** 执行）所需的大小（以千字节为单位）。这要么来源于优化器统计信息，要么来源于之前的执行记录。
*   `ESTIMATED_ONEPASS_SIZE`：这是估计此工作区为了在单次 **pass** 中执行操作所需的大小（以千字节为单位）。这要么来源于优化器统计信息，要么来源于之前的执行记录。
*   `LAST_TEMPSEG_SIZE`：上次实例化此工作区时创建的临时段大小（以字节为单位）。当 `LAST_EXECUTION` 为 `OPTIMAL` 时，此列为 null。

## Shortcuts

运行时引擎通常会严格遵循 CBO 设定的指令，但有时为了性能，它会发挥主动性，偏离规定的路径。让我们看几个例子。

### Scalar Subquery Caching

再看一下 Listing 3-1 中的查询。Listing 4-7 执行了该查询并检查了运行时行为。

**Listing 4-7. 标量子查询缓存示例**

```sql
SET LINES 200 PAGES 900 SERVEROUT OFF
COLUMN PLAN_TABLE_OUTPUT FORMAT a200

SELECT e.*
      ,d.dname
      ,d.loc
      , (SELECT COUNT (*)
           FROM scott.emp i
          WHERE i.deptno = e.deptno)
          dept_count  FROM scott.emp e, scott.dept d
 WHERE e.deptno = d.deptno;

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'ALLSTATS LAST'));
```

```
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT   |      |      1 |        |     14 |
|   1 |  SORT AGGREGATE    |      |      3 |      1 |      3 |
|*  2 |   TABLE ACCESS FULL| EMP  |      3 |      1 |     14 |
|*  3 |  HASH JOIN         |      |      1 |     14 |     14 |
|   4 |   TABLE ACCESS FULL| DEPT |      1 |      4 |      4 |
|   5 |   TABLE ACCESS FULL| EMP  |      1 |     14 |     14 |
```

`DBMS_XPLAN.DISPLAY` 的输出再次被编辑过，但重要的列已列出。你可以看到操作 5 执行了 1 次并返回了 14 行。这很合理。`SCOTT.EMP` 表中有 14 行。然而，你可能预期来自相关子查询的操作 1 和 2 会执行 14 次。从 `Starts` 列可以看到它们只执行了 3 次！这是因为运行时引擎为每个 `DEPTNO` 值缓存了子查询的结果，因此子查询对于 `EMP` 中出现的 3 个 `DEPTNO` 值中的每一个只执行了一次。提醒一下，`E-Rows` 是操作单次执行估计返回的行数，`A-Rows` 是操作所有三次执行实际返回的行数。

标量子查询缓存仅在子查询的结果已知在多次调用之间不会变化时才会启用。因此，如果子查询调用了例如 `DBMS_RANDOM` 包中的例程，那么子查询缓存就会被关闭。如果子查询包含对用户编写函数的调用，那么标量子查询缓存也会被禁用，除非在函数声明中添加了关键字 `DETERMINISTIC`。在我们讨论函数结果缓存时，我会再稍微详细解释一下这一点。

### Join Shortcuts

有时，如果继续执行没有意义，运行时引擎会中断连接操作。Listing 4-8 创建并连接了两个表。

**Listing 4-8. 连接快捷方式**

```sql
SET LINES 200 PAGES 0 SERVEROUTPUT OFF

CREATE TABLE t1
AS
       SELECT ROWNUM c1
         FROM DUAL
   CONNECT BY LEVEL <= 100;

CREATE TABLE t2
AS
       SELECT ROWNUM c1
         FROM DUAL
   CONNECT BY LEVEL <= 100;

SELECT *
  FROM t1 JOIN t2 USING (c1)
 WHERE c1 = 200;

SELECT *
FROM TABLE (DBMS_XPLAN.display_cursor (format => 'BASIC IOSTATS LAST'));
```

```
| Id  | Operation          | Name | Starts | E-Rows | A-Rows |

|   0 | SELECT STATEMENT   |      |      1 |        |      0 |
|*  1 |  HASH JOIN         |      |      1 |      1 |      0 |
|*  2 |   TABLE ACCESS FULL| T1   |      1 |      1 |      0 |
|*  3 |   TABLE ACCESS FULL| T2   |      0 |      1 |      0 |
```

表 `T1` 和 `T2` 连接的执行计划显示了一个哈希连接，但操作 2，即对表 `T1` 的全表扫描，返回了 0 行。运行时引擎随后意识到，如果 `T1` 没有行，连接就不可能产生任何行。从 `Starts` 列你可以看到操作 3，即对 `T2` 的全表扫描，从未运行。我相信你会同意，这是一个合理的决定，覆盖了 CBO 的指令！

## Result and OCI Caches

结果缓存（**result cache**）在 Oracle Database 11gR1 中引入，旨在避免重复执行相同的查询。结果缓存的使用主要通过 `RESULT_CACHE_MODE` 初始化参数和两个提示（hint）来控制：

*   如果 `RESULT_CACHE_MODE` 设置为 `FORCE`，那么除非代码中提供了 `NO_RESULT_CACHE` 提示，否则在所有有效情况下都会使用结果缓存。
*   如果 `RESULT_CACHE_MODE` 设置为 `MANUAL`（默认值），那么除非代码中提供了 `RESULT_CACHE` 提示并且使用结果缓存是有效的，否则永远不会使用结果缓存。

Listing 4-9 展示了如何使用结果缓存功能。

**Listing 4-9. 由初始化参数强制启用的结果缓存功能**

```sql
BEGIN
   DBMS_RESULT_CACHE.flush;
END;
/

ALTER SESSION SET result_cache_mode=force;

SELECT COUNT (*) FROM scott.emp;

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'ALLSTATS LAST'));

SELECT COUNT (*) FROM scott.emp;

SELECT * FROM TABLE (DBMS_XPLAN.display_cursor (format => 'ALLSTATS LAST'));

ALTER SESSION SET result_cache_mode=manual;
```

```
| Id  | Operation              | Name                       | Starts | E-Rows | A-Rows |
```


## 执行计划与缓存

| Id | Operation              | Name                       | Starts | E-Rows | A-Rows |
|----|------------------------|----------------------------|--------|--------|--------|
|  0 | SELECT STATEMENT       |                            |      1 |        |      1 |
|  1 |  RESULT CACHE          | `2jv3dwym1r7n6fjj959uz360b2` |      1 |        |      1 |
|  2 |   SORT AGGREGATE       |                            |      1 |      1 |      1 |
|  3 |    INDEX FAST FULL SCAN| `PK_EMP`                     |      1 |     14 |     14 |

| Id | Operation              | Name                       | Starts | E-Rows | A-Rows |
|----|------------------------|----------------------------|--------|--------|--------|
|  0 | SELECT STATEMENT       |                            |      1 |        |      1 |
|  1 |  RESULT CACHE          | `2jv3dwym1r7n6fjj959uz360b2` |      1 |        |      1 |
|  2 |   SORT AGGREGATE       |                            |      0 |      1 |      0 |
|  3 |    INDEX FAST FULL SCAN| `PK_EMP`                     |      0 |     14 |      0 |

为了运行一个可重复的示例，我首先使用 `DBMS_RESULT_CACHE` 包刷新了结果缓存。然后我执行了一个简单的查询两次。您可以看到执行计划现在有了一个新的 `RESULT_CACHE` 操作，该操作导致第一次执行的结果被保存。在刷新调用后进行的第二次调用中，操作 2 和 3 都没有执行，结果已从结果缓存中检索。

我必须说，我还没有利用结果缓存。处理查询被多次执行的常用方法是只执行它们一次！在有一次我无法阻止语句的多次执行的情况下，因为它们是自动生成的，而结果缓存的使用无效，因为查询中出现了表达式 `TRUNC(SYSDATE)`，该表达式的非确定性使得结果缓存的使用无效。尽管如此，我确信在某些情况下，结果缓存将是最后的救命稻草！

OCI 缓存比服务器端结果缓存更进一步，它将数据存储在客户端。一方面，这可能会提供出色的性能，因为它完全避免了与服务器进行任何通信。另一方面，结果的一致性无法保证，因此它的有用性甚至比服务器端结果缓存更有限！

## 函数结果缓存

假设一个函数调用对于相同的参数集总是返回相同的结果，那么您可以在函数声明中提供 `RESULT_CACHE` 关键字。这将导致返回值与相关参数值一起保存在系统范围内的缓存中。即使函数的结果仅在某些基础表的内容保持不变的情况下才是确定的，您仍然可以使用函数结果缓存。当函数结果缓存首次在 Oracle Database 11gR1 中引入时，您必须指定一个 `RELIES_ON` 子句来指定函数确定性所依赖的对象。从 Oracle Database 11gR2 开始，系统会自动完成此操作，如果指定了 `RELIES_ON` 子句，它将被忽略。

函数结果缓存的概念与我在标量子查询缓存上下文中简要提到的确定性函数的概念非常相似：

*   无论 `DETERMINISTIC` 还是 `RESULT_CACHE` 关键字，都不会阻止在不同参数被提供时对函数的多次调用。
*   声明为 `DETERMINISTIC` 的函数将只避免在单个 SQL 语句调用内对具有相同参数值的函数进行多次调用。`RESULT_CACHE` 关键字可以跨语句和跨会话节省调用。
*   当函数被声明为 `DETERMINISTIC` 时，不会检查基础数据在语句中间的变化。从这个意义上说，`DETERMINISTIC` 不如 `RESULT_CACHE` 可靠。另一方面，您可能不希望基础数据的变化在语句中间可见！
*   `DETERMINISTIC` 函数不消耗系统缓存，并且理论上至少应该比函数结果缓存效率稍高一些。

您可以在函数声明中同时指定 `DETERMINISTIC` 和 `RESULT_CACHE` 关键字，但请记住，`DETERMINISTIC` 并不绝对保证在同一个 SQL 语句中的多次调用会返回相同的结果。标量子查询缓存是有限的，在某些情况下，早期调用的结果可能会从缓存中被移除。

## 事务一致性与函数调用

为这样一本书准备示例的一个有趣之处在于，有时作为作者，我自己也学到了新东西！我以前认为 `READ_COMMITTED` 隔离级别将保证单个 SQL 语句的结果是一致的，即使不能提供 `SERIALIZABLE` 的事务级别保证。然而，事实证明，除非事务隔离级别是 `SERIALIZABLE`，否则重复的递归 SQL 调用（如在函数调用中）*不能*保证产生一致的结果！

Listing 4-10 展示了一个同时使用 `DETERMINISTIC` 和 `RESULT_CACHE` 关键字声明的函数的声明和调用。

Listing 4-10. 通过缓存函数结果进行性能改进

```sql
CREATE TABLE business_dates
(
   location        VARCHAR2 (20)
  ,business_date   DATE
);

INSERT INTO business_dates (location, business_date)
     VALUES ('Americas', DATE '2013-06-03');

INSERT INTO business_dates (location, business_date)
     VALUES ('Europe', DATE '2013-06-04');

INSERT INTO business_dates (location, business_date)
     VALUES ('Asia', DATE '2013-06-04');

CREATE OR REPLACE FUNCTION get_business_date (p_location VARCHAR2)
   RETURN DATE
   DETERMINISTIC
   RESULT_CACHE
IS
   v_date   DATE;
   dummy    PLS_INTEGER;
BEGIN
   DBMS_LOCK.sleep (5);

SELECT business_date
     INTO v_date
     FROM business_dates
    WHERE location = p_location;

RETURN v_date;
END get_business_date;
/

CREATE TABLE transactions
AS
       SELECT ROWNUM rn
             ,DECODE (MOD (ROWNUM - 1, 3),  0, 'Americas',  1, 'Europe',  'Asia')
                 location
             ,DECODE (MOD (ROWNUM - 1, 2)
                     ,0, DATE '2013-06-03'
                     ,DATE '2013-06-04')
                 transaction_date
         FROM DUAL
   CONNECT BY LEVEL <= 20;

SELECT *
    FROM transactions
   WHERE transaction_date = get_business_date (location);
```

此代码创建了一个 `BUSINESS_DATE` 表，该表非常真实地将 `Americas` 的 `BUSINESS_DATE` 设置为比 `Europe` 和 `Asia` 的晚一天。然后从 `TRANSACTIONS` 表中进行选择，挑选与该 `LOCATION` 的 `BUSINESS_DATE` 匹配的行。当然，在现实生活中，这个简单的例子可能最好通过表连接来完成，但同样在现实生活中，函数中的逻辑可能比这里提供的更复杂。

如果您运行此脚本，其中函数内部有一个刻意的 5 秒暂停，您将看到它仅在 15 秒后（即仅三次函数调用后）返回。这表明该函数每个区域只被调用了一次。

## 总结

CBO 创建了一个它认为在当时信息下是最优的执行计划。优化的一个关键部分是理解 CBO 认为将要发生的情况与实际发生的情况之间的差异。本章解释了运行时引擎如何解释 CBO 给出的执行计划，如何跟踪运行时引擎的性能，以及各种形式的缓存如何影响其行为。

您现在已经掌握了足够的基本概念，我可以向您概述我对 SQL 语句优化的方法，这就是我将在第 5 章中介绍的内容。

## 第 5 章

![image](img/frontdot.jpg)

## 调优简介


# 建立索引还是不建立索引？

在 Exadata 实施过程中，我们争论最多的一个问题就是是否要删除索引。声称在 Exadata 上不需要任何索引的说法，在某种程度上加剧了这个问题。使用索引的访问路径通常无法利用 Exadata 特有的优化。是的，你没看错——这通常是因为在优化器选择对索引执行快速全扫描的情况下可以发生卸载，但这并不是索引最常见的用法模式。更常见的模式是使用索引范围扫描来从表中检索相对较少的记录，而目前这种操作是无法卸载的。一般来说，你会希望在选择性谓词上使用索引范围扫描。然而，由于 Exadata 在扫描磁盘方面非常高效，在许多情况下，基于索引的访问路径不再比基于扫描的访问操作更快。这个查询执行的频率开始扮演重要的角色。如果你能在几秒钟内扫描一个数百万行的表，那无疑是快的。然而，如果你在一次合并操作中需要执行这个扫描 10,000 次，那么在查找表上加一个索引可能会加快速度。这确实是一个需要我们重新审视的问题：何时我们想使用索引，何时我们又期望全扫描表现更好。

当 Exadata 刚开始出现在客户现场时，我们常听到的一种说法是索引不再必要，应该被删除。对于纯粹的数据仓库工作负载，对于分析性索引来说，这可能确实是相当好的建议。然而，我们很少看到可以称之为“纯粹数据仓库”的工作负载。大多数系统都混合了多种访问模式，一组语句追求低延迟，而另一组则追求高吞吐量。在这些情况下，删除所有索引是行不通的。这就是为什么将讨论保留到本节的原因。

## 混合工作负载的问题

对于混合工作负载，即需要为特定语句集合保留索引的情况，其问题在于优化器在选择使用还是忽略它们方面，并不如人们期望的那样智能。不过，或许有一种方法可以绕过这种情况，那就是创造性地使用不可见索引。这个 11g 特性允许你在制定执行计划时对优化器隐藏索引。索引仍然存在并且也会被维护，因此你可能需要重新审视索引的使用情况。并非每个索引都需要删除，同样，也并非每个分析性索引都需要保留。

下面是一个相对简单的实现示例，展示了如何在数据库中保留索引，但只选择性地使用它们。`dbm01` 数据库已被修改，并创建并启动了两个服务。`DSSSRV`，顾名思义，是用户在进行决策支持查询或那些对吞吐量要求高、对低延迟需求较低的查询时应使用的服务。正如你所想，对于 `OLTPSRV` 则相反。通过该服务连接的用户非常关心低延迟，但对带宽不太在意。可以编写一个小的 PL/SQL 过程来检查会话使用了哪个服务连接，并更改参数 `optimizer_use_invisible_indexes`。表 `T1_WITH_INDEX` 上的索引是不可见的：

```sql
SQL> select index_name, visibility
  2  from user_indexes
  3  where index_name = 'I_T1_WITH_INDEXES_1';

INDEX_NAME                     VISIBILIT
------------------------------ ---------
I_T1_WITH_INDEXES_1            INVISIBLE
```

这里使用的小过程只是检查服务名称并更改优化器对该索引的可见性：

```sql
SQL> create procedure check_service is
  2  begin
  3   if lower(sys_context('userenv','service:name')) = 'dsssrv' then
  4    execute immediate 'alter session set optimizer_use_invisible_indexes = false';
  5   elsif lower(sys_context('userenv','service:name')) = 'oltpsrv' then
  6    execute immediate 'alter session set optimizer_use_invisible_indexes = true';
  7   end if;
  8  end;
  9  /

Procedure created.
```

## SQL 查询示例

以下的执行计划显示，根据会话连接所使用的服务，索引会被使用或忽略。

### 第一个 SQL 示例（OLTPSRV 连接）

第一个示例使用 `OLTPSRV` 连接：

```sql
SQL> select sys_context('userenv','service:name') from dual;

SYS_CONTEXT('USERENV','SERVICE_NAME')
-----------------------------------------------------------------------------------------
oltpsrv

SQL> exec check_service

SQL> select /* oltpsrv */ count(*) from t1_with_index where id between 200 and 400;

COUNT(*)
----------
201

SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
-----------------------------------------------------------------------------------------
SQL_ID  46auh11c8ddts, child number 0
-------------------------------------
select /* oltpsrv */ count(*) from t1_with_index where id between 200
and 400

Plan hash value: 2861271559

-----------------------------------------------------------------------------------------
| Id  | Operation         | Name                | Rows  | Bytes | Cost (%CPU)| Time     |
-----------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |                     |       |       |     3 (100)|          |
|   1 |  SORT AGGREGATE   |                     |     1 |     6 |            |          |
|*  2 |   INDEX RANGE SCAN| I_T1_WITH_INDEXES_1 |   202 |  1212 |     3   (0)| 00:00:01 |
-----------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - access("ID">=200 AND "ID"<=400)

20 rows selected.
```

注意这是由索引驱动的执行计划。

### 第二个 SQL 示例（DSSSRV 连接）

当通过 `DSSSRV` 连接时，情况发生了变化：

```sql
SQL> select sys_context('userenv','service:name') from dual;

SYS_CONTEXT('USERENV','SERVICE_NAME')
--------------------------------------------------------------------------------------------
dsssrv

Elapsed: 00:00:00.00

SQL> exec check_service

PL/SQL procedure successfully completed.

Elapsed: 00:00:00.01

SQL> select /* dsssrv */ count(*) from t1_with_index where id between 200 and 400;

COUNT(*)
----------
201

Elapsed: 00:00:00.56

SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------
SQL_ID  0ym2m0whwsn1y, child number 0
-------------------------------------
select /* dsssrv */ count(*) from t1_with_index where id between 200
and 400

Plan hash value: 1131101492

--------------------------------------------------------------------------------------------
| Id  | Operation                   | Name          | Rows  | Bytes | Cost (%CPU)| Time     |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT            |               |       |       |   452K(100)|          |
|   1 |  SORT AGGREGATE             |               |     1 |     6 |            |          |
|*  2 |   TABLE ACCESS STORAGE FULL | T1_WITH_INDEX |   202 |  1212 |   452K  (1)| 00:00:18 |
--------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   2 - storage(("ID"<=400 AND "ID">=200))
       filter(("ID"<=400 AND "ID">=200))

21 rows selected.
```

与前面的例子不同，你在执行计划中看不到任何索引。如果你考虑将那个小的 PL/SQL 块整理得更美观些，你可以轻松地将其嵌入到一个登录触发器中，并以这种方式控制索引的使用。Exadata 背景下优化器的其他局限性将在下一节中讨论。


### 优化器并不知情

你已经多次在本书中读到，优化器并不知晓它正在 Exadata 上运行。总体而言，指导优化器决策的原则是健全的，无论底层存储平台如何。数据库层的代码是相同的——无论它是否运行在 Exadata 上——这意味着应用程序在执行计划选择方面，在 Exadata 上会表现出相似的行为。因此，如果你停留在相同的版本和相同的内存设置上，你不应期望任何应用程序仅仅因为迁移到 Exadata 就经历大量执行计划的变更。使用相同的软件生成执行计划，对稳定性大有裨益！如果你是从较低的 Oracle 版本迁移到 Exadata，例如在 11.2 到 12.1 的迁移过程中，或者从单实例迁移到 RAC，情况可能会有所不同，但你也会在 Exadata 平台之外预期到类似的变化。

缺点在于，优化器并不知晓 Exadata 拥有一些优化措施，这些优化措施可以使全表扫描的性能远优于其他平台——除了你将在下一节读到的`EXADATA`系统统计信息。因此，拥有许多索引的混合负载系统使得优化器的工作更具挑战性。事实上，正如你可能预料的那样，在有索引可用的情况下，优化器倾向于优先选择基于索引的计划，而非基于全表扫描的计划，尽管基于全表扫描的计划往往快得多。

有几种方法可以应对优化器倾向于索引访问而非全表扫描的特性。系统统计信息、优化器参数和提示都作为潜在的解决方案浮现出来。你可以在以下章节中了解更多相关信息。

#### 系统统计信息

系统统计信息向优化器提供有关“系统”的额外信息，包括执行一次单块读（典型于索引查找）和一次多块读（典型于全表扫描）分别需要多长时间。这似乎是一种理想机制，可以通过提供优化器做出正确决策所需的额外信息来操控它。不幸的是，智能扫描并非基于传统的多块读；事实上，智能扫描可以比多块读快上几个数量级。因此，在这种情况下，修改系统统计信息可能并非最佳选择。

事实上，在讨论 Exadata 部署时，是否在`WORKLOAD`模式下收集系统统计信息的问题经常出现。基于上述原因，收集它们可能不是一个明智的想法，因为它可能引入计划退化。引入`WORKLOAD`统计信息也可能产生深远影响。

然而，根据`DOC ID 1274318.1`，对于数据库版本 11.2.0.2 `BP18` 和 11.2.0.3 `BP8` 及更新版本，存在另一种选择：以 Exadata 方式收集统计信息。My Oracle Support 上的文档明确指出，这不是一个通用建议，而是应该仔细评估。要启用 Exadata 系统统计信息，你可以使用以下命令：

```
SQL> exec DBMS_STATS.GATHER_SYSTEM_STATS('EXADATA')
```

作为此调用的结果，数据库引擎被告知它可以在 Exadata 上通过单次请求读取更多数据，从而降低全表扫描的成本。但这并不会阻止优化器选择索引。成本计算模型的变化正是为什么你应该在仔细测试后才引入此变更的原因！前述说明还建议，如果应用程序是从头开始在 Exadata 上开发的，那么收集`Exadata 统计信息`的效果可以更容易控制，任何不利的副作用都可以在上线前的测试中被发现。无论哪种方式——都需要仔细测试。

#### 优化器参数

有几个初始化参数可以推动优化器趋向或远离使用索引。参数 `OPTIMZER_INDEX_CACHING` 和 `OPTIMIZER_INDEX_COST_ADJ` 都可用于此目的。虽然这些是可能影响优化器核心功能的大型调控旋钮，但它们的设计目的正是为了让索引对优化器更具吸引力或更不具吸引力。在某些情况下，以有限的方式使用这些参数是可行的方法，例如在运行大型批处理过程之前，使用 `alter session` 命令。这些参数也可以使用 `OPT_PARAM` 提示在语句级别设置。以下是一个非常简单的例子：

```
SQL> show parameter optimizer_index

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
optimizer_index_caching              integer     0
optimizer_index_cost_adj             integer     100

SQL> select /*+ parallel(2) gather_plan_statistics monitor chap 17 -f */
  2    count(*), a.state
  3    from bigt a, t1_sml b
  4    where a.id = b.id
  5    and b.state = 'RARE'
  6    group by a.state
  7    /
no rows selected

Elapsed: 00:00:25.46

SQL> select * from table(dbms_xplan.display_cursor);

PLAN_TABLE_OUTPUT
--------------------------------------------------------------------------------------------
SQL_ID  4q6vqy2r1yn5w, child number 0
-------------------------------------
select /*+ parallel(2) gather_plan_statistics monitor chap 17 -f */
count(*), a.state from bigt a, t1_sml b where a.id = b.id and b.state =
'RARE' group by a.state

Plan hash value: 1484706486

--------------------------------------------------------------------------------------------
| Id  | Operation                              | Name          | Rows  | Bytes | Cost  |
--------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                       |               |       |       |   262K|
|   1 |  PX COORDINATOR                        |               |       |       |       |
|   2 |   PX SEND QC (RANDOM)                  | :TQ10003      |     1 |    32 |   262K|
|   3 |    HASH GROUP BY                       |               |     1 |    32 |   262K|
|   4 |     PX RECEIVE                         |               |     1 |    32 |   262K|
|   5 |      PX SEND HASH                      | :TQ10002      |     1 |    32 |   262K|
|   6 |       HASH GROUP BY                    |               |     1 |    32 |   262K|
|*  7 |        HASH JOIN                       |               |     1 |    32 |   262K|
|   8 |         JOIN FILTER CREATE             | :BF0000       |     8 |   128 |   262K|
|   9 |          PX RECEIVE                    |               |     8 |   128 |   262K|
|  10 |           PX SEND BROADCAST            | :TQ10001      |     8 |   128 |   262K|
|  11 |            TABLE ACCESS BY INDEX ROWID BATCHED| T1_SML       |     8 |   128 |     3 |
|  12 |             SORT CLUSTER BY ROWID      |               |     8 |       |     3 |
|  13 |              BUFFER SORT               |               |       |       |     3 |
|  14 |               PX RECEIVE               |               |     8 |       |     3 |
|  15 |                PX SEND HASH (BLOCK ADDRESS)   | :TQ10000      |     8 |       |     3 |
|  16 |                 PX SELECTOR            |               |       |       |     3 |
|* 17 |                  INDEX RANGE SCAN      | T1_SML_STATE  |     8 |       |     2 |
|  18 |         JOIN FILTER USE                | :BF0000       |   100M|  1525M|   262K|
|  19 |          PX BLOCK ITERATOR             |               |   100M|  1525M|   262K|
|* 20 |           TABLE ACCESS STORAGE FULL    | BIGT          |   100M|  1525M|   262K|
--------------------------------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
   7 - access("A"."ID"="B"."ID")
```


`17 - access("B"."STATE"='RARE')`

`20 - storage(:Z>=:Z AND :Z<=:Z AND SYS_OP_BLOOM_FILTER(:BF0000,"A"."ID"))`

`filter(SYS_OP_BLOOM_FILTER(:BF0000,"A"."ID"))`

`Note`

`-----`

`- 使用的动态统计信息：动态采样 (level=AUTO)`

`- 由于提示(Hint)，并行度为 2`

`SQL> alter session set optimizer_index_cost_adj=10000;`

`Session altered.`

`SQL> select /*+ parallel(2) gather_plan_statistics monitor chap` `17` `-f */`

`2  count(*), a.state`

`3  from bigt a, t1_sml b`

`4  where a.id = b.id`

`5  and b.state = 'RARE'`

`6  group by a.state`

`7  /`

`no rows selected`

`Elapsed: 00:00:15.40`

`SQL> select * from table(dbms_xplan.display_cursor);`

`PLAN_TABLE_OUTPUT`

`------------------------------------------------------------------------------------`

`SQL_ID  4q6vqy2r1yn5w, child number 1`

`-------------------------------------`

`select /*+ parallel(2) gather_plan_statistics monitor chap` `17` `-f */`

`count(*), a.state from bigt a, t1_sml b where a.id = b.id and b.state =`

`'RARE' group by a.state`

`Plan hash value: 3199786897`

`------------------------------------------------------------------------------------`

`| Id  | Operation                           | Name     | Rows  | Bytes | Cost (%CPU)|`

`------------------------------------------------------------------------------------`

`|   0 | SELECT STATEMENT                    |          |       |       |  2510K(100)|`

`|   1 |  PX COORDINATOR                     |          |       |       |            |`

`|   2 |   PX SEND QC (RANDOM)               | :TQ10001 |     1 |    32 |  2510K  (1)|`

`|   3 |    HASH GROUP BY                    |          |     1 |    32 |  2510K  (1)|`

`|   4 |     PX RECEIVE                      |          |     1 |    32 |  2510K  (1)|`

`|   5 |      PX SEND HASH                   | :TQ10000 |     1 |    32 |  2510K  (1)|`

`|   6 |       HASH GROUP BY                 |          |     1 |    32 |  2510K  (1)|`

`|*  7 |        HASH JOIN                    |          |     1 |    32 |  2510K  (1)|`

`|   8 |         JOIN FILTER CREATE          | :BF0000  |     8 |   128 |   137   (4)|`

`|*  9 |          TABLE ACCESS STORAGE FULL  | T1_SML   |     8 |   128 |   137   (4)|`

`|  10 |         JOIN FILTER USE             | :BF0000  |   100M|  1525M|  2510K  (1)|`

`|  11 |          PX BLOCK ITERATOR          |          |   100M|  1525M|  2510K  (1)|`

`|* 12 |           TABLE ACCESS STORAGE FULL | BIGT     |   100M|  1525M|  2510K  (1)|`

`------------------------------------------------------------------------------------`

`Predicate Information (identified by operation id):`

`---------------------------------------------------`

`7 - access("A"."ID"="B"."ID")`

`9 - storage("B"."STATE"='RARE')`

`filter("B"."STATE"='RARE')`

`12 - storage(:Z>=:Z AND :Z<=:Z AND SYS_OP_BLOOM_FILTER(:BF0000,"A"."ID"))`

`filter(SYS_OP_BLOOM_FILTER(:BF0000,"A"."ID"))`

`Note`

`-----`

`- 使用的动态统计信息：动态采样 (level=AUTO)`

`- 由于提示(Hint)，并行度为 2`

在这个简单的例子中，通过`alter session`将优化器推离索引，导致优化器选择了一个明显更快的执行计划。计划显示，运行时间缩短的结果是进行了全表扫描，而不是使用索引。

#### 提示（Hints）

当然，提示也可以用来帮助优化器做出正确的选择，但这多少有些棘手。在前面提到的混合负载场景中尤其如此。尽管如此，告诉 Oracle 你希望执行哈希连接或忽略某个特定索引，是一个可行的选项。正如前一节提到的，`OPT_PARAM`提示对于设置一些可能影响优化器决策的初始化参数也很有用。SQL 补丁可以通过向你无法控制的代码中注入提示来提供帮助。在获得修复方案之前，Oracle 12c 引入的自适应优化应该会减少使用提示来影响连接方法的必要性。

### 使用资源管理器（Resource Manager）

不幸的是，人们仍普遍认为 Oracle 数据库无法配置成同时充分处理数据仓库（DW）和在线事务处理（OLTP）工作负载。而且，确实，将它们放在不同的系统上确实使它们更易于管理。这种方法的缺点是成本高昂。许多公司将其大部分计算资源用于在平台之间移动数据。Exadata 的强大功能使得将这些环境结合起来颇具吸引力。请记住，Exadata 具有在其他平台上不可用的、用于在多个数据库之间划分资源的附加功能。I/O 资源管理器可以防止长时间运行的数据仓库查询削弱在同一系统上运行的、对延迟敏感的语句的性能。充分理解 Oracle 的资源管理功能，应该会改变你对在混合负载或整合环境中什么是可能的看法。资源管理在第 7 章中有深入介绍。

## 总结

Exadata 不同于传统部署的 Oracle 数据库。为了充分利用它，你需要以不同的方式思考。这并不意味着将应用程序迁移到 Exadata 时必须重写它，但这是一个对其进行整体审查的好机会。在当今世界，DBA 管理着数十或数百个数据库的情况相当常见。“了解你的数据”在这种情况下往往成了一种奢望。DBA 可能被分配一个需要关闭的问题工单，而大量的待处理工单常常使得只要系统“平稳运行”，就无法对根本原因进行更深入的分析。

当决定将数据库迁移到 Exadata 时，这个决定通常意味着升级到更新的 Oracle 版本。平台变更和数据库版本变更是审查数据库环境以获取进一步性能提升的最佳时机。如果可能，并且你在迁移到 Exadata 时没有面临巨大的时间压力，我们鼓励你不要在迁移成功后就停止对系统的工作，而是继续拓展可能的边界。当使用智能扫描（Smart Scans）时，Exadata 系统非常强大，你应该在可行且有意义的地方利用这种性能。



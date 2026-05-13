# 破解索引神话

## 神话：索引中的空间永远不会被重用

我想一劳永逸地破除这个神话：索引中的空间*是*会被重用的。这个神话是这样的：你有一个表`T`，其中有一列`X`。你在某个时间点向表中插入了值`X=5`。之后你删除了它。神话认为，除非你后来将`X=5`重新插入索引，否则`X=5`所使用的空间将永远不会被重用。该神话声称，一旦一个索引槽位被使用，它将永远存在，并且只能被相同的值重用。一个推论是，空闲空间永远不会返还给索引结构，并且块永远不会被重用。这同样不是真的。

这个神话的第一部分很容易被证伪。我们只需要创建一个像这样的表：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t ( x int, constraint t_pk primary key(x) );
Table created.
SQL> insert into t values (1);
1 row created.
SQL> insert into t values (2);
1 row created.
SQL> insert into t values (9999999999);
1 row created.
SQL> analyze index t_pk validate structure;
Index analyzed.
SQL> select lf_blks, br_blks, btree_space from index_stats;
LF_BLKS    BR_BLKS BTREE_SPACE
---------- ---------- -----------
1          0        7996
```

那么，根据这个神话，如果我从`T`中删除`X=2`，除非我重新插入数字 2，否则该空间将永远不会被重用。目前，这个索引使用了一个叶子块的空间。如果索引键条目在删除后永不被重用，并且我持续插入和删除且从不重用一个值，那么这个索引应该会疯狂地增长。让我们看看：

```
SQL> begin
for i in 2 .. 999999
loop
delete from t where x = i;
commit;
insert into t values (i+1);
commit;
end loop;
end;
/
PL/SQL procedure successfully completed.
SQL> analyze index t_pk validate structure;
Index analyzed.
SQL> select lf_blks, br_blks, btree_space from index_stats;
LF_BLKS    BR_BLKS BTREE_SPACE
---------- ---------- -----------
1          0        7996
```

这表明索引中的空间被重用了。然而，像大多数神话一样，其中也包含一粒事实的真相。真相是，最初那个数字 2 所使用的空间将永远保留在那个索引块上。索引不会*合并*自己。这意味着，如果我向表中加载值 1 到 500,000，然后每隔一行删除一次（所有偶数），那么该列的索引中将会有 250,000 个空洞。只有当我重新插入的数据能够放入有空洞的块中时，空间才会被重用。Oracle 不会尝试收缩或压缩索引。这可以通过`ALTER INDEX REBUILD`或`COALESCE`命令来完成。另一方面，如果我向表中加载值 1 到 500,000，然后从表中删除所有值小于等于 250,000 的行，我会发现那些从索引中清除出来的块被放回了索引的`FREELIST`上。这些空间可以完全被重用。

## 神话：区分度最高的元素应该放在最前面

这似乎是常识。如果你要在表`T`的列`C1`和`C2`上创建索引，该表有 100,000 行，你发现`C1`有 100,000 个不同值，`C2`有 25,000 个不同值，你会希望在`T(C1,C2)`上创建索引。这意味着`C1`应该排在前面，这是常识性的做法。事实是，在比较数据向量时（将`C1, C2`视为一个向量），你把哪个放在前面并不重要。考虑下面的例子。我们将基于`ALL_OBJECTS`创建一个表，并在`OWNER`、`OBJECT_TYPE`和`OBJECT_NAME`列（从区分度最低到最高）以及`OBJECT_NAME`、`OBJECT_TYPE`和`OWNER`上创建索引：

```
$ sqlplus eoda/foo@PDB1
SQL> create table t as select * from all_objects;
Table created.
SQL> create index t_idx_1 on t(owner,object_type,object_name);
Index created.
SQL> create index t_idx_2 on t(object_name,object_type,owner);
Index created.
SQL> select count(distinct owner), count(distinct object_type), count(distinct object_name ), count(*)
from t;
COUNT(DISTINCTOWNER) COUNT(DISTINCTOBJECT_TYPE)
COUNT(DISTINCTOBJECT_NAME)   COUNT(*)
------------------ ---------------------- ---------------------- ----------
34                     36                  30813      50253
```

现在，为了说明两者在空间效率上没有差异，我们将测量它们的空间利用率：

```
SQL> analyze index t_idx_1 validate structure;
Index analyzed.
SQL> select btree_space, pct_used, opt_cmpr_count, opt_cmpr_pctsave
from index_stats;
BTREE_SPACE  PCT_USED OPT_CMPR_COUNT OPT_CMPR_PCTSAVE
----------- --------- -------------- ----------------
2526832        89              2               28
SQL> analyze index t_idx_2 validate structure;
Index analyzed.
SQL> select btree_space, pct_used, opt_cmpr_count, opt_cmpr_pctsave
2 from index_stats;
BTREE_SPACE  PCT_USED OPT_CMPR_COUNT OPT_CMPR_PCTSAVE
----------- --------- -------------- ----------------
2510776        90              0                0
```

它们使用的空间几乎完全相同——在这方面没有重大差异。然而，如果我们使用索引键压缩，第一个索引的*可压缩性*要高得多，正如`OPT_CMPR_PCTSAVE`值所证明的那样。有一个论点支持将索引中的列按从区分度最低到最高的顺序排列。现在让我们看看它们的性能如何，以确定哪个索引总体上更高效。为了测试这一点，我们将使用一个带有提示查询的 PL/SQL 块（以便使用一个索引或另一个索引）：

```
SQL> alter session set sql_trace=true;
Session altered.
SQL> declare
cnt int;
begin
for x in ( select /*+FULL(t)*/ owner, object_type, object_name from t )
loop
select /*+ INDEX( t t_idx_1 ) */ count(*) into cnt
from t
where object_name = x.object_name
and object_type = x.object_type
and owner = x.owner;
select /*+ INDEX( t t_idx_2 ) */ count(*) into cnt
from t
where object_name = x.object_name
and object_type = x.object_type
and owner = x.owner;
end loop;
end;
/
PL/SQL procedure successfully completed.
```

这些查询通过索引读取表中的每一行。`TKPROF`报告显示了以下内容：



```
SELECT /*+ INDEX( t t_idx_1 ) */ COUNT(*) FROM T
WHERE OBJECT_NAME = :B3 AND OBJECT_TYPE = :B2 AND OWNER = :B1
调用         次数       CPU 耗时    耗时       磁盘读    查询一致性    当前模式    处理行数
-------    ------      ------     ---------   --------   ----------   ----------   ----------
解析           1         0.00       0.00           0            0            0            0
执行       50253         0.69       0.67           0            0            0            0
获取       50253         0.46       0.49           0       100850            0        50253
-------    ------      ------     ---------   --------   ----------   ----------   ----------
总计      100507         1.15       1.16           0       100850            0        50253
...
首次行数   平均行数   最大行数   行源操作
---------- ---------- ----------  -----------------------------------------
1          1          1 排序聚合 (cr=2 pr=0 pw=0 time=16 us)
1          1          1  索引范围扫描 T_IDX_1 (cr=2 pr=0 pw=0 time=12...
***************************************************************************
SELECT /*+ INDEX( t t_idx_2 ) */ COUNT(*) FROM T
WHERE OBJECT_NAME = :B3 AND OBJECT_TYPE = :B2 AND OWNER = :B1
调用         次数       CPU 耗时    耗时       磁盘读    查询一致性    当前模式    处理行数
-------    ------      ------     ---------   --------   ----------   ----------   ----------
解析           1         0.00       0.00           0            0            0            0
执行       50253         0.68       0.66           0            0            0            0
获取       50253         0.48       0.48           0       100834            0        50253
-------    ------      ------     ---------   --------   ----------   ----------   ----------
总计      100507         1.16       1.15           0       100834            0        50253
...
首次行数   平均行数   最大行数    行源操作
---------- ---------- ---------- -----------------------------------------
1          1          1 排序聚合 (cr=2 pr=0 pw=0 time=13 us)
1          1          1 索引范围扫描 T_IDX_2 (cr=2 pr=0...
```

它们处理了完全相同的行数和非常相似的数据块数量（细微差异来自于表中行的偶然排序以及 Oracle 所做的相应优化），消耗了等量的 CPU 时间，并且运行的耗时大致相同（再次运行此相同测试，`CPU`和`ELAPSED`数字会略有不同，但平均值会是一样的）。将索引列按照其区分度顺序放置并不能带来固有的效率提升，而且如前所述，使用索引键压缩时，反而有理由将选择性最低的列放在前面。如果你在索引上使用`COMPRESS 2`运行前面的示例，鉴于本例中查询的特性，你会发现第一个索引的 I/O 量大约是第二个索引的三分之二。

然而，事实是，将列`C1`置于`C2`之前的决定*必须*由索引的使用方式来驱动。如果你有许多类似下面的查询，将索引建在`T(C2,C1)`上更有意义：

```
select * from t where c1 = :x and c2 = :y;
select * from t where c2 = :y;
```

这单个索引可以被上述任何一个查询使用。此外，使用索引键压缩（我们已在讨论 IOT 时涉及，稍后将进一步探讨），如果`C2`列在前，我们可以构建一个更小的索引。这是因为在索引中，`C2`的每个值平均会重复四次。如果`C1`和`C2`的平均长度都是 10 字节，该索引的条目名义上为 2,000,000 字节（100,000 × 20）。对`(C2, C1)`使用索引键压缩，我们可以将此索引缩小到 1,250,000 字节（100,000 × 12.5），因为`C2`四分之三的重复值可以被抑制。

在 Oracle 5（是的，版本 5）中，确实有理由将最具区分度的列放在索引前面。这与版本 5 实现索引压缩（不同于索引键压缩）的方式有关。该特性在版本 6 中随着行级锁的引入而被移除。自那时起，将最具区分度的条目放在索引前面会使索引更小或更高效的说法就不再成立了。它看起来似乎会，但实际上不会。有了索引键压缩，就有充分的理由反其道而行之，因为它可以使索引更小。然而，如前所述，这应该由你*如何*使用索引来驱动。

## 自动索引

从 19c 版本开始，Oracle 数据库附带了*自动索引*功能。这允许你将一些索引管理功能委托给数据库。此功能根据你的应用程序工作负载来创建、重建和删除索引。由此功能管理的索引被称为*自动索引*。以下是自动索引的主要方面：

*   识别可能改善 SQL 语句性能的、待索引的列。这些被称为候选索引。
*   自动创建带有`SYS_AI`前缀的不可见索引。这些新索引被创建为不可见，因此优化器最初不会考虑使用它们。
*   测试不可见索引以确保它们能改善 SQL 语句的性能。如果性能得到提升，则索引将变为可见。如果没有性能增益，则索引将被标记为不可用，并在之后被删除。
*   自动删除未使用的索引。

注意
自动索引目前仅在安装于 Oracle Exadata 机器上的企业版中可用。在非 Exadata 硬件系统上，如果你尝试执行与此功能相关的包，将会收到`ORA-40216: feature not supported`消息。对于非 Exadata 系统，你可以通过将隐藏参数`_exadata_feature_on`设置为`TRUE`来启用此功能。但是，你应该仅在获得 Oracle 许可的情况下设置此参数，因为这涉及到许可问题。

一旦启用了自动索引，数据库将管理索引的创建、重建和删除。启用此功能后，DBA 的工作将是：

*   通过设置各种参数来管理配置
*   审查系统生成的报告

介绍完毕，接下来让我们谈谈 DBA 需要做些什么来管理和配置自动索引。你会高兴地看到，作为 Oracle DBA，你的工作并没有被完全取代。



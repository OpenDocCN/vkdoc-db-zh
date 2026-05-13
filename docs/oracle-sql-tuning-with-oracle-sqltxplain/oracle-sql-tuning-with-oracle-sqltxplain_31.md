# 第六章

![image](img/frontdot.jpg)

## 通过概要文件强制执行计划

有时，最好的做法是先修复问题，之后再去分析哪里出了错并找出根本原因。我指的是，当面临生产时间压力需要完成某项任务时，任何有效的方法都足够好。核心思路是让业务先恢复并运行起来，然后可以在闲暇时坐下来研究问题所在，弄清楚如何防止其再次发生，并制定一个计划，在需要的情况下，于计划内的停机期间以可控的方式实施永久性修复。管理层通常对那些想在行动前弄清问题及其解决方案的每个细节的完美主义者并不十分宽容；如果他们表现出宽容，那通常是因为他们口袋里还有另一个系统维持着灯火通明和业务运转。

我即将描述的场景虽然罕见，但确实会发生。一个夜间批处理作业——一个对时间要求严格（必须在某个时间前完成）的作业——突然开始运行得久得多。联机业务时段受到影响，管理报告未能按时生成（甚至完全没有生成），数据库管理员和开发人员对此束手无策。什么都没变过（这从来都不是真的），没有向生产环境发布新版本，没有参数更改，没有统计信息收集的变更。这时，你常常会听到那句无奈的呼喊：“它到底在搞什么？”

当你想正儿八经地修复一个问题时，这是一个好问题；但当你需要系统恢复到之前的行为模式时，有一条捷径。这种不理解问题本身而直接修复问题的技巧，常被称为“一级严重性杀手”。这是因为 Oracle 的一级严重性服务请求是最高优先级的：如果有办法解决问题，那就尽管去做。理解可以留到以后。

那么，该怎么做呢？`SQLT` 拥有完成这项工作的工具（大部分时候）。作为 `SQLT` 操作流程的一部分，`SQLT` 会生成一个使用 SQL 概要文件的脚本。这个 SQL 概要文件（如果它对应的是那个“良好”的执行计划）可以被应用到有问题的 SQL 上，从而强制该 SQL “正确”运行，无论系统上发生了什么其他情况（错误的统计信息、更改的参数等）。尽管引入了 SQL 计划管理，SQL 概要文件仍然有用。不过，在未来的 Oracle 版本中，可以期待 SQL 计划基线成为一个更好的产品。现在，让我们先看看如何利用 SQL 概要文件。

### 什么是 SQL 概要文件？

尽管本章标题是“通过概要文件强制执行计划”，但 SQL 概要文件并不完全等同于强制一个执行计划。它们更像是“超级提示”。使用普通提示时，优化器总有可能找到另一种执行查询的方式，而你的提示并非你所愿。另一方面，超级提示对执行计划施加了如此多的约束，以至于优化器（通常）别无选择。“概要文件”这个词在这里相当形象。优化器见过这条 SQL 语句多次通过，并确定了它的某种行为模式：它以特定的顺序访问行，并始终使用某个特定的索引。所有这些概要信息都被收集并为该查询保留，如果它的新行为不理想，就可以用这些信息在未来“强制”该语句以相同方式行为。

SQL 概要文件会是坏事吗？让我们用一个类比来探讨。每个人都有自己坚持的日常惯例。起床、刷牙、吃早餐、上班、开车回家、看报纸等等。这种“概要文件”相当简单。确保上班前刷牙，否则你可能得回家来刷。如果有人抹去了你的记忆（也许是一本调优的书从架子上掉下来砸到了你的头），你忘记了自己的日常惯例，在记忆恢复之前，你可能会想要一份关于你先前惯例的提醒。如果其他情况都没变，这种强制的惯例，或者说“概要文件”，可能是件好事。然而，假设你还搬了新家，现在在家工作。你的概要文件现在可能就完全过时了。开车上班不再有意义。由此可见，概要文件在短期内让一切恢复正常可能是好的，但时不时你需要检查它是否仍然合理。

那么，一个超级提示长什么样呢？这里有一个：

```
h := SYS.SQLPROF_ATTR(
  '[BEGIN_OUTLINE_DATA]',
  '[IGNORE_OPTIM_EMBEDDED_HINTS]',
  '[OPTIMIZER_FEATURES_ENABLE('11.2.0.1')]',
  '[DB_VERSION('11.2.0.1')]',
  '[ALL_ROWS]',
  '[OUTLINE_LEAF(@"SEL$E9AF9BDE")]',
  '[MERGE(@"SEL$CE1D94FA")]',
  '[OUTLINE(@"SEL$98588D1C")]',
  '[UNNEST(@"SEL$6")]',
  '[OUTLINE(@"SEL$CE1D94FA")]',
  '[OUTLINE(@"SEL$C50F9DEF")]',
  '[OUTLINE(@"SEL$6")]',
  '[OUTLINE(@"SEL$81719215")]',
  '[MERGE(@"SEL$EE94F965")]',
  '[OUTLINE(@"SEL$5")]',
  '[OUTLINE(@"SEL$EE94F965")]',
  '[MERGE(@"SEL$9E43CB6E")]',
  '[OUTLINE(@"SEL$4")]',
  '[OUTLINE(@"SEL$9E43CB6E")]',
  '[MERGE(@"SEL$58A6D7F6")]',
  '[OUTLINE(@"SEL$3")]',
  '[OUTLINE(@"SEL$58A6D7F6")]',
  '[MERGE(@"SEL$1")]',
  '[OUTLINE(@"SEL$2")]',
  '[OUTLINE(@"SEL$1")]',
  '[INDEX_RS_ASC(@"SEL$E9AF9BDE" "TABLE1"@"SEL$4" "TABLE1"."TABLE_NAME" "TABLE1"."STATUS" "TABLE1"."CATEGORY"))]',
  '[INDEX_RS_ASC(@"SEL$E9AF9BDE" "TABLE2"@"SEL$3" ("TABLE2"."COL2" TABLE2"."COL1" "TABLE2"."COL3"))]',
  '[INDEX(@"SEL$E9AF9BDE" "TABLE3"@"SEL$2" ("TABLE3"."COL2"  TABLE3"."COL1"))]',
  '[FULL(@"SEL$E9AF9BDE" "TABLE4"@"SEL$1")]',
  '[INDEX_RS_ASC(@"SEL$E9AF9BDE" "TABLE1"@"SEL$6" ("TABLE1"."TABLE_NAME" "TABLE1"."STATUS" "TABLE1"."CATEGORY"))]',
  '[INDEX_RS_ASC(@"SEL$E9AF9BDE" "TABLE2"@"SEL$6" ("TABLE2"."COL2" "TABLE2"."COL1" "TABLE2"."COL3"))]',
  '[LEADING(@"SEL$E9AF9BDE" "TABLE1"@"SEL$4" "TABLE2"@"SEL$3" "TABLE3"@"SEL$2" "TABLE4"@"SEL$1" "TABLE1"@"SEL$6" "TABLE2"@"SEL$6")]',
  '[USE_NL(@"SEL$E9AF9BDE" "TABLE2"@"SEL$3")]',
  '[USE_NL(@"SEL$E9AF9BDE" "TABLE3"@"SEL$2")]',
  '[NLJ_BATCHING(@"SEL$E9AF9BDE" "TABLE3"@"SEL$2")]',
  '[USE_HASH(@"SEL$E9AF9BDE" "TABLE4"@"SEL$1")]',
  '[USE_MERGE_CARTESIAN(@"SEL$E9AF9BDE" "TABLE1"@"SEL$6")]',
  '[USE_NL(@"SEL$E9AF9BDE" "TABLE2"@"SEL$6")]',
  '[PX_JOIN_FILTER(@"SEL$E9AF9BDE" "TABLE4"@"SEL$1")]',
  '[USE_HASH_AGGREGATION(@"SEL$E9AF9BDE")]',
  '[END_OUTLINE_DATA]');
```

注意这个提示比你通常用于调优 SQL 语句的普通提示要冗长得多。尽管如此，在标准的前导部分（包括 `BEGIN_OUTLINE_DATA`、`IGNORE_OPTIM_EMBEDDED_HINTS`、`OPTIMIZER_FEATURES_ENABLE('11.2.0.1')` 和 `DB_VERSION('11.2.0.1')`）之后，仍然有一些可识别的部分。让我们列出更熟悉的部分：

*   `ALL_ROWS`
*   `MERGE`
*   `INDEX_RS_ASC`
*   `FULL`
*   `USE_NL`
*   `NLJ_BATCHING`
*   `PX_JOIN_FILTERING`
*   `USE_MERGE_CARTESIAN`

你还会注意到，引用的是诸如 `SEL$3` 和 `SEL$E9AF9BDE` 这样晦涩的对象（它们被正式称为“查询块名”）。记住，优化器在进行更详细的成本基准调优之前，已经对 SQL 进行了转换（通过使用如第五章所述的查询转换）。在转换过程中，查询的各个部分会获得这些奇特的名称（它们是难以识别的）。概要文件提示随后就引用了这些名称。



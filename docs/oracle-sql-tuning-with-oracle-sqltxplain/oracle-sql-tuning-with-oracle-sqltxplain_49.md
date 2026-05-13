# Oracle CBO 隐藏参数指南

## 隐藏参数介绍
有些错误可以通过将此隐藏参数设置为 `FALSE` 来规避。其默认值是 `TRUE`。仅仅因为在执行计划中看到哈希连接并且语句崩溃，并不意味着将此参数设置为 `false` 就是解决方案。请提交服务请求并向支持团队提供相关信息。可能存在其他因素。将此值设置为 `FALSE` 将禁用哈希连接。

## _optimizer_extended_cursor_sharing_rel
我们已在第 7 章讨论了禁用自适应游标共享。我们通过以下命令禁用 ACS：

```sql
SQL> alter system set "_optimizer_extended_cursor_sharing_rel"=NONE scope=both;
SQL> alter system set "_optimizer_extended_cursor_sharing"=none scope=both;
```

这两个参数都需要禁用 ACS。`_optimizer_extended_cursor_sharing_rel` 的默认值是 `SIMPLE`，`_optimizer_extended_cursor_sharing` 的默认值是 `UDO`。

## _optimizer_cartesian_enabled
如果设置为 `FALSE`，则禁用笛卡尔连接。这可能是一种有用的调试方法，用于在 Oracle 支持人员的帮助下调试失败的语句。默认值是 `TRUE`。

## _optimizer_cost_model
此参数改变了 CBO 对活动进行成本估算的基础。它可以设置为 `cpu`、`io` 或 `choose`。这个名字有点误导性：通过设置 `cpu`，我们并不是在优化以减少 CPU 使用率，我们仍然是在像以前一样优化以减少成本，但现在将 CPU 时间纳入了 I/O 操作的计算中。`choose` 值允许优化器根据 `sys.aux_stats$` 中可用的统计信息进行选择。默认值是 `choose`。

## _optimizer_ignore_hints
这个隐藏参数允许优化器忽略嵌入的提示。默认值是 `FALSE`。如果您觉得提示没有产生最佳计划，并想在会话级别禁用它们，可以尝试此操作：

```sql
SQL> alter session set "_optimizer_ignore_hints"=TRUE;
```

## _optimizer_max_permutations
如果您觉得优化器工作得不够努力，总可以选择将 `_optimizer_max_permutations` 设置为默认值 2,000 以外的值。此参数控制优化器在连接多个表时，每个查询块将尝试的不同排列组合数量。在某些情况下，此值可以高达 80,000。该值会被 `_optimizer_search_limit` 覆盖，其中 `_optimizer_search_limit` 是连接排列的阶乘数。

## _optimizer_use_feedback
控制第 8 章讨论的基数反馈功能。默认为 `TRUE`。设置为 `FALSE` 以禁用此功能。

## _optimizer_search_limit
此参数的默认值是 `5`。这是将考虑的最大笛卡尔连接的阶乘数。5!（读作“5 的阶乘”）相当于 5x4x3x2x1，等于 120。

### 全参数列表
为什么包含一个除非 Oracle 支持人员指示否则我们作为 DBA 不应更改的参数的完整列表？有两个可能的答案。

*   Oracle 支持可能会为某些调查建议某个参数，能够至少对参数的作用有一个简要描述是有用的。无论如何，在应用于任何数据库之前，您都应该向 Oracle 支持人员索要此描述。此列表是您获取此信息的后备。
*   仔细阅读并研究这些参数中的每一个，或者在可丢弃的数据库中进行尝试，可以让您深入了解优化器的工作原理及其为我们所做的事情。这种更深入的理解会成为一个巩固关于优化器某些方面清晰知识的故事。`_optimizer_max_permutations` 就是一个例子。在我遇到这个参数之前，我很高兴地假设优化器尝试了 *所有* 连接排列。稍加思考就会让我摒弃这种看法。但现在我知道连接选择的数量是有限制的，以及如何控制它。

以下是 CBO 参数列表（版本 11.2.0.1），是本附录开头讨论的 SQLT 查询的示例输出：

```sql
SQL> select name, description from sqlt$_v$parameter_cbo order by name;
```




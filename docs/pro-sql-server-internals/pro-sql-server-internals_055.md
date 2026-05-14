# 第 4 章 ■ 特殊索引与存储特性

这种显著的性能影响主要与用户定义函数调用的开销有关。然而，即使不使用用户定义函数，仅由于计算本身，你仍会受到性能影响（尽管影响范围较小）。

■ **注意** 我们将在[第 11 章](http://dx.doi.org/10.1007/978-1-4842-1964-5_11)“用户定义函数”中更深入地讨论用户定义函数及其性能影响。

使用用户定义函数的计算列会阻止查询优化器生成并行执行计划，即使查询并未引用它们。这是查询优化器的设计限制之一。如果我们运行清单 4-24 所示的查询，就可以看到这种行为。代码使用了未文档化的跟踪标志 `T8649`，该标志在可能的情况下强制 SQL Server 生成并行执行计划。和往常一样，请谨慎使用未文档化的跟踪标志，不要在生产环境中使用它们。

### 清单 4-24. 计算列与并行执行计划

```
select count(*) from dbo.NonPersistedColumn option (querytraceon 8649);
select count(*) from dbo.PersistedColumn option (querytraceon 8649);
select count(*) from dbo.InputData option (querytraceon 8649);
```

正如你在图 4-13 中看到的，SQL Server 唯一能够生成并行执行计划的情况是在没有计算列的表上。值得一提的是，SQL Server 能够为




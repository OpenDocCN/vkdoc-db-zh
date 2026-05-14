# 第 17 章：查询重编译

在清除了过程缓存并且现有的计划指南 `MyGoodSOQLGuide` 就位后，重新运行查询。它将使用计划指南来生成如图 17-20 所示的执行计划。为了确认此计划是否可以被保留，首先删除那个强制了索引查找操作的计划指南。

```sql
EXECUTE sp_control_plan_guide
    @operation = 'Drop',
    @name = N'MyGoodSQLGuide' ;
```

如果此时重新运行查询，它将恢复到其原始的执行计划。然而，当前在计划缓存中，保存着如图 17-20 所示的计划。要保留它，请运行以下脚本：

```sql
DECLARE @plan_handle VARBINARY(64),
        @start_offset INT ;

SELECT @plan_handle = deqs.plan_handle,
       @start_offset = deqs.statement_start_offset
FROM   sys.dm_exec_query_stats AS deqs
       CROSS APPLY sys.dm_exec_sql_text(sql_handle)
       CROSS APPLY sys.dm_exec_text_query_plan(deqs.plan_handle,
                                              deqs.statement_start_offset,
                                              deqs.statement_end_offset) AS qp
WHERE  text LIKE N'SELECT soh.SalesOrderNumber%'

EXECUTE sp_create_plan_guide_from_handle
    @name = N'ForcedPlanGuide',
    @plan_handle = @plan_handle,
    @statement_start_offset = @start_offset ;
GO
```

[www.it-ebooks.info](http://www.it-ebooks.info/)

这将根据当前缓存中的执行计划创建一个计划指南。为确保其有效，请再次清除缓存。这样，查询将必须生成一个新的执行计划。重新运行查询，并观察执行计划。由于使用了 `sp_create_plan_guide_from_handle` 创建的计划指南，它将与图 17-20b 中显示的计划相同。

计划指南是控制 SQL 查询和存储过程行为的有效机制，但你只应在对执行计划、数据以及系统结构有透彻理解时才使用它们。

## 总结

正如你在本章所学，查询重编译既可能有益也可能损害性能。如果重编译能生成更好的执行计划，就会提升存储过程的性能。然而，如果重编译只是重新生成了相同的计划，那么它只会消耗额外的 CPU 周期，而不会改进处理策略。因此，你应该仔细审视重编译，以确定其有用性。你可以使用扩展事件来识别是哪条存储过程语句引起了重编译，并且可以从扩展事件输出中的 `recompile_clause` 数据列值来确定原因。一旦确定了重编译的原因，你就可以应用不同的技术来避免不必要的重编译。

到目前为止，你已经了解了如何从正确的索引和计划缓存中获益。然而，这些技术带来的性能提升取决于查询的设计方式。SQL Server 的基于成本的优化器会处理许多查询设计问题。但在设计查询时，你仍应遵循一系列最佳实践。在下一章中，我将介绍一些影响性能的常见查询设计问题。

[www.it-ebooks.info](http://www.it-ebooks.info/)


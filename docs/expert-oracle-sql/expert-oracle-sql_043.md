# 第四章

![image](img/frontdot.jpg)

## 运行时引擎

第三章侧重于 CBO 制定的计划。现在是时候看看这些计划在实际中如何运作了。我将解释收集和分析运行时数据的传统方法，以及一种更新的、互补的方法，称为 SQL 性能监视器。要充分理解 SQL 性能，理解工作区（workareas）的作用以及运行时引擎用于最小化开销的捷径非常重要，这两个主题都将在本章中介绍。

## 收集操作级别的运行时数据

正如你将在第五章中学到的，我们几乎总是需要获取 SQL 语句在操作级别的行为分解，以确定如何最好地优化该 SQL 语句。可以收集的一些操作级统计信息包括：

*   该操作在语句执行过程中运行了多少次
*   运行了多长时间
*   执行了多少次磁盘操作
*   进行了多少次逻辑读取
*   该操作是否完全在内存中执行

这些统计信息默认情况下不会收集。有三种方法可以触发收集：

*   在你的 SQL 语句中添加 `GATHER_PLAN_STATISTICS` 优化器提示：收集将仅针对该语句进行。
*   将初始化参数 `STATISTICS_LEVEL` 更改为 `ALL`：如果在会话级别设置，则该会话的所有 SQL 语句都会受到影响。如果 `STATISTICS_LEVEL` 在系统级别设置为 `ALL`，则实例中所有会话的所有 SQL 语句都将启用收集。
*   启用 SQL 跟踪：与 `STATISTICS_LEVEL` 参数一样，SQL 跟踪可以在会话或系统级别启用。

让我们依次看看这些选项。

## GATHER_PLAN_STATISTICS 提示

这个优化器提示已经存在很长时间了，但在 Oracle Database 12cR1 中才被首次记录。清单 4-1 通过添加 `GATHER_PLAN_STATISTICS` 提示更新了清单 3-2。

清单 4-1. 在语句级别收集操作数据

```sql
SELECT /*+ gather_plan_statistics */
       'Count of sales: ' || COUNT (*) cnt
  FROM sh.sales s JOIN sh.customers c USING (cust_id)
 WHERE cust_last_name = 'Ruddy';
```

请注意，与所有优化器提示一样，你可以使用大写或小写。

## 设置 STATISTICS_LEVEL=ALL




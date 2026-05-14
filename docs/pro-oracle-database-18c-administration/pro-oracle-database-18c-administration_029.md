# 物化视图的维护操作

## 检查刷新能力
以下表格展示了刷新能力的相关信息：
```
CAPABILITY_NAME                P MSGTXT                     RELATED_TEXT
------------------------------ - -------------------------- ------------
REFRESH_FAST_AFTER_ANY_DML     Y
REFRESH_FAST_AFTER_INSERT      Y
REFRESH_FAST_AFTER_ONETAB_DML  Y
```

执行以下语句来检查快速刷新是否有效：
```
SQL> exec dbms_mview.refresh('SALES_MV','F');
PL/SQL procedure successfully completed.
```

`EXPLAIN_MVIEW`过程是一个强大的工具，用于确定刷新能力是否可行。如果不可行，它可以解释原因以及如何潜在地解决问题。

## 查看物化视图定义
要快速查看物化视图所基于的 SQL 查询，可以从`DBA/ALL/USER_MVIEWS`的`QUERY`列中进行选择。如果使用 SQL*Plus，首先需要将`LONG`变量设置为足够大的值以显示`LONG`列的完整内容：
```
SQL> set long 5000
SQL> select query from dba_mviews where mview_name=UPPER('&&mview_name');
```

要查看重新创建物化视图所需的完整 DDL，可以使用`DBMS_METADATA`包（如果使用 SQL*Plus，同样需要将`LONG`变量设置为较大的值）：
```
SQL> select dbms_metadata.get_ddl('MATERIALIZED_VIEW','SALES_MV') from dual;
```

以下是此示例的部分输出：
```
DBMS_METADATA.GET_DDL('MATERIALIZED_VIEW','SALES_MV')

CREATE MATERIALIZED VIEW "MV_MAINT"."SALES_MV" ("SALES_ROWID", "REGION_ROWID",
"SALES_ID", "REG_DESC")
ORGANIZATION HEAP PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255
```

此输出显示了 Oracle 认为重新创建物化视图所需的 DDL。这通常是生成与物化视图关联的最可靠方法。

## 删除物化视图
有时可能需要删除一个物化视图。也许某个视图不再使用，或者可能需要删除并重新创建物化视图以更改其所基于的基础查询（例如添加列）。使用`DROP MATERIALIZED VIEW`命令来删除物化视图；例如：
```
SQL> drop materialized view sales_mv;
```

删除物化视图时，物化视图对象、表对象以及任何相应的索引也会被删除。删除物化视图不会影响任何物化视图日志——物化视图日志仅依赖于主表。

也可以指定保留底层表。如果在排除故障时需要删除物化视图定义但保留物化视图表和数据，可能需要这样做；例如：
```
SQL> drop materialized view sales_mv preserve table;
```

在此场景中，后续也可以通过使用`ON PREBUILT TABLE`子句构建物化视图，将底层表用作物化视图的基础。

如果物化视图最初是使用`ON PREBUILT TABLE`子句创建的，那么当删除物化视图时，底层表不会被删除。如果希望同时删除底层表，则必须使用`DROP TABLE`语句：
```
SQL> drop materialized view sales_mv;
SQL> drop table sales_mv;
```

## 修改物化视图
以下部分描述了与物化视图相关的一些常见维护任务。涵盖的主题包括如何修改物化视图以反映在物化视图初始创建后某个时间点应用于基表的列更改，以及修改诸如日志记录和并行度等属性。

### 修改基表 DDL 并传播到物化视图
一个常见的任务涉及向基表添加列或从基表删除列（因为业务需求已变更）。在向基表添加列或从中删除列后，希望这些 DDL 更改能反映在所有依赖的物化视图中。有几种选项可以将基表列更改传播到依赖的物化视图：

*   使用新的列定义删除并重新创建物化视图。
*   删除物化视图但保留底层表，修改物化视图表，然后使用`ON PREBUILT TABLE`子句重新创建物化视图（包含新的列更改）。
*   如果物化视图最初是使用`ON PREBUILT TABLE`子句创建的，则删除物化视图对象，修改物化视图表，然后使用`ON PREBUILT TABLE`子句重新创建物化视图（包含新的列更改）。

采用上述任何选项，都需要删除并重新创建物化视图，以使其包含基表中的新列更改。接下来将描述这些方法。

### 重新创建物化视图以反映基表修改
使用之前创建的`SALES`表，假设已创建了一个物化视图日志和一个物化视图，如下所示：
```
SQL> create materialized view log on sales with primary key;
--
SQL> create materialized view sales_mv
refresh with primary key
fast on demand as
select sales_id ,sales_amt, sales_dtt
from sales;
```

之后，向基表添加了一个列：
```
SQL> alter table sales add(sales_loc varchar2(30));
```

希望基表的修改能反映在物化视图中。如何完成这个任务？我们知道物化视图包含一个存储结果的底层表。决定直接修改底层物化视图表：
```
SQL> alter table sales_mv add(sales_loc varchar2(30));
```

修改成功。接下来刷新物化视图，但发现新增的列并未被刷新。要理解原因，回想一下物化视图是一个将其结果存储在底层表中的 SQL 查询。因此，要修改物化视图，必须更改其所基于的 SQL 查询。因为没有`ALTER MATERIALIZED VIEW ADD/DROP/MODIFY <column>`语句，所以必须执行以下操作来在物化视图中添加/删除列：

1.  修改基表。
2.  删除并重新创建物化视图以反映基表的更改。
```
SQL> drop materialized view sales_mv;
--
SQL> create materialized view sales_mv
refresh with primary key
complete on demand as
select sales_id, sales_amt, sales_dtt, sales_loc
from sales;
```

如果涉及大量数据，此方法可能需要很长时间。在物化视图重建期间，任何访问该物化视图的应用程序都将面临停机时间。如果工作在数据仓库环境中，由于完全刷新物化视图所需的时间很长，可能需要考虑不删除底层表。下一节将讨论此选项。

### 修改物化视图但保留底层表
删除物化视图时，可以选择保留底层表及其数据。在数据仓库环境中处理大型物化视图时，这种方法可能是有利的。步骤如下：

1.  修改基表。
2.  删除物化视图，但保留底层表。
3.  修改底层表。
4.  使用`ON PREBUILT TABLE`子句重新创建物化视图。

以下是一个简单的示例来说明此过程：
```
SQL> alter table sales add(sales_loc varchar2(30));
```

删除物化视图，但指定要保留底层表：
```
SQL> drop materialized view sales_mv preserve table;
```

现在，修改底层表：
```
SQL> alter table sales_mv add(sales_loc varchar2(30));
```

接下来，使用`ON PREBUILT TABLE`子句创建物化视图：
```
SQL> create materialized view sales_mv
on prebuilt table
refresh with primary key
complete on demand as
select sales_id, sales_amt, sales_dtt, sales_loc
from sales;
```

这样就可以在不删除并完全刷新数据的情况下重新定义物化视图。需要注意的是，如果在物化视图重建操作期间有任何针对基表的 DML 活动，那么当尝试刷新物化视图时，这些事务不会反映在物化视图中。在数据仓库环境中，通常有一个已知的基表加载计划，因此应该能够在维护窗口内执行物化视图的修改，此时基表中没有事务发生。

### 在预建表上修改物化视图



# 手动刷新过程

当你选择手动刷新物化视图时，需要使用包 `dbms_mview` 中的以下过程之一：

*   `refresh`：此过程通过参数 `list` 指定一个逗号分隔的列表来刷新其中的物化视图。例如，以下调用刷新模式 `sh` 中的物化视图 `sales_mv` 和 `cal_month_sales_mv`：
    ```sql
    dbms_mview.refresh(list => 'sh.sales_mv,sh.cal_month_sales_mv')
    ```
*   `refresh_all_mviews`：此过程刷新数据库中存储的所有物化视图，但标记为永不刷新的除外。输出参数 `number_of_failures` 返回处理期间发生的失败次数。
    ```sql
    dbms_mview.refresh_all_mviews(number_of_failures => :r)
    ```
*   `refresh_dependent`：此过程刷新依赖于通过参数 `list` 指定的基表（逗号分隔列表）的物化视图。输出参数 `number_of_failures` 返回处理期间发生的失败次数。例如，以下调用刷新模式 `sh` 中存储的所有依赖于表 `sales` 的物化视图：
    ```sql
    dbms_mview.refresh_dependent(number_of_failures => :r, list => 'sh.sales')
    ```

所有这些过程也支持参数 `method` 和 `atomic_refresh`。前者指定刷新的执行方式（'`c`' 表示完全，'`f`' 表示快速，'`?`' 表示强制），后者指定刷新是否在单个事务中执行。如果参数 `atomic_refresh` 设置为 `FALSE`，则不使用单个事务。因此，对于完全刷新，物化视图将被截断而不是被删除。一方面，刷新速度更快。另一方面，如果在刷新运行期间另一个会话查询物化视图，查询可能会返回错误结果（未选择任何行）。

---

**注意** 在 Oracle9*i* 中，由于错误 3168840，即使参数 `atomic_refresh` 设置为 `TRUE`（所有版本中的默认值），在对单个物化视图执行完全刷新期间，也会执行 `TRUNCATE` 语句。但是，如果同时刷新多个物化视图，则刷新会按预期以原子方式工作。因此，要解决此错误，可以创建一个刷新组，其中包含需要刷新的物化视图和一个仅为了拥有第二个视图而创建的“虚拟”物化视图。你可以使用脚本 `atomic_refresh.sql` 来重现此行为。

---


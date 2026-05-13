# 图 11-2 显示箭头指向下方，因为连接操作会调用其子节点来访问表。不过，你可能更倾向于将箭头绘制为指向上方；这样更能反映数据的流向。

左深连接树引出了一套简单的优化器提示，用于指定连接顺序和连接方法：

*   `LEADING` 提示指定了连接中最前面的几个（或全部）行源。例如，`LEADING(T1 T2)` 强制 CBO 以 `T1` 和 `T2` 开始连接。`T1` 必须是驱动行源，`T2` 是探测行源。接下来是连接 `T3` 还是 `T4`，则由 CBO 决定。
*   提示 `USE_NL`、`USE_HASH`、`USE_MERGE` 和 `USE_MERGE_CARTESIAN` 用于指定连接方法。为了标识提示所指的连接，我们使用探测行源的名称。

这一切都有些抽象，但希望 清单 11-9 中的示例能使事情变得清晰。

## 清单 11-9. 完全指定连接顺序和连接方法

```sql
SELECT /*+
leading (t4 t3 t2 t1)
use_nl(t3)
use_nl(t2)
use_nl(t1)
*/
       *
  FROM t1
      ,t2
      ,t3
      ,t4
 WHERE t1.c1 = t2.c2 AND t2.c2 = t3.c3 AND t3.c3 = t4.c4;

| Id  | Operation            | Name |

|   0 | SELECT STATEMENT     |      |
|   1 |  NESTED LOOPS        |      |
|   2 |   NESTED LOOPS       |      |
|   3 |    NESTED LOOPS      |      |
|   4 |     TABLE ACCESS FULL| T4   |
|   5 |     TABLE ACCESS FULL| T3   |
|   6 |    TABLE ACCESS FULL | T2   |
|   7 |   TABLE ACCESS FULL  | T1   |
```

清单 11-9 中的 `LEADING` 提示指定了所有表，因此连接顺序固定为 `((T4 ![image](img/right-arrow.jpg) T3) ![image](img/right-arrow.jpg) T2) ![image](img/right-arrow.jpg) T1`。在提示连接机制时，三个连接通过它们各自的探测行源来识别：`T3`、`T2` 和 `T1`。在这种情况下，指定 `T4` 的提示将毫无意义，因为它在任何连接中都不是探测行源。


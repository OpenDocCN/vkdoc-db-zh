# 直方图

到目前为止对列统计信息的解释意味着，CBO 必须以相同的方式处理大多数涉及列名和字面值的等值谓词，而不管提供的值是什么。清单 9-7 显示了这可能带来的问题。

## 清单 9-7. 缺失直方图的示例

```sql
SELECT *
  FROM statement
 WHERE transaction_amount = 8;
```

```
| Id  | Operation         | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
|---|---|---|---|---|---|---|
|   0 | SELECT STATEMENT  |           |     2 |   106 |     8   (0)| 00:00:01 |
|*  1 |  TABLE ACCESS FULL| STATEMENT |     2 |   106 |     8   (0)| 00:00:01 |
```

在`STATEMENT`表中，`TRANSACTION_AMOUNT`有 262 个不同的值，范围从 5 到 15200。CBO（这次是正确的）假设 8 是这 262 个值之一，但假设只有大约 500/262 行会有`TRANSACTION_AMOUNT = 8`。实际上，有 125 行满足`TRANSACTION_AMOUNT = 8`。在这种情况下，提高 CBO 估计准确性的方法是定义直方图。在创建直方图之前，我想定义我称之为*直方图规范*的东西，它记录了我们希望 CBO 做出的不同估计。对于`TRANSACTION_AMOUNT`，我建议以下直方图规范：

对于`STATEMENT`中的每 500 行，CBO 应假设其中 125 行的`TRANSACTION_AMOUNT`值为 8，并且 CBO 应假设对于任何其他提供的值，只有 2 行。

清单 9-8 展示了我们如何构建一个直方图来让 CBO 做出这些假设。

## 清单 9-8. 在 TRANSACTION_AMOUNT 上创建直方图


```
DECLARE
   srec   DBMS_STATS.statrec;
BEGIN
   FOR r
      IN (SELECT *
            FROM all_tab_cols
           WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
                 AND table_name = 'STATEMENT'
                 AND column_name = 'TRANSACTION_AMOUNT')
   LOOP
      srec.epc := 3;
      srec.bkvals := DBMS_STATS.numarray (600, 400, 600);
      DBMS_STATS.prepare_column_values (srec
                                       ,DBMS_STATS.numarray (-1e7, 8, 1e7));
      DBMS_STATS.set_column_stats (ownname   => r.owner
                                  ,tabname   => r.table_name
                                  ,colname   => r.column_name
                                  ,distcnt   => 3
                                  ,density   => 1 / 250
                                  ,nullcnt   => 0
                                  ,srec      => srec
                                  ,avgclen   => r.avg_col_len);
   END LOOP;
END;
/

SELECT *
  FROM statement
 WHERE transaction_amount = 8;

| Id  | Operation         | Name      | Rows  | Bytes | Time     |
| --- | ----------------- | --------- | ----- | ----- | -------- |
|   0 | SELECT STATEMENT  |           |   125 |  6875 | 00:00:01 |
|   1 |  TABLE ACCESS FULL| STATEMENT |   125 |  6875 | 00:00:01 |

SELECT *
  FROM statement
 WHERE transaction_amount = 1640;

| Id  | Operation         | Name      | Rows  | Bytes | Time     |
| --- | ----------------- | --------- | ----- | ----- | -------- |
|   0 | SELECT STATEMENT  |           |     2 |   110 | 00:00:01 |
|   1 |  TABLE ACCESS FULL| STATEMENT |     2 |   110 | 00:00:01 |
```

这个 PL/SQL "循环"实际上只会调用一次`DBMS_STATS.SET_COLUMN_STATS`，因为对`ALL_TAB_COLS`³的查询只返回一行。在`TRANSACTION_AMOUNT`上定义直方图的关键是我在粗体中突出显示的`SREC`参数。这个结构的设置如下：

1.  设置`SREC.EPC`。`EPC`是*End Point Count*（端点计数）的缩写，在这里定义了对象列的不同值的数量。我建议将`EPC`设置为比*直方图规范*定义的列值数量多两个。在我们的例子中，直方图规范中唯一的值是 8，所以我们设置`EPC`为 3。
2.  设置`SREC.BKVALS`。`BKVALS`是*bucket values*（桶值）的缩写，这里的三个桶值列表表示每个端点的假设行数。因此，理论上 600 行有一个`TRANSACTION_AMOUNT`值，400 行有第二个值，600 行有第三个值。这三个值一起表明表中有 1600 行（`600+400+600=1600`）。实际上，这些值的实际解释是：对于表中的*每 1600 行*，600 行具有第一个（尚未指定的）`TRANSACTION_AMOUNT`值，400 行具有第二个值，600 行具有第三个值。
3.  调用`DBMS_STATS.PREPARE_COLUMN_VALUES`来为每个端点定义实际的对象列值。第一个端点故意设置得非常低，中间端点是我们直方图规范中的值 8，最后一个值故意设置得非常高。在我们有序值集的开头和结尾添加极低和极高值的原因是，我们为什么将`SREC.EPC`设置为比直方图规范中的值数量多两个。我们永远不会在谓词中使用这些荒谬的值（否则它们就不荒谬了），所以唯一值得注意的统计信息是：每 1600 行中有 400 行的值为 8。这意味着当我们从 500 行中选择时，CBO 给出的基数估计是`500 x 400 / 1600 = 125`。
4.  调用`DBMS_STATS.SET_COLUMN_STATS`。有两个参数我想请你注意。
    *   **DISTCNT** 设置为 3。我们为列定义了三个值`(-1e7, 8, 和 1e7)`，所以我们需要保持一致，将`DISTCNT`设置为与`SREC.EPC`相同的值。这种安排确保我们获得所谓的*频率直方图*。
    *   **DENSITY** 设置为 1/250。当定义了频率直方图时，该统计信息表示谓词中未在直方图中指定的值的选择性。清单 9-8 中提供的值 1/250 表明，对于 500 个非空行，当谓词中出现三个提供值以外的任何值时，CBO 应假设有 500/250 = 2 行匹配该相等谓词。

直方图创建后，我们可以看到，当提供`TRANSACTION_AMOUNT = 8`谓词时，CBO 给出的基数估计是 125；当谓词中提供任何其他字面值时，基数估计是 2。

以这种方式创建直方图通常被称为*伪造直方图*。我不喜欢这个术语，因为它暗示收集的（非伪造的）直方图在某种程度上更优越。其实不然。例如，当我在我的 12cR1 数据库上允许为`TRANSACTION_AMOUNT`自动创建直方图时，我得到了一个具有 254 个端点的直方图。理论上，这意味着对于涉及`TRANSACTION_AMOUNT`相等谓词的每条语句，我可能会得到 255 个不同的执行计划（每个定义的端点一个，加上直方图中未指定值的一个）。通过使用为测试而定义的直方图规范手动创建频率直方图，你可以限制测试用例的数量。在清单 9-8 的例子中，每条语句只有两个测试用例：`TRANSACTION_AMOUNT = 8` 和 `TRANSACTION_AMOUNT = <任何其他值>`。

以此方式手动创建直方图时，请牢记以下几点：


# 大纲数据

```sql
/*+
      BEGIN_OUTLINE_DATA
      FULL(@"SEL$1" "T"@"SEL$1")
      OUTLINE_LEAF(@"SEL$1")
      ALL_ROWS
      OPTIMIZER_FEATURES_ENABLE('10.2.0.3')
      IGNORE_OPTIM_EMBEDDED_HINTS
      END_OUTLINE_DATA
  */
```

## 谓词信息（由操作 ID 识别）

```text
   2 - filter("N">=6 AND "N"<=19)
```

## 列投影信息（由操作 ID 识别）

```text
   1 - (#keys=0) COUNT(*)[22]
```

## 注释

```text
   - 此语句使用了动态采样
```

以下查询展示了如何使用修饰符，从由原始值 `basic` 和 `typical` 生成的默认输出中添加或移除信息。由于它们基于与前面示例相同的查询，您可以比较输出以查看有何不同。以下是脚本 `display.sql` 生成的输出摘录：

```sql
SQL> SELECT * FROM table(dbms_xplan.display(NULL,NULL,'basic +predicate',NULL));
```

```text
PLAN_TABLE_OUTPUT
---------------------------------------------------

Plan hash value: 2966233522

-----------------------------------
| Id  | Operation         | Name |
-----------------------------------
|   0 | SELECT STATEMENT  |      |
|   1 |  SORT AGGREGATE   |      |
|*  2 |   TABLE ACCESS FULL| T    |
-----------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------
2 - filter("N">=6 AND "N"<=19)
```

```sql
SQL> SELECT *
  2 FROM table(dbms_xplan.display(NULL,NULL,'typical -bytes -note',NULL));
```

```text
PLAN_TABLE_OUTPUT
-------------------------------------------------------------------

Plan hash value: 2966233522

-------------------------------------------------------------------
| Id  | Operation         | Name | Rows  | Cost (%CPU)| Time     |
-------------------------------------------------------------------
|   0 | SELECT STATEMENT  |      |     1 |     5   (0)| 00:00:01 |
|   1 |  SORT AGGREGATE   |      |     1 |            |          |
|*  2 |   TABLE ACCESS FULL| T    |    14 |     5   (0)| 00:00:01 |
-------------------------------------------------------------------

Predicate Information (identified by operation id):
---------------------------------------------------

   2 - filter("N">=6 AND "N"<=19)
```


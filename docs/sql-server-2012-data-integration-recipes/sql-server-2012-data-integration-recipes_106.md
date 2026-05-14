# 主键

如果您需要验证要导入的表的主键，请尝试以下 SQL：

```sql
SELECT    A.Table_name, C.Column_name
FROM      ALL_CONSTRAINTS A
          JOIN ALL_CONS_COLUMNS C
          ON A.CONSTRAINT_NAME = C.CONSTRAINT_NAME
WHERE     CONSTRAINT_TYPE = 'P'
```

您可以扩展 `WHERE` 子句，将输出限制到您希望分析的表。您也可以调整 `WHERE` 子句中的 `CONSTRAINT_TYPE` 来获取以下信息：

*   `P`：返回主键约束。
*   `C`：返回 `NOT NULL` 或检查约束。
*   `R`：返回外键约束。
*   `U`：返回唯一键约束。

### 索引

无需深入探讨 Oracle 索引的无数细节，了解源数据中哪些列被索引仍然很有用。如果您使用暂存表来转换数据，这一点尤其合适；这些信息可以让您“预先了解”应索引哪些字段以加速数据暂存过程。SQL 如下：

```sql
SELECT ALL_IND_COLUMNS.INDEX_NAME, ALL_IND_COLUMNS.TABLE_OWNER,
      ALL_IND_COLUMNS.TABLE_NAME, ALL_IND_COLUMNS.COLUMN_NAME
FROM     ALL_INDEXES INNER JOIN ALL_IND_COLUMNS
        ON ALL_INDEXES.INDEX_NAME = ALL_IND_COLUMNS.INDEX_NAME;
```

如果一个索引名称出现多次，那是因为您面对的是一个多列索引。

请不要犹豫进一步查看 `ALL_INDEXES` 表，看看是否有其他元数据可以为您提供帮助。通常，这对于所有 Oracle 的数据字典表都是如此。它们提供了大量有用的信息。


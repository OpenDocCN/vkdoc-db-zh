# 列统计信息

以下查询展示了如何获取表中最重要的列统计信息：

```sql
SQL> SELECT column_name AS "NAME",
  2        num_distinct AS "#DST",
  3        low_value,
  4        high_value,
  5        density AS "DENS",
  6        num_nulls AS "#NULL",
  7        avg_col_len AS "AVGLEN",
  8        histogram,
  9        num_buckets AS "#BKT"
 10 FROM user_tab_col_statistics
 11 WHERE table_name = 'T';
```

```
NAME  #DST LOW_VALUE      HIGH_VALUE        DENS #NULL AVGLEN HISTOGRAM        #BKT
---- ----- -------------- -------------- ------- ----- ------ --------------- -----
ID    1000 C102           C20B            .00100     0      4 NONE                1
VAL1   431 C103           C2213E          .00254   503      3 HEIGHT BALANCED   254
VAL2     6 C20202         C20207          .00050     0      4 FREQUENCY           6
VAL3     6 C20202         C20207          .00050     0      4 FREQUENCY           6
PAD   1000 202623436F2943 7E79514A202D49  .00100     0    251 HEIGHT BALANCED   254
           7334237B426574 4649366C744E25
           336E4A5B302E4F 3F36264C692755
           4B53236932303A 7A57737C6D4B22
           21215F46       59414C44
```

## LOW_VALUE 和 HIGH_VALUE 格式

不幸的是，`low_value` 和 `high_value` 这两个列的值不易解读。实际上，它们显示的值是根据数据库引擎用于存储数据的内部表示形式。要将它们转换为人类可读的值，有两种可能性。

首先，包 `utl_raw` 提供了函数 `cast_to_binary_double`、`cast_to_binary_float`、`cast_to_binary_integer`、`cast_to_number`、`cast_to_nvarchar2`、`cast_to_raw` 和 `cast_to_varchar2`。正如函数名所示，对于每种特定的数据类型，都有一个对应的函数用于将内部值转换为实际值。要获取列 `val1` 的最小值和最大值，可以使用以下查询：

```sql
SQL> SELECT utl_raw.cast_to_number(low_value) AS low_value,
  2        utl_raw.cast_to_number(high_value) AS high_value
 3 FROM user_tab_col_statistics
 4 WHERE table_name = 'T'
 5 AND column_name = 'VAL1';
```

```
LOW_VALUE HIGH_VALUE
--------- ----------
        2       3261
```

其次，包 `dbms_stats` 提供了过程 `convert_raw_value`（该过程被多次重载）、`convert_raw_value_nvarchar` 和 `convert_raw_value_rowid`。由于过程不能直接在 SQL 语句中使用，因此通常仅在 PL/SQL 程序中使用。在以下示例中，PL/SQL 块的目的与前一个查询相同：

```sql
SQL> DECLARE
  2   l_low_value user_tab_col_statistics.low_value%TYPE;
  3   l_high_value user_tab_col_statistics.high_value%TYPE;
  4   l_val1 t.val1%TYPE;
  5 BEGIN
  6   SELECT low_value, high_value
  7   INTO l_low_value, l_high_value
  8   FROM user_tab_col_statistics
  9   WHERE table_name = 'T'
 10   AND column_name = 'VAL1';
 11
 12   dbms_stats.convert_raw_value(l_low_value, l_val1);
 13   dbms_output.put_line('low_value: ' || l_val1);
 14   dbms_stats.convert_raw_value(l_high_value, l_val1);
 15   dbms_output.put_line('high_value: ' || l_val1);
 16 END;
 17 /
```

```
low_value: 2
high_value: 3261
```

以下是此查询返回的列统计信息的解释：

*   `num_distinct` 是列中不同值的数量。
*   `low_value` 是列中的最低值。它以内部表示形式显示。请注意，对于字符串列（在示例中为 `pad` 列），仅使用前 32 个字节。
*   `high_value` 是列中的最高值。它以内部表示形式显示。请注意，对于字符串列（在示例中为 `pad` 列），仅使用前 32 个字节。
*   `density` 是一个介于 0 和 1 之间的十进制数。接近 0 的值表示对该列的限制会过滤掉大多数行。接近 1 的值表示对该列的限制几乎不会过滤掉任何行。如果没有直方图，`density` 是 `1/num_distinct`。如果存在直方图，则计算方式不同，并取决于直方图的类型。
*   `num_nulls` 是列中存储的 `NULL` 值的数量。
*   `avg_col_len` 是列的平均大小（以字节为单位）。
*   `histogram` 指示列是否可用直方图，如果可用，则指示其类型。有效值为 `NONE`（表示无直方图）、`FREQUENCY` 和 `HEIGHT BALANCED`。此列自 Oracle Database 10*g* 起可用。
*   `num_buckets` 是直方图中的桶数。桶（在统计学中也称为 `category`）是同类值的分组。正如我们将在下一节中看到的，直方图由至少一个桶组成。如果没有可用的直方图，则设置为 1。最大桶数为 254。


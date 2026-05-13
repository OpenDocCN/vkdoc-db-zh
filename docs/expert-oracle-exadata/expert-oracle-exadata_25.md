# Oracle SQL 性能分析

## SQL 查询与执行计划

`SQL_ID          CHILD OFFLOAD IO_SAVED_%   AVG_ETIME SQL_TEXT`
`------------- ------ ------- ---------- ---------- ----------------------------------------`
`2s4ff4z9f8q7c       0 No              .00      16.45 select /* fctest001 */ count(*) from SOE`

```sql
SQL> @dplan
Enter value for sql_id: 2s4ff4z9f8q7c
Enter value for child_no: 0
```

### 执行计划输出

`PLAN_TABLE_OUTPUT`
`-------------------------------------------------------------------------------------------`
`SQL_ID  2s4ff4z9f8q7c, child number 0`
`-------------------------------------`
`select /* fctest001 */ count(*) from SOE.ORDER_ITEMS where order_id < 0`
`Plan hash value: 643311209`
`---------------------------------------------------------------------------------------`
`| Id  | Operation             | Name          | Rows  | Bytes | Cost (%CPU)| Time     |`
`---------------------------------------------------------------------------------------`
`|   0 | SELECT STATEMENT      |               |       |       | 26355 (100)|          |`
`|   1 |  SORT AGGREGATE       |               |     1 |     6 |            |          |`
`|*  2 |   INDEX FAST FULL SCAN| ITEM_ORDER_IX |     3 |    18 | 26355   (1)| 00:00:02 |`
`---------------------------------------------------------------------------------------`
`Predicate Information (identified by operation id):`
`---------------------------------------------------`
`   2 - filter("ORDER_ID"<0)`
`19 rows selected.`

## 分析与解释

等一下——为什么那个查询没有进行卸载？索引快速全扫描不是候选者吗？它们确实是，但这里你看不到的是，`ITEM_ORDER_IX`是一个反向键索引，这在至少 12.1.2.1.x 版本中是不可卸载的。

## 查询索引信息

```sql
SQL> select owner, index_name, index_type, table_owner, table_name
  2  from dba_indexes
  3  where index_name = 'ITEM_ORDER_IX'
  4  and owner = 'SOE';
```

查询结果显示`ITEM_ORDER_IX`的类型为`NORMAL/ REV`。

`OWNER      INDEX_NAME           INDEX_TYPE                       TABLE_OWNER      TABLE_NAME`
`---------- -------------------- ------------------------------ --------------- ---------------`
`SOE        ITEM_ORDER_IX        NORMAL/ REV                     SOE              ORDER_ITEMS`

## 强制全表扫描以验证 IO 统计

为了最终证明物理 IO 统计信息可以包括闪存缓存命中以及存储索引节省，已使用提示强制进行全表扫描：

```sql
SQL> select /*+ full(oit) */ /* fctest004  */ count(*) from SOE.ORDER_ITEMS oit
  2  where order_id < 0;
```

`COUNT(*)`
`----------`
`0`
`Elapsed: 00:00:00.11`

## 会话级 IO 统计查询

使用您之前见过的查询的一个轻微变体，您可以看到不同的情况：

```sql
SQL> select sid, name, value from v$sesstat natural join v$statname
  2  where sid = (264)
  3  and name in (
  4    'cell physical IO bytes eligible for predicate offload',
  5    'cell flash cache read hits',
  6    'cell overwrites in flash cache',
  7    'cell partial writes in flash cache',
  8    'cell physical IO bytes saved by storage index',
  9    'cell writes to flash cache',
 10    'physical read IO requests',
 11    'physical read bytes',
 12    'physical read requests optimized',
 13    'physical read total bytes optimized',
 14    'physical write requests optimized',
 15    'physical write total bytes optimized')
 16  order by name;
```

查询结果统计信息如下：

`SID NAME                                                                  VALUE`
`---------- ---------------------------------------------------------------- ------------------`
`264 cell flash cache read hits                                                61.00`
`264 cell overwrites in flash cache                                              .00`
`264 cell partial writes in flash cache                                          .00`
`264 cell physical IO bytes eligible for predicate offload          3,050,086,400.00`
`264 cell physical IO bytes saved by storage index                   3,048,513,536.00`
`264 cell writes to flash cache                                                  .00`
`264 physical read IO requests                                               23,365.00`



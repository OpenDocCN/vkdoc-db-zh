# 自动数据优化 vs. 手动生命周期管理

在 Oracle 12c 之前，执行维护任务是数据库管理员的职责。在许多情况下，维护包括实施**信息生命周期管理**（ILM）策略。通过妥善实施的变更管理，这过去运行良好。然而，在变更管理实施不那么严格的情况下，维护 ILM 策略可能会变得困难。

Oracle 12c 通过允许自动应用 ILM 策略，将数据库管理员从许多此类任务中解放出来。正如你可以在接下来的章节中看到的，你可以向一个段添加一个声明性策略，并将实施工作交给 Oracle。此功能需要**高级压缩许可**。

**自动数据优化**（ADO）主要基于使用情况跟踪。一个所谓的**热图**在行和段级别跟踪访问和数据修改。热图不仅是 ADO 的基础，还可以通过数据字典视图进行查询，并通过一组 PL/SQL 应用程序编程接口（API）进行操作。

广义上说，管理员有两个选项，值得庆幸的是它们并非互斥。第一个选项是压缩；另一个选项是使用分层子句。不过，第一步是启用热图。但要小心：启用热图跟踪已经要求你拥有**高级压缩选项**的许可。

```
SQL> alter system set heat_map = on sid='*' scope=both;

系统已变更。
```

Oracle 提供了多个视图，你可以用它们来了解更多关于数据库中热图跟踪的信息：

```
SQL> SELECT table_name
  2  FROM dict
  3  WHERE table_name LIKE 'V%HEAT%MAP%'
  4  OR table_name LIKE 'DBA%HEAT%MAP%';

TABLE_NAME
------------------------------
DBA_HEATMAP_TOP_OBJECTS
DBA_HEATMAP_TOP_TABLESPACES
DBA_HEAT_MAP_SEG_HISTOGRAM
DBA_HEAT_MAP_SEGMENT
V$HEAT_MAP_SEGMENT
```

`V$HEAT_MAP_SEGMENT`是提供实时访问热图的数据字典视图，但它不像`DBA_HEATMAP_TOP_OBJECTS`那样具有“所有者”列。你可以通过`OBJECT_ID`和`DATA_OBJECT_ID`轻松地与`DBA_OBJECTS`进行连接。

## ADO 的示例用例

为了更好地说明 ADO 的概念和实施，将使用以下演示。一个名为`ADODEMO`的简单表将用于此目的。将向其附加多个信息生命周期管理策略，并展示其效果。让我们从表定义开始：

```
CREATE TABLE adodemo (
id, t_pad, date_created, state
)
enable row movement
partition by range (date_created)
INTERVAL(NUMTOYMINTERVAL(1, 'MONTH'))
(
partition p_manual values less than (to_date('01.01.2005','dd.mm.yyyy'))
)
AS -- 感谢 Jonathan Lewis!
WITH v1 AS  (
  SELECT rownum n FROM dual CONNECT BY level <= 10000
)
SELECT
  rownum id,
  rpad(rownum,1999) t_pad,
  TRUNC(sysdate) - 180 + dbms_random.value(0,180) date_created,
  CASE
    WHEN mod(rownum,100000) = 0
    THEN CAST('RARE' AS VARCHAR2(12))
    WHEN mod(rownum,10000) = 0
    THEN CAST('FAIRLY RARE' AS VARCHAR2(12))
    WHEN mod(rownum,1000) = 0
    THEN CAST('NOT RARE' AS VARCHAR2(12))
    WHEN mod(rownum,100) = 0
    THEN CAST('COMMON' AS   VARCHAR2(12))
    ELSE CAST('THE REST' AS VARCHAR2(12))
  END state
FROM v1,
  v1
WHERE rownum <= 1e7;
```

该表具有一千万行，但相当窄。`ADODEMO`表主要从分区角度值得关注。你在前一章节中读到分区对于 HCC 的适当实施是多么重要。请注意，该表最初完全没有启用压缩。这是有意为之：使用 ADO，你无需自己指定压缩——把它留给 Oracle。在我们的 12.1.0.2.0 测试平台上，该表具有以下物理属性：

```
SQL> select table_name,partition_name,num_rows,last_analyzed
  2  from dba_tab_partitions where table_owner = 'MARTIN' and table_name = 'ADODEMO';

TABLE_NAME                     PARTITION_NAME           NUM_ROWS LAST_ANAL
------------------------------ -------------------- ---------- ---------
ADODEMO                        P_MANUAL
ADODEMO                        SYS_P796                  1721062 09-OCT-14
ADODEMO                        SYS_P797                  1111364 09-OCT-14
ADODEMO                        SYS_P798                  1724376 09-OCT-14
ADODEMO                        SYS_P799                  1664931 09-OCT-14
ADODEMO                        SYS_P800                  1720722 09-OCT-14
ADODEMO                        SYS_P801                   389333 09-OCT-14
ADODEMO                        SYS_P802                  1668212 09-OCT-14
已选择 8 行。
```

由于是基于`MONTH`的间隔分区，你无需过多关注分区定义的细节。一个默认分区就足够了；Oracle 会在需要时创建新的分区。



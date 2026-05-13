# 显式游标与批量处理

在需要一次处理成千上万（或数百万）条记录的情况下，你无法一次性处理所有记录。当然，你或许会*尝试*。但性能下降会非常严重，以至于你不想这么做。当必须将一个非常大的数据集分成较小的块进行处理时，批量处理是满足这一需求的最佳方法之一。请参考清单 10-3 中的批量处理示例。

**清单 10-3.** 使用显式游标执行批量处理

```sql
CREATE OR REPLACE PROCEDURE refresh_store_feed AS
   TYPE prod_array      IS TABLE OF store_products%ROWTYPE INDEX BY BINARY_INTEGER;
   l_prod               prod_array;
   CURSOR c IS
      SELECT  product
        FROM  listed_products@some_remote_site;
   BEGIN
   OPEN C;
   LOOP
   FETCH C BULK COLLECT INTO l_prod LIMIT 100;
   FOR i IN 1 .. l_csi.COUNT
   LOOP
      /*    ... 在这里对 l_csi(i) 执行一些无法用 SQL 完成的过程性代码 ... */
   END LOOP;
      FORALL i IN 1 .. l_csi.COUNT
         INSERT INTO store_products (product) VALUES (l_prod(i));
   EXIT WHEN c%NOTFOUND;
   END LOOP;
   CLOSE C;
   END;
   /
```

由于像清单 10-3 所示的那种包含分段处理的批量处理无法使用隐式游标完成，因此你必须使用显式游标来完成任务。在上面的例子中，你可能会问：“如果目标仅仅是从一个表查询并插入到另一个表，为什么不直接使用 `INSERT AS SELECT` 方法？”这是一个合理的问题，因为在不得不使用 PL/SQL 之前先使用 SQL 通常是完成任务的最快方式。但在清单 10-3 概述的情况下并非如此。如果你必须在数据被选出来后——但在插入之前——对其进行某种中间处理，并且你希望以批量方式执行该任务，那么你将需要一个显式游标。此外，虽然可以使用隐式游标进行数组提取（从 Oracle10g 开始，隐式游标会自动执行 100 行的数组提取），并且可以处理它们（读取，然后执行 DML），但你无法批量执行 DML。只有在使用 `FORALL` 功能时，才能批量执行 DML。而且只有在使用显式游标时，才能使用 `FORALL` 功能。

另一个需要使用显式游标的情况本质上是有些动态的。这听起来可能违反直觉，但当你直到运行时才知道查询将是什么时，就需要一种特殊类型的显式游标来帮助你确定实际要处理的结果集。在这种情况下，你将使用的显式游标类型称为 REF（*reference* 的缩写）游标。如果你需要向客户端返回一个结果集，这是最简单的方法之一。REF 游标提供了常规 PL/SQL 游标无法提供的灵活性。


# LOB 存储特性说明

## 性能对比

```
INSERT INTO T (ID, IN_ROW) VALUES ( S.NEXTVAL, 'Hello World' )
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute    100      0.00       0.00          0          4        317         100
Fetch        0      0.00       0.00          0          0          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total      101      0.00       0.00          0          4        317         100
********************************************************************************
INSERT INTO T (ID,OUT_ROW) VALUES ( S.NEXTVAL, 'Hello World' )
call     count       cpu    elapsed       disk      query    current        rows
------- ------  -------- ---------- ---------- ---------- ----------  ----------
Parse        1      0.00       0.00          0          0          0           0
Execute    100      0.02       0.61          0          4        440         100
Fetch        0      0.00       0.00          0          0          0           0
------- ------  -------- ---------- ---------- ---------- ----------  ----------
total      101      0.02       0.61          0          4        440         100
...
Elapsed times include waiting on following events:
Event waited on                             Times   Max. Wait  Total Waited
----------------------------------------   Waited  ----------  ------------
direct path write                             100        0.01          0.60
```

## 使用建议

请注意观察到的 I/O 使用率增加，包括读取和写入。总而言之，这表明如果你使用`CLOB`，并且许多字符串预期能放入行内（即小于 4000 字节），那么使用默认的`ENABLE STORAGE IN ROW`是个好主意。

### CHUNK 子句

LOB 以块为单位存储；指向 LOB 数据的索引指向单个数据块。*块*是逻辑上连续的块集合，是 LOB 分配的最小单位，而通常一个块是最小的分配单位。`CHUNK`大小必须是你的 Oracle 块大小的整数倍——这是唯一有效的值。

> **注意**
>
> `CHUNK`子句仅适用于 BasicFiles。`CHUNK`子句出现在 SecureFiles 的语法中只是为了向后兼容。

你必须从两个角度仔细选择`CHUNK`大小。首先，每个 LOB 实例（每个存储在行外的 LOB 值）将至少消耗一个`CHUNK`。一个`CHUNK`由一个 LOB 值使用。如果一个表有 100 行，每行有一个包含 7KB 数据的 LOB，那么可以肯定将分配 100 个块。如果你将`CHUNK`大小设置为 32KB，你将分配 100 个 32KB 的块。如果你将`CHUNK`大小设置为 8KB，你将（可能）分配 100 个 8KB 的块。关键在于，一个块只被一个 LOB 条目使用（两个 LOB 不会使用同一个`CHUNK`）。如果你选择的块大小不符合你预期的 LOB 大小，最终可能会浪费大量空间。例如，如果你有一个平均 LOB 大小为 7KB 的表，并使用了 32KB 的`CHUNK`大小，你将为每个 LOB 实例浪费大约 25KB 的空间。另一方面，如果你使用 8KB 的`CHUNK`，你将最小化任何类型的浪费。

当你要最小化每个 LOB 实例的`CHUNK`数量时，也需要小心。正如你所看到的，有一个`LOBINDEX`用于指向各个块，你拥有的块越多，这个索引就越大。如果你有一个 4MB 的 LOB 并使用 8KB 的`CHUNK`，你将需要至少 512 个`CHUNK`来存储该信息。这意味着你需要至少足够的`LOBINDEX`条目来指向这些块。直到你记住这是每个 LOB 实例的，这听起来可能不多；如果你有数千个 4MB 的 LOB，你现在就有成千上万的条目。这也将影响你的检索性能，因为读取和管理许多小块比读取较少但较大的块需要更长时间。最终目标是使用一个`CHUNK`大小，既能最小化浪费，又能高效地存储你的数据。

### RETENTION 子句

`RETENTION`子句根据你使用的是 SecureFiles 还是 BasicFiles 而有所不同。如果你回顾“内部 LOB”部分开头`DBMS_METADATA`的输出，请注意，对于 SecureFiles LOB 的`CREATE TABLE`语句中没有`RETENTION`子句，而对于 BasicFiles LOB 则有一个。这是因为`RETENTION`对于 SecureFiles 是自动启用的。

`RETENTION`用于控制 LOB 的读一致性。在后续的小节中，我将详细介绍`RETENTION`在 SecureFiles 和 BasicFiles 之间的处理方式有何不同。




# 第三十三章 ■ 基于列的存储与批处理模式执行

## 图 33-21. `sys.column_store_segments` 输出

图 33-21 显示了该查询的部分输出。第 8 列没有显示列名，它代表的是`OrderId`列，该列是聚集索引的一部分，但并未在列存储索引中显式定义。

输出中的列代表以下含义：

-   `column_id`是索引中一个列的 ID，可以与`sys.index_columns`视图进行联接。如你所见，只有显式包含在索引中的列才有对应的`sys.index_columns`行。
-   `partition_id`引用一个行组（因此也是一个段）所属的分区。你可以将其与`sys.partitions`视图联接，以获取索引的`object_id`。
-   `segment_id`是段的 ID，基本上也是行组的 ID。分区中的第一个段/行组的 ID 为 0。
-   `version`表示列存储段的格式。SQL Server 2012、2014 和 2016 返回 1 作为其值。
-   `encoding_type`表示此段使用的编码类型。它可以是以下四个值之一：
    - 值编码对应的`encoding_type = 1`
    - 非字符串的字典编码对应的`encoding_type = 2`
    - 字符串值的字典编码对应的`encoding_type = 3`
    - 无编码对应的`encoding_type = 4`
-   `row_count`表示段中的行数。
-   `has_null`指示数据是否包含空值。
-   `magnitude`是用于值编码的量级。对于其他编码类型，它返回-1。
-   `min_data_id`和`max_data_id`分别表示段内某一列的最小值和最大值。SQL Server 在查询执行期间会分析这些值，并排除那些不存储满足查询谓词的值的段。此过程的工作方式类似于分区表中的分区消除。
-   `null_value`表示用于指示空值的值。
-   `on_disk_size`指示段的大小（以字节为单位）。

## `sys.column_store_dictionaries`

`sys.column_store_dictionaries`视图提供了有关列存储索引所使用的字典的信息。代码清单 33-12 显示了可用于检查字典列表的代码。

图 33-22 展示了查询输出。

![图 33-22. `sys.column_store_dictionaries` 输出](img/index-687_1.jpg)

### 图 33-22. `sys.column_store_dictionaries` 输出

### 代码清单 33-12. 检查`sys.column_store_dictionaries`视图

```sql
select p.partition_number as [partition], c.name as [column], d.column_id, d.dictionary_id
,d.version, d.type, d.last_id, d.entry_count
,convert(decimal(12,3),d.on_disk_size / 1024.0 / 1024.0) as [Size MB]
from sys.column_store_dictionaries d join sys.partitions p on
p.partition_id = d.partition_id
join sys.indexes i on
p.object_id = i.object_id
left join sys.index_columns ic on
i.index_id = ic.index_id and
i.object_id = ic.object_id and
d.column_id = ic.index_column_id
left join sys.columns c on
ic.column_id = c.column_id and
ic.object_id = c.object_id
where i.name = 'IDX_FactSales_ColumnStore'
order by p.partition_number, d.column_id
```

输出中的列代表以下含义：

-   `column_id`是索引中一个列的 ID。
-   `dictionary_id`是字典的 ID。
-   `version`表示字典格式。SQL Server 2012、2014 和 2016 返回 1 作为其值。
-   `type`表示字典中存储的值的类型。它可以是以下三个值之一：
    - 包含整数值的字典由`type = 1`指定
    - 包含字符串值的字典由`type = 3`指定
    - 包含浮点值的字典由`type = 4`指定
-   `last_id`是字典中的最后一个数据 ID。
-   `entry_count`包含字典中的条目数。
-   `on_disk_size`指示字典的大小（以字节为单位）。


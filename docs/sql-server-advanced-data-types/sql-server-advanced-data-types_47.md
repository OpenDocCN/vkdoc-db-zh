# 第 4 章 查询与分解 XML

当使用 `nodes()` XQuery 方法分解数据时，它可以与 `value()` 方法（将数据分解为关系型结果）或 `query()` 方法（将数据分解为更小的 XML 文档）结合使用。您也可以将 `nodes()` 与 `value()` 和 `query()` 的组合一起使用。`nodes()` 方法最大的优点是它通过 `CROSS APPLY` 操作符可以轻松地应用于列。


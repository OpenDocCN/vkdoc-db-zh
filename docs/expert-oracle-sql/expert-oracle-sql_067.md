# 创建对象统计信息总结

到目前为止，我已经解释了五种可以将对象统计信息加载到数据字典中的不同方式。主要机制是使用诸如 `DBMS_STATS.GATHER_TABLE_STATS` 之类的存储过程直接收集对象统计信息。我们还可以使用 `DBMS_STATS.IMPORT_TABLE_STATS` 或 `DBMS_STATS.TRANSFER_STATS` 加载预先制作好的统计信息。第四种选择是使用诸如 `DBMS_STATS.SET_TABLE_STATS` 之类的存储过程将对象统计信息设置为特定值。最后，对象统计信息可以通过 DDL 或 DML 操作（例如 `CREATE INDEX`）生成。

既然我们已经将对象统计信息加载到数据库中，能够查看它们就太好了。现在让我们来处理这个问题。


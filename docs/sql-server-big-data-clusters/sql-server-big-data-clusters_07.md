# 从 HDFS 导入 airports.csv 文件 (PySpark)
df_airports = spark.read.format('csv').options(header='true', inferSchema="true").load('/Flight_Delays/airports.csv')
```

**清单 6-3** 将 CSV 数据导入数据框

如果一切顺利，您应该会得到一个包含来自 airports.csv 文件数据的 Spark 数据框。

从示例代码中可以看出，我们向 `spark.read.format` 命令提供了许多选项。最重要的一个是我们正在读取的文件类型；在我们的例子中，这是一个 CSV 文件。我们提供的其他选项告诉 Spark 如何处理该 CSV 文件。通过设置选项 `header='true'`，我们指定 CSV 文件有一个包含列名的标题行。选项 `inferSchema='true'` 帮助我们自动检测每一列中使用的数据类型。如果我们不指定此选项，或者将其设置为 false，所有列都将被设置为字符串数据类型，而不是我们期望的数据类型（例如，数值数据的整数数据类型）。如果您不使用 `inferSchema='true'` 选项，或者 `inferSchema` 配置了错误的数据类型，则必须在导入 CSV 文件之前定义模式，并将其作为选项提供给 `spark.read.format` 命令，如清单 6-4 的示例代码所示。

```python

# SQL Server 大数据集群

## 引言

## 减少数据冗余与开发时间

## Linux 上的 SQL Server

## Spark

## 在 Azure 上部署

## 通过 Azure Data Studio 部署大数据集群

## 从 HDFS 导入 airports.csv 文件 (PySpark)

## 从 HDFS 导入 airports.csv 文件 (PySpark)

## 手动设置模式并提供给 spark.read 函数

## 我们需要导入 pyspark.sql.types 库

## 模式声明后，我们可以将其提供给 spark.read 函数

## 显示 df_airports 数据框中的行数

## 显示 df_flights 数据框的模式 (PySpark)

## 让我们返回第一行

## 选择前十行，以表格结构返回

## 我们也可以从数据框中选择特定的列

## 我们也可以在一个或多个列上进行排序

## 根据列值过滤结果

## 除了过滤单个值，我们还可以使用 IN 来提供多个过滤项

## 我们需要从 pyspark.sql.functions 导入 col 函数

## 声明一个包含城市名称的列表

## 过滤数据框

## 更新一行

## 我们需要从 pyspark.sql.functions 导入 col 和 when 函数

## 选择整个数据框，但在 IATA_CODE = "COD" 的地方，将 CITY 值设置为 "Cody" 而不是 "Jackson"

## 将结果存储在新的 df_airports_updated 数据框中

## 返回结果，过滤 IATA_CODE == "COD" 的行

## 移除一行

## 选择整个数据框，除了 IATA_CODE = "COD" 的行

## 将结果存储在新的 df_airports_removed 数据框中

## 返回结果，过滤 IATA_CODE == "COD" 的行

## 按城市统计机场数量并按计数降序排序

## 同样将其他 CSV 文件导入数据框

## 让我们使用内连接在航空公司代码上连接 df_airlines 和 df_flights 数据框

## 打印连接后数据框的模式

## 我们将再次连接两个数据框，但这次删除 df_flights 数据框的 AIRLINE 列

## 打印连接后数据框的模式

## 从连接后的数据框中选择若干列

## 从 df_flightinfo 创建一个新的 df_flightinfo_times 数据框

## 增加一个新列，该列通过计划时间和实际耗时进行计算

## 返回前十行

## 显示最大时间差值

## 显示最小时间差值

## 显示平均时间差值

## 我们可以使用单个命令生成特定列的汇总统计信息

## 选择所有延误超过 60 分钟的航班

## 按航空公司对 Time_diff 数据进行分组，并返回计划时间与实际耗时之间的平均差值

## 数据框执行

## 我们的数据框在 Spark 节点上的存储使用情况

## 使用数据库内机器学习模型为数据评分

## 7. 机器学习模型训练与评估

## YAML 参数配置与应用部署

## 大数据集群故障排除、升级与删除
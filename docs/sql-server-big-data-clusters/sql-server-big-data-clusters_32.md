# 同样将其他 CSV 文件导入数据框
df_airlines = spark.read.format('csv').options(header='true', inferSchema="true").load('/Flight_Delays/airlines.csv')
df_flights = spark.read.format('csv').options(header='true', inferSchema="true").load('/Flight_Delays/flights.csv')
```
代码清单 6-17
将 airlines.csv 和 flights.csv 导入数据框

执行代码清单 6-17 后，我们应该在 PySpark 笔记本中拥有三个独立的数据框：`df_airports`、`df_airlines` 和 `df_flights`。

要连接两个数据框，我们必须提供用于连接这两个数据框的键。如果此键在两个数据框上相同，我们不必在连接中显式设置映射（我们只需要将列名作为参数提供）。然而，我们认为始终描述连接映射是一个好习惯，以使代码更易于理解。另外，在我们使用的示例数据集中，我们需要连接的列在数据框上具有不同的列名，这就需要显式的连接映射。

代码清单 6-18 的示例代码将使用内连接将 `df_flights` 和 `df_airlines` 数据框连接在一起，并输出一个名为 `df_flightinfo` 的新数据框。我们返回新创建数据框的模式，以查看两个数据框是如何连接在一起的（图 6-16）。

![../images/480532_2_En_6_Chapter/480532_2_En_6_Fig16_HTML.jpg](img/480532_2_En_6_Fig16_HTML.jpg)
图 6-16
df_flightinfo 数据框的模式，它是 df_flights 和 df_airlines 的连接结果

```
from pyspark.sql.functions import *

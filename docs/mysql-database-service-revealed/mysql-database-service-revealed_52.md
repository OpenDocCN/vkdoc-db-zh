# 循环遍历数据，仅打印规格名称。
for shape_summary in list_shapes_response.data:
print(shape_summary.name)
清单 9-17
列出 MySQL 规格脚本
```

当你执行这段代码时，你会看到 MySQL 规格列表的一个摘录，如清单 9-18 所示。

```
C:\Users\cbell>python list_mysql_shapes.py
Enter the SSH key passphrase:
VM.Standard.E2.1
VM.Standard.E2.2
VM.Standard.E2.4
VM.Standard.E2.8
MySQL.VM.Standard.E3.1.8GB
MySQL.VM.Standard.E3.1.16GB
MySQL.VM.Standard.E3.2.32GB
MySQL.VM.Standard.E3.4.64GB
MySQL.VM.Standard.E3.8.128GB
MySQL.VM.Standard.E3.16.256GB
MySQL.VM.Standard.E3.24.384GB
MySQL.VM.Standard.E3.32.512GB
MySQL.VM.Standard.E3.48.768GB
MySQL.VM.Standard.E3.64.1024GB
...
MySQL.VM.Optimized3.1.8GB
MySQL.VM.Optimized3.1.16GB
MySQL.VM.Optimized3.2.32GB
MySQL.VM.Optimized3.4.64GB
MySQL.VM.Optimized3.8.128GB
MySQL.VM.Optimized3.16.256GB
MySQL.HeatWave.BM.Standard.E3
清单 9-18
列出 MySQL 规格脚本的输出
```

现在，让我们稍微增加一点复杂度。看看如何控制数据库系统。

### 停止/启动数据库系统

在处理数据库系统时，我们将使用 `oci.mysql.DbSystemClient` 类及其方法。要停止数据库系统，我们使用 `stop_db_system()` 方法；要启动数据库系统，我们使用 `start_db_system()` 方法。正如你所看到的，API 方法通常非常直观。

这个例子稍微复杂一些，因为我们必须以另一个名为 `stop_db_system_details()` 的类的形式提供一个数据库系统 ID（OCID）来执行停止或启动操作。这个类使用一个模型来填充数据。在这个案例中，我们需要数据库系统 OCID 和关闭类型。

我们还将通过首先列出我们 compartment 中的数据库系统并选择一个具有特定名称的来使它更复杂一些。你可能会惊讶于记住 OCIDs 有多困难，但记住一个数据库系统名称要容易得多。我们将显示一个列表供用户选择要控制的数据库系统。要列出数据库系统，我们使用 `list_db_systems()` 方法并传入 compartment ID，我们已经将其保存在一个环境文件中。

总结一下，我们将导入需要的库，读取配置文件，从用户读取 SSH 密钥密码短语，从环境变量读取 compartment ID，然后从用户读取数据库系统名称。一旦我们有了这些信息，我们就可以调用方法列出该 compartment 中的数据库系统，选择一个与名称匹配的，然后停止或启动它。

有趣的是，这些 API 方法不像我们在 CLI 示例中看到的那样返回一个工作请求 ID。相反，该方法会等待直到操作完成。清单 9-19 展示了这个示例的完整代码。文件名为 `stop_db_system.py`。在继续下一个示例之前，花点时间研究一下它。

```
import getpass
import os
import sys

# Check response codes as needed and exit
Listing 9-16
Basic Python API Script
```

您可以使用以下命令执行此脚本。您应该会看到它运行然后无错误地退出。如果未正常退出，请在继续之前检查您的配置文件和 Python SDK 安装，确保一切已正确安装和配置：

```
python3 basic_api.py
```

现在，让我们看看在 CLI 部分使用过的一些示例，了解如何使用 Python SDK 完成相同的操作。让我们从使用 `oci.mysql.MysqlaasClient` 类列出 MySQL 形状开始。


### 列出 MySQL 规格

要列出 MySQL 规格，我们使用 `oci.mysql.MysqlaasClient` 类中的 `list_shapes()` 方法，该方法接受 compartment ID 作为参数，这在许多 API 方法中都很常见。

由于我们会经常使用 compartment ID，可以将其保存在环境变量中，并从 Python 脚本中读取。在 macOS 或 Linux 中设置环境变量，请使用 `export` 命令，如下所示：

```
export COMPARTMENT_ID=ocid1.compartment.MASKED
```

对于 Windows，我们使用 `set` 命令，如下所示：

```
set COMPARTMENT_ID=ocid1.compartment.MASKED
```

在 Python 中，你可以使用 `os` 模块读取环境变量，如下所示：

```
import os
COMPARTMENT_ID = os.getenv('COMPARTMENT_ID')
```

调用 API 获取规格列表的代码如下：

```
list_shapes_response = mysql_client.list_shapes(compartment_id=COMPARTMENT_ID)
```

这会返回一个 `ShapeSummary` 类的列表，我们可以用它来遍历规格并打印名称，这比 CLI 的 JSON 输出要整洁得多（并且行数更少）。你可以在 [`https://docs.oracle.com/en-us/iaas/tools/python/2.79.0/api/mysql/models/oci.mysql.models.ShapeSummary.xhtml#oci.mysql.models.ShapeSummary`](https://docs.oracle.com/en-us/iaas/tools/python/2.79.0/api/mysql/models/oci.mysql.models.ShapeSummary.xhtml%2523oci.mysql.models.ShapeSummary) 找到 `ShapeSummary` 类的描述。

好的，现在我们来看完整的代码。清单 9-17 展示了这个示例的完整代码。文件名为 `list_mysql_shapes.py`。

```
import getpass
import os

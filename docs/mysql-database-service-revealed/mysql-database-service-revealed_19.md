# Close the cursor and connection
cur.close()
cnx.close()
```

**清单 3-10** MySQL 连接与查询示例

一旦你输入了代码，就可以执行它以查看结果，如下所示：

> **提示**
> 确保你的 MySQL 服务器正在运行，并且你在字典中为服务器提供了正确的密码和主机名。

```bash
% python3 ./mysql_connect.py
animals
greenhouse
information_schema
mysql
performance_schema
plant_monitoring
sakila
sys
world
world_x
```

根据你安装或创建的示例数据库或其他数据库，你的结果可能会有所不同，但你至少应该看到 `plant_monitoring`、`mysql`、`information_schema` 和 `performance_schema`。

如果你遇到如下所示的错误，请务必检查字典中的凭据，确保你使用了正确的主机名（或 IP 地址）、端口、用户和密码。

```bash
Error: 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

现在让我们看看如何插入数据。

## 向表中插入数据

现在让我们看看如何向表中插入一些数据。我们将此示例代码命名为 `mysql_insert.py`。

在这个例子中，我们只是想从文件中读取数据并将其插入到一个表中。我们将使用与之前示例相同的代码连接到服务器，并使用游标执行查询。不同之处在于，我们将使用一个文件以逗号分隔值格式（`.csv`）读取示例数据。这是一种在各种应用程序中常用的格式。

对于文件中的每一行，我们解码字段，然后使用列中的数据形成一条 `INSERT` 命令。我们将再次使用游标类的 `execute()` 方法来执行插入数据的查询。由于没有结果，我们不获取任何内容。但是，在插入完所有行后，我们必须调用游标类的 `commit()` 方法来提交更改。清单 3-11 展示了此示例的完整代码。请花点时间阅读以明确其逻辑。

```python
"""mysql_insert.py"""
#

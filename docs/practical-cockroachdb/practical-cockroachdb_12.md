# 第 5 章 与 CockroachDB 交互

```
[['Ada Lovelace'], ['Alice Ball'], ['Rosalind Franklin']],
```

)

结束

带有参数的查询应该始终进行参数化。在 Ruby 中，你使用 `exec_params` 函数将参数传递给查询。然后，我传递一个多维数组作为参数，每个数组包含要插入的一行的字段。

现在让我们读出数据。`exec`/`exec_params` 函数也可以用来返回数据。在下面的数据中，我传递一个代码块，查询结果以行的形式在其中可用。对于返回的每一行，我获取 `"id"` 和 `"name"` 列：

```ruby
conn.transaction do |tx|
  tx.exec('SELECT id, name FROM person') do |result|
    result.each do |row|
      puts row.values_at('id', 'name')
    end
  end
end
```

最后，我们将运行应用程序：

```
$ ruby main.rb
```

```
2bef0b0a-3b57-4f85-b0a2-58cf5f6ab7e4
["Ada Lovelace"]
5096e818-3236-4697-b919-8695fde1581d
["Rosalind Franklin"]
cf5b1f2d-f4af-4aab-8c67-3a4ced6f6c07
["Alice Ball"]
```

#### Crystal 示例

接下来是 Crystal！我最近在几个个人项目中使用了 Crystal（包括以 CockroachDB 为后端的项目），所以我想确保涵盖它。

让我们准备环境：

```
$ crystal init app crystal_example
$ cd crystal_example
```

接下来，我们将引入一个依赖项，以帮助我们与 CockroachDB 通信。打开 `shard.yml` 文件并粘贴以下内容：

```yaml
dependencies:
  pg:
    github: will/crystal-pg
```

依赖项添加到我们的 shard 文件中后，`shards` 命令就会知道该获取什么。现在调用它来引入我们的数据库驱动程序：

```
$ shards install
```

在 `src/crystal_example.cr` 文件中，让我们连接到数据库：

```crystal
require "db"
require "pg"

db = PG.connect "postgresql://demo:demo43995@localhost:26257/defaultdb?auth_methods=cleartext"

db.exec "INSERT INTO person (name) VALUES ($1), ($2), ($3)",
  "Florence R. Sabin", "Flossie Wong-Staal", "Marie Maynard Daly"

db.query "SELECT id, name FROM person" do |rs|
  rs.each do
    id, name = rs.read(UUID, String)
    puts "#{id} #{name}"
  end
end
```

最后，我们将运行应用程序：

```
$ crystal run src/crystal_example.cr
```

```
3bab85db-8ccb-4e9f-b1ba-3cf8ed1f0fc8 Florence R. Sabin
70f34efc-3c8a-4308-bdc6-dddafaccea57 Flossie Wong-Staal
cd37624f-2037-440e-b420-84a64a855792 Marie Maynard Daly
```

#### C# 示例

接下来是 C#。让我们准备环境。在这个示例中，我将使用 .NET 6 和 C# 10：

```
$ dotnet new console --name cs_example
$ cd cs_example
```

接下来，我们将引入一些 NuGet 包依赖项。请注意，Dapper 并非使用 CockroachDB 的必需依赖项；它只是本例中的一个偏好选择：

```
$ dotnet add package System.Data.SqlClient
$ dotnet add package Dapper
$ dotnet add package Npgsql
```

现在是代码部分。为保持代码简洁，我省略了异常处理：

```csharp
using Dapper;
using System.Data.SqlClient;
using Npgsql;

var connectionString = "Server=localhost:26257;Database=defaultdb;UserId=demo;Password=demo43995";

using (var connection = new NpgsqlConnection(connectionString))
{
    var people = new List<Person>()
    {
        new Person { Name = "Kamala Harris" },
        new Person { Name = "Jacinda Ardern" },
        new Person { Name = "Christine Lagarde" },
    };

    connection.Execute("INSERT INTO person (name) VALUES (@Name)", people);

    foreach (var person in connection.Query<Person>("SELECT * FROM person"))
    {
        Console.WriteLine($"{person.ID} {person.Name}");
    }
}

public class Person
{
    public Guid? ID { get; set; }
    public string? Name { get; set; }
}
```

最后，我们将运行应用程序：

```
$ dotnet run
```

```
4e363f35-4d3b-49dd-b647-7136949b1219 Christine Lagarde
5eecb866-a07f-47b4-9b45-86628864e778 Jacinda Ardern
7e466bd2-1a19-465f-9c70-9fb5f077fe79 Jacinda Ardern
```

#### 数据库设计

到目前为止，我们设计的数据库模式始终保持简单，以帮助演示特定的数据库功能，如数据类型。在本节中，我们将创建的数据库模式将更准确地反映你在实际环境中创建的内容。

#### 数据库设计

我们将从查看 CockroachDB 的顶层对象——数据库开始。到目前为止，为了简单起见，我们都是在 `defaultdb` 上创建表。现在是创建我们自己的数据库的时候了。

在创建数据库之前，一个重要的决定是它的位置。它将位于单个区域，还是跨越多个区域？

如果你的数据库将保留在单个区域内，可以按如下方式创建：

```sql
CREATE DATABASE db_name;
```

如果名为 `"db_name"` 的数据库已存在，前面的命令将返回错误。以下命令仅在同名数据库尚不存在时才会创建数据库：

```sql
CREATE DATABASE IF NOT EXISTS db_name;
```

如果你的数据库将跨越多个区域，则需要在创建数据库时向 CockroachDB 提供一些区域提示。这些提示包括数据库的主区域以及它要在其中运行的任何其他区域。

为了演示这一点，我将重用前面章节中的一个命令，该命令创建一个跨越三个区域的九节点 CockroachDB 集群：

```
$ cockroach demo \
--no-example-database \
--nodes 12 \
--insecure \
--demo-locality=region=us-east1,az=a:region=us-east1,az=b:region=us-east1,az=c:region=europe-north1,az=a:region=europe-north1,az=b:region=europe-north1,az=c:region=europe-west1,az=a:region=europe-west1,az=b:region=europe-west1,az=c:region=europe-west3,az=a:region=europe-west3,az=b:region=europe-west3,az=c
```

获取区域信息确认我们创建了预期的集群拓扑：

```
root@:26257/defaultdb> SHOW REGIONS;
```

```
  region    |   zones    | database_names | primary_region_of
------------+------------+----------------+--------------------
  europe-north1 | {a,b,c} | {}             | {}
  europe-west1  | {a,b,c} | {}             | {}
  europe-west3  | {a,b,c} | {}             | {}
  us-east1      | {a,b,c} | {}             | {}
```

让我们创建一个跨越欧洲区域的数据库，看看这会如何影响我们的区域视图：

```sql
CREATE DATABASE db_name
PRIMARY REGION "europe-west1"
REGIONS = "europe-west1", "europe-west3", "europe-north1"
SURVIVE REGION failure;
```

```
root@:26257/defaultdb> SHOW REGIONS;
```

```
  region    |   zones    | database_names | primary_region_of
------------+------------+----------------+--------------------
  europe-north1 | {a,b,c} | {db_name}      | {}
  europe-west1  | {a,b,c} | {db_name}      | {db_name}
  europe-west3  | {a,b,c} | {db_name}      | {}
  us-east1      | {a,b,c} | {}             | {}
```

`SHOW REGIONS` 的结果确认我们的 `"db_name"` 数据库在欧洲区域运行，并且主区域为 `"europe-west1"`，正如我们配置的那样。

`SHOW REGIONS` 命令接受额外的参数以进一步缩小搜索范围。以下命令仅显示 `"db_name"` 数据库的区域：

```
root@:26257/defaultdb> SHOW REGIONS FROM DATABASE db_name;
```

```
  database  |    region    | primary |   zones
------------+--------------+---------+-----------
  db_name   | europe-west1 |  true   | {a,b,c}
  db_name   | europe-north1|  false  | {a,b,c}
  db_name   | europe-west3 |  false  | {a,b,c}
```

#### 模式设计

如果你需要在逻辑上分离数据库对象（例如表），那么创建用户定义的模式是一个不错的选择。在以下示例中，我们将假设我们已经决定使用用户定义的模式来逻辑分离数据库对象。使用自定义模式是可选的。如果你不指定，将使用默认的 `"public"` 模式。

在这个例子中，我们正在构建一个数据库来支持一个简单的在线零售业务。支持该在线业务的业务领域如下：

- **零售** – 构建和运营网站的团队
- **制造** – 生产产品的团队
- **财务** – 管理公司账户的团队


在这样的业务场景中，模式（schemas）非常合理，因为每个业务领域都很可能需要一张 "orders" 表。零售团队需要记录客户订单，制造团队需要记录原材料订单，而财务部门则希望将公司卡发生的交易也作为订单来记录。


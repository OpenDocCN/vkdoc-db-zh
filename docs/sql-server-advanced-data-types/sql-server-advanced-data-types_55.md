# 第 6 章 理解 JSON

```json
"PhoneNumber": "(415) 555-0102",
"EmailAddress": "hudsonh@wideworldimporters.com"
},
{
"PersonID": 14,
"FullName": "Lily Code",
"PreferredName": "Lily",
"LogonName": "lilyc@wideworldimporters.com",
"PhoneNumber": "(415) 555-0102",
"EmailAddress": "lilyc@wideworldimporters.com"
},
{
"PersonID": 15,
"FullName": "Taj Shand",
"PreferredName": "Taj",
"LogonName": "tajs@wideworldimporters.com",
"PhoneNumber": "(415) 555-0102",
"EmailAddress": "tajs@wideworldimporters.com"
},
{
"PersonID": 16,
"FullName": "Archer Lamble",
"PreferredName": "Archer",
"LogonName": "archerl@wideworldimporters.com",
"PhoneNumber": "(415) 555-0102",
"EmailAddress": "archerl@wideworldimporters.com"
},
{
"PersonID": 20,
"FullName": "Jack Potter",
"PreferredName": "Jack",
"LogonName": "jackp@wideworldimporters.com",
"PhoneNumber": "(415) 555-0102",
"EmailAddress": "jackp@wideworldimporters.com"
}
]
}
```

与 XML 文档中使用元素不同，JSON 文档仅由 JSON 对象数组中的名称/值对组成。同时，添加了一个名为`SalesPeople`的根节点。

关于 JSON 文档，最明显的观察是其字符数要短得多，这主要归因于缺少结束标签。由此带来的主要后果是，文档更易于解析，并且在应用程序层之间需要传输的信息更少。可以说，该文档对人类也更具可读性。这些优势使 JSON 成为应用程序开发人员非常流行的选择。

JSON 文档的主要缺点是它不能像 XML 那样绑定到模式。其影响是，虽然可以解析文档以确保其语法有效，但无法（在没有自定义代码的情况下）在发送前确保其符合接收方期望的合约。

除了上述差异，尽管它们的外观不同，但这些文档实际上非常相似。它们都是自描述的、可扩展的文档，可用于数据交换。两种格式都被广泛使用，并且与许多`REST` API 和 Web 服务端点配合使用。

**提示**: `REST`是表述性状态传递的缩写。它是一种提供统一接口的架构风格，通常在应用程序的层之间使用。它提供了一种无状态的方法，实现了客户端/服务器分离。

## JSON 使用场景

`SQL Server`中有许多 JSON 数据的用例。以下部分将介绍其中一些潜在用途。

### 使用 REST API 的 N 层应用程序

现代应用程序通常在客户端包含大量逻辑。应用程序的应用层通常必须包含复杂的代码，甚至多个子层，以协调客户端和后端`RDBMS`之间的对话。这是因为您必须使用对象关系映射器来执行数据库查询，将这些结果写入数据传输对象，然后将结果序列化为 JSON 格式，之后才能将其发送给客户端。

然而，借助`SQL Server`中的 JSON 支持，您可以简单地将数据从`SQL Server`公开到`REST` API，并以 JSON 格式返回数据，这意味着应用层可以直接将数据原样发送给客户端。虽然这种做法可能会遭到中层纯粹主义者的反对，但它无疑允许架构师简化应用程序设计。

### 数据去规范化

当数据需要高频率更新时，使用规范化数据模型是完美的选择。在规范化模型中，数据被拆分到多个表中，这些表通过主键和外键约束连接在一起，旨在只存储一次数据。例如，如果客户有多个地址，那么关于客户的核心详细信息（如姓名和电话号码）可能存储在一个名为`Customers`的表中。他们的地址随后可能存储在一个单独的名为`CustomerAddresses`的表中，该表包含一个外键（`CustomerID`），该外键连接到`Customers`表的主键。这意味着客户的核心详细信息只存储一次，而不必为每个地址重复这些详细信息。

**提示**: 虽然对规范化的详细讨论超出了本书的范围，但完整的讨论可以在*Expert Scripting and Automation for SQL Server DBAs* (Apress, 2016)中找到。

然而，传统的规范化模型在某些情况下会导致问题。例如，当数据分散在多个表中并在`SELECT`语句中连接在一起时，由于需要匹配主键和外键值，性能可能会下降。此外，当跨多个表更新数据时，必须使用事务以确保一致的更新。这可能导致锁定问题（如果使用了悲观隔离级别）或`IO`性能问题（如果使用了乐观隔离级别）。

**提示**: 关于事务隔离级别的完整讨论可以在*Pro SQL Server Administration* (Apress, 2015)中找到。

为了解决这个问题，数据架构师有时会使用`NoSQL`结构来存储诸如客户等实体的详细信息，以便逻辑实体可以作为单条记录存储。巧合的是，JSON 通常是这些记录使用的格式。然而，当`NoSQL`数据必须与仍以关系格式存储的数据结合时，这种方法又会产生自己的问题。

JSON 可以通过允许将 JSON 记录存储在`SQL Server`的表中来帮助解决这类建模挑战，这意味着可以更新单个表，同时该表仍然可以轻松地与关系数据连接。

### 配置即代码

在 DevOps 环境中，有将基础设施即代码、平台即代码和配置即代码的要求。本质上，整个虚拟资产都将用代码编写，以便在数据中心之间，或者更常见的是在数据中心和云之间具有高度的可移植性。在发生灾难（例如数据中心丢失）时，它也具有高度的可恢复性。

`SQL Server`管理整合到 DevOps 领域一直进展缓慢，因为`SQL Server`与期望状态配置工具（如`Chef`和`Puppet`）之间缺乏交叉技能。不过，`DBA`领域正慢慢开始参与其中，作为`SQL Server`平台即代码方法的一部分，通常会在中央管理服务器上使用配置管理数据库（`CMDB`），以存储环境内成员服务器（`SQL Server`虚拟机）的详细信息。

（在此上下文中）中央管理服务器指的是一个`SQL Server`实例，用于帮助`DBA`管理其余的`SQL Server`资产。它通常是`SQL Server Agent`主/目标作业配置中的主服务器，并且可能安装了其他管理功能，例如用作`SQL Server`监控中心的管理数据仓库。

然而，当使用平台即代码方法时，应同时应用配置即代码方法。在`SQL Server`领域，这涉及能够从存储在源代码管理提供程序（如`GitHub`或`TFS`）中的代码重建`CMDB`。

对于`SQL Server``CMDB`的配置即代码可以轻松地通过一个循环流程来管理。该流程的第一步是一个`Server Agent`作业，它会定期运行并将表中的数据导出到操作系统中的 JSON 文件。然后，这些文件将被推送到源代码管理存储库并签入。

当诸如`Chef`或`Puppet`之类的期望状态配置管理工具构建新的中央管理服务器时，它将查看源代码管理存储库以找到包含配置的 JSON 文件


[data] 它将创建配置数据库，然后使用存储库中的数据重新填充表。这为配置数据库提供了一个易于配置的 RPO（恢复点目标），而无需应对通常与企业备份管理工具相关的挑战，以及从磁带机器人恢复数据时常有的等待时间。这也意味着 SQL Server 的核心管理工具可以轻松地迁移到云端或其他数据中心，有助于使 SQL Server 环境具有极高的可移植性。

`提示` 期望状态管理工具（如 `puppet` 和 `Chef`）的原理是，它们在服务器上定期运行，并附带一个描述服务器期望状态的配置清单。在 Windows 层面，这可能包括用于禁用访客账户或确保用户权限分配配置正确的代码。在 SQL Server 层面，它可能会检查特定登录名是否存在或 `xp_cmdshell` 是否已禁用。首先，该工具会检查资源是否已按预期配置。如果没有，它将纠正配置。这意味着，如果对服务器进行了未经授权的更改，下次应用配置清单时该更改将被修正。其结果是，Windows 工程师和数据库管理员也能确保其服务器的状态。


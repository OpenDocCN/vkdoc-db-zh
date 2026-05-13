# 设置阈值与打包插件

## 定义阈值与告警信息

默认的警告和严重阈值可以通过使用 `WARNING` 和 `CRITICAL` 属性在 `Condition` 元素中定义。就像收集计划一样，EM12c 用户可以修改这些阈值。警告和严重阈值都是可选的，如果找不到一个合理良好的默认值，通常可以不定义它们。值 `"NotDefined"` 等同于完全没有定义值，正如 清单 10-13 中针对 `"non_nice"` 列条件所示。我倾向于直接跳过定义，以保持元数据定义的简洁，就像 `Response` 指标的 `Status` 列没有 `WARNING` 阈值一样。

您还可以看到，可以定义列特定的告警信息，并使用像 `%columnName%` 这样的占位符变量来包含列名、最新值以及阈值。由于百分号 (`%`) 具有特殊含义，您应使用双百分号 (`%%`) 来在消息中包含真正的百分号。同样地，您可以定义当条件被清除时在 EM12c 中生成的自定义消息。还可以通过指定属性 `OCCURRENCES` 来设置在触发告警之前，该条件连续发生的次数。

只要在目标默认收集文件中指定了条件操作符，就可以在 EM12c 控制台中自定义阈值以及发生次数。但是，操作符本身无法更改。因此，请特别注意选择正确的条件操作符。

### 打包插件

现在您已经了解了最小化 EM12c 插件的所有组件，还需要放置由目标类型元数据中定义的指标收集提取器所调用的操作系统脚本。我建议使用 Perl 编写操作系统脚本，并使用随 Oracle Management Agents 安装的 Perl 版本。这是最独立于平台的方法（尽管出于简洁性考虑，Oracle 提供的示例仅在 Linux 上工作——Perl 脚本只是调用一些操作系统实用程序来收集数据）。脚本放置在代理端暂存区的 `scripts` 目录中，并创建一个与目标类型同名的子目录。因此，在我们的示例中，`data_collector.pl` 文件被放置在 `<stage>/agent/scripts/sample_host1` 中，其中 `sample_host1` 是目标类型。

所有组件和脚本就位后，您现在可以打包插件归档文件了。您可以在打包之前运行插件验证，就像之前在 清单 10-7 中所做的那样。但是，插件创建过程本身包含一个验证步骤，因此不必单独运行验证。

在部署示例插件时，我使用了脚本 `build_sample_plugin.sh`，它不过是 `empdk create_plugin` 命令的一个小包装器。部署插件的完整命令如下：

```
empdk create_plugin -stage_dir <stage_dir> -out_dir <output_dir>
```

两个必需参数定义了暂存目录的位置以及生成 OPAR 文件的输出目录。有关 `empdk create_plugin` 的可选参数的详细信息，请参阅 *Extensibility Programmer's Reference*（第 13 章，第 13.4 节）。

在打包插件期间，EDK 会在您的暂存区中创建额外的文件，这些文件将被打包到插件归档中。例如，它会创建在部署期间将目标类型定义导入到 OMS 中的 PL/SQL 代码。您无需担心或维护它们；每次创建插件归档时，它们都会被重新创建。

创建的 OPAR 文件用于将插件分发给其他用户（EM12c 管理员），无论您是为自己的组织还是为外部用户创建它，也无论它是免费插件还是商业插件。其部署过程与之前概述的示例插件部署过程相同，也在 *Extensibility Programmer's Reference*（第 13 章，第 13.5 和 13.6 节）中有详细描述。最终用户关于插件管理的文档可在 *Oracle Enterprise Manager Cloud Control 12c Administrator's Guide* 的第 22 章中找到（如果您直接向最终用户分发 OPAR 文件，请参阅第 12.4.3.2 小节的“导入插件归档”部分）。

对于这三个主机示例插件，所有步骤都已在前面完成，因此初始示例已经部署且目标已创建。请注意，`oracle.samples.xsh1` 插件实现了更多功能，您将在本章后面部分学习，但到目前为止您所学的内容基本上是任何带有新目标类型的插件的核心。

## 代理的指标浏览器

使用代理的指标浏览器 (Metric Browser) 接口来查看代理实际收集的内容，甚至查看所收集指标的一些调试信息，通常很有用。这在故障排除时特别有用，当不清楚问题是出在代理端的指标收集环节，还是发生在上传到 OMS 端并加载到存储库的过程中时。要为代理激活指标浏览器，请以运行代理的操作系统用户身份运行以下命令：

```
emctl setproperty agent -name _enableMetricBrowser -value true
```

要确定指标浏览器的 URL，请运行命令 `emctl status agent`。`Agent URL` 行将显示为 `http://host:port/emd/main/` 格式。修改此 URL，在 `emd` 和 `main` 之间添加 `/browser/` 字符串，如下所示：`http://host:port/emd/browser/main`。如果您的代理是安全的，它可能使用 `https` 而不是 `http`，因此在那种情况下请使用 `https` URL。

在使用临时列和计算列时，使用指标浏览器特别有用，这些内容将在后面描述。这里还提供了一个完整的指标定义 XML 片段，其中所有属性都显式定义，甚至包括默认值——这是学习一些未文档化选项的好方法。指标浏览器让您能够访问任何目标类型，而不仅仅是您自己的目标，因此您可以查看 Oracle 自身目标的完整（通常很复杂的）指标定义，这些定义可以为您提供关于如何创建自己指标的绝佳思路。还有其他关于指标收集的调试信息可用，但不幸的是，它尚未在任何文档中详细说明，而且到目前为止我还没有使用它的需要。

## 指标扩展的内部机制

您可能已经注意到，定义目标类型指标与定义指标扩展 (Metric Extensions) 类似，只是采用 XML 格式。实际上，指标扩展功能本质上是用户界面，用于在没有创建全新插件和目标类型的负担下，向现有目标添加新指标。在幕后，指标扩展与您在目标类型元数据中看到的 XML 指标定义是相同的。

所有指标扩展都存储在存储库数据库的 `SYSMAN` 模式中的表 `EM_MEXT_VERSIONS` 里。`METADATA_DEFINITION` 列存储的不是别的，正是作为指标扩展定义的 XML `Metric` 元素。相关的 `CollectionItem` 元素位于该表的 `COLLECTION_DEFINITION` 列中。

`清单 10-14` 包含了我创建的第一个 `"Open Cursors, %"` 指标扩展的 XML 定义。这里为了可读性进行了格式化。

`清单 10-14`. 指标扩展的 XML 表示


```
<Metric NAME="ME$open_cursors_pct" TYPE="TABLE">
  <Display>
    <Label NLSID="NLS_METRIC_oracle_databaseME$open_cursors_pct">打开的游标数，百分比</Label>
    <Description NLSID="NLS_DESCRIPTION_oracle_databaseME$open_cursors_pct">
      打开的游标数相对于 open_cursors 初始化参数的百分比。
      仅返回会话的最大值。</Description>
  </Display>
  <TableDescriptor>
    <ColumnDescriptor NAME="open_cursors_pct" TYPE="NUMBER">
      <Display>
        <Label NLSID="NLS_COLUMN_oracle_databaseME$open_cursors_pctopen_cursors_pct">
           打开的游标数</Label>
        <Unit NLSID="mext_unit_nlsid_ME$open_cursors_pct_open_cursors_pct">%</Unit>
      </Display>
      <CategoryValue CLASS="Default" CATEGORY_NAME="Capacity">
    </CategoryValue>
  </ColumnDescriptor>
</TableDescriptor>
  <QueryDescriptor FETCHLET_ID="SQL">
    <Property NAME="STATEMENT" SCOPE="GLOBAL" OPTIONAL="TRUE">
  SELECT ROUND (c.open_cursors / p.value * 100) open_cursors_pct
  FROM (SELECT   sid, COUNT (*) open_cursors
            FROM v$open_cursor
        GROUP BY sid
        ORDER BY 2 DESC) c,
       v$parameter p
  WHERE p.name = 'open_cursors' AND ROWNUM = 1;</Property>
    <Property NAME="MachineName" SCOPE="INSTANCE" OPTIONAL="TRUE">MachineName</Property>
    <Property NAME="Port" SCOPE="INSTANCE" OPTIONAL="TRUE">Port</Property>
    <Property NAME="SID" SCOPE="INSTANCE" OPTIONAL="TRUE">SID</Property>
    <Property NAME="UserName" SCOPE="INSTANCE" OPTIONAL="TRUE">UserName</Property>
    <Property NAME="password" SCOPE="INSTANCE" OPTIONAL="TRUE">password</Property>
    <Property NAME="Role" SCOPE="INSTANCE" OPTIONAL="TRUE">Role</Property>
    <CredentialRef NAME="SQLCreds">
  </CredentialRef>
</QueryDescriptor>
</Metric>
```

如果你发现从头开始创建 XML 指标定义具有挑战性，可以先创建一个指标扩展作为原型，然后直接从存储库中提取 XML 定义，将其复制到你的目标类型元数据定义 XML 文件中，并根据需要进行修改。原始的 XML 定义为你提供了更大的灵活性。

## EM12c 获取组件

EM12c 自带了十几个有文档说明的获取组件。以下是可用获取组件的简要概述：

### OS 命令获取组件
这类获取组件允许你运行操作系统命令和脚本，并将指标值作为输出传递。你已经看过示例了，这三个 OS 命令获取组件的功能与你在“指标扩展”部分看到的 OS 命令适配器相同。OS 命令由代理使用运行代理本身的同一操作系统用户执行，该代理位于代理安装的主机上。为了尽可能确保平台兼容性，我建议你使用 Perl 脚本作为命令，并使用代理附带的 Perl。脚本和任何依赖项可以添加到插件中，并在插件部署到代理上时进行分发。请注意，如果依赖项无法简单地复制，而需要在主机上进行编译或任何其他配置，那么这些依赖项很可能需要在运行代理的主机上预先安装。这会使部署复杂化，应尽量避免。例如，如果你想通过 Perl 脚本使用 `DBI` 连接到非 Oracle 数据库，你需要编译所需的 `DBD` 模块，如 `DBD::MySQL`，这需要在插件部署之前，在主机的每个代理上手动完成。因此，更推荐使用 Perl 本地模块（例如此示例中的 `Net::MySQL`——这是我用于 MySQL 插件的模块）。

### SQL 获取组件
此获取组件允许你针对 Oracle 数据库运行 SQL 语句或 `PL/SQL` 块，并返回一个值表。可以为 `SQL` 和 `PL/SQL` 指定多个输入参数。对于 `PL/SQL` 块，必须定义一个输出参数，并以 `PL/SQL REF CURSOR` 形式或定义为对象数组的命名类型的形式从 `PL/SQL` 返回。该获取组件的属性定义了数据库连接和凭据，以及 `SQL` 参数、要运行的内联 `SQL` 语句或 `SQL` 文件，以及要检索的最大行数。此获取组件不能用于针对非 Oracle 数据库运行 `SQL`。此获取组件的一些用例包括从 Oracle 数据库中的状态表收集应用指标，或监控在数据库内部运行的应用，例如 Oracle Application Express (`APEX`) 应用。

### SNMP 获取组件
此获取组件允许从 `SNMP` 目标收集指标。基于 `SNMP` 获取组件的指标可以从多个对象标识符 (`OID`) 收集值。每个 `OID` 可以引用单个变量或多个实例（例如，每个网络接口一个实例）。该获取组件还支持 `PINGMODE`，这是一种创建响应指标的简单方法，如果 `SNMP` 端口响应则返回 1，如果超时则返回 0。这比生成指标收集错误要好。该获取组件支持基于团体字符串的 `SNMP v1` 和 `v2` 协议，以及更安全的 `v3` 协议，后者需要凭据而不是团体字符串。显而易见的用例是监控 Oracle 代理原生不支持的远程网络设备或远程主机（例如 Ubuntu Linux），或者因某些原因无法安装代理的设备。正如你可能猜到的，指标扩展的 `SNMP` 适配器就是基于此获取组件。然而，`SNMP` 适配器的配置非常有限，以至于在指标扩展的上下文中几乎无用。

### HTTP 数据获取组件
这类获取组件与 OS 命令获取组件非常相似，但它通过使用指定的 `URL` 执行 `HTTP GET` 请求来获取输出，而不是执行 OS 命令。输出可以是原始值、值行或基于分隔符的标记化值行——所有选项与 OS 命令获取组件相同。

### URLXML 获取组件
此获取组件可以处理从 `HTTP URL` 检索的 `XML` 响应，并根据预定义的模式匹配提取表格数据。它仅支持 `HTTP GET` 请求，也可以配置代理服务器，类似于 HTTP 数据获取组件。


请注意，模式被定义为从分层的 XML 结构中提取行，因为指标接受的是值表格。每个值是一个数字或字符串，因此无法返回和处理复杂类型。你需要将数据集合扁平化。

*   `REST fetchlet`：此组件提供了从 RESTful 数据源收集数据的方法。它支持`GET`和`POST` HTTP 方法以及基本身份验证。响应格式可以是`XML`或`JavaScript 对象表示法`（`JSON`），并可以通过使用`XPath`或`JSONPath`转换为用于指标的表格化呈现。同时支持`HTTPS`。
*   `URL Timing fetchlet`：这是一种用于测试 Web 应用程序性能的有用收集机制。它支持简单的代理，并可用于`HTTP`和`HTTPS`协议。该 fetchlet 支持`HTTP`的基本身份验证。它具备定义连接超时和重试次数的能力，并且在配置所收集的指标方面非常灵活。该 fetchlet 可以接受多个 URL。对于每个 URL，它不仅获取页面本身，还获取页面所需的任何内容，如图像、样式表等。它可以提供每个 URL 或每个 URL 的详细统计信息。有几十个统计指标，从明显的如接收的总字节数和检索页面的总时间，到高级统计如`DNS`解析时间或每个请求接收第一个字节前的平均时间。
*   `JDBC fetchlet`：此组件与`URL Timing fetchlet`类似，因为它收集针对任何支持`JDBC`连接的数据库执行指定 SQL 语句的时间信息。它收集诸如请求状态、总时间、检索行数等指标，以及更精细的计时指标，如独立的连接时间和获取时间。遗憾的是，这个`JDBC fetchlet`不返回`SELECT`命令执行结果所检索到的值。为了更好地反映其目的，将其称为`JDBC Timing fetchlet`可能更合适，但 Oracle 可扩展性文档并非如此命名。
*   `Web Services and JMX fetchlets`：这些组件提供了使用 Web 服务或 Java 管理扩展（`JMX`）来检索数据的机制。两者都是被广泛采用的行业标准。Web 服务是一种通用的分布式消息传递和远程执行标准，而`JMX`是专门为 Java 应用程序可管理性设计的标准。虽然每个 Java 虚拟机（`JVM`）实现通常都支持`JMX`，但应用程序需要进行插桩，以便将其指标报告给`JVM`，从而使监控工具能够访问这些指标。Oracle 的应用服务器支持`JMX`，许多其他软件产品如 IBM WebSphere 和 Red Hat JBoss 也同样支持。许多 Apache 项目也支持`JMX`——例如 Apache Tomcat 或 Hadoop 生态系统中的 Apache ZooKeeper。事实上，Hadoop 的核心组件`NameNode`和`JobTracker`都进行了插桩，以便通过`JMX`暴露其运行时指标。*可扩展性程序员参考手册*的第 20 章详细介绍了如何基于 Web 服务和`JMX`创建新的目标类型。
*   `WBEM fetchlet`：此收集器用于从支持基于 Web 的企业管理（`WBEM`）标准的组件检索指标。该标准在业界被广泛采用，并得到许多开源和专有软件的支持。`WBEM`标准比过时的`SNMP`协议先进得多，因为它包含所有元数据、更安全，并且在不同供应商之间兼容性更好。`WBEM fetchlet`本质上是一个通用信息模型（`CIM`）客户端，它连接到`CIM 对象管理器`（`CIMOM`），通常称为`CIM 服务器`。Windows 管理接口（`WMI`）是微软对`WBEM/CIM`的实现。`CIMOM`服务器可以嵌入在设备或服务器中（例如某些 Cisco 设备或 Windows 或 HP-UX）。

此外，还有一个与`WBEM fetchlet`相似的`WS-Management fetchlet`，但它基于`WS-Management`标准。微软 Windows 远程管理（`WinRM`）就是`WS-Management`实现的一个例子。
*   `DMS Fetchlet`：此收集器允许使用动态监控服务（例如，在 Oracle 应用服务器和融合中间件中实现）。它允许应用程序进行插桩，以通过`DMS`展示运行时指标，然后代理可以使用非常高效的收集机制。与基于 Shell 脚本的指标不同，`DMS fetchlet`收集器不会派生任何进程，因此在资源消耗方面非常高效。`DMS API`于 2001 年作为 Java 规范请求 138：性能指标插桩提交给了 Java 社区进程。然而，它并未在业界广泛采用，并于 2010 年被撤回。



![image](img/sq.jpg) **注意** 你可以在*《可扩展性程序员参考手册》*的第 20 章找到关于 fetchlet 的完整描述。

此外，一些 fetchlet 并未被文档记录（因此，不能保证在将来甚至当前版本中仍能工作）。例如，`JDBCSQL` fetchlet 与 JDBC 定时 fetchlet 不同，它允许通过 JDBC 连接到任何带有 JDBC 驱动的数据库来收集 SQL 返回的值，而不仅限于 Oracle。但是，请谨慎依赖未记录的 fetchlet，因为任何 EM12c 补丁集或补丁都可能影响这些 fetchlet 的功能或可用性，即使它们目前看起来工作正常。

## 计算指标列

最可靠且同时最简单的收集机制是无状态的——它们返回的值仅源自被监控目标的当前状态，而不考虑之前的收集。一个典型的例子是当前活动数据库会话的数量。然而，我们经常需要计算指标相对于之前收集值的增量差异。例如，你可能希望收集累积计数器的当前值，例如自数据库实例启动以来的用户提交次数，或者如你将在主机示例中看到的，CPU 核心在某种模式下工作的累积 CPU ticks 数（例如在 Linux 中使用 `vmstat -s`）。这些累积值本身并不有用，但可以用于指示自上次收集以来的差异。

让我们看一下来自 sample_host1 目标类型的 CPU 利用率收集示例（参见 清单 10-15）。

**清单 10-15**. 使用 `TRANSIENT` 和 `COMPUTE_EXPR` 列

```xml
<Metric NAME="CPUPerf" TYPE="TABLE">
  <TableDescriptor>
      <ColumnDescriptor NAME="non_nice_t" TYPE="NUMBER" TRANSIENT="TRUE"/>
      <ColumnDescriptor NAME="nice_t" TYPE="NUMBER" TRANSIENT="TRUE"/>
      <ColumnDescriptor NAME="system_t" TYPE="NUMBER" TRANSIENT="TRUE"/>
      <ColumnDescriptor NAME="idle_t" TYPE="NUMBER" TRANSIENT="TRUE"/>
      <ColumnDescriptor NAME="io_wait_t" TYPE="NUMBER" TRANSIENT="TRUE"/>
      <ColumnDescriptor NAME="irq_t" TYPE="NUMBER" TRANSIENT="TRUE"/>
      <ColumnDescriptor NAME="non_nice" TYPE="NUMBER"
                        COMPUTE_EXPR="100.0 * (non_nice_t - _non_nice_t)/(
                                                (non_nice_t - _non_nice_t) +
                                                (nice_t - _nice_t) +
                                                (system_t - _system_t) +
                                                (idle_t - _idle_t) +
                                                (io_wait_t - _io_wait_t) +
                                                (irq_t - _irq_t)
                                               )">
        <Display>...</Display>
      </ColumnDescriptor>
      <ColumnDescriptor NAME="nice" TYPE="NUMBER"
                        COMPUTE_EXPR="100.0 * (nice_t - _nice_t)/(
                                                (non_nice_t - _non_nice_t) +
                                                (nice_t - _nice_t) +
                                                (system_t - _system_t) +
                                                (idle_t - _idle_t) +
                                                (io_wait_t - _io_wait_t) +
                                                (irq_t - _irq_t)
                                               )">
      ...
  </TableDescriptor>
</Metric>
```

首先，你会看到前七列被标记了属性 `TRANSIENT="TRUE"`。这告诉代理，这些列的值实际上不会被发送到 OMS 存储到仓库中，而是被收集起来以便用于计算其他列的值。定义了 `COMPUTE_EXPR` 属性的列，其值不是来自 fetchlet，而是使用该定义的表达式来计算值。计算表达式可以通过名称引用现有的列，并且这些被引用的列必须在 `TableDescriptor` 中先定义，然后才能在 `COMPUTE_EXPR` 中被引用，因为列值是按从上到下的顺序依次计算的。

要引用前一次收集中列的值，只需在列名前加上一个下划线符号 (`_`)。因此，如果当前值是 `nice_t`，那么前一次收集的值就是 `_nice_t`。于是，差值就是 `nice_t - _nice_t`，这表示自上次收集以来在用户模式下执行 niced 进程所花费的 CPU ticks 数。将自上次收集以来的所有 CPU ticks 相加，就得到了自上次收集以来的 CPU ticks 总数。CPU 执行 niced 用户进程的时间百分比，就是 niced CPU ticks 与总 ticks 数的比值，通过乘以 100 归一化为百分比。这同样适用于其他所有 CPU 模式，如 system 或 idle CPU。

让我们看另一个例子：计算用户回滚和提交的次数。Oracle 数据库保留了自数据库启动以来的提交和回滚计数器，但纯粹基于累积计数器的计算对于监控并不有用。有用的是两次收集间隔之间平均每秒的提交和回滚次数。因此，如果前一次提交数是在时间 `t0` 收集的 `c0`，当前收集是在 `t1` 收集的 `c1`，那么每秒提交次数就是 `(c1–c0)/(t1–t0)`。我不会在指标定义中特别引用时间戳，因为 EM12c 会隐式添加时间戳（有一种方法可以显式收集它，但很少需要）。因此，我们无法从 `COMPUTE_EXPR` 定义中访问该时间戳值。然而，我们真正需要的是时间间隔。

尽管你在默认收集定义中指定了间隔，但你不应该将这个值硬编码到 `COMPUTE_EXPR` 中，因为 (1) 用户可以自定义它，并且 (2) 实际间隔可能略有不同，因为配置的间隔只是一个目标值。幸运的是，EM12c 有一个内置变量 `__interval`（注意是双下划线前缀），它返回自上次收集以来的秒数。因此，一个包含每秒提交和回滚次数的指标可能如 清单 10-16 所示。

**清单 10-16**. 在 `COMPUTE_EXPR` 列中使用 `__interval`

```xml
<Metric NAME="transactions_stat" TYPE="TABLE">
  <TableDescriptor>
    <ColumnDescriptor NAME="commits" TYPE="NUMBER" TRANSIENT="TRUE"/>
    <ColumnDescriptor NAME="commits_ratio" TYPE="NUMBER"
                      COMPUTE_EXPR="(commits - _commits) / __interval">
      <Display>...</Display>
    </ColumnDescriptor>
    <ColumnDescriptor NAME="rollbacks" TYPE="NUMBER" TRANSIENT="TRUE"/>
    <ColumnDescriptor NAME="rollbacks_ratio" TYPE="NUMBER"
                      COMPUTE_EXPR="(rollbacks - _rollbacks) / __interval">
      <Display>...</Display>
    </ColumnDescriptor>
  </TableDescriptor>
  <QueryDescriptor>...</QueryDescriptor>
</Metric>
```



**清单 10-16** 中的示例并非来自示例主机目标，但请设想 `QueryDescriptor` 使用了一个 SQL fetchlet，该 fetchlet 返回两列数据：自启动以来数据库实例的提交和回滚累积次数。假设在第一次采集时，fetchlet 返回的提交列值为 1,000。因为是首次采集，之前没有 `_commits` 或 `__interval` 的值，且计算列尚未计算。在 60 秒后进行的第二次采集中，设想 fetchlet 返回的提交数为 1,300。这种情况下，`commit_ratio` 列将计算为 `(1,300 – 1,000)/60`，即每秒 5 次提交。回滚列也适用相同的逻辑。

## 使用计算表达式 (COMPUTE_EXPR)

请注意，`QueryDescriptor` 中定义的 fetchlet 应返回值，并跳过计算列。并非必须将所有计算列的定义延迟到 `TableDescriptor` 的末尾，您可以将计算列放在它们所引用的列之后。计算列由代理计算，您可以在 Metric Browser 中清晰地监控此过程。代理会跟踪 `TRANSIENT` 列的值，但不会将其提交给 OMS，因此不会增大 EM12c 存储库的大小。

计算表达式也可用于转换单位（例如，如果不需要精确度，可从字节转换为千字节或兆字节），或者当某些指标返回块（blocks）时，将其转换为实际字节数。

### 处理重置的累积计数器

我发现，当处理可能随时重置的累积计数器（如同实例统计信息在实例重启时会被重置）时，使用一个技巧很有用，否则可能导致负差值，进而产生巨大的负比率。相反，应跳过此类采集。这对于使您的指标采集值得信赖至关重要。

`COMPUTE_EXPR` 支持一种条件运算符，形式为 `{条件} ? {真值} : {假值}`。如果先前采集的值高于当前值，则应返回一个空值，我倾向于使用空值而非 0，因为它表明采集数据只是缺失。因为 `COMPUTE_EXPR` 不希望公式中出现 `NULL`（无值），我使用了一个技巧来为列生成空值：我通过除以 0 来引发异常。因此，我的公式看起来像 `_val > val ? 1/0 : (val - _val)/__interval`。我了解到 EM12c 会捕获此异常并假定该列值为空。此方法至少从 Grid Control 10.2 开始有效，并且在 12.1.0.2 中仍然有效。但是，请使用任何新的补丁集测试您的指标采集；因为此行为未文档化，不能保证它不会改变。我很希望能有一个显式的表达式来返回类似 `NULL` 的值。

## 关键列 (Key Columns)

就像在指标扩展中一样，可以将某些列声明为关键列，并将从 fetchlet 返回的多行作为表中的多行进行处理。**清单 10-17** 展示了一个按 CPU 统计指标的示例，其中 CPU 编号被定义为 `KEY` 列。结果与指标扩展中的关键列相同。完整的 XML 片段位于 `sample_host1` 目标类型的目标类型元数据中。

**清单 10-17**.  关键列定义示例

```
<Metric NAME="CPUProcessorPerf" TYPE="TABLE">
  <TableDescriptor>
      <ColumnDescriptor NAME="CPUNumber" TYPE="NUMBER" IS_KEY="TRUE"/>
      <ColumnDescriptor NAME="CPUUser" TYPE="NUMBER" IS_KEY="FALSE"/>
      ...
  </TableDescriptor>
  ...
</Metric>
```

## 动态属性 (Dynamic Properties)

我目前只介绍了 EM12c 用户在添加目标实例时定义的静态实例属性。然而，存在一种机制可以动态收集属性，这些属性之后可在指标采集的插件配置中使用。例如，您可能希望收集目标版本或其运行的平台信息，以便随后收集依赖于平台的指标（例如，Linux 和 Windows 平台通常需要不同处理）。您也可以使用动态属性在目标主页上显示（稍后您将了解这部分内容）。

就像静态实例属性一样，动态属性在 `InstanceProperties` 下定义为一个 `DynamicProperties` 元素。该元素结构与 `Metric` 元素相似，但没有 `TableDescriptor`。属性列表在 `PROP_LIST` 属性中定义为分号分隔的列表（例如，`"prop1;prop2;prop3"`）。它还有一个特殊的格式 `ROW`，替代通常用于指标的 `TABLE`。与指标类似，`DynamicProperties` 包含一个 `QueryDescriptor`，定义了用于收集 `PROP_LIST` 中列出属性的 fetchlet。该 fetchlet 应仅返回一行，其值与 `PROP_LIST` 属性相对应。**清单 10-18** 展示了一个来自我们一直用作示例的 `sample_host1` 目标的动态属性的简化片段。

**清单 10-18**.  来自 sample_host1 目标的启动日期动态属性示例

```
<InstanceProperties>
  <InstanceProperty ...>...</InstanceProperty>
  <DynamicProperties NAME="HostSampleBootDate" PROP_LIST="bootDate" FORMAT="ROW">
    <QueryDescriptor FETCHLET_ID="OSLineToken">
      <Property .../>
      <Property NAME="script" SCOPE="GLOBAL">
          %scriptsDir%/sample_host1/data_collector.pl --collect BootDate --fake "%fake%"
      </Property>
      <Property .../>
    </QueryDescriptor>
  </DynamicProperties>
</InstanceProperties>
```

然后，可以在指标采集的 fetchlet 配置中，通过 `SCOPE="INSTANCE"` 的 `Property` 元素引用动态属性——就像引用任何其他静态属性一样。

您可以在 *可扩展性程序员参考手册* 的第 3.3.6.2 节中找到关于 `DynamicProperties` 的更多信息。然而，官方文档相当稀少。您可以在 `<EDK>/doc/partnersdk/mrs/emcore/targetType` 下的 XML 模式文档中找到更多细节。

动态属性的最佳用途之一是定义类别属性，这些属性继而用于目标类型元数据中使用 `ValidIf` 元素的条件定义中。此元素可以定义为多个元素（如 `Metric` 或 `QueryDescriptor`）的子元素，以便仅当定义的类别属性匹配某些选择时才激活某些指标。另一个用例是根据类别属性的值配置不同的 fetchlet 来收集指标值。不幸的是，截至 EM12c 版本 12.1.0.2.0，此功能尚未正式文档化，因此依赖它时应谨慎。然而，它自 Grid Control 10g 版本就已可用，您可以在 Oracle 开箱即用的插件中看到它的使用方式（例如，在其 `rac_database` 目标类型中）。您可以在 `<AGENT_BASE>/plugins/oracle.sysman.db.agent.plugin_12.1.0.2.0/metadata/rac_database.xml` 中找到 `rac_database` 目标类型的元数据定义。请注意，Oracle 通过包含其他文件的 XML 内容来使用 XML 功能，以方便重用相同的定义。例如，RAC 数据库、ASM 实例和数据库实例的数据库版本计算是相同的，因此它被定义在 `dyn_props.xmlp` 文件中，并从多个目标类型元数据定义文件中引用。



请注意，Oracle 的 `sample_host` 目标还包含定义相关链接的其他动态属性。我认为这部分示例自 10g/11g Grid Control 以来就未被移除，且现在已不再相关。在 EM12c 之前，可扩展性框架的功能很少用于自定义目标的主页面，而向主页面的相关链接部分添加自定义链接是其中之一。EM12c 拥有丰富的目标用户界面定义选项，不再使用通过动态属性定义的相关链接，因此所有 `sample_host` 演示目标中的这个动态属性都会被忽略，并且应该已被移除。

## 报告

EM12c 提供了两种报告机制：

*   `Information Publisher`：这与 Grid Control 10g/11g 版本中存在的机制相同，该机制拥有 `Information Publisher` PL/SQL API，并且过去需要使用 PL/SQL 创建报告定义。EM12c 现在的 `Information Publisher` 报告定义是基于 XML 的，并且能够通过使用 EM12c 控制台设计报告以及通过使用 `emcli` 导出报告。为了在 10g/11g 中获得类似的功能，我不得不创建一个自定义提取器，从使用控制台 UI 开发的报告中生成基于 PL/SQL API 的定义。EM12c 仍然会在后台将 XML 转换为 PL/SQL 调用，但与那些冗长繁琐、几乎无法在不尝试执行的情况下进行语法验证的 PL/SQL 定义相比，XML 定义的便利性是巨大的。有关创建和自定义 `Information Publisher` 报告的详细文档（包括 XML 元素的细节），请参阅《*可扩展性程序员参考手册*》的第 4 章。
*   `BI Publisher`：EM12c 可以与 Oracle Business Intelligence Publisher 集成，以获得更高级的报告功能。然而，与 `BI Publisher` 的集成可能需要为 Oracle Business Intelligence Publisher 购买许可，根据我的观察，这阻碍了 `BI Publisher` 在 EM12c 中的采用。如果你正在为外部用户（而不仅仅是你自己的组织）开发插件，你可能不应该依赖目标 EM12c 环境中 `BI Publisher` 的可用性，因此请务必使用 `Information Publisher` 提供功能相当的报告。《*可扩展性程序员参考手册*》的第 5 章提供了关于创建 `BI Publisher` 报告的非常简要的说明，但其文档质量和深度远不及 `Information Publisher` API 的文档。它还要求读者预先了解 Oracle Business Intelligence Publisher 产品。

尽管《*可扩展性程序员参考手册*》的第 4 章指出 `Information Publisher` 已弃用，但现实情况是，有太多现有的报告和客户依赖它，除非 Oracle 将 `BI Publisher` 默认捆绑在 EM12c 中，否则它在 EM12c 乃至可能的后续版本中都不会消失。我个人对于插件开发的建议是，目前坚持使用 `Information Publisher`，直到 Oracle 提供不需要与额外产品（和许可）集成的报告功能。这也确保了你的插件可以在任何 EM12c 安装中运行，无论其是否与 `BI Publisher` 集成。本节仅涵盖 `Information Publisher` 报告。尽管《*可扩展性程序员参考手册*》的第 5 章涵盖了部署 `BI Publisher` 报告，但没有关于如何将它们与插件归档文件打包在一起的说明。

在 EM12c 中创建 `Information Publisher` 报告相当简单。你可以研究包含在 `oracle.samples.xsh1` 插件中、随 `sample_host1` 目标打包的报告示例。请按照以下步骤使用 `Information Publisher` API 创建新的报告定义并将其与你的插件打包：

1.  当插件的第一个版本（尚未包含报告）和目标类型开发并部署后，使用 EM12c 控制台的报告向导创建报告定义。这将需要创建 SQL 和/或 PL/SQL 来查询存储库视图并以报告组件（如图表、文本标签、链接和表格）所需的格式提取信息（例如目标的指标、配置等）。在这个阶段，你将尽可能接近期望的报告，并通过迭代改进你的报告。有关可用存储库视图的文档，请参阅《*可扩展性程序员参考手册*》第 18 章。
2.  使用 `emcli export_report` 命令提取所设计报告的 XML 定义。
3.  你可以修改并进一步自定义提取出的 XML 报告定义。XML 格式为报告开发者提供了 EM12c 控制台报告向导中不直接提供的额外设计功能。《*可扩展性程序员参考手册*》第 4 章的大部分内容都致力于记录 `Information Publisher` 报告的 XML 元素，你应该将其用作参考。
4.  最后，将创建的报告定义文件放入 `<STAGE>/oms/metadata/reports` 中，并打包新版本的插件。只要报告定义放在正确的位置，EDK 在创建 OPAR 文件时会自动提取报告。

当包含报告的插件部署到 OMS 上时，其报告在 EM12c 控制台的 `Information Publisher` Reports 部分作为 `SYSTEM` 报告可用。这些报告不能被 EM12c 用户删除或修改。但是，可以克隆这些报告以创建一个副本，以便在需要时进行自定义。10g/11g Grid Control 允许将某些 `SYSTEM` 报告标记为直接在目标主页的 Reports 选项卡中可用。此选项卡已从 EM12c 中移除，因为它提供的目标主页面 UI 自定义功能远远超过了简单嵌入报告的旧功能。

`oracle.samples.xsh1` 插件中性能报告类别下的两个报告是很好的研究示例。该插件中的配置报告使用了虚拟数据（从 `DUAL` 表中选择常量），因此不是一个有用的研究示例。

使用元数据注册服务 (MRS) 直接更新存储库中的报告定义而无需重新打包和重新部署插件是很有用的。有关 MRS 的更多详细信息，请参阅本章末尾。

## 企业配置管理

EM12c 可扩展性框架完全支持企业配置管理 (ECM)。虽然 10g/11g Grid Control 支持 ECM 功能，但它未为第三方插件开发者记录并提供官方支持（尽管 Oracle 自己的插件使用了 ECM）。EM12c 为 ECM 添加了基于 XML 的元数据定义，取代了笨拙的基于 PL/SQL 的定义，这为插件开发者提供了简单的支持。

添加配置管理功能有三个步骤：




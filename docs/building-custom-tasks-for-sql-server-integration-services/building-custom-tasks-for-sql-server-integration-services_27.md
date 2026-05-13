# 24. 部署到 Azure-SSIS

在上一章中，我们使用“执行目录包任务”构建了一个 SSIS 框架来执行元数据驱动的 SSIS 包集合。在本章中，我们将构建一个 Azure-SSIS 集成运行时实例，该实例安装“执行目录包任务”，然后将上一章的元数据驱动框架迁移到 Azure。为了实现这个目标，我们将：

*   配置一个 Azure SQL Database 实例来托管 `SSISDB` 数据库和 SSIS 目录
*   配置一个 Azure Data Factory
*   创建并配置部署容器
*   配置一个 Azure-SSIS 集成运行时 (IR)，该运行时配置为安装“执行目录包任务”的 `msi` 文件

在继续之前，让我们先了解一下 Azure Data Factory 的“执行 SSIS 包”活动所呈现的包位置选项。

## 检查包位置选项

可以使用“执行 SSIS 包”活动在 Azure Data Factory 中执行 SSIS 包。在撰写本文时，“包位置”属性列出了五种 SSIS 包源选项，如图 24-1 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig1_HTML.jpg](img/449652_2_En_24_Fig1_HTML.jpg)

图 24-1

“执行 SSIS 包”活动的包位置选项

`SSISDB` 是 SSIS 目录，我们将在本章中使用它。文件系统（包）指存储在 Azure 文件共享中的单个 SSIS 包 `dtsx` 文件。文件系统（项目）指 `ispac` 文件，这些是 SSIS 项目部署文件，包含 SSIS 项目中包含的 SSIS 包的单个 `dtsx` 文件，存储在 Azure 文件共享中。嵌入式包允许开发者上传单个 SSIS 包 `dtsx` 文件。包存储类似于旧的 SSIS 包存储。有关更多信息，请参阅 Microsoft 的 SSIS 包存储文档 ([docs.microsoft.com/en-us/azure/data-factory/azure-ssis-integration-runtime-package-store](https://docs.microsoft.com/en-us/azure/data-factory/azure-ssis-integration-runtime-package-store) [截至 2020 年 10 月 28 日])。

总之，Azure-SSIS IR 支持：

*   运行部署到由 Azure SQL Database 服务器/托管实例托管的 SSIS 目录 (`SSISDB`) 中的包（项目部署模型）
*   运行部署到文件系统、Azure 文件或由 Azure SQL 托管实例托管的 SQL Server 数据库 (`MSDB`) 中的包（包部署模型）

包部署模型支持配置带有*包存储*的 Azure-SSIS IR。包存储在文件系统、Azure 文件或 Azure SQL 托管实例中托管的 `MSDB` 之上提供了一个包管理层。Azure-SSIS IR 包存储允许开发者通过 `SQL Server Management Studio (SSMS)` 导入、导出、删除和运行包，以及监控和停止正在运行的包——类似于旧的 SSIS 包存储。

第一步是配置一个 Azure SQL DB。

### 预配 Azure SQL 数据库

Azure SQL 数据库（或称 Azure SQL DB）是 Azure 提供的一项灵活且可扩展的平台即服务（PaaS）云数据库产品。Azure SQL DB 的大小（和成本）范围从小型到超大型不等。

连接到 Azure 并点击 `新建资源` 以打开 `新建` 窗格，如图 24-2 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig2_HTML.jpg](img/449652_2_En_24_Fig2_HTML.jpg)
*图 24-2：准备预配新的 Azure SQL 数据库*

`新建` 窗格上的 `SQL 数据库` 链接会打开 `创建 SQL 数据库` 窗格。在 Azure 中预配新的“工作区”时，首先要创建一个新的 `资源组`，如图 24-3 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig3_HTML.jpg](img/449652_2_En_24_Fig3_HTML.jpg)
*图 24-3：在预配 Azure SQL 数据库时创建 rgECPT 资源组*

下一步是输入数据库名称。在上一章中，我们重用了 `SSISDB` 数据库。由于在此框架中我们不强制实施引用完整性（外键）或使用 `SSISDB` 的加密功能，因此我们可以预配一个不同的数据库来托管框架元数据，如图 24-4 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig4_HTML.jpg](img/449652_2_En_24_Fig4_HTML.jpg)
*图 24-4：预配 SSISFramework 数据库*

下一步是选择或创建一个新的 SQL Server。图 24-5 显示了在 `服务器` 下拉菜单下方点击“新建”链接后显示的 `新建服务器` 窗格：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig5_HTML.jpg](img/449652_2_En_24_Fig5_HTML.jpg)
*图 24-5：预配新的 SQL 服务器*

将“是否要使用 SQL 弹性池”属性设置为“否”。具有 `Gen5`、`2 个 vCore` 和 `32 GB` 存储的 `常规用途` 服务器配置略超出我们的需求，因此点击 `计算 + 存储` 属性的 `配置数据库` 链接，如图 24-6 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig6_HTML.jpg](img/449652_2_En_24_Fig6_HTML.jpg)
*图 24-6：准备配置 SQL 服务器*

通过选择 `基本配置` 来配置 SQL Server，如图 24-7 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig7_HTML.jpg](img/449652_2_En_24_Fig7_HTML.jpg)
*图 24-7：选择基本配置*

点击 `应用` 按钮返回到 `创建 SQL 数据库` 窗格，该窗格显示如图 24-8 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig8_HTML.jpg](img/449652_2_En_24_Fig8_HTML.jpg)
*图 24-8：准备创建数据库*

下一步是点击窗格底部的 `查看 + 创建` 按钮，这将显示如图 24-9 所示的 `查看` 窗格：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig9_HTML.jpg](img/449652_2_En_24_Fig9_HTML.jpg)
*图 24-9：查看 SQL 数据库和服务器配置*

点击窗格底部的 `创建` 按钮，完成 Azure SQL 数据库的预配。

下一步是预配一个 Azure 数据工厂。

### 预配 Azure 数据工厂

Azure 数据工厂（ADF）是 Microsoft Azure 上的数据集成平台即服务（PaaS）产品。在撰写本文时，ADFv2 是当前版本。

如果你没有 Azure 帐户，请访问 `azure.com` 并创建一个帐户。在撰写本文时，你可以免费注册并获得 12 个月的免费服务。Azure 注册优惠随时间变化，但作者总是看到 Azure 在注册时提供免费内容。

要预配 Azure 数据工厂，请连接到 Azure 门户（`portal.azure.com`）并点击 `创建资源` 选项，如图 24-10 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig10_HTML.jpg](img/449652_2_En_24_Fig10_HTML.jpg)
*图 24-10：创建新资源*

当 `新建` 窗格显示时，在 `搜索市场` 文本框中输入“数据工厂”，然后点击建议中列出的“数据工厂”，如图 24-11 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig11_HTML.jpg](img/449652_2_En_24_Fig11_HTML.jpg)
*图 24-11：选择数据工厂*

点击“数据工厂”会打开 `数据工厂概述` 窗格，如图 24-12 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig12_HTML.jpg](img/449652_2_En_24_Fig12_HTML.jpg)
*图 24-12：准备创建数据工厂*

点击 `创建` 按钮以打开 `创建数据工厂` 窗格。配置 `资源组`、`区域`、`名称` 和 `版本` 属性，然后点击 `Git 配置` 选项卡，如图 24-13 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig13_HTML.jpg](img/449652_2_En_24_Fig13_HTML.jpg)
*图 24-13：配置数据工厂*

选中 `稍后配置 Git` 复选框，然后点击 `查看 + 创建` 按钮，如图 24-14 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig14_HTML.jpg](img/449652_2_En_24_Fig14_HTML.jpg)
*图 24-14：配置 Git 配置选项卡*

Azure 会验证 Azure 数据工厂的配置，并通知你验证是通过还是发现了问题。验证通过后，查看设置，如果接受，则点击 `创建` 按钮，如图 24-15 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig15_HTML.jpg](img/449652_2_En_24_Fig15_HTML.jpg)
*图 24-15：查看设置*

如果一切按计划进行，部署将成功，并且 `概述` 窗格会显示，如图 24-16 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig16_HTML.jpg](img/449652_2_En_24_Fig16_HTML.jpg)
*图 24-16：部署成功*

点击 `转到资源` 按钮以打开 `数据工厂概述`，如图 24-17 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig17_HTML.jpg](img/449652_2_En_24_Fig17_HTML.jpg)
*图 24-17：数据工厂概述*

要配置 Azure 数据工厂的构件，请点击 `创作和监视` 按钮。`数据工厂创作和监视概述` 页面将显示，如图 24-18 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig18_HTML.jpg](img/449652_2_En_24_Fig18_HTML.jpg)
*图 24-18：数据工厂创作和监视概述页面*

Azure 数据工厂已预配完成。下一步是创建和配置部署容器。

## 创建和配置部署容器

Azure Blob Storage 充当 Azure 文件系统。在本节中，我们将创建一个 Azure 存储帐户。Azure 存储帐户允许开发人员创建容器，这些容器在 Azure 中存储文档以供分布式访问。要了解更多信息，请访问 `docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction`。

我们按照以下步骤创建和配置部署容器：

1.  预配一个 Azure 存储帐户。
2.  创建部署容器。
3.  上传 `ExecuteCatalogPackageTask.msi` 文件。
4.  创建并上传 `main.cmd` 文件。

### 配置 Azure 存储账户与部署容器

创建和配置部署容器的步骤始于配置一个 Azure 存储账户。要配置存储账户，请连接到 Azure 门户，然后从菜单中点击“`创建资源`”。如果“`存储账户`”未显示在“`新建`”边栏选项卡的默认屏幕上，请使用“`搜索市场`”功能搜索“`存储账户`”，如图 24-19 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig19_HTML.jpg](img/449652_2_En_24_Fig19_HTML.jpg)

图 24-19：准备添加存储账户

通过选择资源组、输入存储账户名称，并选择位置、性能、账户类型和复制属性来配置新的存储账户，如图 24-20 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig20_HTML.jpg](img/449652_2_En_24_Fig20_HTML.jpg)

图 24-20：配置存储账户属性

点击“`查看 + 创建`”按钮进入审阅边栏选项卡，如图 24-21 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig21_HTML.jpg](img/449652_2_En_24_Fig21_HTML.jpg)

图 24-21：审阅存储账户配置

点击“`创建`”按钮以创建存储账户。

下一步是创建部署容器。

## 创建部署容器

部署容器最初将托管两个文件：`main.cmd` 和 `ExecuteCatalogPackageTask.msi`。`main.cmd` 包含在 Azure-SSIS 节点上安装 Execute Catalog Package Task 的说明。稍后，该容器还将托管包含每次安装 `ExecuteCatalogPackageTask.msi` 安装程序文件详细信息的日志文件。

在开始之前，请浏览到 storageexplorer.com 并下载 Microsoft Azure Storage Explorer，如图 24-22 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig22_HTML.jpg](img/449652_2_En_24_Fig22_HTML.jpg)

图 24-22：下载 Storage Explorer

下载后，安装 Microsoft Azure Storage Explorer。安装完成后，打开 Storage Explorer 并配置您的 Azure 账户。配置好 Azure 账户后，Storage Explorer 应显示您之前配置的存储账户，如图 24-23 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig23_HTML.jpg](img/449652_2_En_24_Fig23_HTML.jpg)

图 24-23：Storage Explorer 中的存储账户

下一步是创建容器。右键单击存储账户，然后点击“`创建 Blob 容器`”，如图 24-24 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig24_HTML.jpg](img/449652_2_En_24_Fig24_HTML.jpg)

图 24-24：创建新容器

当新的未命名容器被添加到存储账户的“`Blob 容器`”节点下时，输入新容器的名称，如图 24-25 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig25_HTML.jpg](img/449652_2_En_24_Fig25_HTML.jpg)

图 24-25：添加新容器

下一步是上传 `ExecuteCatalogPackageTask.msi` 文件。

## 上传 ExecuteCatalogPackageTask.msi 文件

输入新容器名称后按 `Enter` 键。Storage Explorer 会选中新容器并显示“`此 blob 容器中没有可用数据`”。点击“`上传`”按钮，然后从下拉菜单中选择“`上传文件…`”，如图 24-26 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig26_HTML.jpg](img/449652_2_En_24_Fig26_HTML.jpg)

图 24-26：准备上传文件

当“`上传文件`”对话框显示时，点击“`选定的文件`”属性文本框右侧的省略号（...），如图 24-27 中高亮所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig27_HTML.jpg](img/449652_2_En_24_Fig27_HTML.jpg)

图 24-27：准备选择 msi 文件

当“`选择要上传的文件`”对话框显示时，导航到并选择 `ExecuteCatalogPackageTask.msi` 文件，如图 24-28 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig28_HTML.jpg](img/449652_2_En_24_Fig28_HTML.jpg)

图 24-28：选择 msi 文件

点击“`打开`”按钮返回“`上传文件`”对话框，如图 24-29 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig29_HTML.jpg](img/449652_2_En_24_Fig29_HTML.jpg)

图 24-29：准备上传 msi 文件

点击“`上传`”按钮，将 `ExecuteCatalogPackageTask.msi` 文件上传到 Blob 容器，如图 24-30 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig30_HTML.jpg](img/449652_2_En_24_Fig30_HTML.jpg)

图 24-30：msi 文件已上传

下一步是创建 `main.cmd` 文件。

## 创建 Main.cmd 文件

`main.cmd` 文件在每个 Azure-SSIS 节点启动时，在该节点上执行 Windows 安装程序应用程序 `msiexec.exe`。首先，打开您喜欢的文本编辑器并输入清单 24-1 中所示的文本：

```
@echo off
rem execute the msi
msiexec /i %~dp0\ExecuteCatalogPackageTask.msi /passive /l %CUSTOM_SETUP_SCRIPT_LOG_DIR%\ExecuteCatalogPackageTaskInstall.log
echo
echo Execute Catalog Package Task installed.
Listing 24-1
Main.cmd
```

将该文件保存为 `main.cmd`，然后将 `main.cmd` 上传到 Azure Blob 容器，如图 24-31 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig31_HTML.jpg](img/449652_2_En_24_Fig31_HTML.jpg)

图 24-31：main.cmd 文件已上传

您可以在 docs.microsoft.com/en-us/azure/data-factory/how-to-configure-azure-ssis-ir-custom-setup 了解更多关于自定义 Azure-SSIS 集成运行时的信息。

您可以在 docs.microsoft.com/en-us/windows-server/administration/windows-commands/msiexec 了解更多关于 `msiexec` 应用程序的信息。

下一步是预配 Azure-SSIS 集成运行时。


### 配置 Azure-SSIS 集成运行时

要打开 Azure 数据工厂管理页面，请点击 Azure 数据工厂页面中的 `Manage` 链接，如图 24-32 所示：

![图 24-32：Azure 数据工厂管理页面](img/449652_2_En_24_Fig32_HTML.jpg)

要查看 Azure 数据工厂集成运行时，请点击左侧列表中的 `Connections` ➤ `Integration runtimes` 链接，如图 24-33 所示：

![图 24-33：查看 Azure 数据工厂集成运行时](img/449652_2_En_24_Fig33_HTML.jpg)

每当配置 Azure 数据工厂时，一个名为 `AutoResolveIntegrationRuntime` 的 Azure 集成运行时也会被自动配置。集成运行时（IRs）用于调度 ADF 管道活动的执行，通过 ADF 链接服务将内部活动功能连接到外部资源。要了解有关 `AutoResolveIntegrationRuntime` 的更多信息，请访问 `docs.microsoft.com/en-us/azure/data-factory/concepts-integration-runtime`。

点击 `+ New` 链接以创建新的集成运行时。当 `Integration runtime setup` 刀片窗格显示时，选择 `Azure-SSIS` 集成运行时类型，如图 24-34 所示：

![图 24-34：选择 Azure-SSIS 集成运行时](img/449652_2_En_24_Fig34_HTML.jpg)

点击 `Continue` 按钮继续 Azure-SSIS IR 的配置。为 Azure-SSIS 集成运行时输入名称和可选的描述，然后配置 `Location`、`Node Size`、`Node Number` 和 `Edition` 属性，如图 24-35 所示：

![图 24-35：配置 Azure-SSIS 集成运行时](img/449652_2_En_24_Fig35_HTML.jpg)

请注意，如果您的企业已经拥有 SQL Server 许可证，则可以节省成本，如图 24-36 所示：

![图 24-36：如果您的企业拥有 SQL Server 许可证则可节省成本](img/449652_2_En_24_Fig36_HTML.jpg)

点击 `Continue` 按钮，将 Azure-SSIS IR 类型配置为创建 SSIS 目录，如图 24-37 所示：

![图 24-37：配置 Azure-SSIS IR 以创建 SSIS 目录](img/449652_2_En_24_Fig37_HTML.jpg)

配置 Azure-SSIS IR 以连接到本章前面创建的 Azure SQL 数据库，方法是从各自的下拉菜单中选择 `Subscription`、`location` 和 `Catalog database server endpoint` 属性值，然后输入连接信息，如图 24-38 所示：

![图 24-38：为 SSIS 目录配置 Azure-SSIS 属性](img/449652_2_En_24_Fig38_HTML.jpg)

点击 `Continue` 按钮，进入集成运行时设置刀片窗格的“高级设置”部分。

将 SSIS 目录及其数据库 `SSISDB` 添加到之前为 `SSISFramework` 数据库配置的同一个 SQL Server 实例中并非必须。`SSISFramework` 数据库可以位于您的企业 SSIS 框架解决方案可访问的任何位置。

在继续 Azure-SSIS IR 配置之前，返回 Microsoft Azure 存储资源管理器并导航到之前创建的容器。右键单击该容器，然后单击 `Get Shared Access Signature`，如图 24-39 所示：

![图 24-39：准备获取共享访问签名 (SAS)](img/449652_2_En_24_Fig39_HTML.jpg)

将 `Expiry time` 设置为未来的某个日期时间，因为每次启动 Azure-SSIS 节点时都会访问此文件夹。除了默认的 (`Read` 和 `List`) 权限外，请勾选 `Add`、`Create` 和 `Write` 权限复选框，如图 24-40 所示：

![图 24-40：配置 SAS](img/449652_2_En_24_Fig40_HTML.jpg)

点击 `Create` 按钮以创建共享访问签名统一资源标识符 (URI) 和查询字符串，如图 24-41 所示：

![图 24-41：共享访问签名统一资源标识符 (URI) 和查询字符串](img/449652_2_En_24_Fig41_HTML.jpg)

点击 `SAS URI` 文本框旁边的 `Copy` 按钮。

返回到 Azure-SSIS 集成运行时设置刀片窗格，勾选 `Customize your Azure-SSIS Integration Runtime with additional system configurations/component installations` 复选框，然后将 SAS URI 粘贴到 `Custom setup container SAS URI` 文本框中，如图 24-42 所示：

![图 24-42：自定义 Azure-SSIS IR 以安装自定义任务](img/449652_2_En_24_Fig42_HTML.jpg)

粘贴后，SAS URI 将类似于图 24-43 所示（您的值很可能不同）：

![图 24-43：粘贴后的 SAS URI](img/449652_2_En_24_Fig43_HTML.jpg)

点击 `Continue` 按钮以查看 Azure-SSIS IR 配置的摘要，如图 24-44 所示：

![图 24-44：Azure-SSIS 配置摘要](img/449652_2_En_24_Fig44_HTML.jpg)

点击 `Create` 按钮开始创建 Azure-SSIS 集成运行时，如图 24-45 所示：

![图 24-45：启动 Azure-SSIS 集成运行时](img/449652_2_En_24_Fig45_HTML.jpg)

一旦 Azure-SSIS 集成运行时启动，`Manage` ➤ `Integration runtimes` 页面将如图 24-46 所示：

![图 24-46：已启动的 Azure-SSIS 集成运行时](img/449652_2_En_24_Fig46_HTML.jpg)

Azure-SSIS 集成运行时显示状态为 `Running`。这是一个非常好的迹象。




# 25. 在 Azure Data Factory 中测试任务

## 查看安装日志

继续操作前，返回 Storage Explorer 并刷新容器。请注意，容器中会显示一个名为 `main.cmd.log` 的新文件夹，如图 24-47 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig47_HTML.jpg](img/449652_2_En_24_Fig47_HTML.jpg)

图 24-47 `main.cmd.log` 文件夹

双击 `main.cmd.log` 文件夹以查看安装文件夹，如图 24-48 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig48_HTML.jpg](img/449652_2_En_24_Fig48_HTML.jpg)

图 24-48 查看安装文件夹

双击安装文件夹以查看其中包含的四个文件：

* `ExecuteCatalogPackageTaskInstall.log`
* `InstallationSummary.log`
* `stderr.log`
* `stdout.log`

所有文件如图 24-49 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig49_HTML.jpg](img/449652_2_En_24_Fig49_HTML.jpg)

图 24-49 在安装文件夹中查看安装日志文件

双击 `ExecuteCatalogPackageTaskInstall.log` 文件以下载并查看该文件，如图 24-50 所示：

![../images/449652_2_En_24_Chapter/449652_2_En_24_Fig50_HTML.jpg](img/449652_2_En_24_Fig50_HTML.jpg)

图 24-50 查看 `ExecuteCatalogPackageTaskInstall.log` 文件

高亮部分显示在单行上。作者为了美观添加了两个换行符。最重要的文本是 `Installation success or error status: 0.`。

`Installation success or error status: 0.` 表示 `ExecuteCatalogPackageTask.msi` 的安装过程没有错误。

### 结论

在本章中，我们构建了一个包含 Execute Catalog Package Task 安装说明的 Azure-SSIS 集成运行时实例，具体实现了以下步骤：

* 配置一个 Azure SQL Database 实例来托管 SSISDB 数据库和 SSIS 目录
* 配置一个 Azure Data Factory
* 创建并配置一个部署容器
* 配置一个 Azure-SSIS 集成运行时 (IR)，该运行时配置为安装 Execute Catalog Package Task 的 `msi` 文件

下一步是在 Azure-SSIS 节点中测试 Execute Catalog Package Task 的安装情况。

## 部署 TestSSISProject

在上一章中，我们配置了一个 Azure SQL Database、一个 Azure Data Factory、一个存储账户和部署容器，以及一个 Azure-SSIS 集成运行时。在本章中，我们通过以下方式在 Azure Data Factory 中测试 Execute Catalog Package Task 功能：

* 将 `TestSSISProject` 部署到上一章配置的 Azure SQL DB 托管的 SSIS 目录
* 部署 `ECPTFramework` SSIS 项目
* 将框架元数据迁移（Lift and Shift）到 Azure SQL DB（非 SSISDB）
* 再次部署 `ECPTFramework` SSIS 项目
* 进行测试

在 Visual Studio 中打开一个测试用的 SSIS 项目（我的项目名为 `TestSSISProject`）。打开解决方案资源管理器，右键单击项目名称，然后单击 `Deploy`，如图 25-1 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig1_HTML.jpg](img/449652_2_En_25_Fig1_HTML.jpg)

图 25-1 准备部署 `TestSSISProject`

当集成服务部署向导显示时，单击 `Next` 按钮继续，如图 25-2 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig3_HTML.jpg](img/449652_2_En_25_Fig3_HTML.jpg)

图 25-3 在集成服务部署向导中选择目标

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig2_HTML.jpg](img/449652_2_En_25_Fig2_HTML.jpg)

图 25-2 集成服务部署向导

在这个新的 SSIS 目录中，名为 `Test` 的文件夹（尚）不存在，如图 25-4 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig4_HTML.jpg](img/449652_2_En_25_Fig4_HTML.jpg)

图 25-4 在集成服务部署向导中连接到目标 SSIS 目录

单击 `Path` 属性旁边的 `Browse` 按钮。当“Browse for Folder or Project”对话框显示时，单击 `New folder…` 按钮，如图 25-5 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig5_HTML.jpg](img/449652_2_En_25_Fig5_HTML.jpg)

图 25-5 准备在目标 SSIS 目录中创建新文件夹

当“Create New Folder”对话框显示时，输入 `Test` 作为新文件夹的名称，然后单击 `Ok` 按钮关闭“Create New Folder”对话框，如图 25-6 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig6_HTML.jpg](img/449652_2_En_25_Fig6_HTML.jpg)

图 25-6 在目标 SSIS 目录中创建新文件夹

单击 `OK` 按钮关闭“Create New Folder”对话框，如图 25-7 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig7_HTML.jpg](img/449652_2_En_25_Fig7_HTML.jpg)

图 25-7 在目标 SSIS 目录中选择新文件夹

单击 `OK` 按钮关闭“Browse for Folder or Project”对话框，如图 25-8 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig8_HTML.jpg](img/449652_2_En_25_Fig8_HTML.jpg)

图 25-8 已配置的集成服务部署向导目标

单击 `Next` 按钮打开“Validate”页面，该页面会测试连接管理器连接字符串的本地配置，如图 25-9 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig9_HTML.jpg](img/449652_2_En_25_Fig9_HTML.jpg)

图 25-9 验证项目和包连接管理器

在“Review”页面上查看设置，如图 25-10 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig10_HTML.jpg](img/449652_2_En_25_Fig10_HTML.jpg)

图 25-10 准备部署

单击 `Deploy` 按钮将项目部署到 SSIS 目录，如图 25-11 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig11_HTML.jpg](img/449652_2_En_25_Fig11_HTML.jpg)

图 25-11 成功部署

下一步是配置 SSIS 目录项目。


## 配置 TestSSISProject

使用 SSMS 连接到托管你的 Azure-SSIS 集成运行时的 SSIS 目录的 Azure SQL 数据库实例。要打开“配置”对话框，请右键单击 `TestSSISProject`，然后单击“配置”，如图 25-12 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig12_HTML.jpg](img/449652_2_En_25_Fig12_HTML.jpg)

图 25-12

打开 `TestSSISProject` 配置对话框

将打开“配置 – `TestSSISProject`”对话框。在“参数”选项卡上，单击 `RunForSomeTime.dtsx` 容器中 `DelayString` 参数的省略号，如图 25-13 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig13_HTML.jpg](img/449652_2_En_25_Fig13_HTML.jpg)

图 25-13

配置 – `TestSSISProject` 对话框

当 `"[RunForSomeTime.dtsx].[DelayString]"` 参数的“设置参数值”窗口打开时，选择“编辑值”选项，在相应的文本框中输入“00:00:35”（这将添加一个配置字面值），然后单击“确定”按钮，如图 25-14 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig14_HTML.jpg](img/449652_2_En_25_Fig14_HTML.jpg)

图 25-14

使用配置字面值覆盖 `DelayString` 参数值

配置字面值以**粗体**文本装饰显示，如图 25-15 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig15_HTML.jpg](img/449652_2_En_25_Fig15_HTML.jpg)

图 25-15

配置字面值以粗体显示

让我们测试一下！

## 测试配置

要测试配置更改，请使用 SQL Server Management Studio (SSMS) 连接到托管你的 Azure-SSIS 集成运行时所使用的 SSIS 目录的 Azure SQL 数据库实例。导航到 `TestSSISProject` 包虚拟文件夹，右键单击 `RunForSomeTime.dtsx` SSIS 包，然后单击“执行…”，如图 25-16 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig16_HTML.jpg](img/449652_2_En_25_Fig16_HTML.jpg)

图 25-16

准备执行 `RunForSomeTime.dtsx` SSIS 包

当“执行包”对话框显示时，单击“确定”按钮，如图 25-17 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig17_HTML.jpg](img/449652_2_En_25_Fig17_HTML.jpg)

图 25-17

执行 `RunForSomeTime.dtsx` SSIS 包

当提示显示执行报告时，单击“是”按钮，如图 25-18 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig18_HTML.jpg](img/449652_2_En_25_Fig18_HTML.jpg)

图 25-18

单击“是”按钮以打开执行报告

将显示 SSIS 目录“所有执行”报告，并且一段时间后，报告 `RunForSomeTime.dtsx` SSIS 包已完成并成功，如图 25-19 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig19_HTML.jpg](img/449652_2_En_25_Fig19_HTML.jpg)

图 25-19

执行成功！

## 提升和迁移 SSIS 框架

在第 23 章中，我们将 `fw` 架构添加到 `SSISDB` 数据库，然后添加了三个表：

1.  `fw.Applications`

2.  `fw.Packages`

3.  `fw.ApplicationPackages`

这三个表包含 SSIS 执行框架使用的元数据。总结一下框架的定义：*应用程序*是作为一组执行的 SSIS 包的集合。为了执行一组 SSIS 包，框架需要*包*位置元数据。因为一个包可能作为许多应用程序的一部分被执行，*应用程序包*定义了应用程序和包之间的关系。

### 提升与转移框架元数据表

当工程师将数据从本地迁移到云端时，这种操作常被称为*提升与转移*。在本节中，我们将使用第 23 章中相同的 T-SQL 来提升和转移框架的元数据对象与数据，但目标数据库不同。这一次，目标是我们在第 24 章中配置的名为 `SSISFramework` 的 Azure SQL 数据库。

如果以下内容看起来像是从第 23 章复制、粘贴到这里，然后经过了极其细微的编辑，那是因为它确实是从第 23 章复制、粘贴到这里，然后经过了极其细微的编辑。

首先，打开 SQL Server Management Studio (SSMS) 并连接到托管你的 Azure-SSIS 集成运行时所使用的 SSIS 目录数据库 (`SSISDB`) 的 Azure SQL Server 实例。接下来的几个 T-SQL 脚本与第 23 章中的 T-SQL 脚本几乎相同。区别在于：没有 `Use` 指令。在撰写本文时，Azure SQL DB 不支持 `Use` 指令。解决此限制的方法是：在 SSMS 中单击“工作数据库”下拉菜单并选择 `SSISFramework` 数据库，如图 25-20 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig20_HTML.jpg](img/449652_2_En_25_Fig20_HTML.jpg)

**图 25-20**
选择 SSISFramework 工作数据库

在 SSMS 中打开一个新的查询窗口。使用清单 25-1 中的 T-SQL 向 `SSISDB` 数据库添加一个名为 `fw` 的架构：

```sql
print 'Fw schema'
If Not Exists(Select s.[name]
From [sys].[schemas] s
Where s.[name] = N'fw')
begin
print ' - Create fw schema'
declare @sql varchar(30) = 'Create schema fw'
exec(@sql)
print ' - Fw schema created'
end
Else
begin
print ' - Fw schema already exists.'
end
go
```
**清单 25-1**
添加 `fw` 架构

（第一次）执行后，“消息”选项卡显示如图 25-21 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig21_HTML.jpg](img/449652_2_En_25_Fig21_HTML.jpg)

**图 25-21**
创建 `fw` 架构

接下来，使用清单 25-2 中的 T-SQL 在 `fw` 架构中构建 `Applications` 表：

```sql
print 'Fw.Applications table'
If Not Exists(Select s.[name] + '.' + t.[name]
From [sys].[tables] t
Join [sys].[schemas] s
On s.[schema_id] = t.[schema_id]
Where s.[name] = N'fw'
And t.[name] = N'Applications')
begin
print ' - Create fw.Applications table'
Create Table fw.Applications
(ApplicationId int identity(1, 1)
Constraint PK_Applications Primary Key
,ApplicationName nvarchar(130) Not NULL)
print ' - Fw.Applications table created'
end
Else
begin
print ' - Fw.Applications table already exists.'
end
go
```
**清单 25-2**
添加 `fw.Applications` 表

（第一次）执行后，“消息”选项卡显示如图 25-22 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig22_HTML.jpg](img/449652_2_En_25_Fig22_HTML.jpg)

**图 25-22**
创建 `fw.Applications` 表

使用清单 25-3 中的 T-SQL 添加 `Packages` 表：

```sql
print 'Fw.Packages table'
If Not Exists(Select s.[name] + '.' + t.[name]
From [sys].[tables] t
Join [sys].[schemas] s
On s.[schema_id] = t.[schema_id]
Where s.[name] = N'fw'
And t.[name] = N'Packages')
begin
print ' - Create fw.Packages table'
Create Table fw.Packages
(PackageId int identity(1, 1)
Constraint PK_Packages Primary Key
,FolderName nvarchar(130) Not NULL
,ProjectName nvarchar(130) Not NULL
,PackageName nvarchar(260) Not NULL)
print ' - Fw.Packages table created'
end
Else
begin
print ' - Fw.Packages table already exists.'
end
go
```
**清单 25-3**
添加 `Packages` 表

（第一次）执行后，“消息”选项卡显示如图 25-23 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig23_HTML.jpg](img/449652_2_En_25_Fig23_HTML.jpg)

**图 25-23**
创建 `fw.Packages` 表

通过使用清单 25-4 中的 T-SQL 添加 `ApplicationPackages` 表来解决 `Applications` 和 `Packages` 之间的多对多基数关系：

```sql
print 'Fw.ApplicationPackages table'
If Not Exists(Select s.[name] + '.' + t.[name]
From [sys].[tables] t
Join [sys].[schemas] s
On s.[schema_id] = t.[schema_id]
Where s.[name] = N'fw'
And t.[name] = N'ApplicationPackages')
begin
print ' - Create fw.ApplicationPackages table'
Create Table fw.ApplicationPackages
(ApplicationPackageId int identity(1, 1)
Constraint PK_ApplicationPackages Primary Key
,ApplicationId int Not NULL
Constraint FK_ApplicationPackages_Applications
Foreign Key References fw.Applications(ApplicationId)
,PackageId int Not NULL
Constraint FK_ApplicationPackages_Packages
Foreign Key References fw.Packages(PackageId)
,ExecutionOrder int Not NULL
Constraint DF_ApplicationPackages_ExecutionOrder
Default(10)
,Synchronized bit Not NULL
Constraint DF_ApplicationPackages_Synchronized
Default(1)
,MaximumRetries int Not NULL
Constraint DF_ApplicationPackages_MaximumRetries
Default(100)
,RetryIntervalSeconds int Not NULL
Constraint DF_ApplicationPackages_RetryIntervalSeconds
Default(10)
,Use32bit int Not NULL
Constraint DF_ApplicationPackages_Use32bit
Default(0)
,LoggingLevel nvarchar(25) Not NULL
Constraint DF_ApplicationPackages_LoggingLevel
Default('Basic')
)
print ' - Fw.ApplicationPackages table created'
end
Else
begin
print ' - Fw.ApplicationPackages table already exists.'
end
go
```
**清单 25-4**
添加 `ApplicationPackages` 表

（第一次）执行后，“消息”选项卡显示如图 25-24 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig24_HTML.jpg](img/449652_2_En_25_Fig24_HTML.jpg)

**图 25-24**
创建 `fw.ApplicationPackages` 表

下一步是向框架表中添加 `Applications`、`Packages` 和 `ApplicationPackages` 的元数据。


### 升移框架元数据

使用清单 25-5 中的 T-SQL 添加应用程序元数据：

```
Insert Into fw.Applications
(ApplicationName)
Values('Framework Test')
go
Select * From fw.Applications
go
Listing 25-5
添加应用程序元数据
```

执行后，T-SQL 结果如图 25-25 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig25_HTML.jpg](img/449652_2_En_25_Fig25_HTML.jpg)

图 25-25 添加 SSIS 框架应用程序

使用清单 25-6 中的 T-SQL 添加包元数据：

```
Insert Into fw.Packages
(FolderName, ProjectName, PackageName)
Values
('Test', 'TestSSISProject', 'ReportAndSucceed.dtsx')
, ('Test', 'TestSSISProject', 'ReportParameterAndSucceed.dtsx')
, ('Test', 'TestSSISProject', 'RunForSomeTime.dtsx')
go
Select * From fw.Packages
go
Listing 25-6
添加包元数据
```

执行后，T-SQL 结果如图 25-26 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig26_HTML.jpg](img/449652_2_En_25_Fig26_HTML.jpg)

图 25-26 添加 SSIS 框架包

使用清单 25-7 中的 T-SQL 添加应用程序包元数据：

```
Insert Into fw.ApplicationPackages
(ApplicationId, PackageId, ExecutionOrder
, Synchronized, MaximumRetries
, RetryIntervalSeconds, Use32bit, LoggingLevel)
Values
(1, 1, 10, 1, 10, 2, 0, 'Basic')
, (1, 2, 20, 0, 10, 2, 1, 'Performance')
, (1, 3, 30, 1, 30, 2, 1, 'Verbose')
go
Select * From fw.ApplicationPackages
go
Listing 25-7
添加应用程序包元数据
```

请确保 `ApplicationId` 和 `PackageId` 的值与 `fw.Applications` 和 `fw.Packages` 表中由标识值自动生成的对应值匹配。

执行后，T-SQL 结果如图 25-27 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig27_HTML.jpg](img/449652_2_En_25_Fig27_HTML.jpg)

图 25-27 添加 SSIS 框架应用程序包

下一步是将我们在前一章开发的 `ECPTFramework` SSIS 项目部署到 Azure-SSIS。

### 部署 ECPTFramework SSIS 项目

在 Visual Studio 中打开 `ECPTFramework` 测试 SSIS 项目——即第 23 章开发的项目。打开解决方案资源管理器，右键单击项目名称，然后单击“部署”，如图 25-28 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig28_HTML.jpg](img/449652_2_En_25_Fig28_HTML.jpg)

图 25-28 部署 ECPTFramework

当集成服务部署向导显示时，按照本章前面“部署 TestSSISProject”部分概述的步骤，将 `ECPTFramework` SSIS 项目部署到托管你在预配 Azure-SSIS 集成运行时所选 `SSISDB` 数据库的 Azure SQL 数据库。在本演示中，我创建了一个名为“SSIS”的新 SSIS 目录文件夹，并将 `ECPTFramework` SSIS 项目部署到该 SSIS 文件夹。

### 配置 ECPTFramework SSIS 项目

一旦 `ECPTFramework` SSIS 项目部署完成，打开 SQL Server Management Studio，并导航到集成服务目录节点中的 `ECPTFramework` SSIS 项目。右键单击 `ECPTFramework` SSIS 项目，然后单击“配置”，如图 25-29 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig29_HTML.jpg](img/449652_2_En_25_Fig29_HTML.jpg)

图 25-29 准备配置 ECPTFramework 项目

要开始配置 `ECPTFramework`，请选择“Framework SSIS Catalog Connection”，然后选择 `ConnectionString` 属性。单击“值”列旁边的省略号，如图 25-30 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig30_HTML.jpg](img/449652_2_En_25_Fig30_HTML.jpg)

图 25-30 准备为 Framework SSIS Catalog ConnectionString 属性配置字面量

Framework SSIS Catalog 连接管理器应指向保存框架元数据的 SQL Server 实例——以及数据库。请记住，我们对此版本的框架做了一个小改动；我们将元数据存储在另一个名为 `SSISFramework` 的数据库中。连接字符串已更新，如图 25-31 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig31_HTML.jpg](img/449652_2_En_25_Fig31_HTML.jpg)

图 25-31 为 Framework SSIS Catalog ConnectionString 属性配置字面量

配置完成后，“Execution SSIS Catalog”的 `ConnectionString` 属性的字面量如图 25-32 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig32_HTML.jpg](img/449652_2_En_25_Fig32_HTML.jpg)

图 25-32 已配置的 Framework SSIS Catalog ConnectionString 属性

对 Framework SSIS Catalog Password 属性重复此过程。

配置完成后，Framework SSIS Catalog 的 `ConnectionString` 和 `Password` 属性的字面量值如图 25-33 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig33_HTML.jpg](img/449652_2_En_25_Fig33_HTML.jpg)

图 25-33 已配置的 Framework SSIS Catalog ConnectionString 和 Password 属性

Execute SSIS Catalog 和 Framework SSIS Catalog 连接管理器已配置完成。

下一步是测试执行 `Parent.dtsx` SSIS 包。

### 让我们测试一下！

在 SQL Server Management Studio 的对象资源管理器中，展开“集成服务目录”节点，导航到 SSIS 文件夹 > `ECPTFramework` 项目 > `Parent.dtsx` 包。右键单击 `Parent.dtsx` SSIS 包，然后单击“执行…”，如图 25-34 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig34_HTML.jpg](img/449652_2_En_25_Fig34_HTML.jpg)

图 25-34 执行 Parent.dtsx SSIS 包

当“执行包”对话框显示时，单击“确定”按钮开始 `Parent.dtsx` 包执行，如图 25-35 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig35_HTML.jpg](img/449652_2_En_25_Fig35_HTML.jpg)

图 25-35 开始执行 Parent.dtsx

打开 SSIS 目录“所有执行”报告以查看结果执行情况，如图 25-36 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig36_HTML.jpg](img/449652_2_En_25_Fig36_HTML.jpg)

图 25-36 Parent.dtsx 失败

`Parent.dtsx` SSIS 包失败。`Parent.dtsx` 的执行导致了 `TestSSISProject\ReportAndSucceed.dtsx` SSIS 包的执行。我们需要更多信息来排查此错误。单击 `Parent.dtsx` 报告行中的“All Messages”链接以打开“消息”报告，如图 25-37 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig37_HTML.jpg](img/449652_2_En_25_Fig37_HTML.jpg)

图 25-37 查看错误消息

错误消息显示：“执行应用程序包：错误：任务上的 Execute 方法返回错误代码 `0x80131904`（找不到存储过程 ‘`xp_msver`’）。Execute 方法必须成功，并使用“out”参数指示结果。” [原文如此]



### 雷德蒙德，我们遇到了一个问题

存储过程 `xp_msver` 在本地 SQL Server 实例中可用。存储过程 `xp_msver` 在 Azure SQL 托管实例中可用。在撰写本文时，存储过程 `xp_msver` 在 Azure SQL DB 实例中 ``不可`` 用。

作者设计了一个解决方案，可能对您有用，也可能没用。然而，在撰写本文时，向 `SSISDB` 数据库添加一个名为 `xp_msver` 的存储过程可以解决此情况。连接到 `SSISDB` 数据库并执行清单 25-8 中的代码以创建 `xp_msver` 存储过程：

```sql
Create or Alter Procedure xp_msver
@arg sql_variant = N'Platform'
,@index int = 4 output
,@name nvarchar(25) = N'Platform' output
,@internalvalue int = NULL output
,@charvalue nvarchar(25) = N'NT x64' output
As
Select @index = 4
, @name = N'Platform'
, @internalvalue = NULL
, @charvalue = N'NT x64'
go
清单 25-8
将 xp_msver 存储过程添加到 SSISDB 数据库
```

部署 `xp_msver` 存储过程后，像之前一样重新执行 `Parent.dtsx` SSIS 包，如图 25-34 和 25-35 所示。刷新 SSIS 目录的“所有执行”报告以查看此执行的结果，如图 25-38 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig38_HTML.jpg](img/449652_2_En_25_Fig38_HTML.jpg)
*图 25-38 框架应用程序执行成功*

下一步是分析我们的测试结果。

### 分析结果

比较元数据配置将让我们知道测试是否真的成功。元数据配置可以通过执行 `Parent.dtsx` 包中“SQL Get Application Packages”的查询来获取，并在清单 25-9 中展示——稍作修改以提供 `@ApplicationName` 参数值：

```sql
Select p.FolderName
, p.ProjectName
, p.PackageName
, Convert(bit, ap.Synchronized) As Synchronized
, ap.MaximumRetries
, ap.RetryIntervalSeconds
, Convert(bit, ap.Use32bit) As Use32bit
, ap.LoggingLevel
From fw.ApplicationPackages ap
Join fw.Applications a
On a.ApplicationId = ap.ApplicationId
Join fw.Packages p
On p.PackageId = ap.PackageId
Where a.ApplicationName = N'Framework Test' -- @ApplicationName
Order By ap.ExecutionOrder
清单 25-9
SSISFramework 元数据
```

粘贴到 SSMS 中时，查询如图 25-39 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig39_HTML.jpg](img/449652_2_En_25_Fig39_HTML.jpg)
*图 25-39 来自 SSISFramework 数据库的框架元数据*

执行时，查询结果如图 25-40 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig40_HTML.jpg](img/449652_2_En_25_Fig40_HTML.jpg)
*图 25-40 元数据查询结果*

比较图 25-38 和 25-40 中标记为“1”的项目，我们注意到第二个 SSIS 包（`ReportParameterAndSucceed.dtsx`）的执行，`Synchronized` 被配置为 `0` —— 即 false。由于第一个 SSIS 包 `ReportAndSucceed.dtsx` 配置为同步运行，第二个 SSIS 包和第三个 SSIS 包（`RunForSomeTime.dtsx`）应该在“执行目录包任务”能够启动它们时立即启动。这正是我们在图 25-38 中看到的情况；`RunForSomeTime.dtsx` 在 `ReportParameterAndSucceed.dtsx` 开始执行后 4 秒启动。

您还可以看到 `RunForSomeTime.dtsx` 的执行 ``不`` 等待 `ReportParameterAndSucceed.dtsx` 执行完成。`ReportParameterAndSucceed.dtsx` 执行了近 13 秒（参见“持续时间”列）。

接下来，我们将分析转向 `Use32bit` 元数据 —— 在图 25-40 中用绿色框标出。SQL Server Management Studio 中内置的 SSIS 目录报告非常棒。还有其他查看数据的方式。使用清单 25-10 中的 T-SQL 查看 `Use32bit` 元数据的使用情况。

```sql
Select execution_id
, folder_name
, project_name
, package_name
, use32bitruntime
From [catalog].executions
Where execution_id >= 10 /* ReportAndSucceed.dtsx execution_id */
Order By execution_id Desc
清单 25-10
查看 Use32bit 执行设置
```

查询结果与我们的元数据配置完全匹配，如图 25-41 所示：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig41_HTML.jpg](img/449652_2_En_25_Fig41_HTML.jpg)
*图 25-41 实际应用中的 Use32bit 元数据*

最后，分析 `Logging Level` 元数据的应用 —— 在图 25-40 中用蓝色框标出。点击每个执行的“概述”报告以查看“使用的参数”表格。作者编辑了图 25-42 的图像，使部分“执行信息”清晰可读，并与“使用的参数”表格处于同一截图中：

![../images/449652_2_En_25_Chapter/449652_2_En_25_Fig42_HTML.jpg](img/449652_2_En_25_Fig42_HTML.jpg)
*图 25-42 在概述报告中查看使用的参数*

请注意，对于此次 `ReportParameterAndSucceed.dtsx` SSIS 包的执行，`LOGGING_LEVEL` 执行参数被设置为 `2`。SSIS 目录日志级别是一个枚举：

*   `0`: 无
*   `1`: 基本
*   `2`: 性能
*   `3`: 详细
*   `4`: 运行时谱系

SSIS 目录用户也可以创建自定义日志级别，前提是他们拥有适当的 SSIS 目录权限。

另请注意，可以查看此 `ReportParameterAndSucceed.dtsx` SSIS 包执行的 `SYNCHRONIZED` 执行参数值，如图 25-42 所示。这两个执行参数都与图 25-40 中的元数据配置相匹配。

尽管这个分析可能比较粗略，但它揭示了我们的框架正按设计运行。

### 结论

在本章中，我们在 Azure Data Factory 的 Azure-SSIS 集成运行时中测试了“执行目录包任务”功能，方法包括：

*   将 `TestSSISProject` 部署到前一章配置的 Azure SQL DB 上托管的 SSIS 目录
*   部署 `ECPTFramework` SSIS 项目
*   将框架元数据迁移到 `SSISDB` 以外的 Azure SQL DB
*   部署 `ECPTFramework` SSIS 项目
*   测试 SSIS 框架的 `Parent.dtsx` 包的执行

如果一切按计划进行，我们便实现了本章记录的结果。

接下来，作者将审视在开发“执行目录包任务”过程中获得的一些经验教训。


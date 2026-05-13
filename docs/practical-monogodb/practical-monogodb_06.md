# 5. MongoDB - 安装与配置

> “MongoDB 是一个跨平台的数据库。”

在本章中，你将了解在 Windows 和 Linux 上安装 MongoDB 的过程。

## 5.1 选择你的版本

MongoDB 可以在大多数平台上运行。所有可用软件包的列表可在 MongoDB 下载页面 [`www.mongodb.org/downloads`](http://www.mongodb.org/downloads) 上找到。

适合你环境的正确版本取决于你的服务器操作系统和处理器类型。MongoDB 支持 32 位和 64 位架构，但建议在生产环境中使用 64 位。

**32 位限制**

这是由于 MongoDB 使用了内存映射文件。这限制了 32 位构建版本只能处理大约 2GB 的数据。出于性能原因，建议在生产环境中使用 64 位构建版本。

在撰写本书时，最新的 MongoDB 生产版本是 3.0.4。适用于 Linux、Windows、Solaris 和 Mac OS X 的 MongoDB 下载包可用。

MongoDB 下载页面分为以下部分：

*   当前稳定版本 (3.0.4) – 2015 年 6 月 16 日
*   先前版本 (稳定版)
*   开发版本 (不稳定版)

当前版本是最近可用的最稳定版本，在撰写本书时为 3.0.4。当新版本发布时，之前的稳定版本将被移至“先前版本”部分。

开发版本，顾名思义，是仍在开发中的版本，因此被标记为不稳定。这些版本可能有额外的功能，但由于仍在开发阶段，可能不稳定。你可以使用开发版本尝试新功能，并向 10gen 提供有关功能和遇到问题的反馈。

## 5.2 在 Linux 上安装 MongoDB

本节介绍如何在 LINUX 系统上安装 MongoDB。在接下来的演示中，我们将使用 Ubuntu Linux 发行版。你可以通过手动方式或通过软件仓库安装 MongoDB。我们将引导你完成这两种选项。



### 5.2.1 通过软件仓库安装

在 **LINUX** 中，软件仓库是包含软件的在线目录。**APT 包管理器**是 **Ubuntu** 上用来安装软件的程序。虽然 **MongoDB** 可能存在于默认仓库中，但版本有可能过时，所以第一步是配置 **APT** 以使用自定义仓库。

执行以下命令以导入 **MongoDB** 的 `public.GPG` 密钥：
```bash
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
```
接下来，使用以下命令创建 `/etc/apt/sources.list.d/mongodb-org-3.0.list` 文件：
```bash
echo "deb [`http://repo.mongodb.org/apt/ubuntu`](http://repo.mongodb.org/apt/ubuntu) \"$(lsb_release -sc)\"/mongodb-org/3.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.0.list
```
最后，使用以下命令重新加载软件仓库：
```bash
sudo apt-get update
```
现在，**APT** 已知晓手动添加的仓库。

接下来，你需要安装软件。应在 **shell** 中执行以下命令来安装 **MongoDB** 当前的稳定版本：
```bash
sudo apt-get install -y mongodb-org
```

你已成功安装 **MongoDB**，整个过程就是这么简单。

### 5.2.2 手动安装

本节将介绍如何手动安装 **MongoDB**。在以下情况下，这些知识很重要：

*   当 **Linux** 发行版不使用 **APT 包管理器**时。
*   当你需要的版本无法通过仓库获得或不属于该仓库时。
*   当你需要同时运行多个 **MongoDB** 版本时。

手动安装的第一步是决定要使用的 **MongoDB** 版本，然后从网站下载。接着，需要使用以下命令解压安装包：
```bash
tar -xvf mongodb-linux-x86_64-3.0.4.tgz
```
这将输出类似以下的内容：
```
mongodb-linux-i686-3.0.4/THIRD-PARTY-NOTICES
mongodb-linux-i686-3.0.4/GNU-AGPL-3.0
mongodb-linux-i686-3.0.4/bin/mongodump
............
mongodb-linux-i686-3.0.4/bin/mongosniff
mongodb-linux-i686-3.0.4/bin/mongod
mongodb-linux-i686-3.0.4/bin/mongos
mongodb-linux-i686-3.0.4/bin/mongo
```
这会将包内容解压到一个新目录，即 `mongodb-linux-x86_64-3.0.4`（位于你当前目录下）。该目录包含许多子目录和文件。主要的可执行文件位于 `bin` 子目录下。

至此，**MongoDB** 安装成功完成。

## 5.3 在 Windows 上安装 MongoDB

在 **Windows** 上安装 **MongoDB** 很简单，只需下载适用于所选 **Windows** 版本的 **msi** 文件并运行安装程序即可。

安装程序将引导你完成 **MongoDB** 的安装。

按照向导操作，你将到达“选择安装类型”屏幕。有两种安装类型可供选择，你可以自定义安装。在此示例中，选择安装类型为“自定义”。

选择“自定义”时需要指定安装目录，因此请指定目录为 `C:\PracticalMongoDB`。

请注意，**MongoDB** 可以从用户选择的任何文件夹运行，因为它是自包含的，不依赖于系统。如果选择了“完全”安装类型，则默认选择的文件夹是 `C:\Program Files\MongoDB`。

单击“下一步”将带你进入“准备安装”屏幕。单击“安装”。

这将开始安装，并在屏幕上显示进度。安装完成后，向导将带你进入完成屏幕。

单击“完成”即结束安装。成功完成上述步骤后，你将拥有一个名为 `C:\PracticalMongoDB` 的目录，其中 `bin` 文件夹包含所有相关应用程序。整个过程就是这么简单。

## 5.4 运行 MongoDB

让我们看看如何开始运行和使用 **MongoDB**。

### 5.4.1 先决条件

需要一个数据文件夹来存储文件。在 **Windows** 上默认是 `C:\data\db`，在 **LINUX** 系统上是 `/data/db`。

这些数据目录不是由 **MongoDB** 创建的，因此在启动 **MongoDB** 之前，需要手动创建数据目录，并确保设置了适当的权限（例如，**MongoDB** 拥有读、写和创建目录的权限）。

如果你在创建文件夹之前启动了 **MongoDB**，它将抛出错误消息并无法运行。

### 5.4.2 启动服务

一旦目录创建完毕且权限到位，执行 `mongod` 应用程序（位于 `bin` 目录下）以启动 **MongoDB** 核心数据库服务。

延续上述安装，可以通过在 **Windows** 中打开命令提示符（需要以管理员身份运行）并执行以下命令来启动：
```bash
c:\> c:\practicalmongodb\bin\mongod.exe
```
在 **Linux** 的情况下，在 **shell** 中启动 `mongod` 进程。

这将在本地主机接口上启动 **MongoDB** 数据库。它将监听来自 `mongo` **shell** 在端口 `27017` 上的连接。

如前所述，需要在启动数据库之前创建文件夹路径，默认为 `c:\data\db`。在启动数据库服务时，也可以通过使用 `–dbpath` 参数提供替代路径。
```bash
C:\> C:\practicalmongodb\bin\mongod.exe --dbpath C:\NewDBPath\DBContents
```

## 5.5 验证安装

相关的可执行文件将位于 `bin` 子目录下。为了验证安装步骤是否成功，可以在 `bin` 目录下检查以下内容：

*   `Mongod`：核心数据库服务器
*   `Mongo`：数据库 **shell**
*   `Mongos`：自动分片进程
*   `Mongoexport`：导出实用程序
*   `Mongoimport`：导入实用程序

除了上述之外，`bin` 文件夹中还有其他可用的应用程序。

`mongo` 应用程序启动 `mongo` **shell**，它提供对数据库内容的访问，并允许你对 **MongoDB** 中的数据执行选择性查询或聚合操作。

如上所述，`mongod` 应用程序用于启动数据库服务或守护进程。

启动应用程序时可以设置多个标志。例如，`–dbpath` 可用于指定数据库文件存储的替代路径。要获取所有可用选项的列表，在启动服务时包含 `--help` 标志。

## 5.6 MongoDB Shell

`mongo` **shell** 是 **MongoDB** 标准发行版的一部分。该 **shell** 为 **MongoDB** 提供了完整的数据库接口，使你能够使用 **JavaScript** 环境处理存储在 **MongoDB** 中的数据，该环境可以完全访问该语言及其所有标准函数。

一旦数据库服务启动，你就可以打开 `mongo` **shell** 并开始使用 **MongoDB**。这可以在 **Linux** 中使用 **Shell** 或在 **Windows** 中使用命令提示符（以管理员身份运行）来完成。

你必须参考可执行文件的确切位置，例如在 **Windows** 环境中位于 `C:\practicalmongodb\bin\` 文件夹。

打开命令提示符（以管理员身份运行）并输入 `mongo.exe`。按 **Enter** 键。这将启动 `mongo` **shell**。
```bash
C:\> C:\practicalmongodb\bin\mongo.exe
```
输出将类似于：
```
MongoDB shell version: 3.0.4
connecting to: test
>
```
如果在启动服务时未指定参数，它会连接到本地主机实例上名为 `test` 的默认数据库。

当连接到一个不存在的数据库时，它将被自动创建。**MongoDB** 提供了在尝试访问尚不存在的数据库时自动创建它的功能。

下一章提供了更多关于使用 `mongo` **shell** 的信息。

## 5.7 保护部署安全

你已经知道如何通过默认配置安装和开始使用 **MongoDB**。接下来，你需要确保存储在数据库中的数据在各方面都是安全的。

在本节中，你将了解如何保护数据安全。你将更改默认安装的配置，以确保数据库更加安全。





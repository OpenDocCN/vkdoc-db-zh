# 使用外部表卸载与加载数据

`NODISCARDFILE` | 指定不应创建废弃文件。
`SKIP` | 在加载前跳过文件中指定数量的记录。
`PREPROCESSOR` | 指定在 Oracle 加载数据之前运行并修改文件内容的用户命名程序。
`MISSING FIELD VALUES ARE NULL` | 将没有数据的字段加载为 `NULL` 值。

## 概述

外部表也可用于从常规数据库表中选择数据并创建二进制转储文件。这被称为数据卸载。此技术的优点是转储文件独立于平台，可用于在不同平台的服务器之间移动大量数据。

在创建转储文件时，还可以加密或压缩数据，或同时进行。这样做为您提供了一种高效且安全的在数据库服务器之间传输数据的方法。

图 14-2 展示了使用外部表卸载和加载数据的组件。在源数据库（数据库 A）上，您使用一个外部表（该表从名为 `INV` 的表中选择数据）创建一个转储文件。创建后，您将转储文件复制到目标服务器（数据库 B），然后使用外部表将该文件加载到数据库中。

![使用外部表进行数据卸载和加载](img/214899_3_En_14_Fig2_HTML.jpg)

图 14-2 使用外部表进行数据卸载和加载

## 使用外部表卸载数据的步骤

以下是一个说明使用外部表卸载数据技术的小例子。步骤如下：

1.  创建一个目录对象，指定您希望在磁盘上放置转储文件的位置。如果您不是使用 DBA 帐户，则将对该目录对象的读写访问权限授予需要访问的数据库用户。

2.  使用 `CREATE TABLE...ORGANIZATION EXTERNAL...AS SELECT` 语句将数据从数据库卸载到转储文件中。

首先，创建一个目录对象。下面的代码创建一个名为 `DP` 的目录对象，指向 `/oradump` 目录：

```sql
SQL> create directory dp as '/oradump';
```

如果您不是使用具有 DBA 权限的用户，请显式地将对目录对象的访问权限授予所需用户：

```sql
SQL> grant read, write on directory dp to larry;
```

## 示例：创建并访问转储文件

此示例依赖于一个名为 `INV` 的表；作为参考，以下是 `INV` 表的 DDL：

```sql
SQL> CREATE TABLE inv
(inv_id NUMBER,
inv_desc VARCHAR2(30));
```

要创建转储文件，请使用 `CREATE TABLE...ORGANIZATION EXTERNAL` 语句的 `ORACLE_DATAPUMP` 访问驱动程序。此示例将 `INV` 表的内容卸载到 `inv.dmp` 文件中：

```sql
SQL> CREATE TABLE inv_et
ORGANIZATION EXTERNAL (
TYPE ORACLE_DATAPUMP
DEFAULT DIRECTORY dp
LOCATION ('inv.dmp')
)
AS SELECT * FROM inv;
```

前面的命令创建了两个东西：

*   一个名为 `INV_ET` 的外部表，基于 `INV` 表的结构和数据。
*   一个名为 `inv.dmp` 的平台无关的转储文件。

现在，您可以将 `inv.dmp` 文件复制到另一个数据库服务器，并基于此转储文件创建一个外部表。远程服务器（您将转储文件复制到的服务器）可以是与您创建文件的服务器不同的平台。例如，您可以在 Windows 机器上创建转储文件，复制到 Unix/Linux 服务器，然后通过外部表从该转储文件中选择数据。在此示例中，外部表名为 `INV_DW`：

```sql
SQL> CREATE TABLE inv_dw
(inv_id number
,inv_desc varchar2(30))
ORGANIZATION EXTERNAL (
TYPE ORACLE_DATAPUMP
DEFAULT DIRECTORY dp
LOCATION ('inv.dmp')
);
```

创建后，您可以从 SQL*Plus 访问外部表数据：

```sql
SQL> select * from inv_dw;
```

您也可以使用转储文件创建常规表并加载数据：

```sql
SQL> create table inv as select * from inv_dw;
```

这提供了一种从一个平台到另一个平台传输数据的简单而高效的机制。

## 启用并行以减少耗时

为了在通过外部表创建转储文件时最大化卸载性能，请使用 `PARALLEL` 子句。此示例并行创建两个转储文件：

```sql
SQL> CREATE TABLE inv_et
ORGANIZATION EXTERNAL (
TYPE ORACLE_DATAPUMP
DEFAULT DIRECTORY dp
LOCATION ('inv1.dmp','inv2.dmp')
)
PARALLEL 2
AS SELECT * FROM inv;
```

要访问转储文件中的数据，请创建一个引用这两个转储文件的不同外部表：

```sql
SQL> CREATE TABLE inv_dw
(inv_id number
,inv_desc varchar2(30))
ORGANIZATION EXTERNAL (
TYPE ORACLE_DATAPUMP
DEFAULT DIRECTORY dp
LOCATION ('inv1.dmp','inv2.dmp')
);
```

您现在可以使用此外部表从转储文件中选择数据：

```sql
SQL> select * from inv_dw;
```

## 压缩转储文件

您可以通过外部表创建压缩的转储文件。例如，使用 `ACCESS PARAMETERS` 子句的 `COMPRESS` 选项：

```sql
SQL> CREATE TABLE inv_et
ORGANIZATION EXTERNAL (
TYPE ORACLE_DATAPUMP
DEFAULT DIRECTORY dp
ACCESS PARAMETERS (COMPRESSION ENABLED BASIC)
LOCATION ('inv1.dmp')
)
AS SELECT * FROM inv;
```

使用此选项时，您应该会看到相当好的压缩率。在我的测试中，压缩后的输出转储文件比原始文件小 10 到 20 倍。您的结果可能会有所不同，具体取决于被压缩的数据类型。

从 Oracle Database 12c 开始，有四个压缩级别：`BASIC`、`LOW`、`MEDIUM` 和 `HIGH`。在使用压缩之前，请确保 `COMPATIBLE` 初始化参数设置为 12.0.0 或更高。

> **注意**
> 使用压缩需要 Oracle 企业版以及高级压缩选项。

## 加密转储文件

您也可以使用外部表创建加密的转储文件。此示例使用 `ACCESS PARAMETERS` 子句的 `ENCRYPTION` 选项：

```sql
SQL> CREATE TABLE inv_et
ORGANIZATION EXTERNAL (
TYPE ORACLE_DATAPUMP
DEFAULT DIRECTORY dp
ACCESS PARAMETERS
(ENCRYPTION ENABLED)
LOCATION ('inv1.dmp')
)
AS SELECT * FROM inv;
```

要使此示例正常工作，您需要有一个安全钱包为您的数据库准备好并处于打开状态。

> **注意**
> 使用加密需要 Oracle 企业版以及高级安全选项。

## 启用 Oracle Wallet

Oracle Wallet 是 Oracle 用于启用加密的机制。钱包是一个包含加密密钥的操作系统文件。钱包通过以下步骤启用：

1.  修改 `SQLNET.ORA` 文件以包含钱包的位置：

    ```ini
    ENCRYPTION_WALLET_LOCATION=
    (SOURCE=(METHOD=FILE) (METHOD_DATA=
    (DIRECTORY=/ora01/app/oracle/product/12.1.0.1/db_1/network/admin)))
    ```

2.  使用 `ALTER SYSTEM` 命令创建钱包文件 (`ewallet.p12`)：

    ```sql
    SQL> alter system set encryption key identified by foo;
    ```

3.  启用加密：

    ```sql
    SQL> alter system set encryption wallet open identified by foo;
    ```

有关实现加密的完整详情，请参阅 Oracle 高级安全管理员指南，该指南可从 Oracle 网站的技术网络区域（[`http://otn.oracle.com`](http://otn.oracle.com)）免费下载。

## 访问参数

您通过 `ACCESS PARAMETERS` 子句启用压缩和加密。表 14-2 包含了 `ORACLE_DATAPUMP` 访问驱动程序可用的所有访问参数列表。

表 14-2 `ORACLE_DATAPUMP` 访问驱动程序的参数

| 访问参数 | 描述 |
| --- | --- |
| `COMPRESSION` | 压缩转储文件；`DISABLED` 是默认值。 |
| `ENCRYPTION` | 加密转储文件；`DISABLED` 是默认值。 |
| `NOLOGFILE` | 抑制日志文件的生成。 |
| `LOGFILE=[directory_object:]logfile_name` | 允许您命名日志文件。 |
| `VERSION` | 指定可以读取转储文件的 Oracle 最低版本。 |


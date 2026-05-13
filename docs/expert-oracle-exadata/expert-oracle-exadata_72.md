# 数据库范围安全配置

使用各数据库的 `show parameter` 命令，获取每个正在配置的数据库的 `DB_UNIQUE_NAME`：

```
SYS:+HR>show parameter db_unique_name
NAME             TYPE        VALUE
---------------- ----------- -----
db_unique_name   string      HR

SYS:+PAY>show parameter db_unique_name
NAME             TYPE        VALUE
---------------- ----------- -----
db_unique_name   string      PAY
```

关闭 ASM 集群中的所有数据库和 ASM 实例。

使用 CellCLI `create key` 命令创建安全密钥：

```
CellCLI> create key
        7548a7d1abffadfef95a53185aba0e98
CellCLI> create key
        8e7105bdbd6ad9fa53d41736a533b9b1
```

`create key` 命令必须为 ASM 集群中的每个数据库运行一次。可以从任何存储单元运行该命令。在数据库 `cellkey.ora` 文件的 `key` 字段中，将为 ASM 集群内的每个数据库分配一个安全密钥。

对于每个数据库，使用步骤 2 中创建的密钥创建一个 `cellkey.ora` 文件。将这些 `cellkey.ora` 文件安装到要配置数据库范围安全的每个数据库服务器的 `ORACLE_HOME/admin/{db_unique_name}/pfile` 目录中。

就像为 ASM 范围安全所做的那样，将文件的所有者设置为在 ASM 软件安装期间指定的用户和组。权限应允许文件所有者读取该文件。例如：

```
# -- Cellkey.ora file for the HR database --#
key=7548a7d1abffadfef95a53185aba0e98
asm=+ASM
realm=customer_A_realm
# --
> chown oracle:dba $ORACLE_HOME/admin/HR/cellkey.ora
> chmod 600 $ORACLE_HOME/admin/HR/cellkey.ora

# -- Cellkey.ora file for the PAY database --#
key=8e7105bdbd6ad9fa53d41736a533b9b1
asm=+ASM
realm=customer_A_realm
# --
> chown oracle:dba $ORACLE_HOME/admin/PAY/cellkey.ora
> chmod 600 $ORACLE_HOME/admin/PAY/cellkey.ora
```

请注意，如果在此文件中定义了 realm，它必须与使用 `alter cell realm=` 命令分配给存储单元的 realm 名称匹配。

使用 CellCLI `assign key` 命令为每个正在配置的数据库分配安全密钥。必须在要让 HR 和 PAY 数据库有权访问的每个存储单元上执行此操作。以下密钥被分配给 HR 和 PAY 数据库的 `DB_UNIQUE_NAME`：

```
CellCLI> ASSIGN KEY -
   FOR HR=’7548a7d1abffadfef95a53185aba0e98’, –
      PAY=’8e7105bdbd6ad9fa53d41736a533b9b1’
Key for HR successfully created
Key for PAY successfully created
```

验证密钥是否已正确分配：

```
CellCLI> list key
         HR      d346792d6adea671d8f33b54c30f1de6
         PAY     cae17e8fdce7511cc02eb7375f5443a8
```

使用 CellCLI `create disk` 或 `alter griddisk` 命令，为每个数据库分配对网格磁盘的访问权限。请注意，ASM 唯一名称与数据库唯一名称一起包含在此分配中。

```
CellCLI> create griddisk DATA_HR_CD_00_cell03, -
                         DATA_HR_CD_01_cell03  -
                size=1282.8125G             -
                availableTo=’+ASM,HR’
CellCLI> create griddisk DATA_PAY_CD_00_cell03, -
                         DATA_PAY_CD_01_cell03  -
                size=1282.8125G             -
                availableTo=’+ASM,PAY’
```

`alter griddisk` 命令可用于更改网格磁盘的安全分配。例如：

```
CellCLI> alter griddisk DATA_HR_CD_00_cell03, -
                        DATA_HR_CD_01_cell03  -
               availableTo=’+ASM,HR’
CellCLI> alter griddisk DATA_PAY_CD_00_cell03, -
                        DATA_PAY_CD_01_cell03  -
               availableTo=’+ASM,PAY’
```

至此，已完成对 HR 和 PAY 数据库的数据库范围安全配置。现在可以重新启动 ASM 集群和数据库。人力资源数据库现在可以访问 `DATA_HR` 网格磁盘，而工资单数据库可以访问 `DATA_PAY` 网格磁盘。

### 移除单元安全

实施后，可以通过更新存储单元上的 ACL 列表并更改网格磁盘的 `availableTo` 属性，根据需要修改单元安全。移除单元安全是一个相当直接的过程，包括撤销数据库安全设置，然后移除 ASM 安全设置。

移除单元安全的第一步是移除数据库范围安全。以下步骤将从系统中移除数据库范围安全：

在移除数据库安全之前，必须关闭数据库和 ASM 集群。

使用 CellCLI 命令 `alter griddisk` 从网格磁盘的 `availableTo` 属性中移除数据库。此命令不会从列表中选择性地移除数据库。它只是重新定义了完整的列表。请注意，此时我们只是从列表中移除数据库。ASM 唯一名称应暂时保留在列表中。这必须在要移除安全性的每个单元上执行。

```
CellCLI> alter griddisk DATA_HR_CD_00_cell03, -
                        DATA_HR_CD_01_cell03  -
               availableTo=’+ASM’
CellCLI> alter griddisk DATA_PAY_CD_00_cell03, -
                        DATA_PAY_CD_01_cell03  -
               availableTo=’+ASM’
```

或者，可以使用以下命令从受保护的网格磁盘中移除所有数据库：

```
CellCLI> alter griddisk all availableTo=’+ASM’
```

假设这些数据库尚未在此单元中的任何其他网格磁盘上配置单元安全，则可以从存储单元上的 ACL 列表中移除安全密钥，如下所示：

```
CellCLI> assign key for HR=’’, PAY=’’
Key for HR successfully dropped
Key for PAY successfully dropped
```

移除位于数据库客户端 `ORACLE_HOME/admin/{db_unique_name}/pfile` 目录中的 `cellkey.ora` 文件。

使用以下 CellCLI 命令验证 HR 和 PAY 数据库是否未分配给任何网格磁盘：

```
CellCLI> list griddisk attributes name, availableTo
```

一旦移除了数据库范围的安全，就可以移除 ASM 范围的安全。这将使系统恢复为默认的开放安全状态。以下步骤移除 ASM 范围的安全。完成后，网格磁盘将对存储网络上的所有 ASM 集群和数据库可用。

在继续此过程之前，请确保数据库范围的安全已完全移除。`list key` 命令应仅显示 ASM 集群的密钥分配。此时不应有任何数据库被分配密钥。`list griddisk` 命令应显示所有网格磁盘的分配名称，分配对象为 ASM 集群 `’+ASM’`。

```
CellCLI> list griddisk attributes name, availableTo
```

接下来，从所有网格磁盘的 `availableTo` 属性中移除 ASM 唯一名称。

```
CellCLI> list griddisk attributes name, availableTo
```

现在，通过运行以下命令从 ACL 中移除 ASM 安全：

```
CellCLI> alter griddisk all assignTo=’’
```

以下命令为选定的网格磁盘移除 ASM 集群分配：

```
CellCLI> alter griddisk DATA_CD_00_cell03, -
                        DATA_CD_01_cell03  -
                        DATA_CD_02_cell03  -
                        DATA_CD_03_cell03  -
               availableTo=’’
```

`list griddisk` 命令应显示没有分配的客户端。运行 `list griddisk` 命令进行验证。

现在可以使用 CellCLI `assign key` 命令从存储单元安全地移除 ASM 集群密钥：

```
CellCLI> list key detail
         name:     +ASM
         key:      196d7983a9a33fccae276e24e7a9f89
CellCLI> assign key for +ASM=’’
Key for +ASM successfully dropped
```

从 ASM 集群中所有数据库服务器的 `/etc/oracle/cell/network-config` 目录中移除 `cellkey.ora` 文件。

至此，ASM 范围安全的移除完成。现在可以重新启动 ASM 集群以及它服务的所有数据库。



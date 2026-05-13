# 矛盾之处

我承认，Oracle 镜像中让我颇为烦恼的一点是 Oracle 清单目录被安装在 `ORACLE_BASE` 之下。Oracle 的 *OFA*（最优灵活架构）将清单目录置于 `ORACLE_BASE` 之外，以支持多个安装。在一个 *Real Application Clusters*（RAC）系统中，符合 OFA 的目录结构可能如下所示：

```
# Oracle 清单
/u01/app/oraInventory
# 数据库 BASE 和 HOME:
/u01/app/oracle
/u01/app/oracle/product/$DB_VERSION/dbhome_1
# 网格基础设施 BASE 和 HOME:
/u01/app/grid
/u01/app/$GRID_VERSION/grid
```

这种独立的、专用的目录结构允许不同的所有权和必要的权限，以支持角色分离，所有这些都由一个单独的清单目录管理。

在容器中采用不同的做法有其合理的原因。查看 Oracle 的某个 Dockerfile，在接近末尾处你会看到如下代码块：

```
USER oracle
COPY --chown=oracle:dba --from=builder $ORACLE_BASE $ORACLE_BASE
USER root
RUN $ORACLE_BASE/oraInventory/orainstRoot.sh && \
$ORACLE_HOME/root.sh
```

`COPY` 命令从先前的构建阶段（通过 `--from=builder` 参数引用）复制整个 `ORACLE_BASE` 目录结构到最终的镜像中。随后的指令在最终镜像中运行安装后的 root 脚本，设置清单目录的权限并注册 `ORACLE_HOME`。

如果清单目录位于 `ORACLE_BASE` 之下，一条命令就能复制所有内容——清单、`ORACLE_BASE` 和 `ORACLE_HOME`。由于每条命令都会在镜像中创建一个独立的层，将清单整合到 `ORACLE_BASE` 之下可以减少层数（并可能减小最终镜像的大小）。对镜像运行 `docker history` 可以看到此次复制操作创建的单一层面。为了清晰起见，我使用了格式化命令只打印“CREATED BY”字段，显示创建该层所用的命令：

```
> docker image history --no-trunc --format "table {{.CreatedBy}}"  oracle/database:19.3.0-ee
CREATED BY
...
/bin/sh -c $ORACLE_BASE/oraInventory/orainstRoot.sh &&     $ORACLE_HOME/root.sh
/bin/sh -c #(nop)  USER root
/bin/sh -c #(nop) COPY --chown=oracle:dbadir:7f7ee78c2762cb56f03228b45ce0c0301ea5fd88c4ac14dd98d74eb5621054e4 in /opt/oracle
/bin/sh -c #(nop)  USER oracle
...
```

这是另一种情况，从最终用户的角度看，容器中的清单位置并不重要。它对客户端没有影响——端点是一个执行数据库操作的数据库。但它在以下情况下确实重要：

*   `Real Application Clusters` 或 `Global Service Manager` 安装
*   拥有多个 `ORACLE_BASE` 目录的系统
*   需要角色分离的环境
*   保持与现有基础设施的一致性
*   防止安装程序错误

在最后一种情况下，如果清单位置位于 `ORACLE_BASE` 之下，Oracle 安装可能会返回以下警告（并伴随一个非零退出码）：

```
Attention: INS-32056: The specified Oracle Base contains the existing Central Inventory location.
Recommendation: Oracle recommends that the Central Inventory location is outside the Oracle Base directory. Specify a different location for the Oracle Base.
```

或

```
[WARNING] [INS-32056] The specified Oracle Base contains the existing Central Inventory location: /opt/oracle/oraInventory.
ACTION: Oracle recommends that the Central Inventory location is outside the Oracle Base directory. Specify a different location for the Oracle Base.
```

在一个*完美*的世界里，自动化流程应该因错误和警告而失败，标记出需要审查的条件，或者包含条件异常处理器来忽略它们。假设警告未被捕获或被视为潜在的故障。那样的话，就无法区分预期的、“可以安全忽略”的警告与可能影响镜像可靠性或准确性的其他警告。

有多种方法可以解决这个问题。在我的脚本中，我为 `ORACLE_INVENTORY` 创建了一个参数和环境变量，并在 Dockerfile 中为清单目录添加了第二条 `COPY` 指令：

```
ARG ORACLE_INVENTORY=/u01/app/oraInventory
ENV ORACLE_INVENTORY=$ORACLE_INVENTORY
```

我们还需要添加一条新的 `COPY` 指令来处理清单：

```
USER oracle
COPY --chown=oracle:dba --from=builder $ORACLE_BASE $ORACLE_BASE
COPY --chown=oracle:dba --from=builder $ORACLE_INVENTORY $ORACLE_INVENTORY
```

然后，更新执行安装后 root 脚本的 `RUN` 命令，将对 `ORACLE_BASE` 的引用改为新的 `ORACLE_INVENTORY` 变量：

```
USER root
RUN $ORACLE_INVENTORY/oraInventory/orainstRoot.sh && \
$ORACLE_HOME/root.sh
```

这解决了 Dockerfile 中的步骤。我们还需要将新的目录位置推送到数据库安装中。数据库安装响应文件 `db_inst.rsp` 通过 `INVENTORY_LOCATION` 条目设置清单路径。将以下行从

```
INVENTORY_LOCATION=###ORACLE_BASE###/oraInventory
```

改为

```
INVENTORY_LOCATION=###ORACLE_INVENTORY###/oraInventory
```

最后一步是在 `installDBBinaries.sh` 脚本中添加一行，用环境中的清单位置替换占位符。修改替换占位符的代码部分，使其如下所示：

```
# 替换占位符
# ---------------------
sed -i -e "s|###ORACLE_EDITION###|$EDITION|g" "$INSTALL_DIR"/"$INSTALL_RSP" && \
sed -i -e "s|###ORACLE_BASE###|$ORACLE_BASE|g" "$INSTALL_DIR"/"$INSTALL_RSP" && \
sed -i -e "s|###ORACLE_HOME###|$ORACLE_HOME|g" "$INSTALL_DIR"/"$INSTALL_RSP" && \
sed -i -e "s|###ORACLE_INVENTORY###|$ORACLE_INVENTORY|g" "$INSTALL_DIR"/"$INSTALL_RSP"
```

更新清单位置涉及一些工作，但对于增强镜像的通用性、保持与现有约定的一致性、或遵守标准和推荐实践来说，这是值得的（有时是必要的）一步。它也消除了容器与“传统”环境之间的一个差异，而这个差异常被反对者用作反对容器的论据！

### 扩展的多租户选项

Oracle 在 12c 版本中引入了多租户数据库选项。多租户架构在 21c 及更高版本中成为强制要求。Oracle 容器镜像创建的多租户数据库包含容器数据库（CDB）和一个可插拔数据库（PDB）。用户可以通过向 `docker run` 命令传递环境变量来设置容器和可插拔数据库的名称。然而，在初始数据库设置期间，没有选项可以创建非容器数据库或自动添加多个可插拔数据库。

#### 创建非 CDB 数据库

在 Oracle 12c 至 19c 版本中，官方可能更倾向于我们创建和使用**容器数据库**。然而，在实际部署中，非 CDB 安装仍占据主导地位，并且在接下来的几年里很可能继续保持多数。因此，我们应当能够选择在容器中创建的架构，这是合理的。第 11 章介绍了向`docker run`传递选项以在容器与非容器架构之间进行选择的步骤。简而言之，它涉及：

*   更新`dbca.rsp.tmpl`模板，将硬编码的`createAsContainerDatabase`值替换为一个占位符。当一个新容器启动时，它会调用`createDB.sh`脚本，并引用**数据库配置助手**（`DBCA`）模板进行数据库创建。该占位符引入了通过替换环境变量来改变数据库创建方法的选项。
*   添加一个`sed`查找/替换命令，根据一个可选的环境变量，有条件地将`true`或`false`代入`DBCA`模板。
*   修改`checkDBStatus.sh`中的数据库健康检查，以便从`v$pdbs`（用于容器数据库）或`v$database`（用于非容器数据库）中读取打开模式的值。
*   有条件地从数据库创建脚本中移除特定于可插拔数据库的命令。

这个模式，无论是全部还是部分，都可以被调整，以在数据库创建和启动过程中添加类似的功能。

#### 创建多个可插拔数据库

可插拔数据库可以通过将多个独立数据库整合到单台主机上的可插拔数据库（PDB）中，从而提升数据库基础设施的效用、容量和使用寿命。与模式类似，PDB 提供了另一种对数据进行分段或隔离的方式，而一个容器数据库容纳两个或更多 PDB 的情况并不少见。将这一实践反映到数据库镜像中是合理的，即在初始容器创建时增加创建多个 PDB 的能力。有两种途径可以实现这一点。第一种方法是更改`DBCA`模板并从环境中读取变量。第二种方法是利用 Docker 入口点目录，在数据库设置完成后运行脚本。

第一种方法与创建非 CDB 数据库类似，即向数据库配置助手模板插入一个占位符，并添加一个`sed`命令，根据环境派生的值有条件地替换该占位符。`DBCA`模板中的关键项是`numberOfPDBs`，其默认值为 1：

```
numberOfPDBs=1
```

将其替换为占位符：

```
numberOfPDBs=###PDB_COUNT###
```

在`createDB.sh`脚本中添加一条`sed`命令以更改占位符。请记住需要验证输入的变量是否为正数、提供一个默认值，并且可能为可插拔数据库的数量设置一个实际上限：

```
if ! [[ $PDB_COUNT =~ '^[0-9]+$' ]]             # PDB_COUNT 必须是数字
then PDB_COUNT=1
elif [ "$PDB_COUNT" -ge 1 -a "$PDB_COUNT" -le 5 ] # PDB_COUNT 必须在 1 到 5 之间
then PDB_COUNT=$PDB_COUNT
else PDB_COUNT=1                                  # 其他所有情况设为默认值
fi
sed -i -e "s|###PDB_COUNT###|$PDB_COUNT|g" "$ORACLE_BASE"/dbca.rsp
```

您可以通过检查`PDB_COUNT`是否为零，来通过同一个变量控制创建 CDB 还是非 CDB 数据库。如果为零，则创建一个非 CDB 数据库。

当`numberOfPDBs`变量大于一时，数据库配置助手会创建容器数据库和第一个可插拔数据库，然后再创建额外的可插拔数据库。当`numberOfPDBs`大于一时，Oracle 会使用响应文件中`pdbName`的值作为前缀，并附加 PDB 的索引。它从模板中的`ORACLE_PDB`环境变量派生`pdbName`的值。请注意此行为，以避免出现意外的可插拔数据库名称！请记住，`ORACLE_PDB`的默认值是`ORCLPDB1`，当 PDB 数量为一时，Oracle 会创建一个名为`ORCLPDB1`的 PDB。但是，如果 PDB 数量为二，Oracle 会将 PDB 名称作为基础值，那么 PDB 名称将是`ORCLPDB11`和`ORCLPDB12`！


#### 使用设置入口点

我们在第 7 章介绍了 Docker 的入口点目录。如果需要快速回顾，入口点是 Docker 中的一个特殊路径，通常是`/docker-entrypoint-initdb.d`，用于挂载本地主机目录。容器启动脚本可能会在这里查找脚本并运行它们。我说它们*可能*会在这里查找，因为这取决于作者——这不是一个自动功能，如果启动脚本不识别该目录（或查看其他地方），Docker 会将入口点挂载的文件视为普通文件！

Oracle 的容器仓库在标准位置`/docker-entrypoint-initdb.d`和`/opt/oracle/scripts`识别入口点。它需要两个子目录——`setup`和`startup`——在其中搜索后缀为`.sh`或`.sql`的脚本。容器内置的自动化功能会检查`setup`目录中的文件，并在初始数据库创建完成后仅运行它们一次^([71)]。`startup`目录中的文件每次容器启动时都会运行——包括在初始数据库设置之后。

以下是使用数据库容器中的入口点时的一些提示和注意事项。

入口点目录中的文件按字母数字顺序处理，无论文件类型如何。在`setup`和`scripts`目录中命名文件时使用数字前缀，以保证其执行顺序：

`01_add_pdb.sh`
`02_add_users.sql`
`03_update_passwords.sql`

后缀为`.sh`的 Shell 脚本以容器的默认用户`oracle`执行。SQL 脚本传递给 SQL*Plus 并以`SYSDBA`用户运行。

对于针对可插拔数据库的 SQL 命令，请记住在脚本中设置容器。

挂载在`startup`目录中的文件每次容器启动时都会运行。它们不是镜像的一部分。对本地主机（或其他容器）中这些文件的更改会反映到共享主机目录的所有容器中，当容器重启时，可能导致意外行为和故障。在多个容器使用共享目录以及更改文件时请记住这一点。

`setup`入口点让我们可以使用`-createPluggabledatabase`选项调用数据库配置助理（Database Configuration Assistant），并通过如清单 13-1 所示的脚本添加可插拔数据库。

```bash
#!/bin/bash
# 创建一个额外的可插拔数据库。
$ORACLE_HOME/bin/dbca -silent -createPluggableDatabase -pdbName PDB2 -sourceDB $ORACLE_SID -createAsClone true -createPDBFrom DEFAULT -pdbAdminPassword "oracle123" || exit 1
cat > $ORACLE_HOME/network/admin/tnsnames.ora
PDB2 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = PDB2)
    )
  )
EOF
$ORACLE_HOME/bin/sqlplus / as sysdba << EOF
alter pluggable database PDB2 save state;
EOF
```

**清单 13-1** 一个用于创建额外可插拔数据库、添加 TNS 条目并设置可插拔数据库状态的设置脚本。将文件保存到目录并挂载到`/dockerfile-entrypoint-initdb.d/setup`或`/opt/oracle/scripts/setup`的卷中。

我将此文件命名为`01_add_pdb.sh`，并运行一个名为`SETUP`的新容器来演示对脚本的调用：

```bash
docker run -d --name SETUP \
-e ORACLE_SID=ORCLPDB \
-e ORACLE_PDB=PDB1 \
-v ~/setup:/opt/oracle/scripts/setup \
oracle/database:19.3.0-ee
```

Docker 调用通常的数据库创建脚本。当 DBCA 完成后，脚本读取`setup`目录的内容并运行后缀为`.sh`或`.sql`的文件。它在设置位置找到并运行了`01_add_pdb.sh`脚本。清单 13-2 中的尾部输出显示了`docker logs`输出的末尾，其中自定义用户脚本运行并创建了可插拔数据库。

```console
Executing user defined scripts
/opt/oracle/runUserScripts.sh: running /opt/oracle/scripts/setup/01_add_pdb.sh
Prepare for db operation
13% complete
Creating Pluggable Database
15% complete
19% complete
23% complete
31% complete
53% complete
Completing Pluggable Database Creation
60% complete
Executing Post Configuration Actions
100% complete
Pluggable database "PDB2" plugged successfully.
Look at the log file "/opt/oracle/cfgtoollogs/dbca/CDB/PDB2/CDB.log" for further details.
SQL*Plus: Release 19.0.0.0.0 - Production on Wed Jan 17 17:55:13 2022
Version 19.3.0.0.0
Copyright (c) 1982, 2019, Oracle.  All rights reserved.
Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
SQL>
Pluggable database altered.
SQL> Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
DONE: Executing user defined scripts
The Oracle base remains unchanged with value /opt/oracle
#########################
DATABASE IS READY TO USE!
#########################
```

**清单 13-2** `docker logs`的输出，显示在容器创建期间自定义用户脚本的执行情况。`01_add_pdb.sh`脚本创建了一个名为`PDB2`的可插拔数据库。完成后，它将控制权交还给`createDB.sh`脚本，后者接着报告数据库已准备好供用户使用。

清单 13-1 中的示例使用固定的名称和密码创建单个可插拔数据库。它适用于有限的、特定的情况。如果我们想创建多个可插拔数据库，或者需要一个更灵活的解决方案来根据运行时条件添加 PDB，该怎么办？

让我们扩展原始脚本，如清单 13-3 所示。脚本不再使用静态的 PDB 名称值，而是检查一个新的环境变量`PDB_LIST`是否已设置。该变量表示一个逗号分隔的一个或多个可插拔数据库名称列表，将在数据库创建完成后添加。脚本遍历列表中的每个元素，并执行与原始版本相同的步骤——创建新的 PDB、添加 TNS 条目并设置 PDB 状态——但是以动态的方式完成。

```bash
#!/bin/bash
# 创建额外的可插拔数据库。
if [ -n "$PDB_LIST" ] # 已定义 PDB 列表
then
  # 捕获现有的 IFS（内部字段分隔符）
  OLDIFS=$IFS
  # 在循环前将 IFS 设置为逗号：
  IFS=,
  # 遍历列表中的 PDB：
  for pdb_name in $PDB_LIST
  do
    # 在循环内部将 IFS 恢复为原始值：
    IFS=$OLDIFS
    # 创建 PDB：
    $ORACLE_HOME/bin/dbca -silent -createPluggableDatabase -pdbName $pdb_name -sourceDB $ORACLE_SID -createAsClone true -createPDBFrom DEFAULT -autoGeneratePasswords || exit 1
    # 添加 TNS 条目：
    cat > $ORACLE_HOME/network/admin/tnsnames.ora
    $pdb_name =
      (DESCRIPTION =
        (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
        (CONNECT_DATA =
          (SERVER = DEDICATED)
          (SERVICE_NAME = $pdb_name)
        )
      )
    EOF
    # 打开 PDB，保存其状态：
    sqlplus / as sysdba << EOF
    alter pluggable database $pdb_name save state;
    EOF
    # 在返回下一个循环步骤前，将 IFS 设置回逗号：
    IFS=,
  done
  # 将 IFS 设置回其原始值：
  IFS=$OLDIFS
fi
```

**清单 13-3** 这个修改后的脚本动态添加可插拔数据库，遍历通过环境变量传递给容器的逗号分隔的 PDB 名称列表。

在删除原始的`SETUP`容器后，我重新运行命令，这次定义`PDB_LIST`变量为`"TEST,DEV"`：

```bash
docker run -d --name SETUP \
-e ORACLE_SID=ORCLPDB \
-e ORACLE_PDB=PDB1 \
-e PDB_LIST="TEST,DEV" \
-v ~/setup:/opt/oracle/scripts/setup \
oracle/database:19.3.0-ee
```



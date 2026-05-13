# 执行用户定义脚本

`createDB.sh` 脚本检查 setup 入口点，并找到了 `01_add_pdb.sh` 脚本，这与之前一样。该脚本检查 `PDB_LIST` 变量是否已设置。如果没有，它会将控制权交还给 `createDB.sh` 脚本。不过在这个例子中该变量是存在的，循环会处理列表中的元素，创建 `TEST` 和 `DEV` 可插拔数据库及 TNS 条目，并保存 PDB 状态。清单 13-4 显示了使用更新后的 PDB 脚本时，新容器的 `docker logs` 输出。

```
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
Pluggable database "TEST" plugged successfully.
Look at the log file "/opt/oracle/cfgtoollogs/dbca/CDB/TEST/CDB.log" for further details.
SQL*Plus: Release 19.0.0.0.0 - Production on Wed Jan 17 20:16:39 2022
Version 19.3.0.0.0
Copyright (c) 1982, 2019, Oracle.  All rights reserved.
Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
SQL>
Pluggable database altered.
SQL> Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
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
Pluggable database "DEV" plugged successfully.
Look at the log file "/opt/oracle/cfgtoollogs/dbca/CDB/DEV/CDB.log" for further details.
SQL*Plus: Release 19.0.0.0.0 - Production on Wed Jan 17 20:20:34 2022
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
Listing 13-4
The tail end of the docker logs output in a container running the updated version of the 01_add_pdb.sh script, showing the creation of two pluggable databases, TEST and DEV
```

### 脚本修改的解释

在修改后的脚本中需要注意的一点是，`dbca` 命令中使用了 `-autoGeneratePasswords` 而不是固定密码。这与在新的容器中创建数据库的事件序列有关：

*   容器启动并执行 `runOracle.sh`。
*   `runOracle.sh` 检查是否存在现有数据库，但没有找到。它调用 `createDB.sh` 并将 `ORACLE_PWD` 的值传递给该脚本。除非作为 `docker run` 命令的一部分显式设置，或者通过 Podman secret 传递，否则 `ORACLE_PWD` 不会被导出或存在于环境中⁷²。
*   `createDB.sh` 创建数据库。对于密码，它首先检查是否存在 Oracle Wallet，如果存在则使用钱包中的值。如果不存在，并且从 `runOracle.sh` 传递过来的 `ORACLE_PWD` 值不为空，则使用该值。否则，它指示 Database Configuration Assistant 生成一个密码。
*   完成后，`createDB.sh` 将控制权交还给 `runOracle.sh`。
*   `runOracle.sh` 接下来调用 `runUserScripts.sh`，但没有传递 `ORACLE_PWD` 的值。
*   `runUserScripts.sh` 发现 `01_add_pdb.sh` 脚本并运行它。

由于 Linux shell 的工作方式，`ORACLE_PWD` 变量不能保证在 `runOracle.sh` 和 `createDB.sh` 脚本之外还有值。如果没有值，`-pdbAdminPassword "$ORACLE_PWD"` 会产生错误。为了简单和简洁，我让 PDB 创建自动生成 PDB 密码。添加类似于 `createDB.sh` 中的逻辑，检查钱包和 Podman secret 的存在，将进一步改进该脚本。

### 创建只读数据库主目录

Oracle Database 18c 新增了创建*只读数据库主目录*的选项，该选项将配置文件从 Oracle 数据库软件目录中移除。在旧版 Oracle 中，`ORACLE_HOME` 混合了软件和配置。`ORACLE_HOME/dbs` 目录是密码文件和数据库启动时使用的参数文件的默认位置。`ORACLE_HOME/network/admin` 目录存放网络配置文件，包括 `listener.ora`、`sqlnet.ora` 和 `tnsnames.ora` 文件。当数据库主目录支持多个实例时，这些特定于数据库的文件并未很好隔离，容易因另一个数据库的操作而被混淆或覆盖。这个缺点在 Docker 将配置文件从数据库主目录移动并复制到 `/opt/oracle/oradata/dbconfig` 下的特殊目录时表现得尤为明显。这对于从普通目录备份克隆数据库至关重要，如第 5 章所述。

随着只读主目录的引入，需要跟踪有关主目录的信息。Oracle 将此记录在一个名为 `orabasetab` 的特殊文件中。它仿照 `/etc/oratab` 文件，位于 `$ORACLE_HOME/install/orabasetab`。在传统的、读写主目录中，该文件内容如下：

```bash
#orabasetab file is used to track Oracle Home associated with Oracle Base
/opt/oracle/product/19c/dbhome_1:/opt/oracle:OraDB19Home1:N:
```

各个字段以冒号分隔，反映了主目录的不同属性。顺序为：

1.  `ORACLE_HOME` 目录，`/opt/oracle/product/19c/dbhome_1`。
2.  主目录使用的 `ORACLE_BASE`，`/opt/oracle`。
3.  唯一的 Oracle 主目录名称，在安装时分配，`OraDB19Home1`。
4.  主目录的只读状态，由 `Y` 或 `N` 表示。

该文件的第四字段设置为 `N`，意味着此主目录不是只读的，其配置文件位于“常规”位置。只读主目录中的数据库和网络配置文件从 `ORACLE_HOME` 移动到 `ORACLE_BASE` 下的新位置。两个新的环境变量，`ORACLE_BASE_CONFIG` 和 `ORACLE_BASE_HOME`，用于跟踪这些目录，它们默认为：

```bash
ORACLE_BASE_CONFIG=$ORACLE_BASE
ORACLE_BASE_HOME=$ORACLE_BASE/homes/
```

`$ORACLE_BASE_HOME` 中的 `HOME NAME` 是 `orabasetab` 中的第三个字段。基于前面的例子，代入 `orabasetab` 中的值，就可以揭示如果这是一个只读主目录，配置文件的基本位置：

```bash
ORACLE_BASE_CONFIG=/opt/oracle
ORACLE_BASE_HOME=/opt/oracle/homes/OraDB19Home1
```

原来存在于 `$ORACLE_HOME/dbs` 下的所有内容——参数文件、服务器参数文件和密码文件——都移动到 `$ORACLE_BASE_CONFIG/dbs`。之前存储在 `$ORACLE_HOME/network/admin` 的文件——TNS、SQL*Net 和侦听器文件——则重新定位到 `$ORACLE_BASE_HOME/network/admin`。

`ORACLE_BASE_CONFIG` 和 `ORACLE_BASE_HOME` 环境变量是可选的，Oracle 提供了两个新的二进制文件来报告它们的位置：

*   `orabaseconfig`，用于报告 `ORACLE_BASE_CONFIG`
*   `orabasehome`，用于报告 `ORACLE_BASE_HOME`

它们通过检查 `orabasetab` 中的第四字段（只读主目录字段）来推导正确的位置。通过这两个环境变量，我们可以推断出数据库主目录中数据库配置的正确路径，无论主目录类型如何。在任何数据库中了解这些目录都很重要，但在容器环境中至关重要，因为它们包含的配置文件必须重新定位到挂载到 `/opt/oracle/oradata` 的卷中，以保留容器的完整功能和能力。请记住，该卷保存了数据库的完整内容——包括其配置——我们开始意识到，修改容器镜像以包含“只读主目录”选项涉及：

*   执行将数据库软件目录转换为只读主目录的操作
*   为配置文件设置正确的目录路径
*   更新脚本，以便在引用配置时使用正确的目标位置

#### 将数据库主目录转换为只读

单个命令 `$ORACLE_HOME/bin/roohctl -enable`（用于只读 Oracle 主目录控制）可以将 21c 之前的数据库主目录转换为只读。它在安装数据库软件后、创建数据库之前执行。在 Docker 生命周期中有两个机会运行此命令：

*   在镜像创建期间，遵循数据库软件安装步骤之后
*   启动新的数据库容器时，在创建数据库之前

在镜像创建期间将 `ORACLE_HOME` 转换为只读，会强制从该镜像运行的每个容器都使用只读主目录。将此选项留到运行时则为我们的容器提供了更大的灵活性，这也是这里采用的解决方案。

正如您可能猜到的那样，我们将通过检查运行时环境变量来控制是否转换主目录。我使用 `ROOH`：

```bash
# Enable read-only Oracle Home:
if [ "${ROOH^^}" = "ENABLE" ]
then $ORACLE_HOME/bin/roohctl -enable
fi
```

此命令从环境中读取 `ROOH` 的值，并使用 bash 变量扩展（`^^` 字符）将其转换为大写。当结果匹配 `ENABLE` 时，脚本将主目录转换为只读。将此命令放在 `createDB.sh` 脚本中，在文件开头附近，在任何运行 Database Configuration Assistant 的操作之前。（我将其放在检查 `INIT_SGA_SIZE` 和 `INIT_PGA_SIZE` 之前。）

#### 解析配置目录

我们将使用 Oracle 新增的两个用于报告配置目录的二进制文件：`orabaseconfig` 和 `orabasehome`。在读写主目录中，它们默认为 `ORACLE_HOME`，但在只读主目录中，它们会报告其在 `ORACLE_BASE` 下的新位置。无论主目录类型是只读还是读写，我们都使用这些命令在脚本中为 `ORACLE_BASE_CONFIG` 和 `ORACLE_BASE_HOME` 分配正确的值。由于我们将重复执行此操作，我选择创建一个函数：

```bash
function setOracleBaseDirs {
export ORACLE_BASE_CONFIG="$($ORACLE_HOME/bin/orabaseconfig)"
export ORACLE_BASE_HOME="$($ORACLE_HOME/bin/orabasehome)"
}
```

该函数应出现在 `createDB.sh` 和 `runOracle.sh` 脚本的顶部，在任何调用 `setOracleBaseDirs` 来填充配置目录的其他函数之前。


#### 更新脚本

现有仓库仅预设了一个可读写的主目录，并依赖“旧”的默认路径 `ORACLE_HOME` 来存放其所有配置文件。我们需要将它们更改为使用新的引用路径：

*   `$ORACLE_HOME/dbs` 更改为 `$ORACLE_BASE_CONFIG/dbs`。
*   `$ORACLE_HOME/network/admin` 更改为 `$ORACLE_BASE_HOME/network/admin`。

在 `createDB.sh` 中，这些路径出现在两个函数里：`setupNetworkConfig` 和 `setupTnsnames`。代码清单 13-5 展示了我所做的修改，在每个函数的开始调用 `setOracleBaseDirs` 并使用新的变量。

```
############## Set the ORACLE_BASE_CONFIG and ORACLE_BASE_HOME variables ##############
function setOracleBaseDirs {
export ORACLE_BASE_CONFIG="$($ORACLE_HOME/bin/orabaseconfig)"
export ORACLE_BASE_HOME="$($ORACLE_HOME/bin/orabasehome)"
}
############## Setting up network related config files (sqlnet.ora, listener.ora) ##############
function setupNetworkConfig {
setOracleBaseDirs
mkdir -p "$ORACLE_BASE_HOME"/network/admin
# sqlnet.ora
echo "NAMES.DIRECTORY_PATH= (TNSNAMES, EZCONNECT, HOSTNAME)" > "$ORACLE_BASE_HOME"/network/admin/sqlnet.ora
# listener.ora
echo "LISTENER =
(DESCRIPTION_LIST =
(DESCRIPTION =
(ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1))
(ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
)
)
DEDICATED_THROUGH_BROKER_LISTENER=ON
DIAG_ADR_ENABLED = off
" > "$ORACLE_BASE_HOME"/network/admin/listener.ora
}
####################### Setting up tnsnames.ora ###########################
function setupTnsnames {
setOracleBaseDirs
mkdir -p "$ORACLE_BASE_HOME"/network/admin
# tnsnames.ora
echo "$ORACLE_SID=localhost:1521/$ORACLE_SID" > "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora
echo "$ORACLE_PDB=
(DESCRIPTION =
(ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
(CONNECT_DATA =
(SERVER = DEDICATED)
(SERVICE_NAME = $ORACLE_PDB)
)
)" >> "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora
}
Listing 13-5
`setOracleBaseDirs` 函数为变量分配适当的值。它在两个网络设置函数中被使用，对 `ORACLE_HOME/network/admin` 的引用更改为 `ORACLE_BASE_HOME/network/admin`，以适配只读主目录。
```

我对代码清单 13-6 中的 `runOracle.sh` 进行了类似的修改，添加了新函数并更新了三个辅助函数 `moveFiles`、`symLinkFiles` 和 `undoSymLinkFiles` 中对 `ORACLE_HOME/dbs` 和 `ORACLE_HOME/network/admin` 的引用。

```
############## Set the ORACLE_BASE_CONFIG and ORACLE_BASE_HOME variables ##############
function setOracleBaseDirs {
export ORACLE_BASE_CONFIG="$($ORACLE_HOME/bin/orabaseconfig)"
export ORACLE_BASE_HOME="$($ORACLE_HOME/bin/orabasehome)"
}
########### Move DB files ############
function moveFiles {
setOracleBaseDirs
if [ ! -d "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID" ]; then
mkdir -p "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
fi;
mv "$ORACLE_BASE_CONFIG"/dbs/spfile"$ORACLE_SID".ora "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
mv "$ORACLE_BASE_CONFIG"/dbs/orapw"$ORACLE_SID" "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
mv "$ORACLE_BASE_HOME"/network/admin/sqlnet.ora "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
mv "$ORACLE_BASE_HOME"/network/admin/listener.ora "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
mv "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
mv "$ORACLE_HOME"/install/.docker_* "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
# oracle user does not have permissions in /etc, hence cp and not mv
cp /etc/oratab "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
symLinkFiles;
}
########### Symbolic link DB files ############
function symLinkFiles {
setOracleBaseDirs
if [ ! -L "$ORACLE_BASE_CONFIG"/dbs/spfile"$ORACLE_SID".ora ]; then
ln -s "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/spfile"$ORACLE_SID".ora "$ORACLE_BASE_CONFIG"/dbs/spfile"$ORACLE_SID".ora
fi;
if [ ! -L "$ORACLE_BASE_CONFIG"/dbs/orapw"$ORACLE_SID" ]; then
ln -s "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/orapw"$ORACLE_SID" "$ORACLE_BASE_CONFIG"/dbs/orapw"$ORACLE_SID"
fi;
if [ ! -L "$ORACLE_BASE_HOME"/network/admin/sqlnet.ora ]; then
ln -s "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/sqlnet.ora "$ORACLE_BASE_HOME"/network/admin/sqlnet.ora
fi;
if [ ! -L "$ORACLE_BASE_HOME"/network/admin/listener.ora ]; then
ln -s "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/listener.ora "$ORACLE_BASE_HOME"/network/admin/listener.ora
fi;
if [ ! -L "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora ]; then
ln -s "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/tnsnames.ora "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora
fi;
# oracle user does not have permissions in /etc, hence cp and not ln
cp "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/oratab /etc/oratab
}
########### Undoing the symbolic links ############
function undoSymLinkFiles {
setOracleBaseDirs
if [ -L "$ORACLE_BASE_CONFIG"/dbs/spfile"$ORACLE_SID".ora ]; then
rm "$ORACLE_BASE_CONFIG"/dbs/spfile"$ORACLE_SID".ora
fi;
if [ -L "$ORACLE_BASE_CONFIG"/dbs/orapw"$ORACLE_SID" ]; then
rm "$ORACLE_BASE_CONFIG"/dbs/orapw"$ORACLE_SID"
fi;
if [ -L "$ORACLE_BASE_HOME"/network/admin/sqlnet.ora ]; then
rm "$ORACLE_BASE_HOME"/network/admin/sqlnet.ora
fi;
if [ -L "$ORACLE_BASE_HOME"/network/admin/listener.ora ]; then
rm "$ORACLE_BASE_HOME"/network/admin/listener.ora
fi;
if [ -L "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora ]; then
rm "$ORACLE_BASE_HOME"/network/admin/tnsnames.ora
fi;
}
Listing 13-6
更新 `runOracle.sh` 脚本以处理只读和可读写主目录中的变量目录路径
```


#### 使用只读主目录运行容器

前面的脚本更改仅存在于仓库中。我们需要重建或重新创建镜像，才能使修改后的脚本对新容器可用。导航到 `docker-images/OracleDatabase/SingleInstance/dockerfiles` 目录并运行 `buildContainerImage.sh` 脚本，以使用新的脚本版本重建镜像：

```
./buildContainerImage.sh -v 19.3.0 -e
```

构建完成后，运行一个新容器并添加 `ROOH=ENABLE` 环境变量以在容器中创建一个只读主目录：

```
docker run -d --name ROOH \
-e ROOH=ENABLE \
oracle/database:19.3.0-ee
```

然后，通过 `docker logs -f ROOH` 尾随容器日志来报告输出。日志的首次输出显示只读主目录及其资产的创建：

```
Enabling Read-Only Oracle home.
Update orabasetab file to enable Read-Only Oracle home.
Orabasetab file has been updated successfully.
Create bootstrap directories for Read-Only Oracle home.
Bootstrap directories have been created successfully.
Bootstrap files have been processed successfully.
Read-Only Oracle home has been enabled successfully.
Check the log file /opt/oracle/cfgtoollogs/roohctl/roohctl-220821PM123432.log for more details.
```

接着是熟悉的监听器启动消息，但有所不同！它引用了位于新位置 `/opt/oracle/homes/OraDB19Home1/network/admin` 下的 `listener.ora`，该位置在 `ORACLE_BASE_HOME` 下：

```
LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 21-AUG-2022 12:34:33
Copyright (c) 1991, 2019, Oracle.  All rights reserved.
Starting /opt/oracle/product/19c/dbhome_1/bin/tnslsnr: please wait...
TNSLSNR for Linux: Version 19.0.0.0.0 - Production
System parameter file is /opt/oracle/homes/OraDB19Home1/network/admin/listener.ora
Log messages written to /opt/oracle/diag/tnslsnr/d06ab6991050/listener/alert/log.xml
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1)))
Listening on: (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1)))
STATUS of the LISTENER

Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                21-AUG-2022 12:34:46
Uptime                    0 days 0 hr. 0 min. 13 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /opt/oracle/homes/OraDB19Home1/network/admin/listener.ora
Listener Log File         /opt/oracle/diag/tnslsnr/d06ab6991050/listener/alert/log.xml
Listening Endpoints Summary...
(DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1)))
(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
The listener supports no services
The command completed successfully
```

登录容器并查看 `$ORACLE_HOME/install/orabasetab` 的内容。指示主目录是否为只读的第四个字段在新容器中为 `Y`：

```
#orabasetab file is used to track Oracle Home associated with Oracle Base
/opt/oracle/product/19c/dbhome_1:/opt/oracle:OraDB19Home1:Y:
```

在容器的 `$ORACLE_BASE` 目录中，你会看到两个新目录 `dbs` 和 `homes`。一旦数据库创建完成，`runOracle.sh` 脚本会将配置文件移动到 `/opt/oracle/oradata/dbconfig` 并添加链接：

```
bash-4.2$ ls -l $ORACLE_BASE/dbs
total 12
-rw-rw---- 1 oracle oinstall 1544 Jan 21 15:57 hc_ORCLCDB.dat
-rw-r----- 1 oracle oinstall   43 Jan 21 15:56 initORCLCDB.ora
-rw-r----- 1 oracle oinstall   24 Jan 21 14:43 lkORCLCDB
lrwxrwxrwx 1 oracle oinstall   49 Jan 21 15:58 orapwORCLCDB -> /opt/oracle/oradata/dbconfig/ORCLCDB/orapwORCLCDB
lrwxrwxrwx 1 oracle oinstall   54 Jan 21 15:58 spfileORCLCDB.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/spfileORCLCDB.ora
bash-4.2$ ls -l $ORACLE_BASE/homes/OraDB19Home1/network/admin
total 0
lrwxrwxrwx 1 oracle oinstall 49 Jan 21 15:58 listener.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/listener.ora
lrwxrwxrwx 1 oracle oinstall 47 Jan 21 15:58 sqlnet.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/sqlnet.ora
lrwxrwxrwx 1 oracle oinstall 49 Jan 21 15:58 tnsnames.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/tnsnames.ora
bash-4.2$ ls -l /opt/oracle/oradata/dbconfig/ORCLCDB/
total 24
-rw-r--r-- 1 oracle oinstall  234 Jan 21 14:38 listener.ora
-rw-r----- 1 oracle oinstall 2048 Jan 21 14:47 orapwORCLCDB
-rw-r--r-- 1 oracle oinstall  780 Jan 21 15:58 oratab
-rw-r----- 1 oracle oinstall 3584 Jan 21 16:17 spfileORCLCDB.ora
-rw-r--r-- 1 oracle oinstall   54 Jan 21 14:38 sqlnet.ora
-rw-r----- 1 oracle oinstall  197 Jan 21 15:58 tnsnames.ora
```

如果我在创建容器时不设置 `ROOH` 值，这些文件会出现在它们预期的位置：

```
bash-4.2$ ls -l $ORACLE_HOME/dbs
total 12
-rw-rw---- 1 oracle oinstall 1544 Jan 21 17:42 hc_ORCLCDB.dat
-rw-r--r-- 1 oracle dba      3079 May 14  2015 init.ora
-rw-r----- 1 oracle oinstall   24 Jan 21 16:23 lkORCLCDB
lrwxrwxrwx 1 oracle oinstall   49 Jan 21 17:43 orapwORCLCDB -> /opt/oracle/oradata/dbconfig/ORCLCDB/orapwORCLCDB
lrwxrwxrwx 1 oracle oinstall   54 Jan 21 17:43 spfileORCLCDB.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/spfileORCLCDB.ora
bash-4.2$ ls -l $ORACLE_HOME/network/admin
total 8
lrwxrwxrwx 1 oracle oinstall   49 Jan 21 17:43 listener.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/listener.ora
drwxr-xr-x 2 oracle dba      4096 Apr 17  2019 samples
-rw-r--r-- 1 oracle dba      1536 Feb 14  2018 shrept.lst
lrwxrwxrwx 1 oracle oinstall   47 Jan 21 17:43 sqlnet.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/sqlnet.ora
lrwxrwxrwx 1 oracle oinstall   49 Jan 21 17:43 tnsnames.ora -> /opt/oracle/oradata/dbconfig/ORCLCDB/tnsnames.ora
bash-4.2$ ls -l /opt/oracle/oradata/dbconfig/ORCLCDB
total 24
-rw-r--r-- 1 oracle oinstall  234 Jan 21 16:19 listener.ora
-rw-r----- 1 oracle oinstall 2048 Jan 21 16:28 orapwORCLCDB
-rw-r--r-- 1 oracle oinstall  780 Jan 21 17:43 oratab
-rw-r----- 1 oracle oinstall 3584 Jan 21 17:46 spfileORCLCDB.ora
-rw-r--r-- 1 oracle oinstall   54 Jan 21 16:19 sqlnet.ora
-rw-r----- 1 oracle oinstall  197 Jan 21 17:43 tnsnames.ora
```

#### 镜像定制脚本

我运行的是用于不同环境的混合镜像。我的交互式镜像包含我用于查看和编辑文件、执行测试以及收集诊断信息的工具。它们还将环境设置得熟悉且用户友好，包括我最喜欢的命令行提示符、简化导航的别名和快捷方式，以及一个用于定制我的 SQL*Plus 体验的 `login.sql` 文件。我的生产镜像遵循容器原则，旨在最小化占用空间、减少攻击面并消除不必要的包和库。我可以管理专用于不同目的的独立仓库，但这意味着要管理两套代码，增加测试和验证更改的工作量，并引入更多人为错误的机会。通过构建参数，单个仓库可以支持这两个镜像类别。

管理镜像内容存在多种选项，包括：

*   **具有条件操作的脚本：** 当选项有限、定义明确且不会随时间发生重大变化时很有用。
*   **追加或替换值：** 最适用于预先未明确定义的动态输入。
*   **有条件地复制文件：** 适用于广泛、复杂或特定于环境的、相对静态的修改。

以下方法建议应用这些技术，从单个仓库生成定制镜像。

### 条件操作

支持多种结果并减少代码量可以降低维护开销。向脚本添加条件逻辑是实现此目标的策略之一，当需要在有限选项间做出决定且每个选项都有明确操作时，这种方法效果最佳。条件的来源可以是传递给 `docker build` 命令的参数，也可以从环境中获取。

本书一直使用的公共仓库为每个数据库版本（11.2.0.2、12.1.0.2、12.2.0.1、18.3.0、18.4.0、19.3.0 和 21.3.0）都设有单独的目录。这些目录下的脚本结构相同：有些完全一致，有些在格式和风格上略有差异，还有少数引入了特定于版本的功能。例如，如果我运行 `diff` 命令来显示 `installDBBinaries.sh` 的 19c 和 21c 版本之间的差异，会看到：
```
> diff 19.3.0/installDBBinaries.sh 21.3.0/installDBBinaries.sh
24c24
 if [ "$EDITION" != "EE" ] && [ "$EDITION" != "SE2" ]; then
```
除了 `if` 语句的一个细微差别外，它们完全相同。其他地方则有反映代码改进的差异，例如 `runOracle.sh` 脚本中的这个检查：
```
    if [ -a "$ORACLE_HOME"/install/.docker_* ]; then
>       mv "$ORACLE_HOME"/install/.docker_* "$ORACLE_BASE"/oradata/dbconfig/"$ORACLE_SID"/
>    fi;
```
19c 版本尝试移动匹配 `$ORACLE_HOME/install/.docker_*` 的文件。如果没有文件匹配该模式，它会返回错误。21c 版本将移动命令包装在 `if` 语句中，在尝试移动之前先检查是否存在匹配的文件。21c 版本包含了特定于 Oracle 21c Express 版的操作，但脚本大体相同。将 19c 和 21c 仓库合并到一个公共目录，然后向构建过程传递版本号，这是否可行？

将版本分离到独立目录有充分的理由。在前一章讨论 `build context` 时，我们了解到 Docker 会读取构建路径下的所有文件。将 19c 和 21c 合并在一个 Dockerfile 中意味着将两个版本的数据库安装介质存储在同一目录下。这些文件很大，同时放在一个目录中会增加上下文大小和处理时间。

独立目录的缺点在于维护。文件差异大多很小，但 21c 版本中的每一项改进并未都同步到其他版本。随着支持版本数量的增加，跟踪和测试不断扩大的代码库中的修复与改进的需求也在增长。

构建特性（见第 14 章）可以解决上下文问题。这个问题解决后，考虑通过构建参数将版本传递给 Dockerfile 就更加合理。该版本随后可以控制构建过程的独特方面，特别是：

*   在 `setupLinuxEnv.sh` 中应用正确的预安装 RPM 包
*   执行为 21c 启用 Express 版所需的额外检查
*   使用正确的数据库版本和版本标签标记镜像

需要明确的是：将现有脚本转换为处理多个版本是一项复杂的任务，具体取决于你的目标，这只是说明为何可能在设置脚本中嵌入条件检查的一个例子。因此，列出具体的改动并不实际，但我们可以探索说明这种实践的技术。例如，Linux 的 `case` 语句提供了一种非常便捷的方法，可根据版本确定正确的包集合：
```
case $DB_VERSION in
11*)   RPM_LIST="oracle-rdbms-server-11gR2-preinstall unzip" ;;
12.1*) RPM_LIST="oracle-rdbms-server-12cR1-preinstall tar" ;;
12.2*) RPM_LIST="oracle-database-server-12cR2-preinstall" ;;
18*)   RPM_LIST="oracle-database-preinstall-18c" ;;
19*)   RPM_LIST="oracle-database-preinstall-19c" ;;
21*)   RPM_LIST="oracle-database-preinstall-21c" ;;
*)     exit 1 ;;
esac
```
然后，将 `$RPM_LIST` 变量传递给 `yum install` 命令：^(⁷⁵)
```
yum -y install "$RPM_LIST" openssl
```
我想强调，两种方法都非“对”或“错”。编程是一门艺术，团队会选择最适合其风格、背景和目的的方法。在我的实践中，我倾向于为每个版本使用一套脚本，但对于 Oracle 的容器镜像，我承认独立目录更好。代码库向新用户介绍了容器，为每个版本设置一个目录更容易理解。随着你对容器越来越熟悉并扩展镜像库，将版本化构建整合到统一的目录结构下可能会是更好的选择。关于此方法的更多示例（或许还有灵感）可在我的 GitHub 上找到：[`https://github.com/oraclesean/docker-oracle`](https://github.com/oraclesean/docker-oracle)。

## 在 Dockerfile 中追加上一节中的 `yum install` 命令引出了修改镜像的第二个机会，这次与预期环境的性质有关。生产镜像应该精简，不包含可能增加镜像大小或引入漏洞的不必要包。用于实验的镜像有不同的要求，包括编辑器和诊断工具，这些会因需求和个人喜好而异。我喜欢 `vi`；其他人喜欢 `emacs`。你可能希望包含 `tree` 或 `strace`。这里的重点不在于需要安装 `what` 包，而在于列表的可变性质，要解决的挑战是如何最好地包含它们。

一种方法是以 `root` 用户身份登录正在运行的容器：
```
docker exec -it -u root  bash
```
然后 `yum install` 包：
```
# yum -y install vi less tree which
```
这种方法有以下缺点：

*   其手动性质。我必须在每个容器中执行这些步骤。
*   它是需要额外包的活动的先决条件，而且很容易被忘记，直到你发现缺少某个东西。
*   安装会更改容器覆盖文件系统中的文件，增加了容器的大小。

对于独特的添加项，采取这条路线是有意义的。仅仅因为不想向容器添加几兆字节或多运行一个命令，就花费几分钟构建一个包含“这次你需要的那个包”的自定义镜像，这并不实际，特别是当该容器不会存在很长时间时。

当这些工具经常出现在镜像中时，通过构建参数将它们集成到镜像本身中更有意义。

与 `docker run` 中使用的环境变量不同，我们不能“临时编造”构建参数。它们必须通过 Dockerfile 中的 `ARG` 指令来定义。然而，它们不需要值。让我们在 Dockerfile 中创建一个新参数来接受要安装到镜像中的可选包列表：
```
ARG RPM_SUPPLEMENT=""
```
它应该与传递给第一个构建阶段的其他参数一起出现。它是运行 `$INSTALL_DIR/$SETUP_LINUX_FILE` 步骤的阶段。无需通过关联的 `ENV` 命令将其添加到环境中。参数在阶段的上下文中可用，如果在后续步骤中不需要，则不必提升到容器的环境中。

接下来，在 `setupLinuxEnv.sh` 脚本中，更新 `yum install` 命令以包含新参数：
```
yum -y install oracle-database-preinstall-19c openssl $RPM_SUPPLEMENT
```
如果我们没有向构建参数传递任何值，Linux 设置脚本会运行，并在 `yum` 命令末尾附加一个空字符串。但是，如果我们给构建参数传递一个包列表：^(⁷⁶)
```
docker build ...
--build-arg RPM_SUPPLEMENT="vi less tree which"
...
```
`yum` 命令会在变量中看到自定义参数，并安装额外的包。

### 条件文件复制

### 脚本逻辑与安全考量

本节第一部分提出的修改，针对的是脚本使用参数进行 `if-then` 或基于 `case` 的决策的场景。脚本必须内嵌评估条件和根据条件采取行动的逻辑。该部分的例子考虑了将两个或多个版本的脚本整合到单一目录结构中。仓库中许多辅助文件是相同的或足够相似，可以在不同版本间互换，因此将它们合并是有意义的。但其他文件——包括数据库安装脚本——包含针对特定条件的独特步骤。随着 Oracle 推出新版本，对数据库安装采用“一个脚本统治所有”的方法可能并不实用。为每个版本（从 11g 到 21c 及以后）适应 Express、Standard 和 Enterprise 版本之间的差异是一项复杂的任务。针对所有组合测试更新会迅速变得难以掌控。

还有一个安全方面需要考虑。查看 Oracle 数据库容器中 `$ORACLE_BASE` 目录下的文件：

```
/opt/oracle/runOracle.sh
/opt/oracle/createObserver.sh
/opt/oracle/relinkOracleBinary.sh
/opt/oracle/setPassword.sh
/opt/oracle/checkDBStatus.sh
/opt/oracle/runUserScripts.sh
/opt/oracle/configTcps.sh
/opt/oracle/createDB.sh
/opt/oracle/dbca.rsp.tmpl
/opt/oracle/startDB.sh
```

嵌入在脚本中的逻辑揭示了在不同条件下系统如何工作，并可能暴露攻击者可以利用你的数据库基础设施的方式。能够访问一个镜像或一个不安全或“不重要”容器的人可以推断配置，甚至反向工程你环境中的其他容器。

### Dockerfile 示例

也许存在一个折中方案——一组适用于任何版本的共享文件，以及没有过于复杂或泄露信息的条件逻辑的专用文件。为了想象这是如何工作的，让我们看一下 19c Dockerfile 的顶部部分，重点关注清单 13-7 中的参数和环境设置。我通过删除一些对本练习不重要的特殊环境变量来简化它。

```
# Argument to control removal of components not needed after db software installation
ARG SLIMMING=true
ARG INSTALL_FILE_1="LINUX.X64_193000_db_home.zip"
# Environment variables required for this build (do NOT change)
# -------------------------------------------------------------
ENV ORACLE_BASE=/opt/oracle \
ORACLE_HOME=/opt/oracle/product/19c/dbhome_1 \
INSTALL_DIR=/opt/install \
INSTALL_FILE_1=$INSTALL_FILE_1 \
INSTALL_RSP="db_inst.rsp" \
CONFIG_RSP="dbca.rsp.tmpl" \
PWD_FILE="setPassword.sh" \
RUN_FILE="runOracle.sh" \
START_FILE="startDB.sh" \
CREATE_DB_FILE="createDB.sh" \
SETUP_LINUX_FILE="setupLinuxEnv.sh" \
INSTALL_DB_BINARIES_FILE="installDBBinaries.sh" \
# Directory for keeping Oracle Wallet
WALLET_DIR=""
```
清单 13-7
一个 Oracle 19c Dockerfile 的部分内容，展示了参数和环境指令。环境设置侧重于与文件和目录相关的部分。

为了说明问题，假设 `runOracle.sh` 和 `createDB.sh` 脚本在不同数据库版本之间差异很大，将它们的逻辑合并既不实际也不可取。响应文件也不可互换。在清单 13-8 中，我添加了一个数据库版本参数，并将其纳入现有环境中。

```
# Argument to control removal of components not needed after db software installation
ARG SLIMMING=true
ARG INSTALL_FILE_1="LINUX.X64_193000_db_home.zip"
ARG DB_VERSION=19.3.0
# Environment variables required for this build (do NOT change)
# -------------------------------------------------------------
ENV ORACLE_BASE=/opt/oracle \
ORACLE_HOME=/opt/oracle/product/19c/dbhome_1 \
INSTALL_DIR=/opt/install \
INSTALL_FILE_1=$INSTALL_FILE_1 \
INSTALL_RSP="db_inst.${DB_VERSION}.rsp" \
CONFIG_RSP="dbca.rsp.${DB_VERSION}.tmpl" \
PWD_FILE="setPassword.sh" \
RUN_FILE="runOracle.${DB_VERSION}.sh" \
START_FILE="startDB.sh" \
CREATE_DB_FILE="createDB.${DB_VERSION}.sh" \
SETUP_LINUX_FILE="setupLinuxEnv.sh" \
INSTALL_DB_BINARIES_FILE="installDBBinaries.sh" \
# Directory for keeping Oracle Wallet
WALLET_DIR=""
```
清单 13-8
一个修改后的 Dockerfile，它通过一个参数接受数据库版本，并将其插入到环境中定义的选定脚本和模板的名称中。

该 Dockerfile 支持多个版本，而无需向脚本本身添加复杂逻辑。所有版本共有的文件被共享。在必要时替换专用脚本。同样的技术也有助于解决环境差异。用于设置生产系统外观的 `.bashrc` 文件可能与我在实验性容器中使用的有很大不同。19c 和 21c 的 Dockerfile 向容器的 `.bashrc` 文件添加了条目：

```
RUN echo 'ORACLE_SID=${ORACLE_SID:-ORCLCDB}; export ORACLE_SID=${ORACLE_SID^^}' > .bashrc
```

我们可以扩展该命令以包含提示符的自定义：

```
export PS1="[\u - \${ORACLE_SID}] \w\n# "
```

然而，我们现在进入了棘手的领域。尝试用 `echo` 重现这个字符串：

```
# Without quotes: backslash before the dollar sign and trailing whitespace are lost.
echo export PS1="[\u - \${ORACLE_SID}] \w\n# "
export PS1=[\u - ${ORACLE_SID}] \w\n#
# With double quotes, escaping embedded quotes: backslash is lost.
echo "export PS1=\"[\u - \${ORACLE_SID}] \w\n# \""
export PS1="[\u - ${ORACLE_SID}] \w\n# "
# Double-escaped the backslash: works!
echo "export PS1=\"[\u - \\\${ORACLE_SID}] \w\n# \""
export PS1="[\u - \${ORACLE_SID}] \w\n# "
# Single quotes prevent bash from evaluating the string: works!
echo 'export PS1="[\u - \${ORACLE_SID}] \w\n# "'
export PS1="[\u - \${ORACLE_SID}] \w\n# "
```

Bash shell 有一些敏感性和怪癖，这并不总是显而易见的。弄清楚如何让它打印包含特殊字符的字符串可能会令人沮丧！另一种方法是提前为每个环境编写专用文件，然后将它们复制到镜像中。一个直截了当的生产文件 `bashrc.prod`：

```
ORACLE_SID=${ORACLE_SID:-ORCLCDB}
export ORACLE_SID=${ORACLE_SID^^}
```

以及一个用于测试的文件 `bashrc.test`，它设置了 `ORACLE_SID` 并添加了提示符和其他环境变量的设置：

```
ORACLE_SID=${ORACLE_SID:-ORCLCDB}
export ORACLE_SID=${ORACLE_SID^^}
export PS1="[\u - \${ORACLE_SID}] \w\n# "
export ORACLE_BASE_CONFIG="$($ORACLE_HOME/bin/orabaseconfig 2>/dev/null || echo $ORACLE_HOME)"
export ORACLE_BASE_HOME="$($ORACLE_HOME/bin/orabasehome 2>/dev/null || echo $ORACLE_HOME)"
export TNS_ADMIN=$ORACLE_BASE_HOME/network/admin
export ORACLE_PATH=/home/oracle
```

在下面来自 19c Dockerfile 的片段中，我修改了最终的构建阶段以接受一个参数，并将 `RUN` 命令替换为 `COPY`，以将正确的文件插入镜像：

```
#############################################
# -------------------------------------------
# Start new layer for database runtime
# -------------------------------------------
#############################################
FROM base
ARG ENV=prod
USER oracle
COPY --chown=oracle:dba --from=builder $ORACLE_BASE $ORACLE_BASE
USER root
RUN $ORACLE_BASE/oraInventory/orainstRoot.sh && \
$ORACLE_HOME/root.sh
USER oracle
WORKDIR /home/oracle
# Add an environment-specific bashrc file to the image
COPY bashrc.$ENV /home/oracle/.bashrc
```

无论是处理脚本还是配置文件，使用包含镜像中所需的确切代码或文本的文件可能更容易（并且可能更不易出错和令人沮丧）！


### 摘要

随着你处理现成容器镜像的经验日益丰富，你会发现一些你喜欢的东西，以及一些你想要改变的东西。这是学习曲线的一部分，而稳健、容错的工具有助于建立下一步所需的信心和技能。无论是第一辆公路自行车还是一项新技术，持续的练习能形成肌肉记忆和熟悉感。一旦使用工具变得得心应手，你的思维就能完全专注于更清晰地构想你想做的事情，并借此理解你的工具可能如何限制你的进展。

*Breaking Away* 中的四个朋友在寻找自己独特身份的过程中，努力对抗父母和社会的期望。虽然他们无法改变自己是谁或来自哪里，但他们意识到，出身并不决定未来的方向。这对于我们全书所使用的镜像仓库来说也是如此。它们是塑造我们理解如何在容器中运行数据库的基础，但并不阻止我们大胆梦想并规划自己的路线。

本章介绍了一些帮助你迈出下一步处理容器的“配方”。它们解决了你在使用数据库镜像时可能遇到的问题，但它们也具有适应其他需求的灵活性。这就是“配方”的美妙之处——无论是在烹饪还是编码中，它都是艺术与科学的结合。一个问题很少只有一个答案，然而解决一个问题的方法或模式通常也适用于其他问题。我希望本章的示例能激发你的想象力，为你在 Docker 之旅中遇到的挑战构想解决方案！下一章将更深入地探讨 Docker 如何构建镜像，着眼于编写高效、有效的 Dockerfiles。

脚注 1   2   3   4   5   6   7

## 14. 构建镜像

生活中有些奥秘我不需要（甚至不想）去理解。不去了解能为我保留一丝对周遭世界的惊奇感。缝纫机就是其中之一。我会手工缝纫，虽然没什么天赋，但我从未弄明白机器是如何复制这种劳作的。即使能轻易上网找到慢动作、逐针专家解说的 YouTube 视频来揭示整个过程，我也一直抗拒学习缝纫机的基本原理。机器如何将针部分刺入布料，然后从下方固定每一针，这对我来说超出了理解范围。我明明知道得更多，但我能想象到的最佳解释就是机器内部住着一群小精灵，正疯狂地将梭芯线穿过每一针。我满足于不知情——它滋养了我对日常事物的迷恋与欣赏，并提醒我身边充满了睿智而富有创造力的人！或者，魔法真的存在！

有些便利我简直习以为常。细想围绕我的技术，我明白我的笔记本电脑使用 DHCP 来获取网络地址。我不知道细节，但似乎如果有必要我能够弄清楚，而且它不像缝纫机那样充满神秘感。它让加入网络变得容易，并免去了在酒店 WiFi 上猜测可用 IP 地址的麻烦。

自动化和脚本也属于类似范畴。它们让日常任务的执行变得更快、更容易，并让我们“忘记”细节。（坦白说，有时我看代码会想，自己当初怎么那么天真或那么灵感迸发！）将知识编码转化为工具，可以与他人分享，并作为更大、更好事物的构建块。在第 4 章，`buildContainerImage.sh` 脚本正是这样做的。你给它一个数据库版本和版本，它就在后台为你构建并运行 `docker build` 命令。

要获得对镜像创建的更大控制权，意味着需要亲自动手处理构建过程。即使你的计划不包括编写自定义 Dockerfiles，理解构建命令的元素也能让你更好地欣赏幕后发生的事情，并理解其中的奥妙！而且我保证，构建镜像远没有缝纫机那么神秘，也不需要电脑内部有一支小精灵劳作大军！

### 构建命令语法

`docker build` 可能是 Docker 词汇中最直白的命令。其最简形式是有效的：

```
docker build .
```

这看起来似乎没什么！这么简单的东西能做什么呢？这一切都与上下文有关！



#### 上下文

构建镜像需要 *Dockerfile*（指令或配方）和 *构建上下文*（文件、资源或素材）。上下文是指构建运行所在目录及其下的所有内容。Oracle 数据库构建包含软件安装介质、用于管理构建的脚本（准备操作系统和安装数据库）以及容器运行时使用的脚本（创建数据库、启动数据库、报告健康状况等）。

在之前的构建命令中，点（.）代表上下文。这个点或句号是 Linux 中的一个简写字符，指向当前目录^⁷⁷。Docker 使用它在那里找到的任何文件来构建镜像。构建也需要一个 `Dockerfile`，如果未明确指定，Docker 会在上下文中查找名为 “Dockerfile” 的文件。

为了更好地理解上下文，我在一个新目录中创建了一个简单的 `Dockerfile`，如清单 14-1 所示。它以 `alpine` 镜像开始，这是一个在容器中流行的极简 Linux 发行版。`Dockerfile` 运行一个 `COPY` 指令。第一个参数，星号通配符，将所有文件从构建上下文（本地目录）复制到目标位置，即容器的根目录（用斜杠表示）。

```dockerfile
FROM alpine
COPY * /
CMD ls -l /
```

清单 14-1
用于演示上下文的简单 `Dockerfile`

最后的 `CMD` 指令在执行镜像时运行，并列出容器根目录的内容。清单 14-2 包含了运行上述简单命令所产生的输出。

```console
> docker build .
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM alpine
latest: Pulling from library/alpine
213ec9aee27d: Pull complete
Digest: sha256:bc41182d7ef5ffc53a40b044e725193bc10142a1243f395ee852a8d9730fc2ad
Status: Downloaded newer image for alpine:latest
---> 9c6f07244728
Step 2/3 : COPY * /
---> 1d5ae09bcac7
Step 3/3 : CMD ls -l /
---> Running in 7aab4d9ea957
Removing intermediate container 7aab4d9ea957
---> 8cf63acc1f17
Successfully built 8cf63acc1f17
```

清单 14-2
为清单 14-1 所示的 `Dockerfile` 运行 `docker build` 时生成的输出

来看一下输出：

*   “Sending build context to Docker daemon”，读取了上下文（本地目录）中的所有文件并将它们发送到 Docker 守护进程。只有在这之后，它才开始读取 `Dockerfile` 并执行工作。
*   `Dockerfile` 的第一步指示 Docker 拉取 `alpine` 镜像。它将其添加到一个新创建的层 `9c6f07244728`。（每层都以三个破折号和一个右尖括号 `--->` 为前缀。）
*   第二步将文件复制到另一个新层 `1d5ae09bcac7`。
*   第三步在一个中间容器 `7aab4d9ea957` 中运行 `ls -l /` 命令。
*   移除中间容器后，Docker 创建了最后一层 `8cf63acc1f17`。

运行 `docker images` 报告新创建的镜像以及被 `FROM` 指令拉取的 `alpine` 镜像：

```console
> docker images
REPOSITORY        TAG         IMAGE ID       CREATED          SIZE
                  8cf63acc1f17   21 seconds ago   5.54MB
alpine            latest      9c6f07244728   2 weeks ago      5.54MB
oracle/database   19.3.0-ee   1b588736c8c1   2 months ago     6.67GB
```

新容器的镜像 ID 与镜像列表中显示的一致。注意该镜像没有仓库或标签。管理或报告镜像信息、或将镜像作为容器运行的命令，必须使用其 ID（暂时）。让我们将镜像 ID 传递给 `docker history` 命令以查看镜像的各层：

```console
> docker history 8cf63acc1f17
IMAGE          CREATED         CREATED BY                                SIZE   COMMENT
8cf63acc1f17   2 minutes ago   /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "ls -...   0B
1d5ae09bcac7   2 minutes ago   /bin/sh -c #(nop) COPY file:a45abb589c97bd2f...   34B
9c6f07244728   2 weeks ago     /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
      2 weeks ago     /bin/sh -c #(nop) ADD file:2a949686d9886ac7c...   5.54MB
```

历史记录输出从最新的顶部到最旧的底部读取。注意各层的 ID 值与 `docker build` 输出中的对应。其中一层 `1d5ae09bcac7` 的大小为 34 字节，与本目录中 `Dockerfile` 的大小相同：

```console
> ls -l
total 8
-rw-r--r--   1 seanscott  staff    34 Aug 25 10:57 Dockerfile
```

这一层包含一个文件，即通过 `COPY` 命令添加到“基础”alpine 镜像的 `Dockerfile`！

最后一层 `8cf63acc1f17` 是零字节。这是一个元数据层。它不属于镜像，至少容器内的用户无法以某种方式看到它。它在通过 `docker run` 调用镜像时由 Docker 使用。我们可以通过运行该镜像看到这一点：

```console
> docker run 8cf63acc1f17
total 60
-rw-r--r--    1 root     root            34 Aug 25 16:57 Dockerfile
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 bin
drwxr-xr-x    5 root     root           360 Aug 25 17:54 dev
drwxr-xr-x    1 root     root          4096 Aug 25 17:54 etc
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 home
drwxr-xr-x    7 root     root          4096 Aug  9 08:47 lib
drwxr-xr-x    5 root     root          4096 Aug  9 08:47 media
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 mnt
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 opt
dr-xr-xr-x  332 root     root             0 Aug 25 17:54 proc
drwx------    2 root     root          4096 Aug  9 08:47 root
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 run
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 sbin
drwxr-xr-x    2 root     root          4096 Aug  9 08:47 srv
dr-xr-xr-x   13 root     root             0 Aug 25 17:54 sys
drwxrwxrwt    2 root     root          4096 Aug  9 08:47 tmp
drwxr-xr-x    7 root     root          4096 Aug  9 08:47 usr
drwxr-xr-x   12 root     root          4096 Aug  9 08:47 var
>
```

记住，容器执行的功能或服务有点像可执行程序。`CMD` 指令定义了要执行的操作。容器中的其他所有内容——库、包、二进制文件和脚本——只是为了支持其运行时或可执行功能。这解释了为什么与虚拟机或物理主机上的文件系统和操作系统相比，镜像的占用空间通常有限。镜像 `CMD` 指令中固定的操作限制了容器的范围。在这个例子中，`CMD` 运行了一个命令 `ls -l /`。该任务完成后，它退出并将控制权交还给主机命令提示符。

清单 14-2 包含了这些行：

```text
Step 3/3 : CMD ls -l /
---> Running in 7aab4d9ea957
Removing intermediate container 7aab4d9ea957
```

Docker 并没有读取或保存此操作的结果。“中间容器”是 Docker 检查命令语法的地方。

关于层的一个提醒：在这个容器中运行 `ls -l /` 并不是以通常的方式显示容器的文件系统。相反，它显示的是两个层的合并内容：基础 `alpine` 镜像（层 `9c6f07244728`）中的文件系统，以及由 `COPY` 命令创建的层（`1d5ae09bcac7`）中的文件系统。将这些分离成层允许 Docker 在多个容器中重用相同的 `alpine` 层，而无需使用额外空间！




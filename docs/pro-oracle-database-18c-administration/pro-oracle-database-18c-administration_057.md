# 创建与管理 Oracle 恢复目录

## 创建恢复目录

以下步骤概述了创建恢复目录的过程：

### 1. 准备目录数据库
在独立于目标数据库服务器的服务器上创建一个数据库，专门用于恢复目录。确保数据库大小适当；Oracle 推荐的大小通常偏小。以下是初始大小的充分建议：

- `SYSTEM` 表空间：500MB
- `SYSAUX` 表空间：500MB
- `TEMP` 表空间：500MB
- `UNDO` 表空间：500MB
- 在线重做日志：每个 25MB；三组，每组有两个成员，进行多路复用
- `RECCAT` 表空间：500MB

### 2. 创建目录表空间
为恢复目录用户创建一个专用表空间。将其命名为 `RECCAT` 可使其作为恢复目录元数据的容器易于识别。

```sql
SQL> CREATE TABLESPACE reccat
DATAFILE '/u01/dbfile/O12C/reccat01.dbf' SIZE 500M
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 128k
SEGMENT SPACE MANAGEMENT AUTO;
```

### 3. 创建目录所有者用户
创建一个用户来拥有恢复目录对象。一个像 `RCAT` 这样的名称易于识别。授予必要的 `RECOVERY_CATALOG_OWNER` 角色和 `CREATE SESSION` 权限。

```sql
SQL> CREATE USER rcat IDENTIFIED BY foo
TEMPORARY TABLESPACE temp
DEFAULT TABLESPACE reccat
QUOTA UNLIMITED ON reccat;
--
GRANT RECOVERY_CATALOG_OWNER TO rcat;
GRANT CREATE SESSION TO rcat;
```

### 4. 创建目录并验证
通过 `RMAN` 以 `RCAT` 身份连接并创建恢复目录对象。

```sql
$ rman catalog rcat/foo
```

运行 `CREATE CATALOG` 命令。

```sql
RMAN> create catalog;
RMAN> exit;
```

此命令可能需要几分钟。完成后，验证表是否已创建。

```sql
$ sqlplus rcat/foo
SQL> select table_name from user_tables;
```

输出的一小部分示例：

```
TABLE_NAME
------------------------------
DB
NODE
CONF
DBINC
```

## 注册目标数据库
将你的目标数据库注册到恢复目录。确保你可以建立到目录数据库的 Oracle Net 连接，例如通过配置 `TNS_ADMIN/tnsnames.ora` 文件。然后，按如下方式注册：

```sql
$ rman target / catalog rcat/foo@rcat
```

连接后，你应该会看到类似于以下内容的验证消息：

```
connected to target database: O18C (DBID=3423216220)
connected to recovery catalog database
```

运行 `REGISTER DATABASE` 命令：

```sql
RMAN> register database;
```

你现在可以运行备份操作，元数据将同时写入控制文件和恢复目录。请记住每次都要连接到目标和目录：

```sql
$ rman target / catalog rcat/foo@rcat
RMAN> backup database;
```

## 备份恢复目录
确保你包含了备份和恢复恢复目录数据库本身的策略。为了获得最大保护，请将目录数据库运行在 `ARCHIVELOG` 模式下，并使用 `RMAN` 进行备份。
你也可以使用像 Data Pump 这样的工具进行快照。缺点是在 Data Pump 导出之后创建的目录信息可能会丢失。
请注意，如果目录数据库服务器发生故障，你仍然可以使用 `RMAN` 来备份你的目标数据库；你只是无法连接到目录。因此，任何指示 `RMAN` 同时连接到两者的脚本都必须进行修改。
如果你完全丢失了恢复目录且没有备份，你可以从头重新创建它并重新注册目标数据库，但你将丢失所有长期的历史恢复目录元数据。

## 同步恢复目录
如果网络问题导致在你执行备份时目录无法访问，则必须在连接性恢复后重新同步目录。这确保目录知晓任何未存储在其中的操作。

运行以下命令进行同步：

```sql
$ rman target / catalog rcat/foo@rcat
RMAN> resync catalog;
```

仅当你在未连接到目录的情况下执行了备份时才需要重新同步目录。在正常情况下，使用 `RESYNC` 命令是没有必要的。

## 恢复目录版本
为你正在备份的每个目标数据库版本创建一个恢复目录。这可以避免兼容性问题和升级麻烦。当 `rman` 客户端版本与目录版本匹配时，兼容性更顺畅。
拥有多个目录版本可能会导致混淆，但在具有多个不同 Oracle 数据库版本的环境中，通常更为方便。

## 删除恢复目录
如果你不再需要恢复目录，可以将其删除。

### 删除目录对象
以目录所有者身份连接并发出 `DROP CATALOG` 命令：

```sql
$ rman catalog rcat/foo
RMAN> drop catalog;
```

系统将提示你：

```
recovery catalog owner is RCAT
enter DROP CATALOG command again to confirm catalog removal
```

再次输入 `DROP CATALOG` 命令将从目录数据库中删除所有对象。建议在删除操作之前备份目录。

### 删除目录所有者
或者，删除所有者用户。使用 `DBA` 权限连接并发出 `DROP USER` 语句：

```sql
$ sqlplus system/manager
SQL> drop user rcat cascade;
```

`SQL*Plus` 会立即执行，没有第二次提示。请谨慎操作，因为此操作是永久性的。一个好的做法是在删除目录所有者之前对其执行 Data Pump 导出。

## 记录 RMAN 输出



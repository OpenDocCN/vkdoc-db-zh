# 公共用户、角色与容器管理

## 公共用户的权限管理

普通用户必须在每个可插拔数据库内部授予权限。换句话说，如果你在连接到根容器时为公共用户授予权限，这些权限不会级联到 PDB。如果你需要为公共用户授予跨 PDB 的权限，那么请创建一个公共角色，并将其分配给该公共用户。

那么，公共用户有什么用呢？一种情况是跨 PDB 执行不需要`SYSDBA`级别权限的常见 DBA 维护活动。例如，你希望建立一个 DBA 账户，该账户拥有创建用户、授予权限等的权限，但又不想使用像`SYS`（在所有数据库中拥有所有权限）这样的账户。在此场景中，你可以创建一个公共 DBA 用户，并创建一个包含适当权限的 DBA 公共角色。然后，将这个公共角色分配给公共 DBA。

## 创建公共角色

就像你可以创建一个跨越所有 PDB 的公共用户一样，你也可以以同样的方式创建一个公共角色。公共角色提供了一个单一的对象，你可以向其授予在根容器关联的所有可插拔数据库中都有效的权限。

公共角色在根容器中创建，并会自动在所有关联的 PDB 以及未来创建的任何 PDB 中创建。与公共用户类似，公共角色必须以`C##`或`c##`字符串开头；例如：

```sql
SQL> create role c##dbaprivs container = all;
```

接下来，你可以根据需要向公共角色分配权限。这里，将`DBA`角色授予先前创建的角色：

```sql
SQL> grant dba to c##dbaprivs container = all;
```

现在，如果你将这个公共角色分配给一个公共用户，那么当该公共用户连接到与根容器关联的任何可插拔数据库时，该角色关联的权限就会生效：

```sql
SQL> grant c##dbaprivs to c##dba container = all;
```

## 创建本地用户和角色

本地用户是仅针对该容器并且在该容器内“本地”的用户。这特别适用于应用程序用户以及仅在特定 PDB 中被授权的用户。本地用户与数据库用户之间实际上没有区别，它们只是没有“C##”前缀的 CDB 用户。

你可以通过指定`CONTAINER`子句或不指定它来在 PDB 中创建本地用户。首先连接到要为其创建本地用户的 PDB：

```sql
SQL> CREATE USER pdbsys IDENTIFIED BY pass CONTAINER=CURRENT;
SQL> CREATE USER appuser IDENTIFIED BY pass;
```

角色也分为公共角色和本地角色。所有 Oracle 提供的角色都是公共角色，但可以被授予本地用户。

## 报告容器空间

要报告 CDB 内的所有容器（根容器、种子容器和所有 PDB），你必须遵循以下步骤：

*   以具有查看 CDB 级别视图权限的用户身份连接到根容器。
*   确保你的查询在适当的地方使用了 CDB 级别视图。
*   确保你希望报告的任何 PDB 都处于打开状态。如果 PDB 未打开，则不会显示任何信息。

以下是一个使用 CDB 级别视图报告 CDB 内所有容器基本空间使用信息的查询：

```sql
SQL> SET LINES 132 PAGES 100
SQL> COL con_name        FORM A15 HEAD "Container|Name"
SQL> COL tablespace_name FORM A15
SQL> COL fsm             FORM 999,999,999,999 HEAD "Free|Space Meg."
SQL> COL apm             FORM 999,999,999,999 HEAD "Alloc|Space Meg."
--
SQL> COMPUTE SUM OF fsm apm ON REPORT
SQL> BREAK ON REPORT ON con_id ON con_name ON tablespace_name
--
SQL> WITH x AS (SELECT c1.con_id, cf1.tablespace_name, SUM(cf1.bytes)/1024/1024 fsm
FROM cdb_free_space cf1
,v$containers   c1
WHERE cf1.con_id = c1.con_id
GROUP BY c1.con_id, cf1.tablespace_name),
y AS (SELECT c2.con_id, cd.tablespace_name, SUM(cd.bytes)/1024/1024 apm
FROM cdb_data_files cd
,v$containers   c2
WHERE cd.con_id = c2.con_id
GROUP BY c2.con_id
,cd.tablespace_name)
SELECT x.con_id, v.name con_name, x.tablespace_name, x.fsm, y.apm
FROM x, y, v$containers v
WHERE x.con_id          = y.con_id
AND   x.tablespace_name = y.tablespace_name
AND   v.con_id          = y.con_id
UNION
SELECT vc2.con_id, vc2.name, tf.tablespace_name, null, SUM(tf.bytes)/1024/1024
FROM v$containers vc2, cdb_temp_files tf
WHERE vc2.con_id  = tf.con_id
GROUP BY vc2.con_id, vc2.name, tf.tablespace_name
ORDER BY 1, 2;
```

以下是一些示例输出：

```
Container                                   Free            Alloc
CON_ID Name            TABLESPACE_NAME       Space Meg.       Space Meg.
---------- --------------- --------------- ---------------- ----------------
1 CDB$ROOT        SYSAUX                        42              780
SYSTEM                         7              790
TEMP                                           88
UNDOTBS1                     206              230
USERS                          4                5
2 PDB$SEED        SYSAUX                         2              640
SYSTEM                         5              260
TEMP                                           87
********** *************** *************** ---------------- ----------------
sum                                                     266            2,880
```

在尝试查询 CDB 级别视图之前，请确保你想要报告的 PDB 都已打开。如果 PDB 未打开，它将不会出现在报告输出中。默认情况下，PDB 设置为打开；作为参考，以下是再次打开所有可插拔数据库的语句：

```sql
SQL> alter pluggable database all open ;
```

> **提示：**
> 报告单个可插拔数据库的表空间和数据文件使用情况与报告非 CDB 数据库中的空间使用情况没有太大区别。首先，以具有从`DBA`级别视图选择权限的用户身份直接连接到可插拔数据库。然后，运行查询空间相关视图的报告。有关报告非 CDB 中空间使用的查询，请参见第 4 章。

## 切换容器

一旦你以公共用户身份连接到数据库中的任何容器（根容器或 PDB），你就可以使用`ALTER SESSION`命令切换到你已被授予访问权限的另一个容器。例如，要将当前容器设置为名为`SALESPDB`的 PDB，你可以执行以下操作：

```sql
SQL> alter session set container = salespdb;
```

你可以通过指定`CDB$ROOT`切换回根容器：

```sql
SQL> alter session set container = cdb$root;
```

切换容器不需要监听器正在运行或密码文件。只要公共用户拥有权限，用户就会成功切换到新的容器上下文。拥有切换容器的能力在需要连接到 PDB 以排查问题然后再连接回根容器时特别有用。




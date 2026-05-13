# 查看虚拟列和隐藏列的信息



正如我们所看到的，隐藏列和虚拟列的统计信息与常规列相同；CBO（基于成本的优化器）以类似的方式使用这些统计信息。然而，关于隐藏列和虚拟列还有两点需要说明。我想解释扩展统计信息的导出方式，但首先让我们看看如何在数据字典中检查隐藏列和虚拟列的信息。

## 隐藏列和虚拟列的数据字典视图

视图 `ALL_TAB_COLUMNS` 不显示隐藏列，并且只有当隐藏列上存在统计信息时，才会出现在 `ALL_TAB_COL_STATISTICS` 中。有两个视图无论统计信息是否存在都会显示隐藏列的信息：

*   `ALL_TAB_COLS` 显示与 `ALL_TAB_COLUMNS` 类似的信息，但包含隐藏列。`ALL_TAB_COLS` 统计视图包含两个统计列——`VIRTUAL_COLUMN` 和 `HIDDEN_COLUMN`——分别指示对象列是否为虚拟列或隐藏列。
*   `ALL_STAT_EXTENSIONS` 列出所有虚拟列的信息，包括隐藏的虚拟列，无论它们是否与基于函数的索引关联；未隐藏的、显式声明的虚拟列；或与扩展统计信息关联的隐藏列。`ALL_STAT_EXTENSIONS` 包含一个列——`EXTENSION`——其数据类型为 `CLOB`，定义了虚拟列所基于的表达式。

## 导出扩展统计信息

除扩展统计信息外，隐藏列和虚拟列的导出和导入方式与常规列完全相同。但是，每个扩展统计信息在导出表中都有一行额外的记录。

如果不先使用 `CREATE INDEX` 语句创建基于函数的索引，就无法导入与基于函数的索引关联的隐藏列的统计信息。如果不先使用 `CREATE TABLE` 或 `ALTER TABLE` 语句声明该列，就无法导入显式声明的虚拟列的统计信息。创建基于函数的索引和显式声明的虚拟列的 DDL 语句会导致虚拟列所基于的表达式被存储在数据字典中。

另一方面，您可以在不先调用 `DBMS_STATS.CREATE_EXTENDED_STATS` 的情况下导入扩展统计信息；导入操作会隐式创建隐藏的虚拟列。导入统计信息时发生的这种虚拟列隐式创建意味着导出表需要包含虚拟列所基于表达式的信息。此信息保存在导出表的一个额外行中，该行的 `TYPE` 为 `E`，表达式存储在类型为 `CLOB` 的导出列 `CL1` 中。清单 9-13 展示了如何导出和导入扩展统计信息。

清单 9-13. 导出和导入扩展统计信息

```
TRUNCATE TABLE ch9_stats;

BEGIN
   DBMS_STATS.export_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT'
     ,statown   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,stattab   => 'CH9_STATS');

DBMS_STATS.drop_extended_stats (
      ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname     => 'STATEMENT'
     ,extension   => '(TRANSACTION_DATE,POSTING_DATE)');

DBMS_STATS.drop_extended_stats (
      ownname     => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname     => 'STATEMENT'
     ,extension   => q'[(CASE WHEN DESCRIPTION <> 'Flight'
                                      AND TRANSACTION_AMOUNT > 100
                                 THEN 'HIGH' END)]');

DBMS_STATS.delete_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT');
END;
/

SELECT c1, SUBSTR (c4, 1,10) c4, DBMS_LOB.SUBSTR (cl1,60,1) cl1
  FROM ch9_stats
 WHERE TYPE = 'E';

-- 查询语句输出
C1                C4                CL1
STATEMENT        SYS_STU557        SYS_OP_COMBINED_HASH("TRANSACTION_DATE","POSTING_DATE")
STATEMENT        SYS_STUN6S        CASE  WHEN ("DESCRIPTION"<>'Flight' AND "TRANSACTION_AMOUNT"

SELECT extension_name
    FROM all_stat_extensions
   WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
         AND table_name = 'STATEMENT'
ORDER BY extension_name;

-- 查询语句输出

EXTENSION_NAME
AMOUNT_CATEGORY
POSTING_DELAY
SYS_NC00010$

BEGIN
   DBMS_STATS.import_table_stats (
      ownname   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,tabname   => 'STATEMENT'
     ,statown   => SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
     ,stattab   => 'CH9_STATS');
END;
/

SELECT extension_name
    FROM all_stat_extensions
   WHERE     owner = SYS_CONTEXT ('USERENV', 'CURRENT_SCHEMA')
         AND table_name = 'STATEMENT'
ORDER BY extension_name;

-- 查询语句输出

EXTENSION_NAME
AMOUNT_CATEGORY
POSTING_DELAY
SYS_NC00010$
SYS_STU557IEAHGZ6B4RPUZEQ_SX#6
SYS_STUN6S71CM11RD1VMYVLKR9$GJ
```

清单 9-14 将 `STATEMENT` 的对象统计信息导出到一个刚刚截断的导出表中，然后使用 `DBMS_STATS.DROP_EXTENDED_STATISTICS` 过程删除了两个扩展统计信息。为了确保彻底，剩余的统计信息也被删除了。

对导出表的查询显示扩展统计信息表达式的定义已被保存。请注意多列统计信息中 `SYS_OP_COMBINED_HASH` 函数的使用。一旦扩展统计信息被删除，我们可以看到它们在对 `ALL_STAT_EXTENSIONS` 的查询输出中不再可见。

一旦我们导入统计信息，就可以看到扩展统计信息重新出现在 `ALL_STAT_EXTENSIONS` 中。

## 统计信息描述总结

我已经解释了与表、索引和列相关的统计信息是什么，以及 CBO 如何使用它们进行优化。我还解释了 CBO 可以创建和使用的主要不同类型的隐藏列和虚拟列。我尚未做的是解释可以在分区表上使用的各种不同级别的统计信息。现在让我们开始讨论这个话题。

## 统计信息与分区

分区选项是 Oracle 数据库企业版的一项单独许可的功能。利用此选项，您可以对表和索引进行分区，并享受诸多性能和管理上的好处。我将在第 15 章简要探讨分区的性能优势，但现在我想重点介绍与分区表相关的统计信息。

正如我在本章引言中提到的，当使用分区选项时，表、索引和列的统计信息都可以在全局、分区以及（对于复合分区表而言）子分区级别进行维护。与表或索引分区关联的统计信息与全局级别的统计信息完全相同。例如，表分区的 `AVG_ROW_LEN` 统计信息指示该表分区中行的平均大小。索引分区的 `BLEVEL` 统计信息指示从该索引分区的根块到分区内的叶块需要访问多少个块。在分区级别定义的列对象的 `NUM_NULLS` 统计信息指示在指定表分区中该列值为 `NULL` 的行数。

在本节中，我想澄清一些关于 CBO 如何以及何时使用分区和子分区级别统计信息、何时使用全局统计信息的困惑。我还想清楚地解释 CBO 使用分区级统计信息与运行时引擎进行分区消除之间的区别。



# SQL 执行计划优化与临时表统计信息设置

## 执行计划对比

### 初始强制执行计划（针对 3 行临时表）

| Id  | Operation               | Name          | Rows  | Cost (%CPU) |
| --- | ----------------------- | ------------- | ----- | ----------- |
| 0   | MERGE STATEMENT         |               | 3     | 176 (1)     |
| 1   | MERGE                   | PAYMENTS      |       |             |
| 2   | VIEW                    |               |       |             |
| 3   | HASH JOIN OUTER         |               | 3     | 176 (1)     |
| 4   | TABLE ACCESS FULL       | PAYMENTS_TEMP | 3     | 2 (0)       |
| 5   | TABLE ACCESS FULL       | PAYMENTS      | 120K  | 174 (1)     |

### 针对 3 行临时表的 MERGE 语句示例

```sql
MERGE /*+ cardinality(t 10000) leading(t) use_nl(p) */
     INTO  payments p
     USING payments_temp t
        ON (p.payment_id = t.payment_id)
WHEN MATCHED
THEN
   UPDATE SET p.employee_id = t.employee_id
             ,p.special_flag = t.special_flag
             ,p.paygrade = t.paygrade
             ,p.payment_date = t.payment_date
             ,p.job_description = t.job_description
WHEN NOT MATCHED
THEN
   INSERT     (payment_id
              ,special_flag
              ,paygrade
              ,payment_date
              ,job_description)
       VALUES (t.payment_id
              ,t.special_flag
              ,t.paygrade
              ,t.payment_date
              ,t.job_description);
```

### 强制执行计划（针对 10000 行临时表）

| Id  | Operation                      | Name          | Rows  | Cost (%CPU) |
| --- | ------------------------------ | ------------- | ----- | ----------- |
| 0   | MERGE STATEMENT                |               | 10000 | 10004 (1)   |
| 1   | MERGE                         | PAYMENTS      |       |             |
| 2   | VIEW                         |               |       |             |
| 3   | NESTED LOOPS OUTER          |               | 10000 | 10004 (1)   |
| 4   | TABLE ACCESS FULL          | PAYMENTS_TEMP | 10000 | 2 (0)       |
| 5   | TABLE ACCESS BY INDEX ROWID| PAYMENTS      | 1     | 1 (0)       |
| 6   | INDEX UNIQUE SCAN         | PAYMENTS_PK   | 1     | 0 (0)       |

## CBO 成本估算分析

Listing 12-13 展示了当我们在 3 行临时表上强制执行针对 10,000 行临时表的最优计划，以及在 10,000 行临时表上强制执行针对 3 行临时表的最优计划时的 CBO 成本估算。在这个具体示例中，我们可以看到：当应用于 10,000 行表时，嵌套循环连接（`NESTED LOOPS OUTER`）似乎是灾难性的：成本从 176 跃升至 10,000。然而，当临时表中只选择了 3 行时，使用哈希连接（`HASH JOIN`）的后果并不特别严重，从 5 跃升至 176；这是一个重要因素，但可能只会导致一两秒的耗时增加。因此，我们可以伪造临时表统计信息，指示我们的表中有 10,000 行，这样一切就应该没问题了——至少在这个例子中是这样。

如果您从表中选择特定行，您可能希望阻止这些列上的谓词将 CBO 的基数估计值降低到如此程度，以至于使用了嵌套循环连接方法。Listing 20-14 显示，通过将每个列的`NUM_DISTINCT`值设置为 2，并将表的`NUM_ROWS`提高到（比如）20,000，我们可以安排总是（或几乎总是）选择哈希连接。

## 设置统计信息以增加哈希连接机会

Listing 20-14. 设置统计信息以增加哈希连接的机会

```sql
CREATE OR REPLACE PACKAGE tstats
AS
-- Other procedures omitted
PROCEDURE set_temp_table_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name    all_tab_col_statistics.table_name%TYPE
     ,p_numrows       INTEGER DEFAULT 20000
     ,p_numblks       INTEGER DEFAULT 1000
     ,p_avgrlen       INTEGER DEFAULT 400);
-- Other procedures omitted
END tstats;
/

CREATE OR REPLACE PACKAGE BODY tstats
AS
-- Other procedures omitted
PROCEDURE set_temp_table_stats (
      p_owner         all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name    all_tab_col_statistics.table_name%TYPE
     ,p_numrows       INTEGER DEFAULT 20000
     ,p_numblks       INTEGER DEFAULT 1000
     ,p_avgrlen       INTEGER DEFAULT 400)
   IS
      distcnt   NUMBER;
   BEGIN
      DBMS_STATS.unlock_table_stats (ownname   => p_owner
                                    ,tabname   => p_table_name);
      $IF dbms_db_version.version >=12
      $then
         DBMS_STATS.set_table_prefs (ownname   => p_owner
                                    ,tabname   => p_table_name
                                    ,pname     => 'GLOBAL_TEMP_TABLE_STATS'
                                    ,pvalue    => 'SHARED');
      $END
      DBMS_STATS.delete_table_stats (ownname   => p_owner
                                    ,tabname   => p_table_name);
      DBMS_STATS.set_table_stats (ownname         => p_owner
                                 ,tabname         => p_table_name
                                 ,numrows         => p_numrows
                                 ,numblks         => p_numblks
                                 ,avgrlen         => p_avgrlen
                                 ,no_invalidate   => FALSE);
      /*

我们必须现在设置列统计信息以限制谓词对基数计算的影响；
      默认情况下，每个谓词会使基数减少 100 倍。

我们使用 2 作为不同值的数量，以将这个因子减少到 2。
      我们不使用 1，因为像"column_1 <> 'VALUE_1'"这样的谓词会将
      基数减少到 1。

*/
      distcnt := 2;

FOR r IN (SELECT *
                  FROM all_tab_columns
                 WHERE owner = p_owner AND table_name = p_table_name)
      LOOP
         DBMS_STATS.set_column_stats (ownname         => p_owner
                                     ,tabname         => r.table_name
                                     ,colname         => r.column_name
                                     ,distcnt         => distcnt
                                     ,density         => 1 / distcnt
                                     ,srec            => NULL
                                     ,no_invalidate   => FALSE);
      END LOOP;

DBMS_STATS.lock_table_stats (ownname => p_owner, tabname => p_table_name);
   END set_temp_table_stats;
-- Other procedures omitted
END tstats;

EXEC tstats.set_temp_table_stats(p_table_name=>'PAYMENTS_TEMP');
```

## 应用统计信息后的执行计划示例

### 示例 1：无谓词条件的连接查询

```sql
SELECT *
  FROM payments_temp JOIN payments USING (payment_id);
```

| Id  | Operation          | Name          | Rows  | Cost (%CPU) |
| --- | ------------------ | ------------- | ----- | ----------- |
| 0   | SELECT STATEMENT   |               | 20000 | 845 (1)     |
| 1   | HASH JOIN         |               | 20000 | 845 (1)     |
| 2   | TABLE ACCESS FULL| PAYMENTS_TEMP | 20000 | 272 (0)     |
| 3   | TABLE ACCESS FULL| PAYMENTS      | 120K  | 174 (1)     |

### 示例 2：带等值谓词的连接查询

```sql
SELECT *
  FROM payments_temp t JOIN payments p USING (payment_id)
 WHERE t.paygrade = 10 AND t.job_description='INTERN';
```

| Id  | Operation          | Name          | Rows  | Cost (%CPU) |
| --- | ------------------ | ------------- | ----- | ----------- |
| 0   | SELECT STATEMENT   |               | 5000  | 447 (1)     |
| 1   | HASH JOIN         |               | 5000  | 447 (1)     |
| 2   | TABLE ACCESS FULL| PAYMENTS_TEMP | 5000  | 272 (0)     |
| 3   | TABLE ACCESS FULL| PAYMENTS      | 120K  | 174 (1)     |

### 示例 3：带不等值和范围谓词的连接查询

```sql
SELECT *
  FROM payments_temp t JOIN payments p USING (payment_id)
 WHERE t.paygrade != 10 AND t.payment_date > SYSDATE - 31;
```

| Id  | Operation          | Name          | Rows  | Cost (%CPU) |
| --- | ------------------ | ------------- | ----- | ----------- |



|   0 | SELECT STATEMENT   |               |  6697 |   447   (1)|
|---|---|---|---|---|
|   1 |  哈希连接         |               |  6697 |   447   (1)|
|   2 |   全表扫描| PAYMENTS_TEMP |  6697 |   272   (0)|
|   3 |   全表扫描| PAYMENTS      |   120K|   174   (1)|

清单 20-14 展示了 `TSTATS` 包中的另一个过程。该过程用于在临时表上伪造统计信息。过程包含一个用于设置表首选项的条件调用，这是 12c 特有的功能。在为 `PAYMENTS_TEMP` 伪造统计信息后，清单 20-14 展示了使用临时表上各种谓词组合的多种两表连接的执行计划。由于基数估计保持较高，所有情况都使用了哈希连接，并且成本估计保持合理。当然，在实际工作中，你不会依赖 CBO 的估计——你会运行一些真实的工作负载并测量实际的用时。

## 列统计信息的正确取值

在清单 20-14 所示的例子中，通过为所有用于谓词的列设置较低的 `NUM_DISTINCT` 值来保持较高的基数估计。然而，较低的 `NUM_DISTINCT` 值会导致该列在 `GROUP BY` 子句中使用时产生较低的基数。可能需要一些实验，并且在某些情况下你很可能需要借助提示。

这些清单只考察了一条 SQL 语句，而这条语句可能代表也可能不代表你应用中的典型 SQL 语句。尽管如此，当表中的行数变化很大时，哈希连接通常代表最佳的折衷方案。

你不应该将关于临时表统计信息的讨论视为一种规定性的解决方案。相反，你应该将其视作你需要进行的那种分析的模板。如果你进行了分析并发现你确实必须为不同大小的临时表制定不同的执行计划，那么清单 6-6 和 6-7 中展示的方案类型始终可作为最后的手段。

一旦你为所有表（包括临时表）收集或设置了所有统计信息，你的测试系统上所有应用语句的执行计划就应该稳定了。现在该看看如何将这种稳定性复制到其他测试系统以及生产环境了。

## 如何部署 TSTATS

正如我在第 6 章中解释的那样，`TSTATS` 的理念是将对象统计信息视为应用程序代码。就像应用程序代码一样，对象统计信息应置于版本控制之下，并根据变更控制规则，使用自动化部署脚本部署到生产环境和正式测试系统。这如何最好地实现呢？

这个过程始于收集对象统计信息的测试系统。第一步是将你的应用程序中表的对象统计信息导出到使用 `DBMS_STATS.CREATE_STAT_TABLE` 过程创建的导出表中。为了便于讨论，我们假设你的应用程序名为 `AJAX`，并且你的导出表名为 `TSTATS_AJAX`。现在你需要以 `TSTATS_AJAX` 为基础，创建一个置于版本控制下的部署脚本。你可以通过将 `TSTATS_AJAX` 中的行转换为 `insert` 语句或创建一个 SQL*Loader 文件来实现这一点。像 Oracle 的 SQL Developer 和 Dell 的 TOAD 这样的工具都包含了生成此类脚本的功能。你的部署脚本可以在目标系统上创建 `TSTATS_AJAX`，然后使用 SQL*Plus 或 SQL*Loader（视情况而定）直接将数据加载进去。

![image](img/sq.jpg) **注意** 你可能会认为，如果部署脚本包含了应用程序模式中每个表、索引和表列的 `insert` 语句，其大小将会非常巨大。但是，请记住你不会有表分区的统计信息，所以情况可能不会像你想象的那么糟糕。

部署脚本的最后一部分应将数据从 `TSTATS_AJAX` 导入到生产环境或目标测试系统的数据字典中。清单 20-15 展示了 `TSTATS` 包中执行此导入的过程。

清单 20-15。将对象统计信息导入数据字典

```
CREATE OR REPLACE PACKAGE tstats
AS
-- 其他过程已省略
   PROCEDURE import_table_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE
     ,p_statown all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_stat_table all_tab_col_statistics.table_name%TYPE);
-- 其他过程已省略
END tstats;
/

CREATE OR REPLACE PACKAGE BODY tstats
AS
-- 其他过程已省略
PROCEDURE import_table_stats (
      p_owner all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_table_name all_tab_col_statistics.table_name%TYPE
     ,p_statown all_tab_col_statistics.owner%TYPE DEFAULT SYS_CONTEXT (
                                                                   'USERENV'
                                                                  ,'CURRENT_SCHEMA')
     ,p_stat_table all_tab_col_statistics.table_name%TYPE)
   IS
   BEGIN
      DECLARE
         already_up_to_date   EXCEPTION;
         PRAGMA EXCEPTION_INIT (already_up_to_date, -20000);
      BEGIN
         DBMS_STATS.upgrade_stat_table (ownname   => 'DLS'
                                       ,stattab   => 'DLS_TSTATS');
      EXCEPTION
         WHEN already_up_to_date
         THEN
            NULL;
      END;

      DBMS_STATS.unlock_table_stats (ownname   => p_owner
                                    ,tabname   => p_table_name);
      DBMS_STATS.delete_table_stats (ownname         => p_owner
                                    ,tabname         => p_table_name
                                    ,no_invalidate   => FALSE);
      DBMS_STATS.import_table_stats (ownname         => p_owner
                                    ,tabname         => p_table_name
                                    ,statown         => p_statown
                                    ,stattab         => p_stat_table
                                    ,no_invalidate   => FALSE);

-- 对于分区表，目标系统上的（子）分区数量
      -- 可能与源系统上的不匹配。
      FOR r
         IN (SELECT *
               FROM all_tables
              WHERE     owner = p_owner
                    AND table_name = p_table_name
                    AND partitioned = 'YES')
      LOOP
         adjust_global_stats (p_owner, p_table_name,'PMOP');
      END LOOP;

      DBMS_STATS.lock_table_stats (ownname => p_owner, tabname => p_table_name);
   END import_table_stats;
-- 其他过程已省略
END tstats;
/



若目标系统运行的 Oracle 数据库软件版本高于源系统，则会调用 `DBMS_STATS.UPGRADE_STAT_TABLE` 以使 `TSTATS_AJAX` 保持最新状态，若 `TSTATS_AJAX` 已是最新则忽略生成的任何错误。随后，针对 `TSTATS_AJAX` 所引用的每张表执行以下操作：

*   移除任何现有统计信息。这对于防止出现统计信息混杂的情况是必要的。
*   通过调用 `DBMS_STATS.IMPORT_TABLE_STATS` 将对象统计信息导入数据字典。
*   调整分区表的全局统计信息，因为目标系统上的分区数量可能与源系统不匹配。
*   锁定已导入的统计信息。

如果在您应用程序软件的后续版本中希望添加新表或索引，您可以简单地创建一个包含这些额外对象的新版部署脚本，或者也可以为这些对象创建一个专用脚本。您也可能将 `TSTATS.IMPORT_OBJECT_STATS` 调用包含在创建新表或索引的 DML 脚本中。这完全取决于您组织内部署新对象所使用的流程。

## 总结

TSTATS 是一种管理对象统计信息的方法，它与 Oracle 建议的标准方法**根本不同**。尽管如此，TSTATS 并未使用 Oracle 数据库的任何不受支持的特性，并且有效地允许将执行计划的变更置于变更控制流程之下，而无需使用难以（若非不可能）与应用程序代码协调一致的 SQL 语句存储库。

本章提供了一些关于如何设置 TSTATS 的思路。然而，TSTATS 并非一个“即插即用”的工具，在成功地为复杂的商业应用程序部署 TSTATS 之前，通常需要进行大量的分析和定制。希望本章已指明了方向。

我希望您觉得阅读本书是一次富有教育意义且愉快的体验。祝您好运！

### 索引

![images](img/sq.jpg)  A

*   `访问方法`
*   `位图索引访问`
*   `转换计数示例`
*   `索引合并，INLIST`
*   `操作操作`
*   `操作`
*   `物理结构`
*   `受限的 ROWID`
*   `B-树` (*参见* `B-树索引访问`)
*   `簇访问`
    *   `方法`
    *   `操作`
*   `全表扫描`
    *   `操作`
    *   `段全扫描`
    *   `表访问抽样`
*   `ROWID` (*参见* `ROWID`)
*   `TABLE 和 XMLTABLE`
    *   `CAST MULTISET 示例`
    *   `DBMS_XPLAN.display 示例`
    *   `嵌套表`
    *   `表函数`
    *   `用法`
*   `自适应游标共享`, 11gR1
*   `自适应执行计划`
*   `高级调优技术`
    *   `索引快速全扫描`
    *   `索引连接`
    *   `JPPD 转换`
    *   `多列索引`



外连接到内连接转换

调用代码，数据字典查询

PUSH_PRED 提示

次优计划，数据字典视图

ROWID 范围，应用程序编码的并行执行

星型转换

美国国家标准协会 (ANSI)

ANSI。参见美国国家标准协会 (ANSI)

反连接
- 空值感知
- 标准

数组接口
- 绑定变量
- BULK COLLECT 语法
- DBA
- DELETE 和 MERGE
- DML

自动 DOP。参见自动并行度 (auto DOP)

自动并行度 (auto DOP)

自动段空间管理 (ASSM)

自动工作负载存储库 (AWR)

AWR。参见自动工作负载存储库 (AWR)

# B

位图连接索引

块范围粒度

B 树索引访问
- AND_EQUAL
- 索引快速全扫描
- 索引全扫描
    - 降序
    - MIN/MAX 优化
    - 操作
    - 简化视图
- 索引连接
    - 复杂示例
    - HR 示例模式
    - 手动转换
- 索引范围扫描
    - 降序
    - MIN/MAX 优化
- 索引采样快速全扫描
- 索引跳跃扫描
    - 降序变体
    - 操作
- 索引唯一扫描
- 方法

灌木连接
- ANSI 连接
- 执行计划
- 启发式转换
- Oracle 性能

# C

缓存效应
- 描述
- 使用多列索引的索引聚簇
- 使用单列索引的索引聚簇
- 逻辑 I/O 操作

CARDINALITY 和 OPT_ESTIMATE 提示

基于成本的优化器
- 执行计划
- 索引扫描
- 嵌套循环



# 列统计信息

## CBO 计算

*   行源
*   `SH.COUNTRIES`
*   基数错误
*   糟糕的物理设计
*   缓存效应（*参见* 缓存效应）
*   列相关性
*   争用
*   虚假数据类型
*   描述
*   函数
*   信息缺失
*   过时的统计信息
*   `统计信息反馈` 和 `DBMS_STATS.SEED_COL_USAGE` 功能
*   传递闭包
*   不支持的转换
*   基数反馈
*   笛卡尔连接
*   `CBO`。 *参见* 基于成本的优化器 (CBO)
*   列相关性
*   列投影
*   扩展统计信息
    *   表达式统计信息
    *   多列统计信息
*   `HIGH_VALUE` 和 `LOW_VALUE`
*   直方图
    *   `ALL_TAB_COLS`
    *   数据
    *   伪造直方图
    *   收集过程
    *   手动创建
    *   缺失的直方图
    *   `SREC` 参数
    *   `TRANSACTION_AMOUNT`
*   `JOB_DESCRIPTION`
*   `METHOD_OPT`
*   参数
*   `PAYGRADE` 和 `PAYMENT_DATE` 的相关性
*   `PAYMENT_DATE`
*   生产系统
*   `SAMPLE_PAYMENTS`
*   选择与执行计划
*   简单的基数与字节数计算
*   `SQL`
*   `SYS_OP_COMBINED_HASH` 函数
*   基于时间的列
*   `TSTATS_AJAX`
*   `TSTATS.AMEND_TIME_BASED_STATISTICS`
*   虚拟列和隐藏列
*   `CBO`
    *   数据字典视图
    *   导出扩展统计信息
    *   查询
    *   非隐藏的虚拟列

## NUM_DISTINCT=1 的列

*   与密度
*   与范围谓词
*   数据字典
*   数据伪造
*   `DBMS_STATS.UPGRADE_STAT_TABLE`
*   `DML`



*   CBO
*   `DBMS_STATS.COPY_TABLE_STATS`
*   `HIGH_VALUE` 与 `LOW_VALUE`
*   `NULL`
*   `PAYMENTS`
*   `SPECIAL_FLAG`
*   `TSTATS.ADJUST_COLUMN_STATS`
*   基于成本的优化器 (CBO)
*   CBO 成本与实际成本
*   成本，定义
*   成本估计算法
*   准确性
*   对象统计信息
*   DML 或 DDL 语句
*   执行计划
*   成本
*   定义
*   最优的
*   次优的
*   优化过程 (*参见* 优化过程，CBO)
*   RBO
*   游标
    *   缓存
    *   子游标
    *   父游标
*   ![图片](img/sq.jpg)  D
*   数据库管理员 (DBA)
*   数据定义语言 (DDL) 语句
*   数据密集化
    *   避免，分析场景
    *   描述
    *   教科书式解法
*   数据字典视图
    *   描述
    *   复杂视图的效率问题
*   数据流操作符
*   数据操作语言 (DML)
*   数据类型转换
    *   编码问题
    *   隐式转换
*   DBA。*参见* 数据库管理员 (DBA)
*   `DBMS_STATS.COPY_TABLE_STATS`
    *   优势
    *   `BUSINESS_DATE`
    *   `HIGH_VALUE` 与 `LOW_VALUE`
    *   分区统计信息
    *   SQL 语句
    *   `STATEMENT_PART`
    *   `TRANSACTION_DATE`
*   `DBMS_STATS.SEED_COL_USAGE`
*   `DBMS_XPLAN` 格式化选项
    *   ‘BASIC +ALLSTATS’
*   `DBMS_XPLAN` 包
*   DDL。*参见* 数据定义语言 (DDL) 语句
*   并行度 (DOP)
    *   自动 DOP
    *   理想 DOP
*   反规范化
    *   位图连接索引
    *   手动聚合与连接表
    *   物化视图
    *   SQL 性能调优
*   基于磁盘的排序



# m
`手动增加大小，内存`
`内存分配`
`排序运行`
`三趟排序`
`两趟排序`
# E
`DML`。参见 `数据操作语言 (DML)`
`DOP`。参见 `并行度 (DOP)`
`重复排序`
![images](img/sq.jpg)  E
`EBR`。参见 `基于版本的重定义 (EBR)`
`基于版本的重定义 (EBR)`
`企业管理器 (EM)`
`等于与不等于谓词`
`不适当使用，不等运算符`
`MOSTLY_BORING`
`NULL 使用`
# 错误，提示
`CHANGE_DUPKEY_ERROR_INDEX`
`EBR`
`IGNORE_ROW_ON_DUPKEY_INDEX`
`完整性约束`
`ORA-30393 错误`
`行和列`
# Exadata
`Exadata`
# 执行计划
`执行计划`
`自适应计划`
`CBO 计算`
`列投影`
`控制并行执行`
`创建表与环回数据库链接`
`数据流操作符`
`DBMS_XPLAN.DISPLAY`
`DBMS_XPLAN 格式化选项`
`DBMS_XPLAN 函数`
`DDL 语句`
`默认显示`
`显示计划`
`AWR`
`游标`
`DBMS_XPLAN.DISPLAY_CURSOR`
`在游标缓存中`
`pstart 和 pstop 列`
`SH 示例模式与游标缓存`
`分布机制`
`FORCE PARALLEL QUERY`
`全局提示`
`粒度`
`提示数据字典视图`
`多个 DFO 树`
`NO_MERGE 提示`
`注释`
# 操作
`操作`
`协程，概念`
`成本`
`INLIST 迭代器与分区范围操作符`
`交互`
`操作 0: SELECT STATEMENT`
`操作 1: SORT AGGREGATE`
`操作 2: TABLE ACCESS FULL`
`操作 3: HASH JOIN`
`操作 4 和 5: 表全扫描`



# 索引条目

## 条目列表

-   父子关系
-   优化器提示
-   ORCL
-   大纲数据
-   PARALLEL 和 PARALLEL_INDEX
-   并行查询服务器集与 DFO 树
-   绑定变量窥探
-   谓词信息
-   查询块与对象别名
-   远程 SQL
-   结果缓存信息
-   运行 EXPLAIN PLAN
-   SEL$F1D6E378
-   SQL 语言参考手册
-   语句排队与 DOP 降级
-   表队列与 DFO 排序
-   变换

## 相关条目

-   EXPLAIN PLAN. `另请参阅` 执行计划
-   访问谓词
-   EMP 和 DEPT 表
-   执行计划
-   过滤谓词
-   误导性
-   操作表

![images](img/sq.jpg)  F

## F 部分

-   最终状态优化
-   访问方法
-   哈希连接
-   嵌套循环
-   IN 列表迭代
-   执行计划，无迭代
-   执行计划，有迭代
-   连接方法
-   次优执行计划
-   不直观
-   连接顺序
-   不适当
-   简单

## F 部分（续）

-   全表扫描 (FTS)
-   字节数
-   基数
-   成本

![images](img/sq.jpg)  G

## G 部分

-   全局分区索引
-   全局统计信息, Oracle
-   `STATEMENT_PART`
-   `TRANSACTION_DATE`
-   `TSTATS.ADJUST_COLUMN_STATS_V2`

![images](img/sq.jpg)  H

## H 部分

-   哈希连接
-   提示
-   数据与数据库
-   DML 错误日志记录
-   有文档记录与无文档记录
-   `CARDINALITY` 提示
-   `EXPAND_GSET_TO_UNION`
-   `GATHER_PLAN_STATISTICS`
-   `SQL`
-   `EBR`
-   错误
-   `MODEL` 子句
-   优化器提示 (`参阅` 优化器提示)
-   Oracle `SQL`
-   生产环境提示 (`参阅` 生产环境提示)
-   `PUSH_SUBQ` 故事



## I

`运行时引擎提示`

`理想并行度（ideal DOP）`

`理想 DOP`。参见 `理想并行度（ideal DOP）`

`索引压缩`

`索引组织表（IOTs）`

`索引范围扫描与索引全扫描`

`维度表以避免全局索引`

`索引以避免排序`

`本地索引用于数据排序，失败`

`索引统计信息`

`位图索引`

`代价表访问`

`影响因素`

`强聚簇索引`

`弱聚簇索引`

`基于函数的索引`

`带时区的时间戳`

`协调世界时（UTC）`

`索引操作`

`通过多列索引进行嵌套循环访问`

`参数`

`IN 列表迭代`

`感兴趣事务列表（ITL）`

`IOTs`。参见 `索引组织表（IOTs）`

`ITL`。参见 `感兴趣事务列表（ITL）`

## J, K, L

`连接顺序`

`哈希连接完全外连接`

`哈希连接右外连接`

`带哈希连接输入交换`

`不带哈希连接输入交换`

`连接谓词下推（JPPD）转换`

`连接`

`反连接`（参见 `反连接`）

`笛卡尔连接`

`描述`

`哈希连接`

`连接顺序`（参见 `连接顺序`）

`归并连接`

`嵌套循环`（参见 `嵌套循环`）

`并行连接`（参见 `并行连接`）

`半连接`（参见 `半连接`）

`SQL`

`分析函数`

`ANSI`

`笛卡尔连接`

`CBO`

`数据稠密化`

`四表内连接`

`完全外连接`

`输入交换`

`外连接与 ANSI 连接语法`

`专有语法`

`右外连接语法`

`行源`

`销售查询`

`表连接`

`传统语法`

`JPPD`。参见 `连接谓词下推（JPPD）转换`



![images](img/sq.jpg)  M

## M

-   手动段空间管理 (MSSM)
-   内存限制排序
-   合并连接
-   `MODEL` 子句
    -   与 Excel 的比较
    -   `CURRENT_VALUE (CV)`
    -   `DIMENSION BY` 子句
    -   执行计划
    -   `MEASURES` 子句
    -   `MEDIAN`
    -   `PARTITION BY` 子句
    -   `RULES` 子句
    -   电子表格概念

![images](img/sq.jpg)  N

## N

-   嵌套循环
    -   左横向连接
    -   传统
-   非排序聚合函数
    -   索引范围扫描与索引全扫描 (*参见* 索引范围扫描与索引全扫描)
    -   `ROW_NUMBER`
    -   使用 `FIRST_VALUE` 聚合函数排序行
    -   使用 `FIRST` 函数排序
-   非排序聚合函数 (NSAFs)
    -   哈希聚合
    -   在分析中
    -   `SORT GROUP BY`
-   `NSAFs`。 *参见* 非排序聚合函数 (NSAFs)

![images](img/sq.jpg)  O

## O

-   对象统计信息
    -   `CBO`
    -   数据字典信息
    -   信息来源
        -   初始化参数
        -   系统统计信息
    -   列。 *参见* 列统计信息
    -   创建
    -   导出和导入
    -   收集
        -   索引和表
        -   设置
        -   传输
    -   检查
        -   数据字典
        -   导出表
        -   索引 (*参见* 索引统计信息)
        -   锁定
            -   挂起
        -   目的 (*参见* 全表扫描 (FTS))
        -   恢复
    -   统计信息与分区
        -   以及子分区级统计信息
        -   执行计划
        -   收集分区表的统计信息
        -   收集统计信息
        -   全局统计信息
        -   `NUM_DISTINCT` 计算
        -   分区级统计信息，`CBO`
        -   分区大小
        -   概要信息
    -   表。 *参见* 表统计信息
-   对象统计信息与部署
    -   `CBO`



Connor McDonald 的观点

Dave Ensor 悖论

战略性特性

并发执行计划

多个执行计划

多个`SQL_ID`及其创建

倾斜数据与直方图

`SQL`语句

工作负载变化

扩展统计信息

收集统计信息，执行计划

生成虚假对象统计信息

手工构建的直方图

多个`SQL`语句

更改统计信息

物理数据库变更

转换`SQL`代码与添加提示

Oracle 的计划稳定性特性

`SQL`计划基线

`SQL`概要文件

存储提纲

性能管理原则

机场示例

非数据库部署策略

英国皇家邮政示例

`SLA`（服务级别协议）

`PL/SQL`代码

单个`SQL`语句

向`SQL`代码添加提示

物理数据库变更

转换`SQL`代码

`TCF`

`TSTATS`

列统计信息

描述

真实世界性能团队

时间敏感数据，移除

未变更的执行计划

单趟排序

优化过程

并行度

`ALTER SESSION`语句

自动`DOP`（并行度）

最终状态查询优化

理想`DOP`（并行度）

`PARALLEL`优化器

并行查询操作

查询转换

基于成本的转换

启发式转换

简单视图合并

优化器提示

`CARDINALITY`和`OPT_ESTIMATE`提示

`CBO`（基于成本的优化器）

`FIRST_ROWS`和`ALL_ROWS`

对象统计信息提示

`OPT_PARAM`

`PARALLEL`（T）

Oracle

列统计信息（*参见* `列统计信息`）

`DBMS_STATS`



DBMS_STATS.COPY_TABLE_STATS（*参见* `DBMS_STATS.COPY_TABLE_STATS`）

全局统计信息

管理与承诺

重新收集统计信息

表分区

大纲数据

注释

执行计划

标识

`INSERT`

解释提示

`OUTLINE_LEAF` 和 `OUTLINE`

查询块

查询转换提示

SQL 语句

提示类型

![images](img/sq.jpg)  P

## 并行 DDL
并行 DDL

## 并行 DML
并行 DML

## 并行连接
并行连接

### 自适应
自适应

### 布隆过滤
布隆过滤

### 广播分发
广播分发

### 数据缓冲
数据缓冲

### 完全分区智能连接
完全分区智能连接

### 哈希分发
哈希分发

### 部分分区智能连接
部分分区智能连接

### `PQ_DISTRIBUTE`
`PQ_DISTRIBUTE`

### 行源复制
行源复制

## 并行查询
并行查询

### 并行查询服务器集 (PQSS)
并行查询服务器集 (PQSS)

### 通过运行 DFO Q1, 00 和 PQSS2 启动
通过运行 DFO Q1, 00 和 PQSS2 启动

### DFO 树
DFO 树

### 哈希分发
哈希分发

### 哈希连接
哈希连接

### 连接谓词
连接谓词

## 分区粒度
分区粒度

### `PQ_FILTER` 提示参数
`PQ_FILTER` 提示参数

### TQ10001, TQ10002 和 TQ10003
TQ10001, TQ10002 和 TQ10003

## 并行排序
并行排序

## 分区粒度
分区粒度

## 物理数据库设计
物理数据库设计

### 添加索引
添加索引

### 压缩
压缩

### 索引
索引

### `LOB`
`LOB`

### 表
表

### 争用，管理
争用，管理

#### 热点块问题
热点块问题

#### 序列争用
序列争用

### 反规范化
反规范化

### `LOB`s
`LOB`s

### 分区
分区

#### 优势
优势

#### 全表扫描
全表扫描

#### 并行化
并行化

#### 分区智能连接
分区智能连接

### 删除索引
删除索引

#### `DML`
`DML`

#### 外键约束
外键约束

#### `V$OBJECT_USAGE`
`V$OBJECT_USAGE`

### 必需索引识别
必需索引识别

#### 位图索引
位图索引

#### 数据
数据

#### 全局分区索引
全局分区索引



# 性能调优指南

## 索引与优化器

-   索引全扫描
-   索引组织表
-   多列非唯一索引，正确用法
-   列的顺序，多列索引
-   反向键索引
-   单列非唯一 B 树索引，误用
-   执行计划演进
-   PL/SQL
-   PQSS. *参见* 并行查询服务器集 (PQSS)

## 谓词

-   谓词
    -   等式 *vs*. 不等式
    -   当函数应用于列时的连接
    -   分区消除与表达式

## 其他术语

-   前缀表

## 生产提示

-   生产提示
    -   丛林式连接
    -   COUNTRY_SALES_TOTAL
    -   数据库块损坏
    -   数据片段化
    -   执行计划
    -   物化
    -   NO_ELIMINATE_OBY
    -   NO_QUERY_TRANSFORMATION
    -   SH.CUSTOMERS
    -   表空间消耗

## 查询块与对象别名

![images](img/sq.jpg)  Q

-   查询块与对象别名
    -   基于成本的优化器
    -   合并
    -   多种类型
    -   多表插入
    -   简单查询
    -   单表 INSERT 语句
    -   表 T1 和 T2
    -   ALIAS 部分
    -   QB_NAME 提示

## 查询块与子查询

-   查询块与子查询
    -   定义
    -   逻辑顺序
    -   ORDER BY 子句
    -   集合查询块

## 可读性，SQL

![images](img/sq.jpg)  R

-   RBO. *参见* 基于规则的优化器 (RBO)
-   可读性，SQL
    -   DML 语句
    -   内嵌视图
    -   JID
    -   Oracle 数据库
    -   表/数据字典

## 其他术语

-   相对数据块地址
-   远程 SQL

## 重写，SQL 语句

-   重写，SQL 语句
    -   绑定变量
    -   数据字典视图
    -   数据类型转换
    -   表达式，谓词 (*参见* 谓词)
    -   多个相似子查询
    -   临时表 (*参见* 临时表)
    -   UNION、UNION ALL 与 OR

## 根本原因分析与 ROWID

-   根本原因分析
-   ROWID 伪列



## 术语索引

行 ID

批量行访问

大文件表空间

扩展

格式

带 LAST 函数的函数

乐观锁定

Oracle 数据库

行范围，表

小文件表空间的受限 ROWID，部件

单次，表访问

排序更少的列

排序，未索引列

表访问，方法

通用

基于规则的优化器

运行时引擎

操作级数据，显示

一条语句的所有执行 与 操作

`DBMS_XPLAN.DISPLAY_CURSOR`

`V$SQL_PLAN_STATISTICS_ALL`

操作级数据，收集

`GATHER_PLAN_STATISTICS`

SQL 语句

SQL 跟踪

将 `STATISTICS_LEVEL` 设置为 `ALL`

最优、单遍和多遍操作

快捷方式

函数结果缓存

连接快捷方式

OCI 缓存

结果缓存

标量子查询缓存

Snapper，会话级统计信息

SQL 性能监视器

`DBMS_SQLTUNE.REPORT_SQL_MONITOR`

报告

`V$SQL_PLAN_STATISCS_ALL`，缺点

事务一致性与函数调用

工作区

内存分配

操作

`WORKAREA_SIZE_POLICY`，`AUTO`

`WORKAREA_SIZE_POLICY`，`MANUAL`

运行时引擎提示

`APPEND_VALUES` 提示

基于代价的优化器

`DBMS_XPLAN.DISPLAY_CURSOR`

DML 语句

`DRIVING_SITE`

Edition-Based Redefinition

`GATHER_OPTIMIZER_STATISTICS` 提示

`GATHER_PLAN_STATISTICS`

`MONITOR` 提示

`NO_SET_TO_JOIN`

`OPTIMIZER STATISTICS GATHERING`

`ORA-12838`

![images](img/sq.jpg) S

SALES_ANALYTICS

SecureFiles 和 Large Objects 开发人员指南

半连接



-   `null-accepting`
-   标准
-   服务等级协议（SLAs）
-   SLAs. `参见` 服务等级协议（SLAs）

## 排序优化。`另请参阅` 数据密集化

-   添加额外的谓词，优化分析
-   描述
-   基于磁盘的排序
-   重复排序
-   移动平均的首次尝试
-   低效实现，`SALES_ANALYTICS`
-   横向连接，数据字典类型
-   使用一个维度表的横向连接
-   使用两个维度表的横向连接
-   内存限制
-   非排序聚合函数（`参见` 非排序聚合函数）
-   最优
-   `ORDER BY` 子句
-   分页问题
-   并行排序
-   `ROWIDs`（`参见` ROWIDs）

## SQL 特性

-   `数组接口`
-   `AWR`
-   `绑定变量`
-   `CBO`
-   计算
-   `DBA_HIST_SQLTEXT`
-   `DBMS_SQLTUNE_UTIL0`
-   `EMPLOYEE_ID`
-   `HR.EMPLOYEES`
-   连接（`参见` 连接，SQL）
-   字面量
-   `NUL` 字符
-   Oracle 数据库
-   `SELECT` 语句
-   `SQL_ID`
-   子查询因子化（`参见` 子查询因子化，SQL）
-   调优过程
-   `V$SQL`
-   `VARCHAR2` 字符串

## SQL 函数

-   聚合函数
-   与分析结合
-   `GROUP BY` 和分析函数
-   多重排序，比较
-   非排序（`参见` 非排序聚合函数（NSAFs））
-   `PARTITION BY`
-   查询
-   排序

### `分析函数`

-   与聚合分离，使用子查询
-   `COUNT` 函数
-   执行计划
-   `order by` 子子句
-   查询分区子子句
-   `ROWNUM` 和子查询
-   `ROW_NUMBER` 和 `RANK`
-   子查询，使用
-   开窗子子句



categories
单行函数

SQL 性能监控
`DBMS_SQLTUNE.REPORT_SQL_MONITOR`
报告
`V$SQL_PLAN_STATISICS_ALL`，缺点

子查询因子化，SQL
基于成本的优化器 (CBO)
数据字典
`MGR_COUNTS`
多次
`MYPROFITS`
`PEERS`
可读性
递归
`SUBORDINATES`
`UNIT_COST`

![images](img/sq.jpg) T

表压缩
基本
混合列压缩
联机事务处理 (OLTP)
覆盖 PCTFREE

表统计信息
信息，来源 (CBO)
参数
恢复统计信息

TCF. *参见* 基于基数反馈的调优 (TCF)

临时表
描述
工作重复
重复构造，多条 SQL 语句

三遍排序
传递闭包

`TSTATS`
列统计信息
描述
执行计划
ORACLE (*参见* ORACLE)
分区消除异常
基数估计
基于成本的优化器 (CBO)
列和索引
`DBMS_STATS.EXPORT_TABLE_STATS`
动态采样
提取、转换、加载 (ETL) 过程
哈希连接
维护操作
`MERGE` 语句
`NUM_BLOCKS`
`NUM_DISTINCT`
`NUM_PARTITIONS`
永久表
分区维护操作 (PMOP)
统计信息收集过程
`TSTATS.ADJUST_TABLE_STATS`
生产系统
真实世界性能团队
时间敏感数据，移除
`TSTATS` 部署方法

基于基数反馈的调优 (TCF)

调优，SQL 语句
分析
耗时
性能分析
运行时统计信息
语句完成
业务问题
修改代码
添加提示
添加信息
对环境的更改
数据库物理更改
SQL 调优顾问
数据
收集统计信息
SQL 语句
技术问题

两遍排序

![images](img/sq.jpg) U, V, W, X, Y, Z

UNION、UNION ALL 和 OR
避免动态 SQL
不同的结果
不同执行计划的等价语句

协调世界时 (UTC)
不支持的转换

| ![image](img/IOUG_logo.jpg) | 面向全面的技术和数据库专业人士 |

**IOUG** 代表了 **Oracle 技术和数据库专业人士的声音**——通过提供教育、分享**最佳实践**以及提供技术方向和交流机会，帮助您在业务和职业中**提高效率**。

* * *

情境，而不仅仅是内容

IOUG 致力于帮助我们的成员通过实用的内容、以用户为中心的教育以及宝贵的交流和领导机会，站在 Oracle 技术和行业问题的最前沿，成为 `#IOUGenius`：

*   *SELECT Journal* 是我们的季刊，提供关于行业新闻和 Oracle 技术最佳实践的深入、同行评审的文章。
*   我们的 `#IOUGenius` 博客每周精选一个主题，并提供由 Oracle 专业人士和 IOUG 社区驱动的内容。
*   特别兴趣小组为您提供了与同行就您关心的具体问题进行协作的机会，甚至可以在您的组织之外担任领导角色。
*   COLLABORATE 是我们一年一度的机会，可以与三个 Oracle 用户组（IOUG、OAUG 和 Quest）的成员以及 Oracle 社区的顶级专家建立联系。

我们是谁...

...超过 20,000 名数据库专业人士、开发人员、应用和基础设施架构师、商业智能专家和 IT 经理

...一个由用户组成的、为用户服务的社区，分享关于对您和您的组织重要的议题和经验与知识

有兴趣吗？加入 IOUG 的 Oracle 技术和数据库专业人士社区：`www.ioug.org/join`。

独立 Oracle 用户组 | 电话：(312) 245-1579 | 邮箱：`membership@ioug.org`
330 N. Wabash Ave., Suite 2000, Chicago, IL 60611

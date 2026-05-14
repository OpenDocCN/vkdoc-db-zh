# 计划表

```
================================
Plan Table
================================
--------------------------------------+-----------------------------------+
| Id  | Operation           | Name    | Rows  | Bytes | Cost  | Time      |
--------------------------------------+-----------------------------------+
| 0   | SELECT STATEMENT    |         |       |       |     3 |           |
| 1   |  SORT AGGREGATE     |         |     1 |     4 |       |           |
| 2   |   TABLE ACCESS FULL | T       |    15 |    60 |     3 |  00:00:01 |
--------------------------------------+-----------------------------------+
```

## 谓词信息

```
----------------------
2 - filter(("N"<=19 AND "N">=6))
```

## other_xml 列内容

```
===========================
  db_version     : 10.2.0.3
  parse_schema   : OPS$CHA
  plan_hash      : 2966233522
  Outline Data:
  /*+
    BEGIN_OUTLINE_DATA
      IGNORE_OPTIM_EMBEDDED_HINTS
      OPTIMIZER_FEATURES_ENABLE('10.2.0.3')
      ALL_ROWS
      OUTLINE_LEAF(@"SEL$1")
      FULL(@"SEL$1" "T"@"SEL$1")
    END_OUTLINE_DATA
  */
```

## 优化器环境

```
  optimizer_mode_hinted               = false
  optimizer_features_hinted           = 0.0.0
  ...
  ...
  _first_k_rows_dynamic_proration     = true
  _optimizer_native_full_outer_join   = off
  *********************************
  Bug Fix Control Environment
  ***************************
  fix  4611850  = enabled
  fix  4663804  = enabled
  ...
  ...
  fix  4908162  = enabled
  fix  5015557  = enabled
```

## 查询块注册表

```
*********************
SEL$1 0x971d5458 (PARSER) [FINAL]
Optimizer State Dump: call(in-use=23984, alloc=49080),
                      compile(in-use=62200, alloc=108792)
```

`sql_id` 以及执行计划本身之后的所有信息，仅从 Oracle Database `10g` Release 2 开始才可用。初始化参数和错误修复的列表尤其长。因此，从这个版本开始，即使是最简单的 SQL 语句，也会向跟踪文件写入大约 12KB 的数据。生成这样的跟踪文件可能会带来显著的开销。因此，您应该仅在确实需要时才激活事件 `10132`。

可以通过以下方式启用和禁用事件 `10132`：

*   为当前会话启用和禁用事件。
    ```sql
    ALTER SESSION SET events '10132 trace name context forever'
    ALTER SESSION SET events '10132 trace name context off'
    ```
*   为整个数据库启用和禁用事件。警告：此设置不会立即生效，仅对修改后创建的会话生效。
    ```sql
    ALTER SYSTEM SET events '10132 trace name context forever'
    ALTER SYSTEM SET events '10132 trace name context off'
    ```

每个服务器进程将其解析的所有 SQL 语句的信息写入其自己的跟踪文件。这不仅意味着一个跟踪文件可以包含多个 SQL 语句的信息，还意味着当事件在多个会话中启用时，将会使用多个跟踪文件。有关跟踪文件名称和位置的信息，请参阅第 3 章中的“查找跟踪文件”部分。


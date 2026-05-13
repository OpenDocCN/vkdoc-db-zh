# 启用并行性

## 15-2. 为新建对象设置并行度

### 解决方案

为表或索引设置一个高于默认值的并行度（DOP），是启用多进程处理的一种简单直接的方法，能提供更一致和固定的行为。对表或索引启用并行性是通过 DDL 命令完成的。你可以在创建表或索引的 `CREATE` 语句中启用并行性。

对于一个新表，如果你期望有能利用多进程的稳定查询，那么为对象设置一个固定的 DOP 可能比在 SQL 中添加提示（hint）或让 Oracle 为你设置 DOP 更方便。在以下示例中，我们为 `EMP` 表指定了 DOP 为 4：

```sql
CREATE TABLE EMP
(
 EMPNO NUMBER(4) CONSTRAINT PK_EMP PRIMARY KEY,
 ENAME VARCHAR2(10),
 JOB VARCHAR2(9),
 MGR NUMBER(4),
 HIREDATE DATE,
 SAL NUMBER(7,2),
 COMM NUMBER(7,2),
 DEPTNO NUMBER(2) CONSTRAINT FK_DEPTNO REFERENCES DEPT
)
PARALLEL(DEGREE 4);
```

通过在表上放置一个静态的 DOP 值 4，任何访问 `EMP` 表的用户执行的每个查询都将使用 DOP 为 4。

```sql
select * from emp;
```

```
----------------------------------------------------------------------
| Id  | Operation            | Name     |    TQ  |IN-OUT| PQ Distrib |
----------------------------------------------------------------------
|   0 | SELECT STATEMENT     |          |        |      |            |
|   1 |  PX COORDINATOR      |          |        |      |            |
|   2 |   PX SEND QC (RANDOM)| :TQ10000 |  Q1,00 | P->S | QC (RAND)  |
|   3 |    PX BLOCK ITERATOR |          |  Q1,00 | PCWC |            |
|   4 |     TABLE ACCESS FULL| EMP      |  Q1,00 | PCWP |            |
----------------------------------------------------------------------
```

在创建索引时，也可以指定默认的 DOP。在某些情况下，创建索引时使用更高的 DOP 是有益的。对于大型分区表，在 `WHERE` 子句中常用的列上创建二次本地分区索引是很常见的。使用这些索引的一些查询可能会受益于增加 DOP。在以下 DDL 中，我们以 DOP 为 4 创建了这个索引：

```sql
CREATE INDEX EMP_I1
ON EMP (HIREDATE)
LOCAL
PARALLEL(DEGREE 4);
```

### 工作原理

直接在对象上设置并行性有助于多个进程更快地完成任务——无论是加速查询，还是帮助加速索引的创建。为了能够评估为对象设置的合适 DOP，你应该了解数据的访问模式。如果对对象的信息了解越少，设置的 DOP 就应该越保守。为一系列对象设置高 DOP 可能会像它帮助性能一样轻易地损害性能，因此启用对象 DOP 需要经过仔细的规划和考虑。

![images](img/square.jpg) **提示** 如果启用了自动 DOP 并且配置正确（`PARALLEL_DEGREE_POLICY=AUTO`），那么你在对象上设置的并行性将被忽略，优化器会选择要使用的并行度。有关启用自动 DOP 的详细信息，请参见配方 15-10。

## 15-3. 为现有对象启用并行性

### 问题

你有一系列运行缓慢的查询正在访问一组现有的数据库表，并且你希望采取措施来减少查询的执行时间。

### 解决方案

为现有表或索引设置更高的 DOP 是一种更一致和固定的启用多进程的方法。设置表或索引的 DOP 是在 DDL 命令中完成的。你可以使用 `ALTER` 语句来更改表或索引的 DOP。例如，如果你有一个现有表需要更改 DOP 以适应希望利用多进程的用户查询，可以轻松地将其添加到表上，并立即生效。以下示例更改了表的默认 DOP：

```sql
ALTER TABLE EMP
PARALLEL(DEGREE 4);
```

如果过了一段时间，你希望重置表上的 DOP，也可以使用 `ALTER` 语句来完成。查看以下两个示例，了解如何重置表的 DOP：

```sql
ALTER TABLE EMP
PARALLEL(DEGREE 1);
```

```sql
ALTER TABLE EMP
NOPARALLEL;
```

如果你认为一个已存在的索引会受益于更高的 DOP，也可以轻松更改。与表一样，更改会立即生效。以下示例展示了如何更改索引的默认 DOP：

```sql
ALTER INDEX EMP_I1
PARALLEL(DEGREE 4);
```

与表类似，你可以通过以下两种方式之一重置索引的 DOP：

```sql
ALTER INDEX EMP_I4
PARALLEL(DEGREE 1);
```

```sql
ALTER INDEX EMP_I4
NOPARALLEL;
```

### 工作原理

增加现有对象的 DOP 是表明你数据库中访问表的查询已经存在性能问题的标志。监控并行性能是了解为对象或一组对象设置的 DOP 是否合适的关键因素。检查 `V$PQ_TQSTAT` 中的数据以帮助确定已使用的 DOP，或检查 `V$SYSSTAT` 以帮助了解数据库上并行性的使用程度。请参阅配方 15-12 获取使用这些数据字典视图的一些示例。

## 15-4. 实现并行 DML

### 问题

你希望在执行 DML 操作（`INSERT`、`UPDATE`、`MERGE`、`DELETE`）时诱导并行性，以提高性能并减少事务时间。

### 解决方案

如果在数据仓库环境或需要高容量批量事务的大型表环境中操作，并行 DML 可以帮助加快处理速度并减少执行这些操作所需的时间。并行 DML 在数据库中默认是禁用的，必须使用以下语句显式启用：

```sql
ALTER SESSION ENABLE PARALLEL DML;
```

通过指定上述语句，它确实*启用*了会话中并行 DML 的可能性，但并不保证一定会发生。并行 DML 操作只会在特定条件下发生：

*   在 DML 语句中指定了提示（hint）。
*   具有并行属性的表是 DML 语句的一部分。
*   DML 操作满足语句并行运行的适当规则。使用并行 DML 的关键限制在本配方后面部分注明。

在某些情况下，你可能希望强制并行行为，无论你为对象设置了什么并行度，或者无论你在 DML 中放置了什么提示。因此，或者，你可以使用以下语句强制并行 DML：

```sql
ALTER SESSION FORCE PARALLEL DML;
```

作为一般规则，在常规运行的 DML 中强制使用并行 DML 不是一个好的做法，因为它会很快消耗系统资源，导致性能开始下降。最好谨慎使用，它可以帮助处理偶尔的大型 DML 操作。



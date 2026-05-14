# Oracle SQL 跟踪文件分析报告

## 执行摘要

首先，我们观察到 PL/SQL 块（`Statement ID` 1）占据了绝大部分的处理时间。

`Total` | `Number of` | `Duration per`
`Statement ID` `Type` | `Duration` | `%` | `Executions` | `Execution`
------------ | ------------- | -------- | ------- | ----------- | ------------
`1` | PL/SQL | 5.333 | `94.119` | 1 | 5.333
`35` | PL/SQL | 0.010 | 0.184 | 1 | 0.010
------------ | ------------- | -------- | -------
`Total` | | 5.343 | 94.303

## 深入分析 SQL 语句

下一步自然是获取关于处理时间最长的 SQL 语句的详细信息。为了便于引用，TVD$XTAT 为每个 SQL 语句生成一个 ID（即上文摘录中的 `Statement ID` 列）。在输出文件的 HTML 版本中，您可以直接点击该 ID 跳转到对应的 SQL 语句详情。而在纯文本版本中，则需要搜索字符串 `"STATEMENT 1"`。

每个 SQL 语句会提供以下信息：执行环境的通用信息、SQL 语句文本、执行统计信息、执行计划、执行中使用的绑定变量以及等待事件。其中，执行计划、绑定变量和等待事件是可选的，仅当它们在跟踪文件中被记录时才会显示。

### SQL 语句 1 详情

首先显示的是执行环境的通用信息和 SQL 语句文本。请注意，会话属性信息仅在可用时才显示。例如，此处未显示属性 `action name`，因为应用程序未设置它。还需注意，`SQL ID` 仅在 Oracle Database 11*g* 及以上版本中才可用。

`STATEMENT 1`

`Session ID` | `90.6`
--- | ---
`Service Name` | `DBM11106.antognini.ch`
`Module Name` | `SQL*Plus`
`Parsing User` | `33`
`Hash Value` | `2276506700`
`SQL ID` | `3mfcj1a3v1g2c`

```
DECLARE
  l_channel_id sh.sales.channel_id%TYPE := 3;
BEGIN
  FOR c IN (SELECT cust_id, extract(YEAR FROM time_id), sum(amount_sold)
            FROM sh.sales
            WHERE channel_id = l_channel_id
            GROUP BY cust_id, extract(YEAR FROM time_id))
  LOOP
    NULL;
  END LOOP;
END;
```

### 执行统计信息

执行统计信息以表格形式提供了按数据库调用类型聚合的数据。由于表格布局基于 TKPROF 生成的格式，各列的含义相同。然而，此处有两个额外的列：`Misses` 和 `LIO`。前者表示每次调用期间发生的硬解析次数。后者仅是 `Consistent` 和 `Current` 两列之和。还需注意，TVD$XTAT 提供了两个表格。第一个表格包含了与当前语句相关的所有递归 SQL 语句的统计信息。第二个表格与 TKPROF 类似，不包含它们。

#### 包含递归语句的统计信息

`Database Call Statistics with Recursive Statements`
`--------------------------------------------------`

`Call` | `Count` | `Misses` | `CPU` | `Elapsed` | `PIO` | `LIO` | `Consistent` | `Current` | `Rows`
--- | --- | --- | --- | --- | --- | --- | --- | --- | ---
`Parse` | 1 | 1 | 0.024 | 0.162 | 7 | 55 | 55 | 0 | 0
`Execute` | 1 | 0 | 1.430 | 5.483 | 2,726 | 5,239 | 5,239 | 0 | 1
`Fetch` | 0 | 0 | 0.000 | 0.000 | 0 | 0 | 0 | 0 | 0
--- | --- | --- | --- | --- | --- | --- | --- | --- | ---
`Total` | 2 | 1 | 1.454 | 5.645 | 2,733 | 5,294 | 5,294 | 0 | 1

#### 不包含递归语句的统计信息

`Database Call Statistics without Recursive Statements`
`-----------------------------------------------------`
`Call` | `Count` | `Misses` | `CPU` | `Elapsed` | `PIO` | `LIO` | `Consistent` | `Current` | `Rows`
--- | --- | --- | --- | --- | --- | --- | --- | --- | ---
`Parse` | 1 | 1 | 0.007 | 0.046 | 0 | 0 | 0 | 0 | 0
`Execute` | 1 | 0 | 0.011 | 0.007 | 0 | 0 | 0 | 0 | 1
`Fetch` | 0 | 0 | 0.000 | 0.000 | 0 | 0 | 0 | 0 | 0
--- | --- | --- | --- | --- | --- | --- | --- | --- | ---
`Total` | 2 | 1 | 0.018 | 0.053 | 0 | 0 | 0 | 0 | 1

在此案例中，根据执行统计信息，当前 SQL 语句本身消耗的时间很少。下面的资源使用概要文件也印证了这一点。事实上，它显示几乎 100% 的时间都消耗在了递归 SQL 语句上。

### 资源使用概要

`Resource Usage Profile`
`----------------------`

`Component` | `Total Duration` | `%` | `Number of Events` | `Duration per Event`
--- | --- | --- | --- | ---
`recursive statements` | 5.312 | `99.618` | n/a | n/a
`CPU` | 0.018 | 0.338 | n/a | n/a
`SQL*Net message from client` | 0.002 | 0.045 | 1 | 0.002
`SQL*Net message to client` | 0.000 | 0.000 | 1 | 0.000
--- | --- | --- | --- | ---
`Total` | 5.333 | 100.000 | |

## 识别递归 SQL 语句

为了展示这些递归 SQL 语句具体是哪些，资源使用概要后面列出了递归 SQL 语句列表。从这个列表中可以看出，`SELECT` 语句（`Statement ID` 2）大约导致了 98% 的响应时间。请注意，所有其他 SQL 语句都是由数据库引擎自身生成的（例如，在解析阶段），因此被标记为 `SYS recursive`。

`10 recursive statements were executed.`

`Statement ID` | `Type` | `Total Duration` | `%`
--- | --- | --- | ---
`2` | SELECT | 5.229 | `98.061`
`12` | SELECT (SYS recursive) | 0.030 | 0.571
`17` | SELECT (SYS recursive) | 0.024 | 0.453
`32` | SELECT (SYS recursive) | 0.012 | 0.225
`44` | SELECT (SYS recursive) | 0.006 | 0.121
`57` | SELECT (SYS recursive) | 0.003 | 0.056
`63` | SELECT (SYS recursive) | 0.002 | 0.038
`64` | SELECT (SYS recursive) | 0.002 | 0.038
`70` | SELECT (SYS recursive) | 0.002 | 0.037
`106` | SELECT (SYS recursive) | 0.001 | 0.019
--- | --- | --- | ---
`Total` | | 4.819 | 90.362

## 深入分析 SQL 语句 2

由于 SQL 语句 2 是导致响应时间的主要原因，我们需要进一步深入分析其详细信息。其结构与 SQL 语句 1 基本相同，但包含额外信息。在显示执行环境的部分，您可以看到递归级别（请记住，由应用程序执行的 SQL 语句处于级别 0）和父 SQL 语句 ID。这第二部分信息对于不丢失 SQL 语句之间的关系至关重要（而 TKPROF 会丢失！）。

`STATEMENT 2`

`Session ID` | `90.6`
--- | ---
`Service Name` | `DBM11106.antognini.ch`
`Module Name` | `SQL*Plus`
`Parsing User` | `33`
`Recursive Level` | 1
`Parent Statement ID` | 1
`Hash Value` | 1624534809
`SQL ID` | `g4h8jndhd8vst`

```
SELECT CUST_ID, EXTRACT(YEAR FROM TIME_ID), SUM(AMOUNT_SOLD)
FROM SH.SALES
WHERE CHANNEL_ID = :B1
GROUP BY CUST_ID, EXTRACT(YEAR FROM TIME_ID)
```



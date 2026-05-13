# 函数返回布尔值且非空

非空函数模式的一种特定情况是函数的 `RETURN` 数据类型为 `BOOLEAN`。这类函数可以安全地在条件语句中被引用，以实现强大且富有表现力的逻辑。

例如，假设存在一个过程，用于对某种类型 `my_rectype` 的记录执行某些复杂处理，但这些记录必须通过一些复杂的准则才能成功处理。这些准则实际上是处理过程的前置条件，或者可能是该过程不要求但程序逻辑希望强加的附加准则。这些准则可以封装在一个函数中，该函数的契约是测试这些准则并明确返回 `TRUE` 或 `FALSE`。

```sql
FUNCTION ready_to_process (rec1 my_rectype) RETURN BOOLEAN;
```

执行记录处理的代码可能如下所示：

```sql
IF ready_to_process(rec1=>local_rec)
THEN
  process_record(rec1=>local_rec);
ELSE
  log_error;
END IF;
```

再次强调，无需处理 `NULL` 函数输出，再加上富有表现力的函数名，使得此类代码的意图清晰明了。


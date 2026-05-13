# 在 WHERE 子句中处理空值

在`WHERE`子句中，如果你只是想检查一个列是否包含空值，请使用`IS NULL`或`IS NOT NULL`作为比较操作符——例如：

```
SELECT ename , sal
FROM emp
WHERE comm IS NULL
ORDER BY ename;
```

```
ENAME              SAL
---------- ----------
ADAMS            1100
BLAKE            2850
KING             5000
```

```
SELECT ename , sal
FROM emp
WHERE comm IS NOT NULL
ORDER BY ename;
```

```
ENAME              SAL
---------- ----------
ALLEN            1600
MARTIN           1250
```

你也可以在`WHERE`子句中使用`NVL`或`NVL2`函数，就像在`SELECT`语句中一样：

```
SELECT ename , sal
FROM emp
WHERE NVL(comm,0) = 0
ORDER BY ename;
```

```
ENAME              SAL
---------- ----------
ADAMS            1100
BLAKE            2850
KING             5000
```

## 工作原理

最好始终明确处理空值的可能性，因此如果表的一列可为空，就假设空值存在，否则输出结果可能不理想或不可预测。一个快速检查列中是否存在空值的方法是将表中的行数（`COUNT *`）与该列的行数（`COUNT <column_name>`）进行比较。对可为空列的计数将仅计算那些没有空值的行。以下是一个例子：

```
SELECT count(*) FROM emp;
```

```
  COUNT(*)
----------
        14
```

```
SELECT count(comm) FROM emp;
```

```
COUNT(COMM)
-----------
          4
```

这种比较行数与列中值数量的技术是检查列中是否存在空值的一种便捷方法。

另一个在处理空值时非常有用的函数是`COALESCE`函数。使用`COALESCE`，你可以传入一系列值，该函数将返回第一个非`NULL`值。如果`COALESCE`中的所有值都是`NULL`，则返回`NULL`值。这里有一个简单的例子：

```
SELECT coalesce(NULL,'ABC','DEF') FROM dual;
```

```
COA
---
ABC
```

假设你想获取客户的送货地址，如果不存在，则再获取账单地址。使用`COALESCE`，你可以如下例所示实现这一点：

```
SELECT COALESCE(
(SELECT shipping_address FROM customers
WHERE cust_id = 9342),
(SELECT billing_address FROM customers
WHERE cust_id = 9342))
FROM dual;
```

在使用`COALESCE`的语句中，所有参数必须具有相同的数据类型，否则你将收到错误，如下所示：

```
SELECT coalesce(NULL,123,'DEF') FROM dual;
                         *
ERROR at line 1:
ORA-00932: inconsistent datatypes: expected NUMBER got CHAR
```

### 8-12. 搜索部分列值

### 问题

你需要从数据库中某个列搜索一个字符串，但不知道该列数据的确切值。

#### 解决方案

当你对`WHERE`子句中用于筛选的列的数据值不确定时，可以使用`LIKE`操作符。与等号、`BETWEEN`子句或`IN`子句等常规比较操作符不同，`LIKE`操作符允许你基于列数据的*部分字符串*来搜索匹配项。使用`LIKE`子句时，你还需要在数据本身中使用`%`符号或`_`符号来查找所需数据。百分比符号用于替代一个到多个字符。

例如，如果你想查看在 1995 年雇佣的员工列表，而不关心具体日期，就可以使用`LIKE`子句来搜索`hire_date`中包含字符串`1995`的任何匹配项。当对日期或时间戳数据类型使用`LIKE`时，你需要确保所使用的日期格式与你`LIKE`语句中的搜索条件兼容。例如，如果你数据库的默认日期格式是`DD-MON-YY`，那么字符串`1995`与此格式不兼容，将永远找不到匹配项。为了以此方式搜索，需要在发出查询前，在你的会话中设置日期格式：

`alter session set nls_date_format = 'yyyy-mm-dd';`

`Session altered.`

```
SELECT employee_id, last_name, first_name, hire_date
FROM employees
WHERE hire_date LIKE '%1995%'
ORDER BY hire_date;
```

```
EMPLOYEE_ID LAST_NAME                 FIRST_NAME           HIRE_DATE
----------- ------------------------- -------------------- ----------
        122 Kaufling                  Payam                1995-05-01
        115 Khoo                      Alexander            1995-05-18
        137 Ladwig                    Renske               1995-07-14
        141 Rajs                      Trenna               1995-10-17
```

解决需要担心会话日期格式的一个简单方法是在查询中直接使用`TO_CHAR`函数。这种方法的优势在于编码非常简单，无需担心会话的日期格式。请看以下示例：

```
SELECT employee_id, last_name, first_name, hire_date
FROM employees
WHERE to_char(hire_date,'yyyy') = '1995'
ORDER BY hire_date;
```

下划线符号（`_`）用于精确替代一个字符。假设你正在寻找姓氏为“Olsen”或“Olson”的员工，但不确定具体拼写。你可以在单个查询中，结合下划线和`LIKE`子句来找出数据库中所有符合该名称变体的员工：

```
SELECT last_name, first_name, phone_number
FROM employees
WHERE last_name like 'Ols_n';
```

```
LAST_NAME                 FIRST_NAME           PHONE_NUMBER
------------------------- -------------------- --------------------
Olsen                     Christopher          011.44.1344.498718
Olson                     TJ                   650.124.8234
```


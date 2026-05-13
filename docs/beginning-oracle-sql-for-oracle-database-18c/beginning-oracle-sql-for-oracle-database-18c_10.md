# 第十一章：数据排序

## 11.1 当值不完全匹配时使用 BETWEEN

`BETWEEN` 操作符也可用于数据与您指定的上下限值不完全匹配的情况。要查找 `salary`（工资）值在 50,000 到 100,000 之间的 `employee`（员工）记录，您可以编写如下查询：

```sql
SELECT id, last_name, salary
FROM employee
WHERE salary BETWEEN 50000 AND 100000;
```

该查询的结果是：

```
ID   LAST_NAME   SALARY
4    SIMPSON     52000
8    SMITH       62000
```

您会注意到几点。首先，没有值恰好是 50,000 或 100,000，但这没关系。在此范围内的值会被显示出来。此外，`NULL` 值不会被显示。`NULL` 被视为未知值，因此数据库无法判断那些 `NULL` 值是否在此范围内。

## 11.2 对文本值使用 BETWEEN

您也可以将文本值与 `BETWEEN` 关键字一起使用。假设您想查找所有介于 'ANDERSON' 和 'MOON' 之间的 `last_name`（姓氏）值。您可以编写如下查询：

```sql
SELECT id, last_name, salary
FROM employee
WHERE last_name BETWEEN 'ANDERSON' AND 'MOON';
```

我们的结果将显示如下：

```
ID   LAST_NAME   SALARY
1    JONES       20000
3    KING        40000
5    ANDERSON    31000
6    COOPER      NULL
```

这个结果是通过检查每个 `last_name` 值的字符，看它们如果按字母顺序排序，是否在 'ANDERSON' 之后且在 'MOON' 之前来确定的。这三个值都符合。

## 11.3 包含性与排他性检查示例

这是一个使用 `BETWEEN` 子句无法达到预期效果的例子。假设您需要查找所有 `salary` 大于或等于 20000 且小于 40000 的 `employee` 记录。如果他们的 `salary` 是 40000，则不应包含在内。不使用 `BETWEEN` 的查询可能如下所示：

```sql
SELECT id, last_name, salary
FROM employee
WHERE salary >= 20000
AND salary < 40000;
```

请注意，第一个 `WHERE` 条件使用 “>=” 来表示“大于或等于 20000”。第二个条件使用 “<” 来表示“小于 40000”。此查询将返回以下结果：

```
ID   LAST_NAME   SALARY
1    JONES       20000
2    SMITH       35000
5    ANDERSON    31000
```

如果您使用 `BETWEEN` 来编写这个查询，它将看起来与之前的查询相同，但会显示不同的结果。

```sql
SELECT id, last_name, salary
FROM employee
WHERE salary BETWEEN 20000 AND 40000;
```

结果是不同的：

```
ID   LAST_NAME   SALARY
1    JONES       20000
2    SMITH       35000
3    KING        40000
5    ANDERSON    31000
9    PATRICK     40000
```

这是需要注意的地方。如果您需要使用“大于或等于”和“小于或等于”，那么 `BETWEEN` 是适用的，否则它就不适用。您能直接说 `BETWEEN` 20000 和 39999 吗？

可以，但怎么能保证没有 39999.50 的工资呢？您如何有效地找出小于 40000 的最高值？这并不容易做到。日期字段也存在同样的问题，本书后面会讨论。

## 11.4 是否应该使用 BETWEEN？

您已经看到 `BETWEEN` 关键字可以作为“大于或等于”和“小于或等于”逻辑（也称为“包含性”）的替代方案。您应该使用哪种方法？

以下是我的建议：如果您确实在寻找一个值的范围，并且您的规则要求它是包含性的，那么您可以使用 `BETWEEN`。如果不是，我建议使用其他符号。表 10-1 对此规则提供了一个快速参考。

表 10-1：何时使用 `BETWEEN` 操作符

| 是否在寻找值范围？ | 是否包含指定值？ | 使用 BETWEEN 或 运算符 |
| --- | --- | --- |
| 是 | 是 | BETWEEN |
| 是 | 否 | 运算符 |
| 否 | 是 | 运算符 |
| 否 | 否 | 运算符 |

## 11.5 总结

`IN` 关键字允许您指定多个值来进行“等于”检查。它比使用多个 `OR` 关键字更容易输入和阅读，并且可以与任何数据类型一起使用。

`BETWEEN` 关键字是一个有用的概念，它允许您为一个必须在另外两个值之间的值指定 `WHERE` 子句。它是一个易于编写和理解的“大于或等于”与“小于或等于”的组合，但如果需要，无法修改为使用“大于”或“小于”。


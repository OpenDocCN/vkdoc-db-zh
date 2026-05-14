# 第 25 章 ■ 查询优化与执行

## 联接

解析和绑定阶段，并在优化过程中用物理运算符替换它们。例如，一个 `inner join`（内联接）逻辑运算符可以在执行计划中被替换为三种物理联接运算符之一。

本章不可能涵盖所有物理运算符；但是，我们将讨论一些在执行计划中经常遇到的常见运算符。

SQL Server 中有多种物理联接运算符的变体，它们决定了联接谓词的匹配方式以及结果行中包含的内容。然而，就算法而言，只有三种联接类型：`nested loop`（嵌套循环）、`merge`（合并）和 `hash`（哈希）联接。

### 嵌套循环联接

`nested loop join`（嵌套循环联接）是最简单的联接算法。与任何联接类型一样，它接受两个输入，分别称为 `outer`（外表）和 `inner`（内表）。内嵌套循环联接的算法如 清单 25-6 所示，而外嵌套循环联接的算法如 清单 25-7 所示。

```
for each row R1 in outer table
  for each row R2 in inner table
    if R1 joins with R2
      return join (R1, R2)
```

```
for each row R1 in outer table
  for each row R2 in inner table
    if R1 joins with R2
      return join (R1, R2)
    else
      return join (R1, NULL)
```

如你所见，该算法的成本取决于输入的大小，并与其乘积成正比；即外表的大小乘以内表的大小。成本随着输入的大小快速增长；因此，当至少一个输入很小时，嵌套循环联接是高效的。在等式联接谓词的情况下，如果内表的谓词列上有索引也是有益的。这有助于避免执行过程中的 `index scan`（索引扫描）操作。

嵌套循环联接不要求联接键必须具有等式谓词。SQL Server 会评估两个输入中每一行之间的联接谓词。事实上，它甚至不要求必须有联接谓词。例如，`CROSS JOIN`（交叉联接）逻辑运算符将导致一个嵌套循环物理联接，其中两个输入的所有行都被联接在一起。

### 合并联接

`merge join`（合并联接）适用于两个已排序的输入。它一次比较两行，如果相等，则将其联接结果返回给客户端。否则，它会丢弃值较小的那一行，并转到该输入中的下一行。与嵌套循环联接相反，合并联接要求联接键上至少有一个等式谓词。清单 25-8 展示了内合并联接的算法。

```
/* Pre-requirements: Inputs I1 and I2 are sorted */
get first row R1 from input I1
get first row R2 from input I2
while not end of either input
begin
  if R1 joins with R2
  begin
    return join (R1, R2)
    get next row R2 from I2
  end
  else if R1 < R2
    get next row R1 from I1
  else /* R1 > R2 */
    get next row R2 from I2
end
```

合并联接算法的成本与两个输入大小的总和成正比，这使得它在处理大输入时比嵌套循环联接更高效。然而，合并联接要求两个输入都是已排序的，当输入在联接键列上有索引时通常就是这种情况。

在某些情况下，SQL Server 可能会在合并联接之前决定使用 `Sort`（排序）运算符对输入进行排序。显然，在分析时，排序的成本需要与联接运算符的成本一起考虑。你也可以考虑创建索引来预排序数据。

### 哈希联接

与在小型输入上效果最佳的嵌套循环联接，以及在已排序输入上表现出色的合并联接不同，`hash join`（哈希联接）旨在处理大型、未排序的输入。哈希联接算法包含两个不同的阶段。

在第一个阶段，即 `build`（生成）阶段，哈希联接扫描其中一个输入（通常是较小的那个），计算



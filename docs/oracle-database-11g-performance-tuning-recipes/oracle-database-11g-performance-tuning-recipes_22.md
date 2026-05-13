# 在 SELECT 子句中处理空值

在`SELECT`子句中，如果你正在处理包含空值的列数据，有两种 Oracle 提供的函数可以在 SQL 中使用，以将空值转换为更可用的形式。这两个函数是`NVL`和`NVL2`。

![图片](img/square.jpg) 实际上，不仅仅只有`NVL`和`NVL2`这两个函数。然而，它们被广泛使用，是一个很好的起点。

使用`NVL`时，你只需传入列名，以及一个基于该值在数据库中是否为空而你想输出的值。例如，在我们的 employees 表中，并非所有员工都根据其职位获得佣金，该列中这些员工的值为空：

```
SELECT ename, sal, comm
FROM emp
ORDER BY ename;
```

```
ENAME              SAL       COMM
---------- ---------- ----------
ADAMS            1100
ALLEN            1600        300
BLAKE            2850
KING             5000
MARTIN           1250       1400
```

如果我们只是想在佣金列中为没有资格获得佣金的员工显示零，我们可以使用`NVL`函数来实现：

```
SELECT ename , sal , NVL(comm,0) comm
FROM emp
ORDER BY ename;
```

```
ENAME              SAL       COMM
---------- ---------- ----------
ADAMS            1100          0
ALLEN            1600        300
BLAKE            2850          0
KING             5000          0
MARTIN           1250       1400
```

如果我们决定对空值执行算术运算，结果将始终为空；因此，如果我们想计算“总薪酬”为工资加佣金，我们必须应用`NVL`函数，以正确计算并考虑空值。在以下示例中，我们分别使用和不使用`NVL`函数计算这些列的总和。不使用`NVL`，我们会得到一个不正确的结果，这可以在`TOTAL_COMP_NO_NVL`输出字段中看到：

```
SELECT ename , sal , nvl(comm,0) comm, sal+comm total_comp_no_nvl,
sal+NVL(comm,0) total_comp_nvl
FROM emp
ORDER BY ename;
```

```
ENAME              SAL       COMM TOTAL_COMP_NO_NVL TOTAL_COMP_NVL
---------- ---------- ---------- ----------------- --------------
ADAMS            1100          0                                 1100
ALLEN            1600        300               1900            1900
BLAKE            2850          0                                 2850
KING             5000          0                                 5000
MARTIN           1250       1400               2650            2650
```

`NVL2`与`NVL`类似，区别在于`NVL2`接受三个参数——值或列，如果列不为空则返回的值，以及如果列为空则返回的值。例如，如果我们使用前面相同的例子来确定员工是否获得佣金，我们只想为每位员工分配一个值，说明他或她是“有佣金”还是“无佣金”员工。我们可以使用`NVL2`函数实现这一点：

```
SELECT ename , sal ,
NVL2(comm,'Commissioned','Non-Commissioned') comm_status
FROM emp
ORDER BY ename;
```

```
ENAME              SAL COMM_STATUS
---------- ---------- ----------------
ADAMS            1100 Non-Commissioned
ALLEN            1600 Commissioned
BLAKE            2850 Non-Commissioned
KING             5000 Non-Commissioned
MARTIN           1250 Commissioned
```


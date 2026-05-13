# 6.2.4 `aggregate()`

上一节介绍了 `MapReduce` 函数。本节，你将初步了解 MongoDB 的聚合框架。

聚合框架让你无需使用 `MapReduce` 函数即可获取聚合值。从性能上讲，聚合框架比 `MapReduce` 函数更快。你需要始终记住，`MapReduce` 适用于批处理方法，而非实时分析。

接下来，你将使用 `aggregate` 函数描绘上述讨论的两个输出结果。第一个输出是找出男女生的数量，可通过执行以下命令实现：

```
> db.students.aggregate({$group:{_id:"$Gender", totalStudent: {$sum: 1}}})
{ "_id" : "F", "totalStudent" : 6 }
{ "_id" : "M", "totalStudent" : 9 }
>
```

类似地，要找出按班级分类的平均分，可以执行以下命令：

```
> db.students.aggregate({$group:{_id:"$Class", AvgScore: {$avg: "$Score"}}})
{ "_id" : "Biology", "AvgScore" : 90 }
{ "_id" : "C3", "AvgScore" : 90 }
{ "_id" : "Chemistry", "AvgScore" : 90 }
{ "_id" : "C2", "AvgScore" : 93 }
{ "_id" : "C1", "AvgScore" : 85 }
>
```


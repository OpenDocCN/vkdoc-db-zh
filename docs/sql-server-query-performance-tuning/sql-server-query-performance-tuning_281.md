# 第 18 章：查询设计分析
第 18 章：查询设计分析
## 查询设计建议
查询设计建议
## 操作小型结果集
操作小型结果集
### 限制 select_list 中的列数
限制 `select_list` 中的列数
### 使用高度选择性的 WHERE 子句
使用高度选择性的 `WHERE` 子句
## 有效使用索引
有效使用索引
### 避免非 SARGable 搜索条件
避免非 SARGable 搜索条件
#### BETWEEN vs. IN/OR
`BETWEEN` vs. `IN/OR`
### LIKE 条件
`LIKE` 条件
### !< 条件 vs. >= 条件
`!<` 条件 vs. `>=` 条件
### 避免在 WHERE 子句列上使用算术运算符
避免在 `WHERE` 子句列上使用算术运算符
### 避免在 WHERE 子句列上使用函数
避免在 `WHERE` 子句列上使用函数
#### SUBSTRING vs. LIKE
`SUBSTRING` vs. `LIKE`
### 日期部分比较
日期部分比较
### 避免优化器提示
避免优化器提示
### JOIN 提示
`JOIN` 提示
### INDEX 提示
`INDEX` 提示
## 使用域和引用完整性
使用域和引用完整性
### NOT NULL 约束
`NOT NULL` 约束
### 声明性引用完整性
声明性引用完整性
## 小结
小结

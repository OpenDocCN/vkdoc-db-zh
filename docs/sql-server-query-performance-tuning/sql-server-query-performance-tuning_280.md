# 第 17 章：查询重编译
第 17 章：查询重编译
## 重编译的好处与缺点
重编译的好处与缺点
## 识别导致重编译的语句
识别导致重编译的语句
### 分析重编译的原因
分析重编译的原因
#### 架构或绑定更改
架构或绑定更改
#### 统计信息更改
统计信息更改
#### 延迟对象解析
延迟对象解析
#### 因常规表导致的重编译
因常规表导致的重编译
#### 因局部临时表导致的重编译
因局部临时表导致的重编译
#### SET 选项更改
`SET` 选项更改
#### 执行计划老化
执行计划老化
#### 显式调用 sp_recompile
显式调用 `sp_recompile`
#### 显式使用 RECOMPILE
显式使用 `RECOMPILE`
#### CREATE PROCEDURE 语句中的 RECOMPILE 子句
`CREATE PROCEDURE` 语句中的 `RECOMPILE` 子句
#### EXECUTE 语句中的 RECOMPILE 子句
`EXECUTE` 语句中的 `RECOMPILE` 子句
#### 用于控制单个语句的 RECOMPILE 提示
用于控制单个语句的 `RECOMPILE` 提示
## 避免重编译
避免重编译
### 不要混合 DDL 和 DML 语句
不要混合 DDL 和 DML 语句
### 避免由统计信息更改引起的重编译
避免由统计信息更改引起的重编译
#### 使用 KEEPFIXED PLAN 选项
使用 `KEEPFIXED PLAN` 选项
### 在表上禁用自动更新统计信息
在表上禁用 `AUTO_UPDATE_STATISTICS`
#### 使用表变量
使用表变量
#### 避免在存储过程中更改 SET 选项
避免在存储过程中更改 `SET` 选项
#### 使用 OPTIMIZE FOR 查询提示
使用 `OPTIMIZE FOR` 查询提示
#### 使用计划指南
使用计划指南
## 小结
小结

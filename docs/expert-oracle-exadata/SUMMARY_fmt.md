
# Oracle Exadata 高级指南

## 引言

## 乐趣无穷！

## 致谢

## 目录

## 什么是 Exadata？

## Exadata 核心概念：卸载与性能优化

## SQL 执行计划与列投影分析

## Exadata 智能扫描深度解析

## SQL 监控

## 总结

## 3. 混合列式压缩

### 创建 HCC 压缩表时会发生什么？

### 压缩对收集统计信息的影响

### DML 性能

### 压缩

### 自动数据优化 vs. 手动生命周期管理

### 启用 ILM 实现存储分层

## 4. 存储索引

### 空值的特殊优化

## 5. `Exadata 智能闪存缓存`

### Exadata X5-2 存储服务器中的闪存

### 使用闪存作为缓存

### Exadata 写回闪存缓存与智能扫描优化

### 启用回写闪存缓存

### Oracle SQL 性能分析

### Exadata 闪存缓存监控与统计

## 6. Exadata 并行操作

### 数据库并行参数配置与存储层并行化

### IORM 相关指标概览

### 后台进程

### 总结

## 配置 Exadata

### Exadata 配置界面

### 第 4 步：运行 CheckIP 以验证网络就绪状态

### Exadata 安装与升级指南

### 配置数据库服务器

### 扩展 Exadata 存储

### 总结

## 9. Exadata 的恢复

### Exadata 存储监控与 `sundiag.sh` 脚本

### `make_cellboot_usb` 脚本日志

### 单元救援选项

### 单元磁盘故障

### 定位并更换故障磁盘

## 10. Exadata 等待事件

### 单元智能表扫描

### Exadata 等待事件

### 系统 I/O 类中的 Exadata 等待事件

### 等待事件详解

## 第 11 章 Exadata 性能指标

### 在 Oracle 执行计划中识别 Smart Scan 迹象

### 理解 Exadata Smart Scan 指标与性能计数器

### Oracle Exadata 统计信息

### 行迁移与链接：对性能的影响及处理策略

### EHCC 相关计数器

### cellsrvstat 实用程序

## 12. 监控 Exadata 性能

### 使用 V$SQL 和 V$SQLSTATS 监控 SQL 语句

### Exadata IORM 页面可用于简单地按数据库监控时间序列工作负载分布，或配置 IO 资源管理设置。该页面布局清晰，能让您了解各数据库的平均节流时间、利用率、小 IO 延迟和磁盘 IO 目标等关键信息，这些信息是制定通用资源管理和 IORM 计划时所必需的。结合前两个页面（存储服务器网格主页和性能视图），可以更容易地验证性能下降的原因是由于资源争用、容量问题，还是数据库内部的问题。I/O 资源管理器将在第 7 章中更详细地介绍。

### Exadata I/O 性能分析与 iostat 使用

### ExaWatcher 数据存储与配置

### 使用 GetExaWatcherResults.sh 脚本

### 查看 iostat 输出示例

### 理解时间戳格式

## 13. 迁移到 Exadata

### 数据泵

### 现在让我们将注意力转向导入过程

### Oracle 数据迁移：Streams 与 Golden Gate 对比

### 跨平台表空间传输到 Exadata

## 14. 存储布局

### Exadata 存储配置：磁盘组与网格磁盘

### 数据库范围安全配置

## 15. 计算节点布局

### 可扩展性与性能



+   [多 RAC 集群配置](expert-oracle-exadata_75.md)
+   [典型 Exadata 配置](expert-oracle-exadata_76.md)
+   [16. Exadata 补丁管理](expert-oracle-exadata_77.md)
+   [Exadata 补丁应用指南](expert-oracle-exadata_78.md)
+   [存储服务器补丁详解](expert-oracle-exadata_79.md)
+   [使用 `dbnodeupdate.sh` 应用补丁](expert-oracle-exadata_80.md)
+   [升级 Exadata 计算节点](expert-oracle-exadata_81.md)
+   [17. 摒弃一些我们曾自以为知道的东西](expert-oracle-exadata_82.md)
+   [Exadata 性能测试：单元格关闭与直接路径读取的影响](expert-oracle-exadata_83.md)
+   [执行统计信息分析：Flash Cache 优化效果](expert-oracle-exadata_84.md)
+   [建立索引还是不建立索引？](expert-oracle-exadata_85.md)
+   [`CELLCLI` 与 `DCLI`](expert-oracle-exadata_86.md)
+   [在数据库中使用 `cellcli` XML 输出](expert-oracle-exadata_87.md)
+   [在线 Exadata 资源](expert-oracle-exadata_88.md)
+   [`exachk`](expert-oracle-exadata_89.md)
+   [索引](expert-oracle-exadata_90.md)

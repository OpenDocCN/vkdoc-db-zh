# R 绘图与表格代码（适用于 Visual Studio 的 R 工具）

首先，关于本附录的几点说明：

*   本附录提供了第 5 章中示例的完整代码，但你仍然需要从 `Weather_Sample.csv` 文件将数据导入 RTVS，才能实际运行。
*   你可能不需要运行 `install.packages()` 引用，但它们已列出以备不时之需。如果已经安装过相应的包，则无需再次安装。

### 按机场 ID 的平均温度（绘图）

此代码以图形格式给出按机场 ID 的平均温度。请注意，`mean()` 函数包含了 `DryBulbFarenheit` 属性，表明关键字段是温度而非风速。另外，`ggplot2` 的引用表明这是绘图代码，而非表格代码。

```r
install.packages("data.table")
library(data.table)
Weather_Sample <- data.table(Weather_Sample)
chart_by_ID <- as.data.frame(Weather_Sample[, mean(DryBulbFarenheit, na.rm = TRUE), by = AirportID])
install.packages("ggplot2")
library(ggplot2)
ggplot(chart_by_ID, aes(x = AirportID, y = V1)) + geom_point(stat = "identity") + geom_smooth(method = "lm", formula = y ∼ splines::bs(x, 3)) + scale_x_continuous(name = "Airport ID") + scale_y_continuous(name = "Average Temperature") +
geom_text(aes(label = AirportID), size = 3, vjust = 1.0) +
geom_text(aes(label = round(V1, digits = 2)), size = 3, vjust = 2.0)
```

### 按机场 ID 的平均温度（表格）

此代码以表格格式给出按机场 ID 的平均温度。请注意，`mean()` 函数包含了 `DryBulbFarenheit` 属性，表明关键字段是温度而非风速。另外，没有 `ggplot2` 的引用表明这是表格代码，而非绘图代码。

```r
install.packages("data.table")
library(data.table)
Weather_Sample <- data.table(Weather_Sample)
setkey(Weather_Sample, AirportID)
avg_temperature_by_ID <- as.data.frame(Weather_Sample[, mean(DryBulbFarenheit, na.rm = TRUE), by = AirportID])
avg_temperature_by_ID
```

### 按机场 ID 的平均风速（绘图）

此代码以图形格式给出按机场 ID 的平均风速。请注意，`mean()` 函数包含了 `WindSpeed` 属性，表明关键字段是风速而非温度。另外，`ggplot2` 的引用表明这是绘图代码，而非表格代码。

```r
install.packages("data.table")
library(data.table)
Weather_Sample <- data.table(Weather_Sample)
chart_by_ID <- as.data.frame(Weather_Sample[, mean(WindSpeed, na.rm = TRUE), by = AirportID])
install.packages("ggplot2")
library(ggplot2)
ggplot(chart_by_ID, aes(x = AirportID, y = V1)) + geom_point(stat = "identity") + geom_smooth(method = "lm", formula = y ∼ splines::bs(x, 3)) + scale_x_continuous(name = "Airport ID") + scale_y_continuous(name = "Average Wind Speed") +
geom_text(aes(label = AirportID), size = 3, vjust = 1.0) +
geom_text(aes(label = round(V1, digits = 2)), size = 3, vjust = 2.0)
```

### 按机场 ID 的平均风速（表格）

此代码以表格格式给出按机场 ID 的平均风速。请注意，`mean()` 函数包含了 `WindSpeed` 属性，表明关键字段是风速而非温度。另外，没有 `ggplot2` 的引用表明这是表格代码，而非绘图代码。

```r
install.packages("data.table")
library(data.table)
Weather_Sample <- data.table(Weather_Sample)
setkey(Weather_Sample, AirportID)
# OBJECT NAME          DATA FRAME    DATASET NAME          DATA COLUMN REMOVE N/A    GROUP BY COLUMN
avg_windspeed_by_ID <- as.data.frame(Weather_Sample[, mean(WindSpeed, na.rm = TRUE), by = AirportID])
avg_windspeed_by_ID
```

## 索引

**A, B, C**
*   验收测试

**D**
*   数据库
*   数据库引擎配置
*   数据目录
*   FILESTREAM
*   TempDB
*   数据配置（报表生成器）
*   连接属性
*   数据集属性
*   数据源属性
*   已更新的数据源
*   标题值
*   数据目录
*   下载 Microsoft R Open
*   目标位置
*   安装
*   安装数学内核库 (Intel© MKL)
*   MKL 许可信息

**E, F, G, H**
*   电子邮件设置
*   加密密钥
*   执行账户

**I, J, K**
*   初始界面设计
*   安装（请参阅 设置和安装）

**L**
*   线性回归
*   数据集
*   模型
*   predicted_values 的 ggplot 图
*   模型系数
*   语法

**M, N, O**
*   Microsoft R Open 安装
*   模型对象
*   系数
*   `head(predicted_values)`
*   predicted_values 数据集
*   `$residuals`

**P, Q**
*   包管理器
*   可用包
*   已安装包
*   已加载包
*   RUnit
*   绘图
*   diamonds 数据集
*   ggplot2
*   对数尺度
*   按机场 ID 的温度
*   按机场 ID 的风速
*   数据集，导入
*   数据集准备
*   已绘制的信息
*   脚本窗格
*   Power BI 集成
*   项目定义阶段
*   ggplot2
*   初始界面设计
*   包管理器
*   README 文件
*   需求收集
*   软件变更请求流程
*   解决方案资源管理器
*   螺旋式开发

**R**
*   回归诊断
*   报表生成器
*   数据配置（请参阅 数据配置，报表生成器）
*   动态图像属性
*   尺寸选项
*   已更新的值
*   按机场 ID 的温度
*   按机场 ID 的风速
*   空白报表
*   边框属性
*   属性
*   报表生成器安装
*   二进制数据
*   数据库和表
*   天气数据
*   字符分隔符部分
*   数据源
*   目标位置
*   消失的警告
*   平面文件源
*   导入/导出数据
*   已填充的值
*   查询执行
*   保存并运行
*   源和目标信息
*   已更新的目标
*   Reporting Services 配置
*   报表服务器
*   管理缓存界面
*   数据源
*   依赖项
*   属性
*   安全性
*   快照
*   订阅
*   保存视图
*   R 模型执行
*   foreign 库
*   ggplot2 包
*   readme 文件
*   R 交互窗口
*   适用于 Visual Studio 的 R 工具 (RTVS)
*   将 R 更改为 Microsoft R 客户端
*   数据
*   数据科学设置
*   文档和示例
*   下载 Community 2015
*   下载 Microsoft R Open
*   编辑器选项
*   反馈
*   免费 Visual Studio
*   安装 Microsoft R Client
*   Microsoft R 产品
*   图
*   R 文档
*   readme 文件
*   diamonds 数据集
*   执行
*   foreign 库
*   ggplot2 包
*   对数尺度
*   模型数据集
*   predicted_values 数据集
*   R 交互会话
*   调查/新闻
*   表格代码
*   Visual Studio Community
*   Windows
*   工作目录

**S**
*   横向部署
*   服务账户
*   服务验证
*   设置和安装
*   配置
*   数据库引擎实例
*   Reporting Services 服务器
*   数据目录
*   功能选择
*   文件夹结构
*   安装窗口
*   安装规则
*   许可条款
*   Microsoft R Open
*   选项
*   产品密钥
*   准备安装界面
*   RTVS（请参阅 适用于 Visual Studio 的 R 工具 (RTVS)）
*   SQL Server 2016（请参阅 SQL Server 2016）
*   TempDB
*   软件变更请求流程
*   验收测试
*   管理员批准
*   归档
*   代码设计文档
*   安装
*   回归测试
*   请求提交
*   单元测试
*   软件需求文档
*   螺旋式开发流程
*   SQLRUserGroupSQL 2016RS
*   SQL Server
*   SQL Server 安装完成
*   同意安装
*   Microsoft R Open
*   数据库引擎配置
*   数据目录
*   FILESTREAM
*   TempDB
*   功能选择
*   安装规则
*   实例配置
*   许可条款
*   产品密钥
*   准备安装
*   Reporting Services 配置
*   服务器配置
*   SQL Server Data Tools (SSDT)
*   SQL Server 管理工具
*   连接到服务器
*   安装中心
*   安装界面
*   行号设置
*   本地用户和组
*   R 安装和通信
*   SQLRUserGroupSQL2016RS 详细信息
*   SSMS 下载
*   SSMS 安装
*   SSDT（请参阅 SQL Server Data Tools (SSDT)）
*   订阅设置

**T**
*   TempDB

**U, V**
*   单元测试

**W, X, Y, Z**
*   Web 门户 URL
*   Web 服务 URL
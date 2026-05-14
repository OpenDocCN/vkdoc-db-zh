# 图 13-13 用于筛选和排序文件加载的已完成程序包

你现在可以运行该程序包了。但是，在运行此程序包之前，请记得在“变量”窗口中添加 `Directory`、文件筛选器（`FileFilter`）和排序列（`SortColumn`）的实际数据。在本例中，具体如下：

| FileSource: | C:\SQL2012DIRecipes\CH13\MultipleFlatFiles |
| FileFilter: | Stock*.Csv |
| SortColumn: | CreationTime |

或者，如果你是从命令行、SQL Server 代理作业或 SSIS 目录运行程序包，请记得相应地添加对这些变量的引用。


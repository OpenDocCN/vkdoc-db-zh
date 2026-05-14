# 13-4. 对文件加载进行排序和筛选

## 问题
您需要选择一组特定的文件，并根据它们的属性按定义的顺序加载。

## 解决方案
使用 `ADO.NET` 数据集和 `SSIS 脚本` 任务。

在此示例中，我假定您有一个包含 CSV 格式平面文件的目录需要加载到 `SQL Server` 中，并且按从最旧文件开始的顺序加载非常重要。以下步骤解释了实现此目的的方法。

1.  打开 `SSIS` 并创建一个新的包。我建议将其命名为 `OrderAndFilterFileLoad`，因为这正是它要做的。
2.  在包级别添加以下变量：

    | 变量名称 | 类型 | 值 | 注释 |
    | --- | --- | --- | --- |
    | `ADOFilteredAndSortedTable` | Object |  | 保存已排序文件结果列表的 SSIS 对象。 |
    | `FileName` | String |  | Foreach 循环容器用于加载文件（按正确的排序顺序）的变量。 |
    | `FileFilter` | String |  | 用于保存文件名以获取源文件的系统属性的变量，可用于对最终列表排序。 |
    | `SortColumn` | String |  | 充当排序键的列。 |
    | `FileSource` | String |  | 源目录。 |
    | `ADOTable` | Object |  | 保存初始未排序文件列表的 SSIS 对象。 |

3.  在 `Control Flow` 选项卡上添加一个 `Script` 任务。将其命名为 `PopulateRecordset`，然后双击进行编辑。添加以下读写变量：

    | `ADOFilteredAndSortedTable` |
    | --- |
    | `ADOTable` |

4.  单击 `Edit Script` 按钮。
5.  在 `SSIS Script` 窗口中，在 `Imports` 区域添加以下指令：
    ```
    Imports System.IO
    ```
6.  用以下代码替换 `Main` 方法。
    此脚本在配方末尾有说明 (`C:\SQL2012DIRecipes\CH13\OrderedFilteredFileLoad.Vb`)：
    ```vb
    Public Sub Main()
        '声明所有变量
        Dim FileSource As String = Dts.Variables("FileSource").Value.ToString
        Dim FileFilter As String = Dts.Variables("FileFilter").Value.ToString
        Dim SortColumn As String = Dts.Variables("SortColumn").Value.ToString

        Dim MainDS As New System.Data.DataSet
        Dim MainTable As New System.Data.DataTable
        Dim MainRow As System.Data.DataRow
        Dim MainCol As New System.Data.DataColumn

        Dim dirInfo As New System.IO.DirectoryInfo(FileSource)
        Dim fileSystemInfo As System.IO.FileSystemInfo
        Dim FileCounter As Int16 = 0
        Dim FileName As String
        Dim FileFullName As String
        Dim FileSize As Long
        Dim FileExtension As String
        Dim CreationTime As Date
        Dim DirectoryName As String
        Dim LastWriteTime As Date

        '定义表结构
        MainDS.Tables.Add(MainTable)
        MainTable.Columns.Add("FileName", System.Type.GetType("System.String"))        ' 0
        MainTable.Columns.Add("DateAdded", System.Type.GetType("System.DateTime"))     ' 1
        MainTable.Columns.Add("DateLoaded", System.Type.GetType("System.DateTime"))    ' 2
        MainTable.Columns.Add("FileSize", System.Type.GetType("System.Int32"))         ' 3
        MainTable.Columns.Add("CreationTime", System.Type.GetType("System.DateTime"))  ' 4
        MainTable.Columns.Add("FileExtension", System.Type.GetType("System.String"))   ' 5
        MainTable.Columns.Add("DirectoryName", System.Type.GetType("System.String"))   ' 6
        MainTable.Columns.Add("LastWriteTime", System.Type.GetType("System.DateTime")) ' 7
        '遍历目录，并向 ADO 表添加记录
        For Each fileSystemInfo In dirInfo.GetFileSystemInfos(FileFilter)
            FileName = fileSystemInfo.Name
            FileFullName = fileSystemInfo.FullName

            Dim fileDetail As New FileInfo(FileFullName)

            FileSize = fileDetail.Length
            CreationTime = fileDetail.CreationTime
            FileExtension = fileDetail.Extension
            DirectoryName = fileDetail.DirectoryName
            LastWriteTime = fileDetail.LastWriteTime

            MainRow = MainTable.NewRow()

            MainRow(0) = FileName
            MainRow(1) = Now()
            MainRow(2) = CDate("01-01-1900")
            MainRow(3) = FileSize
            MainRow(4) = CreationTime
            MainRow(5) = FileExtension
            MainRow(6) = DirectoryName
            MainRow(7) = LastWriteTime

            MainTable.Rows.Add(MainRow)

            FileCounter = CShort(FileCounter + 1)
        Next

        '创建 ADOLoopTable - 用于实际的批处理循环
        Dim SortedFilteredDS As System.Data.DataSet = MainDS.Clone
        Dim SortedFilteredRows As DataRow() = _
            MainDS.Tables(0).Select("FileSize > 1", SortColumn & " ASC")
        Dim SortedFilteredTable As DataTable = SortedFilteredDS.Tables(0)

        For Each ClonedFilteredRow As DataRow In SortedFilteredRows
            SortedFilteredTable.ImportRow(ClonedFilteredRow)
        Next

        '将表转换为 SSIS 对象：
        Dts.Variables("ADOFilteredAndSortedTable").Value = CType(SortedFilteredDS, Object)

        Dts.TaskResult = ScriptResults.Success
    End Sub
    ```
7.  关闭 `SSIS Script` 窗口，并单击 `OK` 以关闭 `Script` 任务编辑器。
8.  在 `Control Flow` 选项卡上添加一个 `Foreach Loop` 容器，并将其命名为 `Load Files`。将 `Populate Recordset` 脚本任务连接到此容器，然后双击以编辑 `Foreach Loop` 容器。
9.  在左侧选择 `Collection`，并将枚举器类型选为 `For Each ADO Enumerator`。生成的对话框应如图 13-12 所示。




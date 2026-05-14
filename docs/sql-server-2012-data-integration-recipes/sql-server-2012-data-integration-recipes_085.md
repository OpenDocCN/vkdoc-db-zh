# 7-12. 将大型数据集导出为 XML

## 问题

你希望将一个大型数据集导出为 XML 文件。

## 解决方案

使用 `BCP` 和 `T-SQL`，并将文件分批导出，以避免创建单个庞大的 XML 输出记录。

实现此功能的方法是运行以下 `T-SQL` 代码片段来批量导出 `XML`，这样可以确保处理大型数据集（`C:\SQL2012DIRecipes\CH07\LargeXMLOutput.sql`）：

```sql
-- 实例化临时表，将顺序 ID 映射到数据源中的唯一键
IF OBJECT_ID('Tempdb..#IDSet') IS NOT NULL DROP TABLE #IDSet;
SELECT ID, ROW_NUMBER() OVER (ORDER BY ID) AS ROWNO INTO #IDSet FROM dbo.Client;

-- 创建临时表以保存 XML 输出和排序信息
IF OBJECT_ID('dbo.XMLOutput') IS NOT NULL DROP TABLE XMLOutput;
CREATE TABLE XMLOutput (XMLCol NVARCHAR(MAX), SortID INT);

-- 循环变量
DECLARE @LowerThreshold INT = 0
DECLARE @UpperThreshold INT
DECLARE @Range INT = 20000

SELECT @UpperThreshold = MAX(ROWNO) FROM #IDSet;

-- 添加根元素
INSERT INTO XMLOutput (XMLCol, SortID) VALUES (' <Root> ', 0);

-- 循环遍历数据并输出 XML
WHILE @LowerThreshold <= @UpperThreshold
BEGIN
    INSERT INTO XMLOutput (XMLCol, SortID)
    SELECT A.ClientXML, @LowerThreshold
    FROM (
        SELECT C.ID, ClientName, Country, Town
        FROM CarSales.dbo.Client C
        INNER JOIN #IDSet S ON C.ID = S.ID
        WHERE ROWNO > @LowerThreshold AND ROWNO <= @LowerThreshold + @Range
        FOR XML PATH('Client')
    ) A (ClientXML);

    SET @LowerThreshold = @LowerThreshold + @Range
END;

-- 添加闭合的根元素
INSERT INTO XMLOutput (XMLCol, SortID) VALUES ('</ROOT > ', @LowerThreshold + 1);

-- 导出数据
EXEC Master..xp_cmdshell 'BCP "SELECT XMLCol  FROM CarSales.dbo.XMLOutput" QUERYOUT C:\SQL2012DIRecipes\CH07\XMLOut.xml -w -SADAM_01\Adam -T'

-- 清理表
DROP TABLE XMLOutput
```

## 工作原理

如果你执行带有 `FOR XML` 语句的 `SELECT` 查询，你会发现只输出一条记录。对于较大的数据集，输出唯一的一条记录可能会有问题，因为当使用 `BCP` 或 `SSIS` 将数据集写入磁盘时，它们可能会导致问题。因此，我们还需要研究一种将查询分解为多条记录，然后使用 `BCP` 导出每条记录的方法，如前一节所述。

本节中的代码工作原理如下：

*   首先，它为源数据的一系列子集创建了 `XML`。
*   然后，将这些数据添加到数据库中的一个临时表中。
*   接着，分别添加开头和结尾的 `ROOT` 元素。
*   最后，按正确顺序输出临时表的内容，将 `XML` 包裹在 `ROOT` 元素中。

`#IDSet` 临时表用于将源 `ID` 映射到一组连续的 `ID`，以便对数据进行分块处理。在所有情况下可能并非绝对必要，但它通常确保提取的子集大小几乎相同，并避免了当使用源 `ID` 将数据分解为块时，如果源数据中存在许多缺失 `ID` 而导致处理几乎为空的子集的开销。当源数据没有单调递增的 `ID`，和/或是多列唯一 `ID` 时，它也至关重要，因为它可以将更复杂的源 `ID` 映射到一个简单的数字 `ID`。

我建议你将 `@Range` 变量设置为一个在速度（即最少的数据库 `SELECT` 调用次数）和效率（即 `BCP` 不会因输出过大而失败）之间取得最佳平衡的值。只有通过试错，你才能找到一个最优的记录数，使得生成的 `XML` 片段可以被 `BCP` 成功导出而不失败。记录集越小，过程越慢（因为会调用多次 `SELECT...FOR XML`）。这将取决于 `XML` 片段的大小，因此请准备好进行一些迭代测试，直到获得一个优化的过程。

如果你想调整此代码以将输出分解为多个 `XML` 文件，你需要：

*   设置一个 `SQL` 变量作为计数器，该计数器在每次游标迭代时递增。
*   将 `BCP` 输出脚本设置为一个 `SQL` 变量——并使用先前创建的变量作为文件名的一部分。
*   在游标内添加头部和尾部的 `<Root>` 元素，并在每次游标迭代时执行 `BCP` 输出。

## 提示、技巧和陷阱

*   在测试和调试时，最好注释掉最后的 `DROP TABLE XMLOutput` 语句，并检查输出表是否包含你期望找到的数据。
*   我几乎能听到 `T-SQL` 纯粹主义者对在生产代码中使用游标发出的惊恐喘息，我想补充几句辩解的话。首先，游标并非总是邪恶的——它们可以是一个简单的解决方案，因此偶尔使用它们不一定是立即反感的理由。其次，根据我的经验，在像这样导出 `XML` 时，即使对于相当大的数据集，游标也只循环几次，因此性能影响可以忽略不计。


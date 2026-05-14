# 使用变更跟踪进行增量数据管理

## 步骤

1.  创建用于接收变更的目标表。
    ```sql
    Client_CT      (      
        ID INT IDENTITY(1,1) NOT NULL,
        ClientName NVARCHAR(150) NULL,
        Country TINYINT NULL,
        Town VARCHAR(50) NULL,
        County VARCHAR(50) NULL,
        Address1 VARCHAR(50) NULL,
        Address2 VARCHAR(50) NULL,
        ClientType VARCHAR(20) NULL,
        ClientSize VARCHAR(10) NULL,
    ) ;
    GO
    ```

2.  启用要跟踪的表的变更跟踪功能，使用以下 T-SQL 代码片段（`C:\SQL2012DIRecipes\CH11\ChangeTrackingClient.Sql`）：
    ```sql
    USE CarSales;
    GO
    
    ALTER TABLE CarSales.dbo.Client
    ENABLE CHANGE_TRACKING ;
    GO
    ```

3.  向正在跟踪的表（本例中为 `dbo.Client`）添加扩展属性 `LAST_SYNC_VERSION`，该属性将用于存储上次同步的数据版本号。你可以使用以下 T-SQL 代码片段完成此操作——此属性只能添加一次（`C:\SQL2012DIRecipes\CH11\LastSynchVersionProperty.Sql`）：
    ```sql
    USE CarSales;
    GO
    
    EXECUTE sys.sp_addextendedproperty
        @level0type =  N'SCHEMA'
        ,@level0name =  dbo
        ,@level1type =  N'TABLE'
        ,@level1name =  Client
        ,@name =  LAST_SYNC_VERSION
        ,@value =  0
    ;
    ```

4.  运行以下 T-SQL 代码以将相关的插入、删除和更新操作应用到目标表（`C:\SQL2012DIRecipes\CH11\ChangeTrackingProcess.Sql`）：
    ```sql
    USE CarSales;
    GO
    
    BEGIN TRY
    
        DECLARE @LAST_SYNC_VERSION BIGINT
        DECLARE @CURRENT_VERSION BIGINT
        SET @CURRENT_VERSION =  CHANGE_TRACKING_CURRENT_VERSION()
    
        SELECT    @LAST_SYNC_VERSION =  CAST(value AS BIGINT)
        FROM      sys.extended_properties
        WHERE     major_id =  OBJECT_ID('dbo.Client')
                  AND name =  N'LAST_SYNC_VERSION'
    
        -- 确保更新不会丢失数据的测试...
    
        DECLARE @MIN_VALID_VERSION BIGINT
    
        SELECT @MIN_VALID_VERSION =  CHANGE_TRACKING_MIN_VALID_VERSION(OBJECT_ID('dbo.Client'));
    
        IF @LAST_SYNC_VERSION >= @MIN_VALID_VERSION
        BEGIN
    
            -- 插入
            INSERT INTO   CarSales_Staging.dbo.Client_CT
                         (ID,ClientName,Country,Town,County, Address1,Address2, ClientType,ClientSize)
            SELECT        SRC.ID,ClientName,Country,Town,County,Address1,Address2, ClientType,ClientSize
            FROM          dbo.Client SRC
            INNER JOIN    CHANGETABLE(CHANGES Client, @LAST_SYNC_VERSION) AS CT
                         ON SRC.ID =  CT.ID
            WHERE         CT.SYS_CHANGE_OPERATION =  'I'
    
            -- 删除
            DELETE FROM   DST
            FROM           CarSales_Staging.dbo.Client_CT DST
                         INNER JOIN    CHANGETABLE(CHANGES Client, @LAST_SYNC_VERSION) AS CT
                         ON CT.ID =  DST.ID
            WHERE         CT.SYS_CHANGE_OPERATION =  'D'
    
            -- 更新
            UPDATE        DST
            SET           DST.ClientName =  SRC.ClientName
                         ,DST.Country =  SRC.Country
                         ,DST.Town =  SRC.Town
                         ,DST.County =  SRC.County
                         ,DST.Address1 =  SRC.Address1
                         ,DST.Address2 =  SRC.Address2
                         ,DST.ClientType =  SRC.ClientType
                         ,DST.ClientSize =  SRC.ClientSize
            FROM          CarSales_Staging.dbo.Client_CT DST
                         INNER JOIN    dbo.Client SRC
                         ON DST.ID =  SRC.ID
                         INNER JOIN    CHANGETABLE(CHANGES Client, @LAST_SYNC_VERSION) AS CT
                         ON SRC.ID =  CT.ID
            WHERE         CT.SYS_CHANGE_OPERATION =  'U'
        END
    
        -- 在 UPSERT/DELETE 操作后
        EXECUTE sys.sp_updateextendedproperty
            @level0type         = N'SCHEMA'
            ,@level0name        = dbo
            ,@level1type        = N'TABLE'
            ,@level1name        = Client
            ,@name              = LAST_SYNC_VERSION
            ,@value             = @CURRENT_VERSION
        ;
    
    END TRY
    
    BEGIN CATCH
    
        -- 在此处添加错误日志记录
    
    END CATCH
    ```

## 工作原理

对于 SQL Server 开发人员和 DBA 来说幸运的是，Microsoft 在 SQL Server 2008 中引入了一个轻量级的解决方案来解决增量数据管理问题。这个解决方案称为变更跟踪，它不需要任何触发器或手工制作的表。唯一的要求是在源表和目标表上有一个主键。

它的优点如下：
*   它是一个轻量级的解决方案，对服务器的额外开销很小，磁盘空间需求也很少。
*   不需要更改源表。
*   它是一个近乎实时的解决方案。
*   它在很大程度上是自我管理的，因为你可以为要跟踪的增量数据设置一个保留期，SQL Server 会负责清理这些历史数据。然而，这并不意味着数据同步是自动的，只是说管理变更跟踪对象在很大程度上是为你处理的。
*   它适用于所有版本的 SQL Server。

然而，它有几个限制。主要的一个是变更跟踪不支持从目标“拉取”数据同步——如果你尝试这样做，会收到以下错误消息：
```
Msg 22106, Level 16, State 1, Line 3
The CHANGETABLE function does not support remote data sources.
```
因此，这需要一些调整——具体来说，需要在源服务器上使用存储过程来隔离并提供数据用于插入、删除和更新。其他缺点包括：
*   必须在源表上删除 `PRIMARY KEY` 约束之前禁用变更跟踪。
*   非主键列的数据类型更改不会被跟踪。
*   强制执行主键的索引不能被删除或禁用。
*   截断源表需要重新初始化变更跟踪。这本质上意味着必须先为源表禁用变更跟踪，然后再重新启用它。

![image](img/sq.jpg) **注意**   Microsoft 建议你启用快照隔离，这意味着 TempDB 将被大量使用，因此你需要确保 TempDB 配置正确。然而，也有使用快照隔离的替代方案。有关完整详情，请参阅联机丛书 (BOL)。

尽管如此，变更跟踪是一个健壮的解决方案，非常适合许多增量数据场景。它在以下情况下使用效果最佳：
*   你希望在隔离增量数据时避免使用触发器或用户表。
*   当它添加到源系统的（尽管很低）开销是可接受的。

我不会提供关于使用变更跟踪所能完成的所有工作的详尽描述，也不会详细介绍它在生产环境中是如何管理的。如果你需要这些信息，BOL 很好地描述了它，并且在线上和印刷品中都有许多关于这个主题的优秀参考资料。毕竟，这本书是坚定地专注于 ETL 的。

设置变更跟踪很容易。首先，必须在源数据库上设置变更跟踪，然后任何你希望跟踪其 DML 操作的表都必须启用变更跟踪。一旦完成，SQL Server 就可以检测到更改的记录，以及更改的类型（插入、更新或删除），并根据这些更改的记录执行任何所需的数据集成过程。

设置好变更跟踪后，你就可以开始使用它了。你需要知道的是，你必须：
*   获取目标数据库中上次成功更新的 DML 操作的版本号。
*   使用命令 `CHANGE_TRACKING_MIN_VALID_VERSION` 检查自该版本号以来在你的变更跟踪历史记录中是否有数据。


## 提示、技巧和陷阱

*   存储一个名为 `LAST_SYNC_VERSION` 属性的原因是，在检测差异时，你需要告诉 SQL Server，你想从 DML 修改历史中的哪个点开始执行数据的更新插入/删除操作。由于变更跟踪使用顺序编号来标识被跟踪表上的每一个 DML 操作，因此对于每一个数据提取操作，你都需要知道上一次使用的版本号是多少。我建议将其存储为扩展属性，不过如果你愿意，也可以将其存储在日志表中。
*   变更跟踪的版本编号从 0 开始。
*   指定的保留期必须至少与数据同步之间的最长时间一样长。
*   如果你将 `AUTO_CLEANUP` 设置为 `OFF`，那么你必须在某个时候将其重置为 `ON` 以清除变更历史。没有其他方法可以清理变更跟踪。
*   在差异跟踪的保留期（使用 `CHANGE_RETENTION` 设置）和数据库效率之间存在不可避免的权衡。保留期越长，回溯时间检测变更就越容易——但数据库会更大、更慢。
*   我更倾向于将执行变更跟踪的 DML 操作包装在事务中，因为这将确保 `Insert`、`Delete` 和 `Update` 命令会自动且完全地执行——或者根本不执行。当然，你应该添加足够的错误捕获和日志记录，以确保任何错误都能及时上报给 DBA，以便在变更保留期过去之前纠正任何错误。否则，只有对源表和目标表进行完全重新同步，才能确保两个数据集完全一致。然而，在同一服务器的数据库间完美工作的事务，在使用链接服务器实现时将需要 MSDTC。
*   要重新同步源表和目标表，只需截断目标表，然后将数据从源表 `INSERT.. SELECT` 到其中（或使用你喜欢的任何其他表加载技术）。这不会对变更跟踪产生任何影响。然后，你必须将 `Last_Synch_Version` 记录（或设置扩展属性）为变更跟踪的 `Current_Version`，这样当两个源同步时，就只会检测到从此刻开始的 DML 操作。
*   要禁用变更跟踪，只需使用以下 T-SQL 代码段：
    ```sql
    ALTER TABLE dbo.Client DISABLE CHANGE_TRACKING;
    ```
    然后
    ```sql
    ALTER DATABASE CarSales SET CHANGE_TRACKING = OFF;
    ```
*   注意，在数据库级别禁用变更跟踪之前，你必须先禁用所有表的变更跟踪。


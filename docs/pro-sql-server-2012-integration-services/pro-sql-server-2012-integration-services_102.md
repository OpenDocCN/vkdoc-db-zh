# 第 13 章 日志记录与审计

#### 包级别审计

我们的包级别审计功能记录了包标识和执行信息，包括一个执行 GUID，您可以用它关联回 SSIS 日志表。这为 SSIS 日志条目添加了上下文，并提供了可用于按包进行故障排除的额外信息。

为了存储包审计信息，我们创建了一个 `AuditPackage` 表，其中包含包信息，包括名称、版本和一个 GUID，该 GUID 可用于将此表中的条目与 SSIS 日志条目关联（如果您使用适用于 SQL Server 的 SSIS 日志提供程序）。以下是创建该表的代码：

```sql
IF OBJECT_ID ('Audit.AuditPackage') IS NOT NULL
    DROP TABLE Audit.AuditPackage;
GO

CREATE TABLE Audit.AuditPackage
(
    AuditPackageID BIGINT NOT NULL IDENTITY(1, 1),
    BatchID BIGINT NOT NULL,
    PackageName NVARCHAR(128),
    ExecutionInstanceGUID UNIQUEIDENTIFIER,
    ProductVersion NVARCHAR(20),
    MachineName NVARCHAR(128),
    VersionMajor INT,
    VersionMinor INT,
    VersionBuild INT,
    LocaleID INT,
    PackageStartTime DATETIMEOFFSET,
    PackageEndTime DATETIMEOFFSET,
    PackageElapsedTimeMS AS (DATEDIFF(MILLISECOND, PackageStartTime, PackageEndTime)),
    CONSTRAINT PK_AUDIT_AUDITPACKAGE PRIMARY KEY CLUSTERED
    (
        AuditPackageID
    )
);
GO
```

我们为包审计创建了两个过程：`StartAuditPackage` 用于启动审计过程，`EndAuditPackage` 用于结束审计。启动过程在每次调用后返回一个唯一的包审计 ID 号。同一个审计 ID 号被传入结束过程以完成审计条目。代码如下：

```sql
IF OBJECT_ID('Audit.StartAuditPackage') IS NOT NULL
    DROP PROCEDURE Audit.StartAuditPackage;
GO

CREATE PROCEDURE Audit.StartAuditPackage
    @ExecutionInstanceGUID UNIQUEIDENTIFIER,
    @ProductVersion NVARCHAR(20),
    @MachineName NVARCHAR(128),
    @VersionMajor INT,
    @VersionMinor INT,
    @VersionBuild INT,
    @LocaleID INT,
    @PackageName NVARCHAR(128),
    @BatchID BIGINT,
    @AuditPackageID BIGINT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    INSERT INTO Audit.AuditPackage
    (
        PackageName,
        BatchID,
        ExecutionInstanceGUID,
        ProductVersion,
        MachineName,
        VersionMajor,
        VersionMinor,
        VersionBuild,
        LocaleID,
        PackageStartTime
    )
    VALUES
    (
        @PackageName,
        @BatchID,
        @ExecutionInstanceGUID,
        @ProductVersion,
        @MachineName,
        @VersionMajor,
        @VersionMinor,
        @VersionBuild,
        @LocaleID,
        SYSDATETIMEOFFSET()
    );
    SET @AuditPackageID = SCOPE_IDENTITY();
END;
GO

IF OBJECT_ID('Audit.EndAuditPackage') IS NOT NULL
    DROP PROCEDURE Audit.EndAuditPackage;
GO

CREATE PROCEDURE Audit.EndAuditPackage
    @AuditPackageID BIGINT
AS
BEGIN
    SET NOCOUNT ON;
    UPDATE Audit.AuditPackage
    SET PackageEndTime = SYSDATETIMEOFFSET()
    WHERE AuditPackageID = @AuditPackageID;
END;
GO
```

#### 向包添加审计



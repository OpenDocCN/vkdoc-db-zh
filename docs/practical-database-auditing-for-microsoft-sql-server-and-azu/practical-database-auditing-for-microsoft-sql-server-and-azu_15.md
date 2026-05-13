# 第三章 什么是 SQL Server 审计？

## 针对不同场景的审计规范

*   **审计特定用户执行的所有操作**
    *   **审计规范名称** – `Audit_user`
        确保筛选审计规范，使其仅捕获该用户账户执行的操作。
    *   **服务器审计规范名称** – `ServerAudit_user`
        确保以相同方式审计所有数据库和服务器级别的操作，并添加 `DATABASE_OBJECT_ACCESS_GROUP` 和 `SCHEMA_OBJECT_ACCESS_GROUP` 操作。

*   **审计特定数据库的架构和权限更改**
    *   **审计规范名称** – `Audit_DatabaseChanges`
    *   **数据库审计规范名称** – `DatabaseAudit_DatabaseChanges`
        确保将所有以 `DATABASE` 和 `SCHEMA` 开头的审计操作添加到你的数据库审计规范中。

*   **审计所有人对表的更改**
    *   **审计规范名称** – `Audit_tblChanges`
    *   **数据库审计规范名称** – `DatabaseAudit_tblChanges`
        确保使用 `INSERT`、`UPDATE`、`DELETE`、`SELECT` 和/或 `EXECUTE` 这些审计操作类型。

在下一章中，你将学习如何在 SQL Server Management Studio 中设置和配置这些审计。


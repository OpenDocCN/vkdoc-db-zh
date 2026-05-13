# 附录 A 数据库审核选项比较

## 审核 AWS RDS

*   **SQL Server 审核** – 类似于 SQL Server 审核，但审核文件需要存储在 AWS 指定的文件夹中。更多信息，请参阅第 15 章“其他云提供商审核选项”。
*   **扩展事件** – 类似于 SQL Server 扩展事件，但 `.xel` 文件需要存储在 AWS 指定的文件夹中。更多信息，请参阅第 15 章“其他云提供商审核选项”。

## Google Cloud

*   仅在运行 SQL Server 审核或扩展事件的虚拟机上可用

## 审核

*   **诊断设置** – 通过 Azure 门户配置。便于将审核数据集中到 Log Analytics 工作区中。更多信息，请参阅第 14 章“审核 Azure SQL 托管实例”。
*   **SQL Server 审核** – 类似于 SQL Server 审核，但审核文件需要存储在 Azure 存储帐户中。唯一的问题是难以循环遍历多个审核文件来一次性查询所有数据。更多信息，请参阅第 14 章“审核 Azure SQL 托管实例”。
*   **扩展事件** – 类似于 SQL Server 扩展事件，但 `.xel` 文件需要存储在 Azure 存储帐户中。唯一的问题是难以循环遍历多个 `.xel` 文件来一次性查询所有 XEvent 数据。更多信息，请参阅第 14 章“审核 Azure SQL 托管实例”。

### 审核 Azure SQL 托管实例

*   集中化，265–268
*   使用诊断设置，246–252
*   报告，265–268
*   使用 SQL Server 审核，251–261
*   XEvents，261–265

### 审核 Azure SQL 数据库

*   类别，26
*   集中化，241–245
*   配置，39，40
*   凭据，236，237
*   创建，38
*   数据库规格，46–51
*   修改，224–226
*   目标，39，40
*   门户，213–224
*   启用，43
*   查询扩展事件，238–240
*   HTML（参见 HTML 报告）及报告，241–245
*   规格，24，44–46
*   存储帐户与容器，230–236

### 审核操作组

*   数据库审核操作组，28–30
*   服务器审核操作组，27，28

### 审核操作，228
### 审核分发，221
### 审核文件，40，44，70，81，184，185，251，254，259，261，269，271，276，301
### 审核，18，*另请参见* 数据库审核

*   Azure SQL 数据库，18，19
*   Azure SQL 托管实例，19
*   选项，295，296
*   数据库问题，6，7
*   启用与配置，214–218
*   Google Cloud，20
*   管理，4
*   修改，224–226
*   官方审查，3

© Josephine Bush 2022
J. Bush，*Microsoft SQL Server 与 Azure SQL 实用数据库审核*，[`doi.org/10.1007/978-1-4842-8634-0`](https://doi.org/10.1007/978-1-4842-8634-0#DOI)

## 索引

### A，B

*   选项，293–295
*   策略，226–229
*   高级设置，99–101，133，143
*   SQL Server，296–298
*   Amazon Web Services (AWS)，20，268，269
*   类型，4，5
*   *另请参见* AWS RDS SQL Server 审核
*   价值，3
*   应用程序日志，40，41，53，54，67，69
*   查看，219–225
*   审核
*   审核 Azure SQL 数据库
*   类别，26
*   集中化，241–245
*   配置，39，40
*   凭据，236，237
*   创建，38
*   扩展事件，237，238
*   数据库规格，46–51
*   修改，224–226
*   目标，39，40
*   门户，213–224
*   启用，43
*   查询扩展事件，238–240
*   HTML（*参见* HTML 报告）
*   及报告，241–245
*   规格，24，44–46
*   存储帐户与
*   审核操作组
*   容器，230–236
*   数据库审核操作组，28–30
*   审核 Azure SQL
*   服务器审核操作组，27，28
*   托管实例
*   审核操作，228
*   集中化，265–268
*   审核分发，221
*   使用诊断设置，246–252
*   审核文件，40，44，70，81，184，185，251，
*   报告，265–268
*   254，259，261，269，271，276，301
*   使用 SQL Server 审核，251–261
*   审核，18，*另请参见* 数据库审核
*   XEvents，261–265
*   Azure SQL 数据库，18，19
*   审核事件操作，140
*   Azure SQL 托管实例，19
*   审核事件，139
*   选项，295，296
*   审核选项，275–279
*   数据库问题，6，7
*   审核用户，183，188，189
*   启用与配置，214–218
*   审核级别操作，27
*   Google Cloud，20
*   审核日志失败，68
*   管理，4
*   审核日志，259
*   修改，224–226
*   审核名称，39，67
*   官方审查，3
*   自动清理，164
*   合规性，5
*   AWS RDS SQL Server 审核
*   压缩，278
*   审核选项，275–279
*   配置，13，119，125，272
*   选项组，272–275
*   配置更改
*   S3 存储桶，269–272
*   历史记录，151–153
*   XEvents，286–288
*   查询，153–155
*   Azure SQL 数据库，18，19
*   SQL Server 审核，155–159
*   SQL Server，213
*   容器，230–236
*   Azure SQL 托管实例，19
*   CSV 表步骤，243，268
*   Azure 存储帐户，237，238，258，
*   259，262，301
*   Azure 存储连接，238，262
*   **D**
*   数据库
*   审核操作组，28–30
*   **C**
*   审核，6
*   C2 审核，15–16
*   问题审核，6，7
*   加州消费者隐私法案
*   数据库审核
*   (CCPA)，5
*   审核 Azure SQL 托管
*   集中式审核数据库，183，188，
*   实例，19
*   190，193
*   审核选项，293–295
*   集中化，241–245，265–268
*   Azure SQL 数据库，18，19
*   集中化审核数据
*   C2 审核，15，16



## 索引

## A

`creation`， 188, 189

`CDC`， 13, 14

`linked server`， 189, 190

`change tracking`， 14, 15

`multiple servers`， 183

`common criteria compliance`， 15, 16

`setting up audits`， 183–187

`extended events`， 10, 11

`Change data capture (CDC)`， 13, 14, 168–172, 294, 295

`SQL Server Audit`， 9, 10, 168–172, 294, 295

`SQL Server configuration`， 11–13

`Change tracking`， 14–15, 164–168, 294, 295, 300

`successful and failed login auditing`， 17, 18

## C

`Channel drop-down`， 94

`Clean up audit data`， 183, 190

`Database audit specification`， 9, 46–51, 156

`Cloud auditing`， 294–295

`columns`， 79, 80

`Cloud shell`， 226, 227

`DML actions`， 74

`Columns`， 54, 55, 79, 80

`multiple audits adding`， 79

`in table`， 128

`objects, schemas or databases`， 74–76

`Common criteria compliance`， 15, 16, 161–164, 294, 295

`querying system views`， 76–79

`suggestions`， 73

`Database credential`， 236, 237

`Export extended event session`， 92

`Database-level actions`， 26

`Extended events (XEvents)`， 10, 11, 237, 238, 293, 295–298

`Database-level auditing`， 213, 218

`advanced settings`， 99–101

`Database mail`， 198

`auditing Azure SQL managed instance`， 261–265

`Data definition language (DDL) auditing`， 6

`AWS RDS SQL Server Audit`， 286–288

`Data manipulation language (DML) auditing`， 6

`category drop-down`， 95

`Data retention settings`， 217

`components`， 90–101

`Data storage screen`， 116

`default sessions`， 89, 90

`DDL triggers`， 180

`deleting`， 135, 136, 150

`Default auditing policy`， 228

`files`， 146

`Default sessions`， 89, 90

`global fields`， 95–97

`Deleting audits`， 57–61, 83–86

`Google Cloud`， 288, 289

`Deletion audits`， 57–60

`library`， 93–95

`extended events`， 150

`modifying`， 131–133, 148, 149

`XEvents`， 135, 136

`new session option`， 119–124

`DevOpsOperationsAudit`， 246

`new session wizard option`， 103–119

`Diagnostic settings`

`predicates`， 95–97

`addition`， 246

`querying`， 125–131

`configure`， 246

`querying data`， 147, 148

`querying audit data`， 248–251

`querying system`， 141–146

`saved`， 247

`scripting`， 137, 138

`SQL Server Audit`， 247, 248

`setting up`， 138–141

`Disabling audits`， 60–61, 86, 87

`SQL scripts`， 137

`@displayname`， 200

`stopping and starting`， 134, 135, 150, 151

`DML actions`， 49, 74

`successful and failed logins`， 178–180

`targets`， 97–99

`templates`， 90–93

## E

`use cases`， 101, 102

`@emailaddress`， 199

`via GUI`， 103

`Engine version`， 273

`.xel file`， 125

`Error: 33204 SQL Server Audit`， 44

`Event hub`， 216

`Event library`， 93–95, 123

## F

`Event retention mode`， 100

`Files on disk`， 125

`Event Viewer`， 53, 54

`Filtering`， 55–57, 82, 83, 285

`Filtering events`， 140

`of actions`， 143

`Filtering values`， 130

`of events`， 142

`Filters`， 145

`Log analytics`， 216, 217, 219, 249, 250

## G

`Log Analytics workspace`， 216–220, 241, 249

`Log File Viewer`， 53

`General Data Protection Regulation (GDPR)`， 5

`Logic app`， 241, 265

`Login auditing`， 17, 18, 176, 294

`Get-AZSqlServerAudit`， 226

`Global fields`， 95–97, 109, 123, 145

`Google Cloud`， 20, 268, 288, 289

## H

`Group button`， 274

`GUI SQL Server (see SQL Server Audit)`

`XEvents (see Extended events (XEvents))`

`Health Insurance Portability and Accountability Act (HIPAA)`， 5

`HTML Reports with PowerShell`， 205–209

## I, J

`with SQL Server Agent`， 197–205

`New option group auditing option`， 275–279

`IAM role name`， 278

`to RDS Instance`， 279–281

`Information Systems`， 5

`New session configure`， 122

`Internal Revenue Service (IRS)`， 3, 4

`dialog box`， 119

`IP addresses`， 221

`events page`， 121

`general page`， 120

`global fields`， 123

`page filter`， 123

## K

`storage`， 124

`Kusto query`， 221–223, 250

`New session wizard option capture screen`， 108

`choose template screen`， 107

## L

`creation`， 104

`Linked server`， 183, 189, 190

`data storage screen`， 116

`Listing INDEX event session screen`， 118

`filters operator drop-down`， 111

`filtering values`， 130

`filters screen`， 110

`SQL Server Audit`， 258–261, 283–286

`global fields`， 109

`view target data`， 126

`group clauses`， 115

`watch live data`， 126, 127

`introduction screen`， 105

`XEvents data`， 147, 148

`modify filters`， 113

`Querying audit logs application log`， 53

`multiple filters`， 112

`columns`， 54, 55

`properties screen`， 106

`cross-section`， 81

`in SSMS`， 103

`summary screen`， 117




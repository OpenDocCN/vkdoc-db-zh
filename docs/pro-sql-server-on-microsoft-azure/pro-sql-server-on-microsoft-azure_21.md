# Execute a query against the SQL Server instance
$dbs = Invoke-Sqlcmd -ServerInstance $sqlserver -Query $sqlquery

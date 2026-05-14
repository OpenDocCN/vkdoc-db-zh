# 检查备份是否直接存储到 BLOB
if ($backups.Result -eq "Disk")
{
    Write-Host "[警告] 在本地磁盘上找到数据库备份" -ForegroundColor Red
    $RulePass = 0
}
if ($RulePass -eq 1)
{
    Write-Host "[信息] 所有数据库备份均存储到 Azure Blobs" -ForegroundColor Green
}

$sqlquery = "if exists (select top 1 backup_size from msdb.dbo.backupset where compressed_backup_size = backup_size)
select 'Uncompressed' as Result
else
select 'Compressed' as Result"

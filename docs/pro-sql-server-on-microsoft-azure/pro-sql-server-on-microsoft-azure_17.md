# 从每个 URI 中检索存储账户
foreach ($vhd in $VHDs)
{
    $Accounts = $Accounts + ((($vhd -split "//")[1]) -split ".blob")[0]
}

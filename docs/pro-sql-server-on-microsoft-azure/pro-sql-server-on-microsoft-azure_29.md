# 找出服务账户的 SID 值
$objUser = New-Object System.Security.Principal.NTAccount($servcice.StartName)
$strSID = $objUser.Translate([System.Security.Principal.SecurityIdentifier])

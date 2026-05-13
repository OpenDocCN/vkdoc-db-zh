# Configure OS settings
Write-Host 'Configure OS settings...'
Start-Process -FilePath 'C:\Windows\System32\powercfg.exe' -ArgumentList '-setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c' -Wait -NoNewWindow
Write-Host 'High Performance power plan configured'
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl\ -Name Win32PrioritySeparation -Value 24
Write-Host 'Optimise for background services configured'

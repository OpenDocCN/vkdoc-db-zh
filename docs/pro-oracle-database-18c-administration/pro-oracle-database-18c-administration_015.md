# 文件系统检查

7 * * * * /orahome/bin/filesp.bsh 1>/orahome/bin/log/filesp.log 2>&1

请注意，本节中使用的 Shell 脚本（`filesp.bsh`）可能需要根据您的环境进行修改。该 Shell 脚本依赖于`df -h`命令的输出，而该命令的输出会因操作系统和版本的不同而有所差异。例如，在 Solaris 系统上，`df -h`的输出如下所示：

```
$ df -h
Filesystem             size   used  avail capacity  Mounted on
/ora01                  50G    42G   8.2G    84%    /ora01
/ora02                  50G    20G    30G    41%    /ora02
/ora03                  50G   4.5G    46G     9%    /ora03
/orahome                30G    24G   6.5G    79%    /orahome
```

Shell 脚本中的这一行用于有选择地报告`df -h`命令输出中的“容量”信息：

```
usedSpc=$(df -h $ml | awk '{print $5}' | grep -v capacity | cut -d "%" -f1 -)
```

对于您的环境，您需要修改前面的这行代码，以正确提取与每个挂载点磁盘剩余空间相关的信息。例如，假设您在 Linux 系统上执行`df -h`命令，并观察到以下输出：

```
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00
222G  162G   49G  77% /
```

这里只有一个挂载点，并且磁盘空间百分比与“Use%”列相关联。因此，要提取相关信息，您需要修改 Shell 脚本中与`usedSpc`相关的代码；例如，

```
df -h / | grep % | grep -v Use | awk '{print $4}' | cut -d "%" -f1 -
```

因此，Shell 脚本将需要修改以下几行，如下所示：

```
mntlist="/"
for ml in $mntlist
do
echo $ml
usedSpc=$(df -h / | grep % | grep -v Use | awk '{print $4}' | cut -d "%" -f1 -)
```


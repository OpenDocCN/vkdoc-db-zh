# Description: (Optional) Command to run on the target.
variable.command=%job_default_shell%

schedule.frequency=IMMEDIATE
```

创建新作业，指定新的属性文件作为输入文件，如下所示：

```
[oracle ~]$ emcli create_job -input_file='property_file:uname_r_oms.job'
Creation of job "UNAME_R_OMS" was successful.
```

现在 `UNAME_R_OMS` 作业是一个活动作业。可以通过指定 `describe_library_job` 动词使用相同的过程在库中创建作业。

## 创建库作业

```
[oracle ~]$ emcli describe_library_job -name='UNAME_A'> uname_a.job
[oracle ~]$ cp uname_a.job uname_r.job
```

编辑 `uname_r.job` 文件。Listing 3-29 显示了脚本内容，并使用 `create_library_job` 动词从脚本创建库作业。

Listing 3-29. 创建 `uname_a.job` 和 `uname_r.job` 库作业属性文件

```
[oracle ~]$ cat uname_r.job


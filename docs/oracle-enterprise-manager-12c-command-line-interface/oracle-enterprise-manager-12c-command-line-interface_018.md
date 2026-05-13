# Description: (Optional) Command to run on the target.
variable.command=%job_default_shell%

schedule.frequency=IMMEDIATE
```

此命令的输出可用作属性文件来创建新作业。

## 通过属性文件创建作业

1.  将文本复制到文件中并根据需要修改变量（Listing 3-27）。
2.  为了省去一步，可以使用 shell 功能将标准输出重定向到文件。
3.  将刚创建的文件复制为新文件，并在标准文本编辑器中打开该新文件（Listing 3-27）。
4.  更改新文件中的作业名称和命令以反映新作业（更改以粗体显示；Listing 3-28）。

Listing 3-27. 创建 `uname_a_oms.job` 作业属性文件并复制

```
[oracle ~]$ emcli describe_job -name='UNAME_A_OMS' > uname_a_oms.job
[oracle ~]$ cp uname_a_oms.job uname_r_oms.job
```

Listing 3-28. 创建 `uname_r_oms.job` 作业属性文件

```
[oracle ~]$ cat uname_r_oms.job


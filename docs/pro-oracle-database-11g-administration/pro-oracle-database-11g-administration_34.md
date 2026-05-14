# 显示最耗 CPU 的进程

```bash
alias topc="ps -e -o pcpu,pid,user,tty,args | sort -n -k 1 -r | head"
```


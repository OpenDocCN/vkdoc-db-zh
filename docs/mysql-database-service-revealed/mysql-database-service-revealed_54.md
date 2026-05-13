# 确定请求的操作：启动/停止
if len(sys.argv) > 1:
is_stop = sys.argv[1].upper() == "STOP"
operation = "Stopping"
else:
is_stop = False
operation = "Starting"

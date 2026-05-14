# 增加一个新列，该列通过计划时间和实际耗时进行计算
df_flightinfo_times = df_flightinfo.withColumn("Time_diff", df_flightinfo.ELAPSED_TIME - df_flightinfo.SCHEDULED_TIME).select("FLIGHT_NUMBER", "AIRLINE", "SCHEDULED_TIME", "ELAPSED_TIME", "Time_diff")

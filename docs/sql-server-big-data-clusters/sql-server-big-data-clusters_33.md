# 让我们使用内连接在航空公司代码上连接 df_airlines 和 df_flights 数据框
df_flightinfo = df_flights.join(df_airlines, df_flights.AIRLINE == df_airlines.IATA_CODE, how="inner")

# 我们将再次连接两个数据框，但这次删除 df_flights 数据框的 AIRLINE 列
df_flightinfo = df_flights.join(df_airlines, df_flights.AIRLINE == df_airlines.IATA_CODE, how="inner").drop(df_flights.AIRLINE)

# 将结果存储在新的 df_airports_removed 数据框中
df_airports_removed = df_airports.filter(df_airports.IATA_CODE  "COD")

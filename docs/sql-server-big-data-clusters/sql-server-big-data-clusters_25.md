# 将结果存储在新的 df_airports_updated 数据框中
df_airports_updated = df_airports.withColumn("CITY", when(col("IATA_CODE") == "COD", "Jackson"))

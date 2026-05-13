# Now insert the data in the table
for row in data:
    cols = row.strip('\n').split(",") # comma-separated row
    INSERT = f"{INSERT_SQL}'{cols[0]}','{cols[1]}',{cols[2]})"
    print(INSERT)
    cur.execute(INSERT)

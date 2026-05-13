# Fetch and print the results
rows = cur.fetchall()
for row in rows:
    print(row[0]) # Print first column only

# 循环遍历数据库系统摘要
db_systems = []
num = 1
print("Compartment 中的数据库系统")
print("-------------------------")
for db_system_summary in list_db_systems_response.data:
if db_system_summary.lifecycle_state != 'DELETED':
print("{0}: {1}".format(num, db_system_summary.display_name))
db_systems.append(db_system_summary)

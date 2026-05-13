# 获取要控制的数据库系统的名称
db_num = int(input("\n 选择哪个数据库系统 (输入序号)? "))
print("\n{0} 名为 '{1}' 的数据库系统\n".format(operation, db_systems[db_num-1].display_name))

# 在单个命令中创建类的一个实例，定义要应用的属性，并指定目标列表筛选器。
myinst = updateProps.updateProps(namefilter='^orcl', typefilter='oracle_dbsys', 
agentfilter='em12cr3\.example\.com:3872', propdict={'LifeCycle Status':'Development'})


# 从环境变量获取 compartment ID
COMPARTMENT_ID = os.getenv('COMPARTMENT_ID')
### 列出 MySQL 规格
list_shapes_response = mysql_client.list_shapes(compartment_id=COMPARTMENT_ID)

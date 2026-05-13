# 循环遍历完整的目标列表
for targ in myobj:
    # 如果目标名称筛选器适用...
    if myreg.search(targ['Target Name']):
        myproprecs = targ['Target Name'] + mydelim + \
                     targ['Target Type'] + mydelim

        # 对于每个目标，循环遍历
        #         目标属性字典
        for propkey, propvalue in myprops.items():
            myproprecprops = propkey + mydelim + propvalue

            # 如果 debug 为 True，则将命令打印到屏幕
            #           而不执行
            if debug:
                mycommand = 'set_target_property_value(' + \
                            'subseparator="' + mysubsep + \
                            '", property_records="' + \
                            myproprecs + \
                            myproprecprops + '")'
                print(mycommand)
```


```python

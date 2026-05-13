#         并执行命令
else:
    print('Target: ' + targ['Target Name'] + \
          ' (' + targ['Target Type'] + \
          ')\n\tProperty: ' + propkey + \
          '\n\tValue:    ' + propvalue)
    set_target_property_value(
        subseparator=mysubsep,
        property_records=myproprecs + myproprecprops)
```

## 更新目标属性类 (`updateProps.py`)

`updateProps.py` 脚本包含几条 `import` 语句和一个类定义。该脚本不包含任何直接执行的命令。因此，直接执行此脚本（即 `emcli @updateProps.py`）并无用处。正确的用法是，在交互模式下，将此脚本作为一个模块与登录脚本一起导入，然后使用该类及其函数来执行命令。在脚本模式下，将创建一个执行脚本，其中包含使用该类及其函数的命令。`updateProps.py` 的导入语句以及登录脚本的导入语句应出现在任何命令执行之前。

为了演示这一点，请创建一个名为 `changeTargProps.py` 的脚本。此脚本将用于使用 `updateProps()` 类来更改目标属性。此示例中的环境要求执行脚本中不包含凭证密码；因此，将使用 `secureStart.py` 登录脚本来建立登录。`changeTargProps.py` 的前两行将是导入语句：

```
import secureStart
import updateProps
```

在脚本执行至此点时，EM CLI 与 OMS 之间应已成功登录，并且 `updateProps()` 类应已导入。脚本中的其余命令使用类函数来更新目标的属性：

```

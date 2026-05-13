# 使用类应用属性更新而不创建实例。
updateProps.updateProps(
    namefilter='^orcl',
    typefilter='oracle_dbsys',
    agentfilter='em12cr3\.example\.com:3872',
    propdict={
              'LifeCycle Status':'Production',
              'Location':'DC1'}
    ).setprops(True)
```

下面的示例是 `updateProps.py` 脚本。`updateProps()` 类包含大量注释，以解释其各个组件的功能及其整体目的。有关此类的更多详细信息，请参见 第 6 章：

```
import emcli
import re
import operator

class updateProps():
    """
       updateProps() 类与 EM CLI 结合使用，
       允许用户将一组定义的目标属性应用到通过
       正则表达式在多个层级筛选的目标列表中。
       这些目标列表筛选层级包括目标名称、
       目标类型和父代理名称。
       可以直接使用该类，或使用该类的实例，
       将属性应用到目标。

筛选后的列表可以指定为：
         - 类参数，并直接将定义的属性应用到它们
         - 创建类实例时的类参数
         - 作为实例一部分的 filt() 函数

定义的目标属性集可以指定为：
         - 类字典参数 'propdict'，并直接将定义的属性
           应用到目标列表
         - 创建类实例时的类字典参数 'propdict'
         - 作为实例一部分的 props() 函数
    """
    def __init__(self, agentfilter='.*', typefilter='.*',
                 namefilter='.*', propdict={}):
        self.targs = []
        self.reloadtargs = True
        self.props(propdict)
        self.__loadtargobjects()
        self.filt(agentfilter=agentfilter, typefilter=typefilter,
                  namefilter=namefilter)
    def __loadtargobjects(self):
        """
           __loadtargobjects() 向 OMS 查询完整的目标列表。
           目标列表随后被缓存在实例变量 'fulltargs' 中，
           每个目标的相应属性被分配给
           'targprops' 实例变量。仅当目标属性已被
           应用时，才会刷新这些变量。
```


```python
        """
        此函数由其他函数调用，绝不应直接调用。
        """
        if self.reloadtargs == True:
            self.reloadtargs = False
            self.fulltargs = \
              emcli.list(resource='Targets').out()['data']
            self.targprops = \
              emcli.list(resource='TargetProperties'
                        ).out()['data']

    def props(self, propdict):
        """
        `props()` 可直接从实例调用，或在定义实例或直接使用类时由 `__init__()` 调用。
        此函数定义一个属性名和值的字典对象，该对象将应用于定义好的目标列表。
        """
        assert isinstance(propdict, dict), \
               'propdict 参数必须是 ' + \
               '包含 ' + \
               '{"property_name":"property_value"} 的字典'
        self.propdict = propdict

    def filt(self, agentfilter='.*', typefilter='.*',
             namefilter='.*',
             sort=('TARGET_TYPE','TARGET_NAME'), show=False):
        """
        `filt()` 可直接从实例调用，或在定义实例或直接使用类时由 `__init__()` 调用。
        此函数通过仅包含属性与定义筛选器匹配的目标来限制目标列表。

        此函数接受以下参数：
            agentfilter - 应用于目标父代理的正则表达式字符串
            typefilter - 应用于目标类型值的正则表达式字符串
            namefilter - 应用于目标名称值的正则表达式字符串
        """
        self.targs = []
        __agentcompfilt = re.compile(agentfilter)
        __typecompfilt = re.compile(typefilter)
        __namecompfilt = re.compile(namefilter)
        self.__loadtargobjects()
        for __inttarg in self.fulltargs:
            if __typecompfilt.search(__inttarg['TARGET_TYPE']) \
               and __namecompfilt.search(
                   __inttarg['TARGET_NAME']) \
               and (__inttarg['EMD_URL'] == None or \
               __agentcompfilt.search(__inttarg['EMD_URL'])):
                self.targs.append(__inttarg)
        __myoperator = operator
        for __myop in sort:
            __myoperator = operator.itemgetter(__myop)
        self.targssort = sorted(self.targs, key=__myoperator)
        if show == True:
            self.show()

    def show(self):
        """
        `show()` 可直接从实例调用，或作为其他某些函数的参数调用。
        打印目标名称、类型以及目标所有当前定义的属性名和值的整齐格式化显示。
        """
        print('%-5s%-40s%s' % (
              ' ', 'TARGET_TYPE'.ljust(40, '.'),
              'TARGET_NAME'))
        print('%-15s%-30s%s\n%s\n' % (
              ' ', 'PROPERTY_NAME'.ljust(30, '.'),
              'PROPERTY_VALUE', '=' * 80))
        for __inttarg in self.targssort:
            print('%-5s%-40s%s' % (
                  ' ', __inttarg['TARGET_TYPE'].ljust(40, '.'),
                  __inttarg['TARGET_NAME']))
            self.__showprops(__inttarg['TARGET_GUID'])
            print('')

    def __showprops(self, guid):
        """
        `__showprops()` 打印具有与传递给函数的 'guid' 变量匹配的唯一 guid 的目标的属性。
        此函数由 `show()` 函数为每个目标调用，以打印目标对应的属性。

        此函数由其他函数调用，绝不应直接调用。
        """
        self.__loadtargobjects()
        for __inttargprops in self.targprops:
            __intpropname = \
              __inttargprops['PROPERTY_NAME'].split('_')
            if __inttargprops['TARGET_GUID'] == guid and \
               __intpropname[0:2] == ['orcl', 'gtp']:
                print('%-15s%-30s%s' %
                      (' ', ' '.join(__intpropname[2:]).ljust(
                       30, '.'),
                       __inttargprops['PROPERTY_VALUE']))

    def setprops(self, show=False):
        """
        `setprops()` 从类或实例直接调用，并调用 EM CLI 函数，该函数将定义的属性集应用于筛选目标列表中的每个目标。
        'show' 布尔参数可设置为 True，以在属性应用后调用 `show()` 函数。
        """
        assert len(self.propdict) > 0, \
               'propdict 参数必须包含 ' + \
               '至少一个属性。请使用 ' + \
               'props() 函数进行修改。'
        self.reloadtargs = True
        __delim = '@#&@#&&'
        __subseparator = 'property_records=' + __delim
        for __inttarg in self.targs:
            for __propkey, __propvalue \
                in self.propdict.items():
                __property_records = __inttarg['TARGET_NAME'] + \
                  __delim + __inttarg['TARGET_TYPE'] + \
                  __delim + __propkey + __delim + __propvalue
                print('目标: ' + __inttarg['TARGET_NAME'] +
                      ' (' + __inttarg['TARGET_TYPE'] +
                      ')\n\t 属性: '
                      + __propkey + '\n\t 值: ' +
                        __propvalue + '\n')
                emcli.set_target_property_value(
                  subseparator=__subseparator,
                  property_records=__property_records)
        if show == True:
            self.show()
```

---

[¹]: `http://www.symantec.com/cluster-server`.
[²]: 数据库服务器上的成员通常包括使用相同 Oracle 主目录的数据库和侦听器。与这些目标相关的驱动器都是该组的成员。包含 Oracle 基目录、归档目标、oradata 文件系统以及其他相关数据的文件系统都包含在故障切换组中。
[³]: 尽管标题如此，但 `relocate_target` 动词并不会物理地重新定位任何目标。其唯一目的是将监控和指标收集从一个代理转移到另一个代理。

## 索引

![images](img/sq.jpg) A

-   `参数文件`
-   `优势`
-   `批处理`
-   `创建与编辑`
-   `生命周期状态属性`

![images](img/sq.jpg) B

-   `BI Publisher (BIP)`

![images](img/sq.jpg) C, D

-   `清理脚本`
-   `命令行`
    -   `管理员账户`
    -   `代理管理任务`
    -   `Enterprise Manager (EM) 代理`
    -   `delete_targets 动词`
        -   `删除任务`
        -   `已转移的目标`
    -   `get_targets 动词`
    -   `help 命令`
    -   `管理服务器`
    -   `sync 命令`

![images](img/sq.jpg) E



# 企业管理器命令行界面 (EM CLI)

## 动词

- EM12c 第 4 版动词
- BI Publisher
- 云服务动词
- 融合中间件动词
- 金丝雀代理动词
- 作业动词
- 杂项动词
- 目标数据动词
- `login` 动词

## 模式

- 命令行模式
- 交互模式
- 脚本模式

## 脚本

- `changeTargProps.py` 脚本
    - `生命周期状态` 属性
- `secureStart.py` 脚本
- `start.py` 脚本
- `updatePropsFunc.py` 脚本
- `updateProps.py` 脚本

## 其他概念

- `高级套件`
- `维护窗口`
- `通信`
- `配置设置`
- `错误代码`
- `get_targets`
    - 命令行模式
    - 交互模式
    - 脚本模式
    - 使用通配符
- `help()` 函数
- `安装`
- `作业操作`
    - `交互` 模式
    - `Jython` 模块
    - `ps` 命令
    - `脚本` 模式
    - `SSL` 认证方法
- `模式`
- `脚本`
- `语法`
- `目标创建`
- `动词`
- `EMCTL` 通信
- `EMCTL` *对比* `EM CLI`

## 组件与相关工具

- 访问
- 集中化
- 客户端软件
- 控制实用程序
- 框架
- 安全网
- 启动和停止
- 动词
- `企业管理器控制实用程序 (EMCTL)`
- ![images](img/sq.jpg)   F, G, H
- `函数库`
    - 代码实现
    - 命令行输入变量
    - 调试代码
    - 回显语句
    - TTY 会话
- `融合中间件置备过程 (FMWPROV)`
- ![images](img/sq.jpg)   I
- `信息发布者 (IP) 报告`
- ![images](img/sq.jpg)   J, K
- `JSON`



# L, M, N

## Jython

Jython

#### 监听器进程

监听器进程

### 登录脚本

登录脚本

# O

## OEM 作业, EM CLI

OEM 作业, EM CLI

## create_job 动词

create_job 动词

### 创建和升级作业

创建和升级作业

### 移除代理

移除代理

### 升级代理

升级代理

## OEM 软件库。参见软件库

OEM 软件库。参见软件库

## Oracle 可扩展性交换中心

Oracle 可扩展性交换中心

### 优势

优势

### “概览”选项卡

“概览”选项卡

### 经典视图

经典视图

### EMC 插件对比

EMC 插件对比

### “贡献”选项卡

“贡献”选项卡

### Fusion ION Accelerator 插件

Fusion ION Accelerator 插件

### 全局访问

全局访问

### 标题页

标题页

### 主页

主页

### 链接

链接

### 卸载和导出功能

卸载和导出功能

### 搜索栏视图

搜索栏视图

### 二级搜索页面

二级搜索页面

### 标签搜索

标签搜索

# P, Q

## 打补丁过程

打补丁过程

### 远程客户端安装

远程客户端安装

### 模板

模板

### 使用 EM CLI 客户端

使用 EM CLI 客户端

## Python

Python

### 你好，世界

你好，世界

### `help()` 函数

help() 函数

### 登录脚本

登录脚本

### 对象

对象

#### 字典

字典

#### EM CLI 列表

EM CLI 列表

#### `len()` 函数

len() 函数

#### 监听器进程

监听器进程

#### 列表

列表

#### 数字和字符串

数字和字符串

### 概述

概述

### 目标属性

目标属性

### `updateProps()` 类

updateProps() 类

### EM CLI 目标

EM CLI 目标

#### `filt()` 函数

filt() 函数

#### `init()` 函数

init() 函数

#### `loadtargobjects()` 函数

loadtargobjects() 函数

#### `props()` 函数

props() 函数

#### `setprops()` 函数

setprops() 函数

#### `show()` 函数

show() 函数

#### `showprops()` 函数

showprops() 函数

# R

## 远程目标安装

远程目标安装

### 关联目录

关联目录

### 下载和部署

下载和部署

### 效率

效率

### `emclikit.jar` 文件

emclikit.jar 文件

### Java 路径

Java 路径

### 要求

要求

### 安全性

安全性

## 报表定义库

报表定义库

# S, T, U

## 计划中断

计划中断

### 管理员参考

管理员参考

### 参数文件创建

参数文件创建

### 批处理

批处理



Enterprise Manager 控制工具 (`EMCTL`)
使用函数库
`bashrc` 与 `.bash_profile`
`CreateBlackout`
`EndBlackout`
`restore_training.sh` 脚本
`getopts` 命令
生命周期状态属性
OEM 管理员
稳健的解决方案
角色管理
沙盒 OEM 服务器
Shell 脚本
`SYSMAN` 模式
安全框架
架构
HTTPS 受信任证书
安全模式
服务器管理脚本
Shell 脚本
命令行实用程序
维护窗口， `EM CLI`
函数
`get_targets` 动词
`name` 参数
`schedule` 参数
安排维护窗口 (*参见* 安排维护窗口)
`targets` 参数
`getopts` 命令
日志文件创建
`echo` 语句
错误处理
密码
软件库
添加/编辑/删除脚本
添加脚本
代理共享文件系统
中央库
创建实体
创建通用组件
脚本的描述性数据和路径信息
`EM12c` 控制台
库的访问
命令行脚本文件夹
新文件夹的信息条目
新建文件夹
`EM12c` 软件库界面
信息发布者 (IP) 报告
多个脚本
新文件位置
OMS 共享文件系统
Oracle 可扩展性交换中心
优势
概览选项卡
经典视图
EMC 插件的比较
贡献选项卡
Fusion ION 加速器插件
全局访问
标题页面
主页
链接
卸载与导出功能
搜索栏视图
二次搜索页面
标签搜索
引用文件的位置
报告定义库
设置菜单下拉列表
存储定义库
脚本的唯一名称
上传脚本
存储定义库
`SYSMAN` 模式
![images](img/sq.jpg)  V
Veritas 集群服务器 (VCS)
代码实现
数据库
监听器目标
OEM 代理
Oracle 企业管理器 (OEM)
程序逻辑
`QUAL` 集群组
`TEST` 节点
![images](img/sq.jpg)  W, X, Y, Z
WebLogic 服务器 (WLS)
WLS 安装
域配置配置文件
环境变量
`FMWPROV` 过程
Java 必需文件 (JRF)
Java 版本
Jython
要求
软件库

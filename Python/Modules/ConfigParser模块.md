[TOC]

# Python之ConfigParser

ConfigParser模块用于读取配置文件内容，配置文件格式类似于my.cnf，可以包含一个或多个节点，每个节点可以包含多个参数，每个参数都以key=value的格式存在。利用配置文件，我们可以灵活的修改相关参数，而不用去大量的代码中搜索替换。

```
[section1]
a=1
b=2

[section2]
c=3
d=4
```

初始化ConfigParser对象

```
import configparser
config = configparser.ConfigParser()
```

常用函数

| 函数                                                         | 描述                                                      |
| ------------------------------------------------------------ | --------------------------------------------------------- |
| config.read("/data/luhx/DBCheck/config.ini",encoding="utf-8") | 读取配置文件                                              |
| config.sections()                                            | 获取配置所有section，以list格式返回                       |
| config.options(section)                                      | 获取指定section下的所有参数key，以list格式返回            |
| config.items(section)                                        | 获取指定section下的所有参数键值对                         |
| config.get(section,option)                                   | 获取指定section下指定key的value值                         |
| config.set(section,option,value)                             | 修改section下某个参数的值                                 |
| config.has_section(section)和config.has_option(section,option) | 判断是否存在section和section下的某个参数，返回true或false |
| config.add_section(section)                                  | 创建section                                               |
| config.remove_section(section)                               | 删除section                                               |

> 修改参数文件后需要写回文件并保存，`config.write(open("config.ini", "w"))`


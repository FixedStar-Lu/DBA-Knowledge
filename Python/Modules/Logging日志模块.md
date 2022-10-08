[TOC]

---

# Logging概述

logging模块是Python提供的一个标准库模块，其实现了一个灵活的事件日志系统，有助于调试程序，分析定位程序异常。

## 日志等级

logging模块默认包含下列日志等级，每个等级输出的内容不同

Level | Desc
-- | --
DEBUG | 日志内容最为详细，常用于调试程序
INFO | 输出关键节点信息
WARNING | 输出一些告警信息，但不影响程序正常运行
ERROR | 因一些严重错误导致部分功能无法正常运行
CRITICAL | 因一些严重错误导致程序无法正常运行

日志等级由上至下日志信息详尽程度是逐步递减的，即DEBUG < INFO < WARNING < ERROR < CRITICAL。程序开发调试过程中我们可以采用DEBUG或INFO获取完整的日志信息；在实际生产上线时，我们应该关注WARNING及以下等级

## 常用函数

函数 | Desc
-- | --
logging.debug(msg,*args,**kwargs) | 创建一条DEBUG级别的日志
logging.info(msg,*args,**kwargs) | 创建一条INFO级别的日志
logging.warning(msg,*args,**kwargs) | 创建一条WARNING级别的日志
logging.error(msg,*args,**kwargs) | 创建一条ERROR级别的日志
logging.critical(msg,*args,**kwargs) | 创建一条CRITICAL级别的日志
logging.log(level,*args,**kwargs) | 创建一条定义级别的日志
logging.basicConfig(**kwargs) | 基础配置：日志级别，日志格式，输出位置等信息

```
import loggging

logging.debug("This is a debug log.")
logging.info("This is a info log.")
logging.warning("This is a warning log.")
logging.error("This is a error log.")
logging.critical("This is a critical log.")
```

输出结果
```
WARNING:root:This is a warning log.
ERROR:root:This is a error log.
CRITICAL:root:This is a critical log.
```
可以看到，DEBUG和INFO等级的日志并未输出，这是因为logging默认的等级为WARNING，只有大于等于WARNING等级的日志才会输出

## Logging配置

**输出格式**

默认输出的格式为 ==LEVEL : 日志器名称 : 日志内容==。这是由BASIC_FORMAT配置决定的，可以利用logging.basicConfig(**kwargs)自定义配置格式。


**参数配置**
Parameter | Desc
-- | --
filename | 日志输出到指定文件名，而不是控制台
filemode | 日志文件的打开模式，默认为a
format | 指定日志内容格式
datefmt | 指定时间类型格式
level | 日志等级
stream | 指定日志输出目标stream，如：sys.stdout、sys.stderr以及网络stream，不能与filename同时指定
style | 指定format格式字符串的风格
handler | 创建多个handler的可迭代对象，并添加到root logger。不能与filename，stream同时使用

**format格式**

属性 | format | Desc
-- | -- | --
asctime | %(asctime)s | 日志发生时间。如：2021-06-27 16:49:45,896
created | %(created)f | 日志发生时间(时间戳)
msecs | %(msecs)d | 日志发生时间(毫秒部分)
levelname | %(levelname)s | 日志级别(字符串)
levelno | %(levelno)s | 日志级别(数字)
name | %(name)s | 所使用的日志器名称，默认root
message | %(message)s | 日志内容
pathname | %(pathname)s | 调用日志函数的源码文件位置
filename | %(filename)s | pathname中的文件名部分
module | %(module)s | filename的名称部分
lineno | %(lineno)d | 调用日志函数的源代码所在的行号
funcName | %(funcName)s | 调用日志函数的函数名
process | %(process)d | 进程ID
processName | %(processName)s | 进程名称
thread | %(thread)d | 线程ID
threadName | %(thread)s | 线程名称

下面就对format参数进行一定配置
```
>>> import logging
>>> LOG_FORMAT = "%(asctime)s - %(levelname)s - %(message)s"
>>> DATE_FORMAT = "%m-%d-%Y %H:%M:%S"
>>> logging.basicConfig(level=logging.DEBUG, format=LOG_FORMAT, datefmt=DATE_FORMAT)
>>> logging.debug("This is a debug log.")
06-27-2021 15:52:10 - DEBUG - This is a debug log.
```

除了上述配置，也可以针对logging.debug等函数的**kwargs参数支持exc_info、stack_info、extra选项
- exc_info：将异常异常信息添加到日志消息中，没有则输出None。默认为true
- stack_info：将堆栈信息输出到日志中，默认为false
- extra：值为dict类型，可以自定义消息格式中包含的字段
```
logging.warning("This is a warning log.", exc_info=True, stack_info=True, extra={'user': 'luhengxing', 'ip':'192.168.56.10'})
```

## Logging使用

**需求场景**

- 将ERROR级别的日志写入到文件中
- 日志文件格式为日期-日志级别-文件名[:行号]-日志信息
- 定时对日志文件进行切割

**代码实现**
```
import logging
import logging.handlers
import datetime

logger = logging.getLogger('ErrorLogger')
logger.setLevel(logging.ERROR)

handler=logging.handlers.TimedRotatingFileHandler('error.log', when='midnight', interval=1, backupCount=7, atTime=datetime.time(0, 0, 0, 0))
handler.setFormatter(logging.Formatter("%(asctime)s - %(levelname)s - %(filename)s[:%(lineno)d] - %(message)s"))

logger.addHandler(handler)
logger.error('error messages')
```
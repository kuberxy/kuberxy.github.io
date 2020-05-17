---
layout: post
title: Python标准库 - Logging模块
date:   2019-09-13 22:36:21 +0800
category: Python
tags: [Python、标准库]
typora-root-url: ../..
---



## 日志级别

### What

日志级别是指事件的严重程度。设置一个级别后，严重程度等于或高于该级别的日志会被输出，低于该级别的日志将被忽略。Python logging模块支持的日志级别如下：

|   级别   | 数值 |
| :------: | :--: |
| CRITICAL |  50  |
|  ERROR   |  40  |
|  WARING  |  30  |
|   INFO   |  20  |
|  DEBUG   |  10  |

对应这些级别，logging模块提供了debug()、inof()、warning()、error()、critical()方法，用于定义相应级别下的日志内容。



### 默认级别

```python
import logging

logging.debug('I am {}'.format('kubxy'))
logging.info('I am {}'.format('kubxy'))
logging.warning('I am {}'.format('kubxy'))
logging.error('I am {}'.format('kubxy'))
logging.critical('I am {}'.format('kubxy'))
```

运行结果：

```python
WARNING:root:I am kubxy
ERROR:root:I am kubxy
CRITICAL:root:I am kubxy
```

因为logging模块默认的日志级别为warining，所以上面的代码没有输出debug和info级别的日志。



### 设置级别

```python
import logging

logging.basicConfig(level=logging.DEBUG)

logging.debug('I am {}'.format('kubxy'))
logging.info('I am {}'.format('kubxy'))
logging.warning('I am {}'.format('kubxy'))
logging.error('I am {}'.format('kubxy'))
logging.critical('I am {}'.format('kubxy'))
```

运行结果：

```python
DEBUG:root:I am kubxy
INFO:root:I am kubxy
WARNING:root:I am kubxy
ERROR:root:I am kubxy
CRITICAL:root:I am kubxy
```

logging模块有很多默认配置，可以通过basicConfig()方法来修改



## 日志格式

### What

为了便于阅读和分析，日志通常是按照固定的格式输出的。如下：是logging模块中常用的格式化字符串

| 格式          | 说明                 |
| ------------- | -------------------- |
| %(message)s   | 日志内容             |
| %(asctime)s   | 日志调用的时间       |
| %(levelname)s | 日志的级别           |
| %(funcName)s  | 日志调用所在的函数名 |
|               |                      |
|               |                      |



### 默认格式

```python
import logging

logging.warning('I am {}'.format('kubxy'))
```

运行结果：

```python
WARNING:root:I am kubxy
```

可以看出，logging模块默认的格式为`%(levelname)s:%(message)s`



### 自定义格式

#### 基本用法

```python
import logging

FORMAT = '%(asctime)s %(levelname)s: %(message)s'
logging.basicConfig(format=FORMAT)

logging.warning('I am {}'.format('kubxy'))
```

运行结果：

```python
2020-05-17 16:18:21,591 WARNING: I am kubxy
```



#### 高级用法

```python
import logging

FORMAT = '%(asctime)-60s#$\tThread info: %(thread)d %(threadName)s %(message)s %(school)s'
logging.basicConfig(format=FORMAT, level=logging.INFO, datefmt="%Y-%m-%dT%H:%M:%S")
# logging.basicConfig(format=FORMAT, level=logging.INFO, datefmt="%Y-%m-%dT%H:%M:%S", filename='./python.log')

d = {'school':'test self define'}
logging.info('I am %s %s', 20,'yesrs old', extra = d)
logging.warning('I am %s %s', 20,'yesrs old', extra = d)
```

运行结果：

```python
2020-05-17T16:22:59                                         #$	Thread info: 3428 MainThread I am 20 yesrs old test self define
2020-05-17T16:22:59                                         #$	Thread info: 3428 MainThread I am 20 yesrs old test self define
```



## Logger Objects

### getLogger()

getLogger(name=None)方法用于创建一个logger实例，如下示例：

```python
import logging

logger = logging.getLogger()
print(type(logger))
print(logger)
print(logger.name)

print('-'*20)

logger = logging.getLogger('my')
print(type(logger))
print(logger)
print(logger.name)

print('-'*20)

logger = logging.getLogger('')
print(type(logger))
print(logger)
print(logger.name)
```

运行结果：

```
<class 'logging.RootLogger'>
<logging.RootLogger object at 0x00A45390>
root
--------------------
<class 'logging.Logger'>
<logging.Logger object at 0x005AD0D0>
my
--------------------
<class 'logging.RootLogger'>
<logging.RootLogger object at 0x00A45390>
root
```

可以看到，当name为None或空时返回root logger

 

### 理解logger实例的树形结构

#### 理解示例

logger实例是一个单根的树形结构。我们看如下示例：

```python
import logging

logging.basicConfig(level=logging.ERROR)

logger = logging.getLogger()
print(type(logger))
print(logger)
print('Here is {}. My root is {}'.format(logger.name,logger.parent))
logger.warning('test')

print('--'*50)

logger1 = logging.getLogger('a')
print(type(logger1))
print(logger1)
print('Here is {}. My root is {}'.format(logger1.name,logger1.parent))
logger1.warning('test')
print()

logger2 = logging.getLogger('a.b')
print(type(logger2))
print(logger2)
print('Here is {}. My root is {}'.format(logger2.name,logger2.parent))
logger2.warning('test')
print()

logger3 = logging.getLogger('a.b.c')
print(type(logger3))
print(logger3)
print('Here is {}. My root is {}'.format(logger3.name,logger3.parent))
logger3.warning('test')

print('--'*50)

logger1 = logging.getLogger('x')
print(type(logger1))
print(logger1)
print('Here is {}. My root is {}'.format(logger1.name,logger1.parent))
logger1.warning('test')
print()

logger2 = logging.getLogger('x.y')
print(type(logger2))
print(logger2)
print('Here is {}. My root is {}'.format(logger2.name,logger2.parent))
logger2.warning('test')
print()

logger3 = logging.getLogger('x.y.z')
print(type(logger3))
print(logger3)
print('Here is {}. My root is {}'.format(logger3.name,logger3.parent))
logger3.error('test')
```

运行结果：

```python
<class 'logging.RootLogger'>
<logging.RootLogger object at 0x00762590>
Here is root. My root is None
----------------------------------------------------------------------------------------------------
<class 'logging.Logger'>
<logging.Logger object at 0x0047DA10>
Here is a. My root is <logging.RootLogger object at 0x00762590>

<class 'logging.Logger'>
<logging.Logger object at 0x007392D0>
Here is a.b. My root is <logging.Logger object at 0x0047DA10>

<class 'logging.Logger'>
<logging.Logger object at 0x00739450>
Here is a.b.c. My root is <logging.Logger object at 0x007392D0>
----------------------------------------------------------------------------------------------------
<class 'logging.Logger'>
<logging.Logger object at 0x00739530>
Here is x. My root is <logging.RootLogger object at 0x00762590>

<class 'logging.Logger'>
<logging.Logger object at 0x00739550>
Here is x.y. My root is <logging.Logger object at 0x00739530>

<class 'logging.Logger'>
<logging.Logger object at 0x00739590>
Here is x.y.z. My root is <logging.Logger object at 0x00739550>
ERROR:x.y.z:test
```



#### 应用示例

```python
import logging

FORMAT = '[%(asctime)s] %(funcName)s: %(message)s'
logging.basicConfig(format=FORMAT)

logger = logging.getLogger(__name__)

def fun1():
    mylogger = logging.getLogger('{}.{}'.format(__name__,'fun1'))
    mylogger.warning('fun1')


def fun2():
    mylogger = logging.getLogger('{}.{}'.format(__name__,'fun2'))
    mylogger.warning('fun2')


fun1()
fun2()
```

运行结果：

```python
[2020-05-17 16:47:10,191] fun1: fun1
[2020-05-17 16:47:10,191] fun2: fun2
```



### 设置logger实例的级别

```python
import logging

FORMAT = '[%(asctime)s] %(message)s'
logging.basicConfig(format=FORMAT, level=logging.INFO)

logger = logging.getLogger(__name__)
logging.info('my level is {}'.format(logger.getEffectiveLevel()))

logger.info('set level')
logger.setLevel(21)
logging.info('parent level is {}'.format(logger.parent.getEffectiveLevel()))
logging.info('my level is {}'.format(logger.getEffectiveLevel()))

logger.info('info: set after')
logger.warning('warning: set after')
```

运行结果：

```python
[2020-05-17 16:56:39,832] my level is 20
[2020-05-17 16:56:39,832] set level
[2020-05-17 16:56:39,832] parent level is 20
[2020-05-17 16:56:39,833] my level is 21
[2020-05-17 16:56:39,833] warning: set after
```



### 设置logger实例的过滤器

#### 示例1

```python
import logging

# root logger
logger = logging.getLogger()
logging.warning('root logger filters {}'.format(logger.filters))

# logger
logger1 = logging.getLogger('s')
logger2 = logging.getLogger('s.s1')
logger3 = logging.getLogger('s.s2')

# filter s
filterer = logging.Filter('s')


logging.warning('\n')
logging.warning('{0} logger1 s add filter s {0}'.format('='*20))
logger1.addFilter(filterer)
logging.warning('logger1 filters {}'.format(logger1.filters))
logger1.warning('logger1 warning output')

logging.warning('\n')
logging.warning('{0} logger2 s.s1 add filter s {0}'.format('='*20))
logger2.addFilter(filterer)
logging.warning('logger2 filters {}'.format(logger2.filters))
logger2.warning('logger2 warning output')

logging.warning('\n')
logging.warning('{0} logger3 s.s2 add filter s {0}'.format('='*20))
logger3.addFilter(filterer)
logging.warning('logger3 filters {}'.format(logger3.filters))
logger3.warning('logger3 warning output')
```

运行结果：

```python
WARNING:root:root logger filters []
WARNING:root:

WARNING:root:==================== logger1 s add filter s ====================
WARNING:root:logger1 filters [<logging.Filter object at 0x00AE93D0>]
WARNING:s:logger1 warning output
WARNING:root:

WARNING:root:==================== logger2 s.s1 add filter s ====================
WARNING:root:logger2 filters [<logging.Filter object at 0x00AE93D0>]
WARNING:s.s1:logger2 warning output
WARNING:root:

WARNING:root:==================== logger3 s.s2 add filter s ====================
WARNING:root:logger3 filters [<logging.Filter object at 0x00AE93D0>]
WARNING:s.s2:logger3 warning output
```



#### 示例2

```python
import logging

# root logger
logger = logging.getLogger()
logging.warning('root logger filters {}'.format(logger.filters))

# logger
logger1 = logging.getLogger('s')
logger2 = logging.getLogger('s.s1')
logger3 = logging.getLogger('s.s2')

# filter s
filterer = logging.Filter('s.s1')


logging.warning('\n')
logging.warning('{0} logger1 s add filter s.s1 {0}'.format('='*20))
logger1.addFilter(filterer)
logging.warning('logger1 filters {}'.format(logger1.filters))
logger1.warning('logger1 warning output')

logging.warning('\n')
logging.warning('{0} logger2 s.s1 add filter s.s1 {0}'.format('='*20))
logger2.addFilter(filterer)
logging.warning('logger2 filters {}'.format(logger2.filters))
logger2.warning('logger2 warning output')

logging.warning('\n')
logging.warning('{0} logger3 s.s2 add filter s.s1 {0}'.format('='*20))
logger3.addFilter(filterer)
logging.warning('logger3 filters {}'.format(logger3.filters))
logger3.warning('logger3 warning output')
```

运行结果：

```python
WARNING:root:root logger filters []
WARNING:root:

WARNING:root:==================== logger1 s add filter s.s1 ====================
WARNING:root:logger1 filters [<logging.Filter object at 0x005F93D0>]
WARNING:root:

WARNING:root:==================== logger2 s.s1 add filter s.s1 ====================
WARNING:root:logger2 filters [<logging.Filter object at 0x005F93D0>]
WARNING:s.s1:logger2 warning output
WARNING:root:

WARNING:root:==================== logger3 s.s2 add filter s.s1 ====================
WARNING:root:logger3 filters [<logging.Filter object at 0x005F93D0>]
```



#### 示例3

```python
import logging

# root logger
logger = logging.getLogger()
logging.warning('root logger filters {}'.format(logger.filters))

# logger
logger1 = logging.getLogger('s')
logger2 = logging.getLogger('s.s1')
logger3 = logging.getLogger('s.s2')

# filter s
filterer = logging.Filter('s.s2')


logging.warning('\n')
logging.warning('{0} logger1 s add filter s.s2 {0}'.format('='*20))
logger1.addFilter(filterer)
logging.warning('logger1 filters {}'.format(logger1.filters))
logger1.warning('logger1 warning output')

logging.warning('\n')
logging.warning('{0} logger2 s.s1 add filter s.s2 {0}'.format('='*20))
logger2.addFilter(filterer)
logging.warning('logger2 filters {}'.format(logger2.filters))
logger2.warning('logger2 warning output')

logging.warning('\n')
logging.warning('{0} logger3 s.s2 add filter s.s2 {0}'.format('='*20))
logger3.addFilter(filterer)
logging.warning('logger3 filters {}'.format(logger3.filters))
logger3.warning('logger3 warning output')
```

运行结果：

```python
WARNING:root:root logger filters []
WARNING:root:

WARNING:root:==================== logger1 s add filter s.s2 ====================
WARNING:root:logger1 filters [<logging.Filter object at 0x005E93F0>]
WARNING:root:

WARNING:root:==================== logger2 s.s1 add filter s.s2 ====================
WARNING:root:logger2 filters [<logging.Filter object at 0x005E93F0>]
WARNING:root:

WARNING:root:==================== logger3 s.s2 add filter s.s2 ====================
WARNING:root:logger3 filters [<logging.Filter object at 0x005E93F0>]
WARNING:s.s2:logger3 warning output
```



#### 示例4

```python
import logging

# root logger
logger = logging.getLogger()
logging.warning('root logger filters {}'.format(logger.filters))

# logger
logger1 = logging.getLogger('s')
logger2 = logging.getLogger('s.s1')
logger3 = logging.getLogger('s.s2')

# filter s
filterer = logging.Filter('s.s3')


logging.warning('\n')
logging.warning('{0} logger1 s add filter s.s3 {0}'.format('='*20))
logger1.addFilter(filterer)
logging.warning('logger1 filters {}'.format(logger1.filters))
logger1.warning('logger1 warning output')

logging.warning('\n')
logging.warning('{0} logger2 s.s1 add filter s.s3 {0}'.format('='*20))
logger2.addFilter(filterer)
logging.warning('logger2 filters {}'.format(logger2.filters))
logger2.warning('logger2 warning output')

logging.warning('\n')
logging.warning('{0} logger3 s.s2 add filter s.s3 {0}'.format('='*20))
logger3.addFilter(filterer)
logging.warning('logger3 filters {}'.format(logger3.filters))
logger3.warning('logger3 warning output')
```

运行结果：

```python
WARNING:root:root logger filters []
WARNING:root:

WARNING:root:==================== logger1 s add filter s.s3 ====================
WARNING:root:logger1 filters [<logging.Filter object at 0x007B93D0>]
WARNING:root:

WARNING:root:==================== logger2 s.s1 add filter s.s3 ====================
WARNING:root:logger2 filters [<logging.Filter object at 0x007B93D0>]
WARNING:root:

WARNING:root:==================== logger3 s.s2 add filter s.s3 ====================
WARNING:root:logger3 filters [<logging.Filter object at 0x007B93D0>]
```



#### 小结

过滤器用于筛选特定的日志记录。比如，当一个过滤器为A.B，它可以处理名为A.B、A.B.C、A.B.C.D、A.B.D等logger生成的日志记录，但不能处理名为A.BB、B.A.B等logger生成的日志记录。



## Handler

### What

我们可以为logger实例add一个handler实例，Handler的作用如下：

- 控制日志输出的位置，可以是控制台、文件

- 设置级别
- 设置格式
- 设置过滤器



Handler的类型：

- StreamHandler - 标准输出/标准错误输出
- FileHandler - 磁盘文件
- NullHandler - 什么都不做



### 设置级别

```python
import logging

FORMAT = '%(asctime)s DEBUG: %(message)s'
logging.basicConfig(format=FORMAT,level=logging.DEBUG)

# root logger
logger = logging.getLogger()
print(logger.handlers)

# logger
logger = logging.getLogger('s')
logger.setLevel(logging.INFO)
print(logger.handlers)

streamhandler = logging.StreamHandler()
streamhandler.setLevel(logging.WARNING)
logger.addHandler(streamhandler)

filehandler = logging.FileHandler('./myloger.log')
filehandler.setLevel(logging.WARNING)
logger.addHandler(filehandler)

print(logger.handlers)
logger.error('my logger')
```



### 设置格式

```python
import logging

FORMAT = '%(asctime)s DEBUG: %(message)s'
logging.basicConfig(format=FORMAT,level=logging.DEBUG)


# logger
logger = logging.getLogger('s')
logger.setLevel(logging.INFO)
print(logger.handlers)

formatter = logging.Formatter('%(asctime)s WARNING: %(message)s')
streamhandler = logging.StreamHandler()
streamhandler.setLevel(logging.WARNING)
streamhandler.setFormatter(formatter)
logger.addHandler(streamhandler)

print(logger.handlers)
logger.error('my logger')
```



### 设置过滤器

```python
import logging

logger1 = logging.getLogger('s')
logger2 = logging.getLogger('s.s1')
logger3 = logging.getLogger('s.s2')


filterer = logging.Filter('s')
# filterer = logging.Filter('s.s1')
# filterer = logging.Filter('s.s2')
# filterer = logging.Filter('s.s3')
handler = logging.StreamHandler()
handler.addFilter(filterer)

logger1.addHandler(handler)
print(logger1.filters)
print(logger1.handlers)
logger1.warning('logger1 warning output')

logger2.addHandler(handler)
print(logger2.filters)
print(logger2.handlers)
logger2.warning('logger2 warning output')

logger3.addHandler(handler)
print(logger3.filters)
print(logger3.handlers)
logger3.warning('logger3 warning output')
```



## Logging Flow

![image-20200517172347191](/2019-09-13-Python-StandardLib-1/assets/images/image-20200517172347191.png)


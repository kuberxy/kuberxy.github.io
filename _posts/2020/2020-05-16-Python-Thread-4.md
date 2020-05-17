---
layout: post
title: Python多线程 - 线程安全问题
date:   2020-05-16 10:02:42 +0800
category: Python
tags: [Python、线程]
typora-root-url: ../..
---





## 线程安全

### 什么是线程安全

线程安全，是多线程环境下的一个概念。它是指当多个线程同时访问同一个对象时，不会出现结果不一致的情况。简单理解就是，多线程环境下的执行结果和单线程环境下的结果是一致的，不需要通过一些额外的手段（比如，锁）来保证结果的一致性。



### 理解print函数

首先，我们来看一下print函数的定义：

```python
def print(self, *args, sep=' ', end='\n', file=None):
    """
    print(value, ..., sep=' ', end='\n', file=sys.stdout, flush=False)
    
    Prints the values to a stream, or to sys.stdout by default.
    Optional keyword arguments:
    file:  a file-like object (stream); defaults to the current sys.stdout.
    sep:   string inserted between values, default a space.
    end:   string appended after the last value, default a newline.
    flush: whether to forcibly flush the stream.
    """
    pass
```

可以看到，print函数用于将value打印到一个stream中。默认情况下，stream为标准输出，多个values之间以空格分隔，并且还会在最后一个value后追加一个换行符。

```python
for i in range(1,5):
    print('test')

print('='*50)

for i in range(1,5):
    print('test',end='=')
```

运行结果：

```python
test
test
test
test
==================================================
test=test=test=test=
```

我们可以看到，即使是一个最简单的print语句，它也包含了两个步骤：

- 打印value
- 追加end



### print函数是线程安全的吗？

我们直接来看如下示例代码：

```python
import threading

def worker():
    for x in range(100):
        print('{} is running.'.format(threading.current_thread().name))


for x in range(1,5):
    name = 'worker {}'.format(x)
    t = threading.Thread(name=name, target=worker)
    t.start()
```

在`ipython`下的运行结果：

```python
worker 1 is running.
worker 1 is running.
worker 1 is running.
worker 1 is running.
worker 3 is running.worker 4 is running.

worker 2 is running.
worker 1 is running.
worker 1 is running.
...
```

> 注：以上测试请在ipython环境下完成。python命令行和pycharm都不能演示出效果。

可以看到，在多线程环境下，print函数的两个步骤并不是原子的。也就是说，在打印value之后，追加end之前会存在切换线程，导致实际的输出结果与单线程环境下的输出结果不一致。因此，可以说print函数是线程不安全的。



### 多线程环境下如何输出日志？

> 经过上面的测试，我们发现print函数是线程不安全的，即在多线程环境下，使用print输出日志信息会存在预料之外的结果。此时，我们改怎么办呢？



#### 方法一：让print不打印换行符

```python
import threading

def worker():
    for x in range(100):
        print('{} is running.\n'.format(threading.current_thread().name),end='')


for x in range(1,5):
    name = 'worker {}'.format(x)
    t = threading.Thread(name=name, target=worker)
    t.start()
```

我们知道，字符串是一个不可变的类型，因此它会作为一个不可分割的整体输出。`end=''`表示打印value后追加一个空字符，这样即使在打印value之后，追加end之前发生了线程切换也不影响输出结果



#### 方法二：使用logging模块

在python标准库中，存在一个专门处理日志的模块logging，它是线程安全的，也是我们在生产环境用来输入日志的工具。

```python
import threading
import logging

def worker():
    for x in range(100):
        logging.warning('{} is running.'.format(threading.current_thread().name))


for x in range(1,5):
    name = 'worker {}'.format(x)
    t = threading.Thread(name=name, target=worker)
    t.start()
```














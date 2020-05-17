---
layout: post
title: Python多线程 - threading库（1）
date:   2020-05-16 10:02:42 +0800
category: Python
tags: [Python、线程]
typora-root-url: ../..
---





在Python中，一个解释器进程就是一个独立的进程，多个线程会共享同一个解释器进程。比如，当我们在Linux命令行下执行`python test.py`时，就会启动一个解释器进程，而在`test.py`中至少会运行一个线程。

在python中，使用多线程通常会借助标准库threading。下面我们就来看看一个线程的基本生命周期。



## 创建（初始化）线程

### 语法

```python
def __init__(self, group=None, target=None, name=None, args=(), kwargs=None, verbose=None)
```

说明：

- target：线程调用的对象，即要执行的目标函数。线程在本质上就是执行函数。

- name：线程的名字。名字可以重复，因为线程ID才是线程的唯一标识。

- args：为目标函数传递实参，元组结构。

- kwargs：为目标函数传递关键字参数，字典结构。



### 示例

```python
import threading

# 目标函数
def worker():
    print('Hello, world')

# 初始化一个线程对象
t = threading.Thread(target=worker)
print(type(t))
print(t)
```

运行结果：

```python
<class 'threading.Thread'>
<Thread(Thread-1, initial)>
```



## 启动线程

### 示例1：单线程

```python
import threading

# 目标函数
def worker():
    print("I'm working")
    print("Fineshed")

# 初始化一个线程对象
t = threading.Thread(target=worker, name='worker')

# 启动线程
t.start()
```

运行结果：

```python
I'm working
Fineshed
```



### 示例2：多线程

```python
import threading
import time

# 目标函数
def worker1():
    for _ in range(10):
        time.sleep(0.5)
        print('welcome1')
    print('Worker1 over')


def worker2():
    for _ in range(10):
        time.sleep(0.5)
        print('welcome2')
    print('Worker2 over')

# 初始化第一个线程对象,并启动它
t = threading.Thread(target=worker1, name='worker')
t.start()

# 初始化第二个线程对象,并启动它
t = threading.Thread(target=worker2, name='worker')
t.start()
```

运行结果：

```python
welcome2
welcome1
welcome2
welcome1
welcome1welcome2

welcome1
welcome2
welcome2
Worker2 over
welcome1
Worker1 over
```

如果我们执行多次，可以看到每次的执行结果会有所不同。



## 终止线程

python没有提供终止线程的方法。一个线程被终止，不外乎以下两种情况：

- 线程函数的语句执行完毕
- 线程函数在执行过程中遇到未处理的异常

python线程没有优先级的概念，也不能被销毁、停止、挂起，因此也就没有恢复一说了。



### 示例1：单线程

```python
import threading
import time

def worker():
    count = 0
    while True:
        if count > 5:
            raise RuntimeError(count) # 未处理的异常，导致线程退出
            # reutrn # 执行完毕，正常返回并退出
        print("I'm working")
        count += 1
        time.sleep(1)

t = threading.Thread(target=worker, name='worker')
t.start()

print('--- end ---')
```

运行结果：

```python
I'm working--- end ---

I'm working
I'm working
I'm working
I'm working
I'm working
Exception in thread worker:
Traceback (most recent call last):
  File "D:\Python27\Lib\threading.py", line 801, in __bootstrap_inner
    self.run()
  File "D:\Python27\Lib\threading.py", line 754, in run
    self.__target(*self.__args, **self.__kwargs)
  File "D:/Project/aliyun/test.py", line 8, in worker
    raise RuntimeError(count)
RuntimeError: 6
```



### 示例2：多线程

```python
import threading
import time

def worker1():
    count = 0
    while True:
        count += 1
        time.sleep(0.5)

        print('welcome1')

        if count >= 5:
            break

    print('worker1 over')

def worker2():
    count = 0
    while True:
        count += 1
        time.sleep(0.5)

        print('welcome2')

        if count > 3:
            raise Exception('Thread Error')

    print('worker2 over')

t = threading.Thread(target=worker1, name='worker1')
t.start()

t = threading.Thread(target=worker2, name='worker2')
t.start()
```

运行结果：

```python
welcome2
welcome1
welcome2
welcome1
welcome2
welcome1
Exception in thread worker2:
welcome2
Traceback (most recent call last):
  File "D:\Python27\Lib\threading.py", line 801, in __bootstrap_inner
    self.run()
  File "D:\Python27\Lib\threading.py", line 754, in run
    self.__target(*self.__args, **self.__kwargs)
  File "D:/Project/aliyun/test.py", line 26, in worker2
    raise Exception('Thread Error')
Exception: Thread Error

welcome1
welcome1
worker1 over
```





## 线程传参

我们知道，线程就是对函数的调用。因此，线程传参在本质上就是函数传参。



### 示例1

```python
import threading
import time

def add(x,y):
    print('{} + {}'.format(x,y,x+y))
    print('current thread id {}'.format(threading.current_thread().ident))

t1 = threading.Thread(target=add, name='add', args=(4,5))
t1.start()

time.sleep(2)

t2 = threading.Thread(target=add, name='add', kwargs={'x':4, 'y':5})
t2.start()

time.sleep(2)

t3 = threading.Thread(target=add, name='add', args=(5,), kwargs={'y':4})
t3.start()
```

运行结果：

```python
4 + 5
current thread id 8808
4 + 5
current thread id 7516
5 + 4

current thread id 6972
```



### 示例2

```python
import threading
import time

def worker1(n):
    count = 0
    while True:
        count += 1
        time.sleep(0.5)

        print('welcome1')

        if count >= n:
            break
    print('worker1 over')

def worker2():
    count = 0
    while True:
        count += 1
        time.sleep(0.5)

        print('welcome2')

        if count > 3:
            raise Exception('Thread over')
    print('worker2 over')

t = threading.Thread(target=worker1, name='w1', args=(5,))
t.start()

t = threading.Thread(target=worker2, name='w1')
t.start()
```

运行结果：

```python
welcome1
welcome2
welcome1
welcome2
welcome1
welcome2
welcome1
welcome2
Exception in thread w1:
Traceback (most recent call last):
  File "D:\Python27\Lib\threading.py", line 801, in __bootstrap_inner
    self.run()
  File "D:\Python27\Lib\threading.py", line 754, in run
    self.__target(*self.__args, **self.__kwargs)
  File "D:/Project/aliyun/test.py", line 25, in worker2
    raise Exception('Thread over')
Exception: Thread over

welcome1
worker1 over
```
















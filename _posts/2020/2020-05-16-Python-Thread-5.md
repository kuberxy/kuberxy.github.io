---
layout: post
title: Python线程同步 - Semaphore
date:   2020-05-16 10:02:42 +0800
category: Python
tags: [Python、线程]
typora-root-url: ../..
---





## 信号量简介

信号量（Semaphore），又称多元信号量。一个初始值为N的信号量，允许N个线程并发访问。



在访问资源之前，线程获取信号量，此时会进行如下操作：

- 将信号量的值减1；
- 如果信号量的值小于0，则进入等待状态；否则继续执行。

在访问完资源之后，线程释放信号量，此时将进行如下操作：

- 将信号量的值加1；
- 如果信号量的值小于1，唤醒一个等待中的线程。



## Semaphore对象

### 初始化一个semaphore对象

Semaphore用于初始化一个semaphore对象，它的语法如下：

```pyton
def __init__(self, value=1)
```

如下示例，创建一个信号量为3的semaphore对象

```python
import threading

s = threading.Semaphore(3)
print(type(s))
print(s)
```

运行结果：

```python
<class 'threading.Semaphore'>
<threading.Semaphore object at 0x0020DF50>
```



### 获取信号量

acquire方法用于获取信号量，其语法如下：

```python
def acquire(self, blocking=True, timeout=None)
```

说明：

- blocking：无法获取到信号量时是否阻塞。True表示阻塞，时间由timeout决定；False表示不阻塞，即立即返回。
- timeout：阻塞的时间。当blocking为False时，不能设定timeout。



如下示例：

```python
import threading
import logging

FORMAT = '[%(asctime)-15s] %(message)s'
logging.basicConfig(level=logging.INFO, format=FORMAT)

s = threading.Semaphore(1)

r1 = s.acquire()
logging.info(type(r1))
logging.info(r1)

r2 = s.acquire(blocking=True, timeout=3)
logging.info(type(r2))
logging.info(r2)

r3 = s.acquire(blocking=False)
logging.info(type(r3))
logging.info(r3)

r4 = s.acquire(blocking=False, timeout=3)
logging.info(type(r4))
logging.info(r4)
```

运行结果：

```
[2020-05-17 10:49:13,894] <class 'bool'>
[2020-05-17 10:49:13,894] True
[2020-05-17 10:49:16,900] <class 'bool'>
[2020-05-17 10:49:16,900] False
[2020-05-17 10:49:16,900] <class 'bool'>
[2020-05-17 10:49:16,900] False
Traceback (most recent call last):
  File "D:/Project/get_api_args/test.py", line 21, in <module>
    r4 = s.acquire(blocking=False, timeout=3)
  File "D:\Python35\Lib\threading.py", line 410, in acquire
    raise ValueError("can't specify timeout for non-blocking acquire")
ValueError: can't specify timeout for non-blocking acquire
```

需要注意的是，信号量永远不会小于0。因为当一个线程在acquire时，如果发现信号量为0，它要么被阻塞，要么就是立即返回。



### 释放信号量

release方法用于释放信号量，其语法如下：

```python
def release(self)
```

示例如下：

```python
import threading
import logging

FORMAT = '[%(asctime)-15s] %(message)s'
logging.basicConfig(level=logging.INFO, format=FORMAT)

s = threading.Semaphore(1)

rel1 = s.release()
logging.info('rel1 type: {}'.format(type(rel1)))
logging.info('rel1 value: {}'.format(rel1))

acq1 = s.acquire()
logging.info('acq1 type: {}'.format(type(acq1)))
logging.info('acq1 value: {}'.format(acq1))

acq2 = s.acquire(blocking=True,timeout=3)
logging.info('acq2 type: {}'.format(type(acq2)))
logging.info('acq2 value: {}'.format(acq2))

rel2 = s.release()
logging.info('rel2 type: {}'.format(type(rel2)))
logging.info('rel2 value: {}'.format(rel2))

acq3 = s.acquire()
logging.info('acq3 type: {}'.format(type(acq3)))
logging.info('acq3 value: {}'.format(acq3))
```

运行结果：

```python
[2020-05-17 11:07:08,308] rel1 type: <class 'NoneType'>
[2020-05-17 11:07:08,309] rel1 value: None
[2020-05-17 11:07:08,309] acq1 type: <class 'bool'>
[2020-05-17 11:07:08,309] acq1 value: True
[2020-05-17 11:07:08,309] acq2 type: <class 'bool'>
[2020-05-17 11:07:08,309] acq2 value: True
[2020-05-17 11:07:08,309] rel2 type: <class 'NoneType'>
[2020-05-17 11:07:08,309] rel2 value: None
[2020-05-17 11:07:08,309] acq3 type: <class 'bool'>
[2020-05-17 11:07:08,309] acq3 value: True
```

可以看到，每调用一次release，信号量就会加1。



### 示例

```python
import threading
import logging
import time

FORMAT = '[%(asctime)-15s] [%(threadName)s %(thread)d] %(message)s'
logging.basicConfig(level=logging.INFO, format=FORMAT)

def worker(s:threading.Semaphore):
    logging.info('in sub thread')
    logging.info('Worker Thread 1th acquire'.format(s.acquire()))
    logging.info('sub thread over')

# 初始化一个Semaphore对象，并设置信号量为3
s = threading.Semaphore(3)

# 主线程连续获取3次信号
logging.info('Main Thread 1th acquire {}'.format(s.acquire()))
logging.info('Main Thread 2th acquire {}'.format(s.acquire()))
logging.info('Main Thread 3th acquire {}'.format(s.acquire()))

# 启动一个工作线程，并获取信号，此时会阻塞
t = threading.Thread(target=worker, args=(s,))
t.start()

logging.info('='*50)
time.sleep(2)

# 非阻塞方式获取信号，由于没有信号此时直接返回False
logging.info('Main Thread 4th acquire {}'.format(s.acquire(False)))
# 阻塞方式获取信号，3秒后超时返回False
logging.info('Main Thread 5th acquire {}'.format(s.acquire(timeout=3)))

# 主线程释放一个信号，此时被阻塞的工作线程会被激活
logging.info('Main Thread release 1th')
s.release()
logging.info('end main')
```

运行结果：

```python
[2020-05-17 11:26:46,369] [MainThread 3244] Main Thread 1th acquire True
[2020-05-17 11:26:46,369] [MainThread 3244] Main Thread 2th acquire True
[2020-05-17 11:26:46,369] [MainThread 3244] Main Thread 3th acquire True
[2020-05-17 11:26:46,370] [Thread-1 2804] in sub thread
[2020-05-17 11:26:46,370] [MainThread 3244] ==================================================
[2020-05-17 11:26:48,370] [MainThread 3244] Main Thread 4th acquire False
[2020-05-17 11:26:51,378] [MainThread 3244] Main Thread 5th acquire False
[2020-05-17 11:26:51,379] [MainThread 3244] Main Thread release 1th
[2020-05-17 11:26:51,380] [MainThread 3244] end main
[2020-05-17 11:26:51,380] [Thread-1 2804] Worker Thread 1th acquire
[2020-05-17 11:26:51,380] [Thread-1 2804] sub thread over
```



## 应用示例：连接池

### 简介

连接池，是一种创建和管理连接的缓冲池技术。在一些场景中，连接资源通常是很有限的，并且建立一个连接的成本也很高，此时就会用到连接池技术。

通常，一个简单的连接池应该是这样的：有一个固定的容量。在需要连接时，能从池中获取连接；在不需要连接时，能将连接还回到池中，以供其他人使用。



### 简单实现

一个真正的连接池的实现会很复杂。这里我们只是为了理解信号量的应用，所以下面会使用伪代码做一个简单的功能实现：

```python
import threading
import logging
import random

FORMAT = '[%(asctime)s] %(thread)d %(threadName)s %(message)s'
logging.basicConfig(level=logging.INFO, format=FORMAT)

# 连接管理器
class Conn:
    # 建立一个连接
    def __init__(self, name):
        self.name = name

    def __repr__(self):
        return self.name

# 连接池
class Pool:
    # 初始化连接池
    def __init__(self, count:int):
        self.count = count
        self.pool = []
        for i in range(self.count):
            conn = self._connect('conn-{}'.format(i))
            self.pool.append(conn)

    # 创建一个连接
    def _connect(self, conn_name):
        return Conn(conn_name)

    # 从池中获取连接
    def get_conn(self):
        if len(self.pool) > 0:
            return self.pool.pop()

    # 将连接还回池中
    def release_conn(self, conn:Conn):
        self.pool.append(conn)


pool = Pool(3)

# 使用连接，并释放
def worker(pool:Pool):
    conn = pool.get_conn()
    logging.info(conn)

    threading.Event().wait(random.randint(1,4))
    pool.release_conn(conn)

for i in range(100):
    threading.Thread(target=worker, name='worker-{}'.format(i), args=(pool,)).start()
```



### 改进

上面的示例在运行过程中虽然没有出错，但我们要知道的是get_conn方法是线程不安全的。如果某一时刻连接池中只有一个连接，而刚好又有多个线程想要获取连接，此时它们判断连接池的长度大于0，于是都调用了pop方法。这样的问题该如何解决呢？

```python
import threading
import logging
import random

FORMAT = '[%(asctime)s] %(thread)d %(threadName)s %(message)s'
logging.basicConfig(level=logging.INFO, format=FORMAT)

# 连接管理器
class Conn:
    # 建立一个连接
    def __init__(self, name):
        self.name = name

    def __repr__(self):
        return self.name

# 连接池
class Pool:
    # 初始化连接池
    def __init__(self, count:int):
        self.semaphore = threading.Semaphore(count)
        self.count = count
        self.pool = []
        for i in range(self.count):
            conn = self._connect('conn-{}'.format(i))
            self.pool.append(conn)

    # 创建一个连接
    def _connect(self, conn_name):
        return Conn(conn_name)

    # 从池中获取连接
    def get_conn(self):
        if self.semaphore.acquire(timeout=3):
            conn = self.pool.pop()
            return conn
        else:
            return 'get conn timeout'

    # 将连接还回池中
    def release_conn(self, conn:Conn):
        self.pool.append(conn)
        self.semaphore.release()


pool = Pool(3)

# 使用连接，并释放
def worker(pool:Pool):
    conn = pool.get_conn()
    logging.info(conn)

    threading.Event().wait(random.randint(1,4))
    pool.release_conn(conn)

for i in range(100):
    threading.Thread(target=worker, name='worker-{}'.format(i), args=(pool,)).start()
```



### 思考

> self.pool.append(conn)是否需要加锁？

我们知道，信号量即使没有被使用，也能进行释放操作。也就是说，在上面的示例中，如果我们多次调用release_conn方法，连接池的容量会超过我们设定的最大值。那么，这个问题该怎么解决呢？

答案是，使用BoundedSemaphore有界信号量。

```python
import threading

S = threading.Semaphore(3)
S.release()

print('~'*20)

BS = threading.BoundedSemaphore(3)
BS.release()
```

运行结果：

```python
~~~~~~~~~~~~~~~~~~~~
Traceback (most recent call last):
  File "D:/Project/get_api_args/test.py", line 9, in <module>
    BS.release()
  File "D:\Python35\Lib\threading.py", line 480, in release
    raise ValueError("Semaphore released too many times")
ValueError: Semaphore released too many times
```

BoundedSemaphore虽然可以解决release超界的问题，但append并没有得到控制，这又该怎么办呢？

```python
def release_conn(self, conn:Conn):
    try:
        self.pool.append(conn)
        self.semaphore.release()
    
```










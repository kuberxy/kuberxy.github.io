---
layout: post
title: Python多线程 - threading库（3）
date:   2020-05-16 10:02:42 +0800
category: Python
tags: [Python、线程]
typora-root-url: ../..
---





对于一个Thread实例来说，它也会有一些自身的属性和方法。



## run() & start()

### run() - 执行线程函数

```python
import threading
import time

class MyThread(threading.Thread):
    def start(self):
        print('start'+'~'*10)
        super().start()

    def run(self):
        print('run'+'~'*10)
        super().run()

def worker():
    count = 0
    while True:
        if (count > 5):
            break
        print('worker running')
        count += 1
        time.sleep(1)

t = MyThread(target=worker, name='worker')
t.run()

print("="*50)

t.run()
```

运行结果：

```python
run~~~~~~~~~~
worker running
worker running
worker running
worker running
worker running
worker running
==================================================
run~~~~~~~~~~
Traceback (most recent call last):
  File "D:\Python35\Lib\threading.py", line 861, in run
    if self._target:
AttributeError: 'MyThread' object has no attribute '_target'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "D:/Project/get_api_args/test.py", line 27, in <module>
    t.run()
  File "D:/Project/get_api_args/test.py", line 11, in run
    super().run()
  File "D:\Python35\Lib\threading.py", line 866, in run
    del self._target, self._args, self._kwargs
AttributeError: _target
```

线程对象调用run方法后，会调用target目标函数，并且run方法只能调用一次



### start() - 启动线程

```python
import threading
import time

class MyThread(threading.Thread):
    def start(self):
        print('start'+'~'*10)
        super().start()

    def run(self):
        print('run'+'~'*10)
        super().run()

def worker():
    count = 0
    while True:
        if count > 5:
            break
        print('worker runing')
        count += 1
        time.sleep(1)

t = MyThread(target=worker, name='worker')

t.start()

time.sleep(5)
print('='*50)

t.start()
```

运行结果：

```python
start~~~~~~~~~~
run~~~~~~~~~~
worker runing
worker runing
worker runing
worker runing
worker runing
==================================================
worker runing
start~~~~~~~~~~
Traceback (most recent call last):
  File "D:/Project/get_api_args/test.py", line 29, in <module>
    t.start()
  File "D:/Project/get_api_args/test.py", line 7, in start
    super().start()
  File "D:\Python35\Lib\threading.py", line 840, in start
    raise RuntimeError("threads can only be started once")
RuntimeError: threads can only be started once
```

start方法会调用run方法，并且同一个线程对象只能调用一次start方法



### run() VS start()

```python
import threading
import time

class MyThread(threading.Thread):
    def start(self):
        print('start'+'~'*10)
        super().start()

    def run(self):
        print('run'+'~'*10)
        super().run()

def worker():
    count = 0
    while True:
        if count > 5:
            break
        print('worker running')
        print('current thread name is {}'.format(threading.current_thread().name))
        count += 1
        time.sleep(1)

t = MyThread(target=worker, name='worker')

# 以下分别执行start/run方法
t.start()   # current thread name is worker
# t.run()   # current thread name is MainThread
```

当使用start方法执行时，运行结果如下：

```python
start~~~~~~~~~~
run~~~~~~~~~~
worker running
current thread name is worker
worker running
current thread name is worker
worker running
current thread name is worker
worker running
current thread name is worker
worker running
current thread name is worker
worker running
current thread name is worker
```

当使用run方法执行时，运行结果如下：

```python
run~~~~~~~~~~
worker running
current thread name is MainThread
worker running
current thread name is MainThread
worker running
current thread name is MainThread
worker running
current thread name is MainThread
worker running
current thread name is MainThread
worker running
current thread name is MainThread
```

我们可以得出如下结论：

- start方法会启动一个子线程来执行目标函数
- run方法则直接在主线程中执行目标函数

因此，启动线程应该使用start方法，因为它才能启动多个线程。



## name - 获取线程的名字

```python
import threading
import time

def worker():
    count = 0
    while True:
        if (count > 5):
            break
        print('current thread name is {}'.format(threading.current_thread().name))
        count += 1
        time.sleep(1)

t = threading.Thread(name='worker', target=worker)
t.start()
```

运行结果：

```python
current thread name is worker
current thread name is worker
current thread name is worker
current thread name is worker
current thread name is worker
current thread name is worker
```



## ident - 获取线程的ID

```python
import threading
import time

def worker():
    pass

t = threading.Thread(name='worker', target=worker)

# 工作线程启动前的线程id
print('id {} start before'.format(t.ident))
t.start()
# 工作线程启动后的线程id
print('id {} start before'.format(t.ident))
```

运行结果：

```python
id None start before
id 7676 start before
```



## is_alive() - 返回线程是否活着

```python
import threading
import time

def worker():
    count = 0
    while True:
        if (count > 5):
            break
        count += 1
        time.sleep(1)

t = threading.Thread(name='worker', target=worker)
t.start()

while True:
    time.sleep(1)
    if t.is_alive():
        print('{} {} alive'.format(t.name, t.ident))
    else:
        print('{} {} dead'.format(t.name, t.ident))
```

运行结果：

```python
worker 1320 alive
worker 1320 alive
worker 1320 alive
worker 1320 alive
worker 1320 alive
worker 1320 alive
worker 1320 dead
worker 1320 dead
...
```










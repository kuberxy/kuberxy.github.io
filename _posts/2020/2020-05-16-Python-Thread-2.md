---
layout: post
title: Python多线程 - threading库（2）
date:   2020-05-16 10:02:42 +0800
category: Python
tags: [Python、线程]
typora-root-url: ../..
---



在threading库中，有一些内建的属性和方法。我们可以通过它们获取当前进程的一些信息。



## 获取当前线程的ID - get_ident()

```python
import threading
import time

def get_thread_info():
    print('id = {}'.format(threading._get_ident()))

def worker():
    count = 0
    get_thread_info()
    while True:
        if count > 5:
            break
        time.sleep(1)
        count += 1
        print("I'm working")

t = threading.Thread(target=worker, name='worker')

print('# main thread:')
get_thread_info()

print('='*50)

print('# child thread:')
t.start()
```

运行结果：

```python
# main thread:
id = 9008
==================================================
# child thread:
id = 2664
I'm working
I'm working
I'm working
I'm working
I'm working
I'm working
```



## 获取主线程对象 - main_thread()

```python
import threading
import time

def get_thread_info():
    print('main thread = {}'.format(threading.main_thread()))

def worker():
    count = 0
    get_thread_info()
    while True:
        if count > 5:
            break
        time.sleep(1)
        count += 1
        print("I'm working")

t = threading.Thread(target=worker, name='worker')

print('# main thread:')
get_thread_info()

print('='*50)

print('# child thread:')
t.start()
```

运行结果：

```python
# main thread:
main thread = <_MainThread(MainThread, started 8976)>
==================================================
# child thread:
main thread = <_MainThread(MainThread, started 8976)>
I'm working
I'm working
I'm working
I'm working
I'm working
I'm working
```



## 获取当前线程对象 - current_thread()

```python
import threading
import time

def get_thread_info():
    print('current thread = {}'.format(threading.current_thread()))

def worker():
    count = 0
    get_thread_info()
    while True:
        if count > 5:
            break
        time.sleep(1)
        count += 1
        print("I'm working")

t = threading.Thread(target=worker, name='worker')

print('# main thread:')
get_thread_info()

print('='*50)

print('# child thread:')
t.start()
```

运行结果：

```python
# main thread:
current thread = <_MainThread(MainThread, started 3792)>
==================================================
# child thread:
current thread = <Thread(worker, started 8768)>
I'm working
I'm working
I'm working
I'm working
I'm working
I'm working
```



## 获取当前处于alive状态的线程个数 - active_count()

```python
import threading
import time

def get_thread_info():
    print('active count = {}'.format(threading.activeCount()))

def worker():
    count = 0
    get_thread_info()
    while True:
        if count > 5:
            break
        time.sleep(1)
        count += 1
        print("I'm working")

t = threading.Thread(target=worker, name='worker')

print('# main thread:')
get_thread_info()

print('='*50)

print('# child thread:')
t.start()
```

运行结果：

```python
# main thread:
active count = 1
==================================================
# child thread:
active count = 2
I'm working
I'm working
I'm working
I'm working
I'm working
I'm working
```



## 获取所有活着的线程对象列表 - enumerate()

```python
import threading
import time

def get_thread_info():
    print('active thread = {}'.format(threading.enumerate()))

def worker():
    count = 0
    get_thread_info()
    while True:
        if count > 5:
            break
        time.sleep(1)
        count += 1
        print("I'm working")

t = threading.Thread(target=worker, name='worker')

print('# main thread:')
get_thread_info()

print('='*50)

print('# child thread:')
t.start()
```

运行结果：

```python
# main thread:
active thread = [<_MainThread(MainThread, started 9136)>]
==================================================
# child thread:
active thread = [<_MainThread(MainThread, started 9136)>, <Thread(worker, started 8924)>]
I'm working
I'm working
I'm working
I'm working
I'm working
I'm working
```


















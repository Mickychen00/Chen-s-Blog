---
layout: post
title: Multiprocessing学习笔记
toc: true
author: Chen Huang
tags: [并行, multiprocess,python]
categories: python
permalink: /:categories/:year/:month/:day/:title.html
---

**Multiprocessing**与**concurrent.futures**都属于Python的内置模块，二者的功能是使得执行者能够并行地运行某一Python软件。

<-more->
## Multiprocessing与Concurrent.futures功能简介
接下来，将以“睡眠”小程序来展示多进程计算的功用。
``` python
import multiprocessing
import time

start = time.perf_counter()
def do_something(seconds):
    print(f'Sleep {seconds} second')
    time.sleep(seconds)
    print('Done Sleeping')

processes = []
for _ in range(10):
    p = multiprocessing.Process(target=do_something,args=[1.5])
    p.start()
    processes.append(p)
for process in processes:
    process.join() #用于进程占用

finish = time.perf_counter()
print(f'Finished in {round(finish-start,2)} second(s)')
```
结果将是：
```
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Sleep 1.5 second
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Done Sleeping
Finished in 2.76 second(s)
```

但是还有一种更简便的方式，便是使用concurrent.futures模块：
``` python
start = time.perf_counter()
def do_something(seconds):
    print(f'Sleep {seconds} second')
    time.sleep(seconds)
    return 'Done Sleeping' #注意此处使用return而非print
with concurrent.futures.ProcessPoolExecutor() as executor:
    f1 = executor.submit(do_something,1)
    f2 = executor.submit(do_something,1)
    print(f1.result())
    print(f2.result())

finish = time.perf_counter()
print(f'Finished in {round(finish-start,2)} second(s)')
```
结果将是：
```
Sleep 1 second
Sleep 1 second
Done Sleeping
Done Sleeping
Finished in 1.06 second(s)
```
可见若使用concurrent.futures模块，将无需使用join来占用进程。

### Concurrent.futures的拓展：以for循环为例
我们可以将上面的concurrent模块的传入参数进行泛化。
``` python
with concurrent.futures.ProcessPoolExecutor() as executor:
    secs = [5,4,3,2,1]
    results = [executor.submit(do_something,sec) for sec in secs] # return的result列表化
    for f in concurrent.futures.as_completed(results):
        print(f.result())
finish = time.perf_counter()
print(f'Finished in {round(finish-start,2)} second(s)')
```
结果将是：
```
Sleep 5 second
Sleep 4 second
Sleep 2 second
Sleep 3 second
<Future at 0x7fcfd6c14278 state=running>
<Future at 0x7fcfd6bc94a8 state=pending>
<Future at 0x7fcfd6bc9550 state=pending>
<Future at 0x7fcfd6c23400 state=pending>
<Future at 0x7fcfd6c23358 state=pending>
Sleep 1 second
Finished in 5.08 second(s)
```
此外，除submit函数外，还可以使用map函数让代码更加简单：
``` python
with concurrent.futures.ProcessPoolExecutor() as executor:
    secs[5,4,3,2,1]
    results = executor.map(do_something, secs)
    for result in results:
        print(result)
finish = time.perf_counter()
print(f'Finished in {round(finish-start,2)} second(s)')
```
结果将是：
```
Sleep 4 second
Sleep 5 second
Sleep 3 second
Sleep 2 second
Sleep 1 second
Done Sleeping...5
Done Sleeping...4
Done Sleeping...3
Done Sleeping...2
Done Sleeping...1
Finished in 5.1 second(s)
```
至此，本文介绍了multiprocessing和concurrent.future模块的基本并行计算功能。二者的相同点在于**传导函数**和**输入参数**，但可能concurrent模块更加简洁。

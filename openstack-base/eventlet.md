+++
date = "2016-12-13T22:09:16+08:00"
title = "Eventlet"

+++

## 协程（coroutines）

协程和线程的相同点：

- 拥有自己独立的栈，局部变量和指令指针。

区别：

- 多个线程可以同时运行，而协程需要彼此协作的运行，任何时刻只有一个协程在执行，只有其显示的挂起时，执行才会暂停。
- 多个线程执行时，顺序不可控；多个协程执行时，顺序由程序控制执行。

协程的优点：

- 每个协程都有自己私有的stack和局部变量 
- 任一时间，只有一个协程在执行，无需对全局变量加锁
- 顺序可控，完全由程序控制执行的顺序

## Evenlet

[Eventlet] 作用于服务的多线程和wsgi server并发处理请求的情况下。

Eventlet 主要依赖于2个库：[greenlet] and select.epoll

greenlet 库是其并发的基础，eventlet 对其进行封装之后，构成了 GreenThread

select 库中的 epoll 则是其默认的网络通信模型. eventlet 中其中网络相关的库函数进行了改写。

GreenThread API:

*GreenThread Spawn*
 
* spawn(func, *args, **kwargs)
* spawn_n(func, *args, **kwargs)
* spawn_after(seconds, func, *args, **kwargs)

*GreenThread Control*

* sleep(seconds=0) 终止当前的绿色线程，以允许其它的绿色线程执行。
* eventlet.GreenPool 创建一个线程池
* eventlet.GreenPile 
* eventlet.Queue 

*Network Convenience Functions*

* connect(addr, family=socket.AF_INET, bind=None)
* listen(addr, family=socket.AF_INET, backlog=50) 
* serve(sock, handle, concurrency=1000) 
* wrap_ssl(sock, *a, **kw) 
*

## 两点:

1. 高性能网络库
2. GreenThread


[Eventlet]: http://eventlet.net/
[greenlet]: http://greenlet.readthedocs.io/en/latest/

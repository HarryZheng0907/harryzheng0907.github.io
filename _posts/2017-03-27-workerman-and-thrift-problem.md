---
layout:     post
title:      "解决Workerman+Thrift做服务会卡住的问题"
date:       2016-03-27 18:52:00
author:     "Harry"
header-img: "img/post-bg-2015.jpg"
tags:
    - Workerman
	- Thrift
---

### 解决Workerman+Thrift做服务会卡住的问题

用Workerman+Thrft做微服务时，发现服务有时会卡住，卡住以后除非重启进程，不然当前进程便再也处理不了其他请求，这会导致前置请求全部TImeout，与该服务相关的请求全部不可用，所以这个问题必须要解决，不然系统将变得非常不稳定。

由于这个问题是不定时出现的，所以无法在开发环境上重现，只能通过其他手段在正式环境上调试。

当进程再次被卡住时，我们便可以开始分析原因了

#### 1. 搞清楚进程卡住时在做什么

搞清楚进程卡住的时候是在做什么，我们可以通过strace来进行初步分析

```shell
strace -p 进程ID
```

![strace执行结果](http://oeii54s39.bkt.clouddn.com/strace%E7%BB%93%E6%9E%9C.png)

通过strace指令，我们可以看到，进程陷入了不断读取socket的死循环中

#### 2.检查导致死循环的socket有没有什么异常

```shell
lsof|grep 进程ID
```

![lsof结果](http://oeii54s39.bkt.clouddn.com/lsof%E7%BB%93%E6%9E%9C.png)

通过lsof指令，我们可以看到，这个socket的name是 can't identify protocol , 这个socket为什么会变成这样，google了一下还是没搞明白，但是可以确认的是，当读到这种状态的socket，程序将陷入死循环

### 3.查看陷入死循环的代码

``` shell
gdb -p 进程名
(gdb) source php源代码/.gdbinit
(gdb) zbacktrace
```

![gdb结果](http://oeii54s39.bkt.clouddn.com/gdb%E7%BB%93%E6%9E%9C.png)

通过这一步可以看到，只是在当前服务作为客户端访问上游服务，读取socket出现死循环

#### 结论

通过以上三步，我们基本上可以确定了，进程卡住的原因是当前服务访问上游服务，socket出现异常，导致feof无法正确判断当前socket是否已经读完，导致程序陷入了死循环

#### 解决方案

一开始的解决方案打算通过is_resource来判断当前socket是否异常，但是发现没有效果，所以最后只能通过判断读取socket时，两秒内都是读取到空字符串的方式，来确定socket是否异常，当发现异常时，直接报错，防止进程卡住
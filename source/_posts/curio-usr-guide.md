---
title: 使用python进行异步编程 -- curio使用指南（一）
date: 2017-11-03 21:55:58
tags:
- curio
- concurrent
categories:
- python
---
# 为什么使用协程

## C10K问题

在互联网开始的早期，使用互联网的人较少，一台服务器同时在线的连接也不是很多，所以最初的服务器设计的时候使用进程或者是线程的方式分配一个TCP连接，这个时候不存在C10K的难题。

当到了Web2.0的时代，互联网不再是单纯的浏览网页了，它开始需要进行交互，随着互联网的进一步发展，用户界面和界面交互都变得非常复杂起来，应用程序的逻辑也随之变的更加复杂，即时通信和在线的实时互动已经变的非常普遍了，假设每个用户都必须要与服务器保持一个或者多个TCP连接，而且每一个TCP连接需要占用一个进程(线程)的资源，这样的话，一个服务器的并发连接数是非常高的，一个普通的大一点网页服务的连接可能就过亿了。进程是操作系统最宝贵的资源，一台机器创建不了这么多进程，如果是C10k就要创建1万个进程，这个是操作系统无法承受的。就算是分布式系统，维持1亿用户在线也需要10万台服务器，成本是巨大的，只有FLAG、BAT这样的公司才有财力购买如此多的服务器。

## 怎么样解决C10K问题

既然有了C10K问题，程序员们就开始行动去解决它。为了解决这一问题，出现了「用同一进程/线程来同时处理若干连接」的思路，也就是I/O多路复用。于是FreeBSD推出了kqueue，Linux推出了epoll，Windows推出了IOCP。这些操作系统提供的功能就是为了解决C10K问题。因为Linux是互联网企业中使用率最高的操作系统，Epoll就成为C10K killer、高并发、高性能、异步非阻塞这些技术的代名词了。

epoll技术的编程模型就是异步非阻塞回调，也可以叫做Reactor，事件驱动，事件轮循（EventLoop）。Epoll就是为了解决C10K问题而生。使用Epoll技术，使得小公司也可以玩高并发。不需要购买很多服务器，有几台服务器就可以服务大量用户。Nginx，libevent，node.js这些就是Epoll时代的产物。

就这样C10K问题解决了，然后又来更高的问题，也就是C100K，C1M等。Epoll既然能解决C10K，解决什么C100K，C1M也是可以的。秘诀就是使用epoll模型，然后多买一些服务器就可以了。但是问题又来了

>异步嵌套回调太TM难写了。尤其是Node.js层层回调，缩进了几十层，要把程序员逼疯了。于是一个新的技术被提出来了，那就是协程（coroutine）。这个技术本质上也是异步非阻塞技术，它是将事件回调进行了包装，让程序员看不到里面的事件循环。程序员就像写阻塞代码一样简单。比如调用 client->recv() 等待接收数据时，就像阻塞代码一样写。实际上是底层库在执行recv时悄悄保存了一个状态，比如代码行数，局部变量的值。然后就跳回到EventLoop中了。什么时候真的数据到来时，它再把刚才保存的代码行数，局部变量值取出来，又开始继续执行。

>这个就像时间禁止的游戏一样，国王对巫师说“我必须马上得到宝物，不然就砍了你的脑袋”，巫师念了一句时间停止的咒语，直到过了1年后勇士们才把宝物送来。这时候巫师解开咒语，把宝物交给国王。这里国王就可以理解成协程，他根本没感觉到时间停止，在他停止到醒来期间发生了什么他不知道，也不关心。

>这就是协程的本质。协程是异步非阻塞的另外一种展现形式。Golang，Erlang，Lua协程都是这个模型。

说的有点远了，关于协程和epoll模型，你可能需要到网上找一些更加详细的资料看看，现在开始我们今天的主题 -- **curio库使用指南**。 
# curio-一个用同步写法进行异步编程的库

## 如何把同步的代码改成异步的

首先看一个同步的例子：

``` python
def handle(id):
    subject = get_subject_from_db(id)
    buyinfo = get_buyinfo(id)
    change = process(subject, buyinfo)
    notify_change(change)
    flush_cache(id)
```
可以看到，需要获取`subject`和`buyinfo`之后才能执行`process`，然后才能执行`notify_change`和`flush_cache`。

如果使用curio异步来写的话，就是:

``` python
import curio

async def handle(id):
    async with TaskGroup() as g:
        subject = await g.spawn(get_subject_from_db, id)
        buyinfo = await g.spawn(get_buyinfo, id)
    
    change = await process(subjetc.result, buginfo)
    await change.join()
    await notifu_change(change.result)
    await flush_cache(id)
```
其实就是把函数包装成一个Task对象或者说future对象，使用spawn可以把函数包装为Task，然后等待函数完成后，从Task的result属性获取返回值。              

下篇我们来聊一聊curio具体有哪些东西和怎么样去使用他们进行异步编程。

# 参考

[聊聊C10K问题及解决方案](http://rango.swoole.com/archives/381)

[Curio官方文档](http://curio.readthedocs.io/en/latest/index.html)




---
title: 使用python进行异步编程 -- curio使用指南（二）
date: 2017-11-04 23:36:17
tags:
- curio
- concurrent
categories:
- python
---
## 一些基本概念

* event_loop 事件循环：程序开启一个无限的循环，程序员会把一些函数注册到事件循环上。当满足事件发生的时候，调用相应的协程函数。

* coroutine 协程：协程对象，指一个使用async关键字定义的函数，它的调用不会立即执行函数，而是会返回一个协程对象。协程对象需要注册到事件循环，由事件循环调用。

* task 任务：一个协程对象就是一个原生可以挂起的函数，任务则是对协程进一步封装，其中包含任务的各种状态。

* future： 代表将来执行或没有执行的任务的结果。它和task上没有本质的区别

## 定义一个协程

定义协程很简单，使用python3.5的关键字async，可以像定义普通的函数一样：

``` python
# hello.py
import curio

async def countdown(n):
    while n > 0:
        print('T-minus', n)
        await curio.sleep(1)
        n -= 1

if __name__ == '__main__':
    curio.run(countdown, 10)
```

使用async定义一个协程(coroutine)，协程也是一种对象。协程不能直接运行，需要把协程加入到事件循环（loop），由后者在适当的时候调用协程。`curio`使用`curio kernel`来运行协程，run()方法可以开始kernel并且初始化Task。

## 创建一个task

协程对象不能直接运行，在运行`kernel`的时候，可以curio.spawn方法将协程包装成为了一个任务（task）对象。所谓task对象是Future类的子类。保存了协程运行后的状态，用于未来获取协程的结果。

``` python
# hello.py
import curio

async def countdown(n):
    while n > 0:
        print('T-minus', n)
        await curio.sleep(1)
        n -= 1

async def kid():
    print('Building the Millenium Falcon in Minecraft')
    await curio.sleep(1000)

async def parent():
    kid_task = await curio.spawn(kid)
    await curio.sleep(5)

    print("Let's go")
    count_task = await curio.spawn(countdown, 10)
    await count_task.join()

    print("We're leaving!")
    await kid_task.join()
    print('Leaving')

if __name__ == '__main__':
    curio.run(parent)
```
在当前程序中，parent()使用curio.spawn()创建新的子任务，当sleep一段时间后，countdown开始运行，join()方法会等待这个Task运行结束，在首先等待`countdown()`完成后，然后程序等待kid()完成，在你运行这个程序时，可以得到下面的结果：

``` bash
Building the Millenium Falcon in Minecraft
Let's go
T-minus 10
T-minus 9
T-minus 8
T-minus 7
T-minus 6
T-minus 5
T-minus 4
T-minus 3
T-minus 2
T-minus 1
We're leaving!
.... hangs ....
```

## curio monitor
在上个程序中的kid()将会阻塞1000秒，而parent的join方法会等待kid()的完成后才会结束。你可以将代码改成下面的样子来开启monitor：

``` python
...
if __name__ == '__main__':
    curio.run(parent, with_monitor=True)
```

运行程序，当程序阻塞在kid()的时候,打开monitor工具：

``` bash
bash % python3 -m curio.monitor
Curio Monitor: 4 tasks running
Type help for commands
curio >
```

使用`ps`命令查看：

``` bash
curio > ps
Task   State        Cycles     Timeout Task
------ ------------ ---------- ------- --------------------------------------------------
1      FUTURE_WAIT  1          None    Monitor.monitor_task
2      READ_WAIT    1          None    Kernel._run_coro.<locals>._kernel_task
3      TASK_JOIN    3          None    parent
4      TIME_SLEEP   1          None    kid
curio >
```

还可以使用`where`查看追踪task：

``` bash
curio > w 3
Stack for Task(id=3, name='parent', <coroutine object parent at 0x1024796d0>, state='TASK_JOIN') (most recent call last):
  File "hello.py", line 23, in parent
    await kid_task.join()
  File "/Users/beazley/Desktop/Projects/curio/curio/task.py", line 106, in join
    await self.wait()
  File "/Users/beazley/Desktop/Projects/curio/curio/task.py", line 117, in wait
    await _scheduler_wait(self.joining, 'TASK_JOIN')
  File "/Users/beazley/Desktop/Projects/curio/curio/traps.py", line 104, in _scheduler_wait
    yield (_trap_sched_wait, sched, state)

curio > w 4
Stack for Task(id=4, name='kid', <coroutine object kid at 0x102479990>, state='TIME_SLEEP') (most recent call last):
  File "hello.py", line 12, in kid
    await curio.sleep(1000)
  File "/Users/beazley/Desktop/Projects/curio/curio/task.py", line 440, in sleep
    return await _sleep(seconds, False)
  File "/Users/beazley/Desktop/Projects/curio/curio/traps.py", line 80, in _sleep
    return (yield (_trap_sleep, clock, absolute))

curio >
```

可以直接取消kid()：

``` bash
curio > cancel 4
Cancelling task 4
*** Connection closed by remote host ***
```

这样手动取消task会抛出TaskCancelled异常，表示程序没有正常运行。因此你需要结束task的时候需要在程序中手动取消：

``` python
async def parent():
    kid_task = await curio.spawn(kid)
    await curio.sleep(5)

    print("Let's go")
    count_task = await curio.spawn(countdown, 10)
    await count_task.join()

    print("We're leaving!")
    try:
        await curio.timeout_after(10, kid_task.join)
    except curio.TaskTimeout:
        print('I warned you!')
        await kid_task.cancel()
    print('Leaving!')
```

当然，你在`parent`取消`kid`的时候，`kid`可以捕捉到这个消除请求并且清除它：

``` python
async def kid():
    try:
        print('Building the Millenium Falcon in Minecraft')
        await curio.sleep(1000)
    except curio.CancelledError:
        print('Fine. Saving my work.')
        raise
```

## 同步机制

curio模块包含多种同步机制，它提供和线程一样的同步机制（`Event, Lock, Semaphore, and Condition`）。看下面使用Event的例子：

``` python
start_evt = curio.Event()

async def kid():
    print('Can I play?')
    await start_evt.wait()

    print('Building the Millenium Falcon in Minecraft')

    async with curio.TaskGroup() as f:
        await f.spawn(friend, 'Max')
        await f.spawn(friend, 'Lillian')
        await f.spawn(friend, 'Thomas')
        try:
            await curio.sleep(1000)
        except curio.CancelledError:
            print('Fine. Saving my work.')
            raise

async def parent():
    kid_task = await curio.spawn(kid)
    await curio.sleep(5)

    print('Yes, go play')
    await start_evt.set()
    await curio.sleep(5)

    print("Let's go")
    count_task = await curio.spawn(countdown, 10)
    await count_task.join()

    print("We're leaving!")
    try:
        await curio.timeout_after(10, kid_task.join)
    except curio.TaskTimeout:
        print('I warned you!')
        await kid_task.cancel()
    print('Leaving!')
```

在程序运行kid()的时候，`await start_evt.wait()`会等待，直到`await start_evt.set()`运行。

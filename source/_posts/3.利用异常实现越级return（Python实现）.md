---
title: 利用异常实现越级return（Python实现）
date: 2019-05-28 14:54:38
author: ChenyangGao
categories: 编程思想
tags: Python
thumbnail: https://cdn.pixabay.com/photo/2014/11/21/15/33/snake-540656_960_720.jpg

---

## 异常是一类消息

在面向对象的世界里，对象之间的交互表现为消息的传递。异常也是一种消息，就我的理解，异常并非错误，它经常用来表明存在错误，但本质上它是一类特殊的消息，关联着一种跳出局部范围的机制。而在<kbd>Python</kbd>的世界里，所有异常都必须是[BaseException](https://docs.python.org/3/library/exceptions.html#BaseException)的子类。
所谓异常是消息，指的是一个异常对象可以携带信息，捕获异常（就得到了一个异常对象）或者传递一个异常对象，你就可以从这个异常对象中提取信息。这个消息有时会被认为是一个指令或者信号，即便仅仅只是捕获到了一个特定类型的异常对象，例如，某个无限循环中有一个捕获异常的语句，当捕获到某个类型的异常对象时，它就认为接收到了退出的信号，快速、优雅地退出。

<!--more-->

## 处理异常的语句是流程控制语句

当异常发生时，异常现场所在的局部范围就将被丢弃，并且把相关统计信息存储在一个异常对象中向上冒泡。所谓的局部范围，包括控制语句内部、局部作用域等层次结构中。
如果定义了一个关于异常的类型系统，那么就会配套实现对异常现场的信息搜集策略、异常的冒泡机制和处理异常的语句。
处理异常的语句，专门针对异常消息，会根据条件来判定，对某个异常是放行还是压下。这种流程控制，设计专门用来控制异常对局部范围的破坏。

## 异常是一种强化版的return

一个函数调用完成时，就会在调用它的地方返回一个值。调用完成，意味着以下3种含义之一：
1. 执行完函数体中的代码，自动返回
2. 遇到return语句，返回
3. 抛出异常

当函数调用完成后，将会销毁调用时所临时创建的局部作用域并回收相关资源。如果函数`a`在它的函数体内调用了函数`b`，而`b`中冒泡了一个异常，未能在`a`中压下，那么就会在`a`冒泡给`a`的调用者。在多层次调用结构中，如果到达顶级层次依然不能被压下，那么程序将会退出。在一些解释器、虚拟机或者某些具有监控功能的执行环境中，程序因为异常退出时，将会自动记录下相关的信息，便于开发者理解问题所在。
抛出异常，未能被压下，将会不断地往上冒泡，并让局部环境不可被再次利用，程序的执行环境将自动回收这个局部环境所占用的资源。不过异常可以引用其它对象，可以被捕获，从中提取信息。正因为可以在退出函数时携带信息冒泡给函数的调用者，所以我说异常是一种强化版的return，调用者完全可以压下异常，从里面取出信息，就像是return给它的一样。

## 在Python中用异常实现一个越级return机制

```python leepfrog.py
import functools


__all__ = ['BubbleReturnSignal', 'leepfrog']


class BubbleReturnSignal(Exception):
    '''使用这个异常作为一个return信号，可以实现在多层
    调用中，快速退出n个调用层，并存储了一个返回值`r`

    :param n: 要跳出的调用层次层数
    :param r: 返回值，默认为None
    '''
    def __init__(self, n: int, r=None):
        self.n = n
        self.r = r

    def test(self):
        if self.n <= 0:
            return self.r
        else:
            self.n -= 1
            raise self


def leepfrog(fn):
    '这个装饰器，使函数自动处理BubbleReturnSignal异常信号，让调用层的退出自动进行'
    def wrapper(*args, **kwds):
        try:
            return fn(*args, **kwds)
        except BubbleReturnSignal as exc:
            return exc.test()
    return functools.update_wrapper(wrapper, fn)
```

> 下面文件中定义的函数，用于测试说明`leepfrog`装饰器函数的功用

```python
import functools
from leepfrog import leepfrog


def test(fn):
    '测试函数，打印一些信息，用于说明'
    def wrapper(*args, **kwds):
        try:
            return fn()
        except Exception as  exc:
            print(f'函数名称: {fn.__qualname__}\t异常参数: {exc.args}\t剩余冒泡: {exc.n}')
            raise
    return functools.update_wrapper(wrapper, fn)

@test
@leepfrog
def foo1():
    return foo2()

@test
@leepfrog
def foo2():
    return foo3()

@test
@leepfrog
def foo3():
    return foo4()

@test
@leepfrog
def foo4():
    return foo5()

@test
@leepfrog
def foo5():
    return bar()

@test
def bar():
    raise BubbleReturnSignal(4, 'value')
```

> 测试结果如下：

```python
>>> value = foo1()
函数名称: bar	异常参数: (4, 'value')	剩余冒泡: 4
函数名称: foo5	异常参数: (4, 'value')	剩余冒泡: 3
函数名称: foo4	异常参数: (4, 'value')	剩余冒泡: 2
函数名称: foo3	异常参数: (4, 'value')	剩余冒泡: 1
函数名称: foo2	异常参数: (4, 'value')	剩余冒泡: 0
>>> value
'value'
```

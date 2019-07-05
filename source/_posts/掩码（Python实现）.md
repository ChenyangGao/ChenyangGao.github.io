---
title: 掩码（Python实现）
author: ChenyangGao
date: 2019-07-04 18:02:35
categories: 编程思想
tags: Python
thumbnail: https://static.boredpanda.com/blog/wp-content/uploads/2018/08/Naamloos-5b80200f6c2e5-png__700.jpg
---

## 什么是掩码
**掩码**<kbd>mask</kbd>，是一个**位模式**，表示从一个二进制串中选出的位的集合。
**掩码**<kbd>mask</kbd>，是一个**0-1序列**，表示从一个二值序列中选出的位置的集合。

**Note** 掩码是一类特殊的**状态**<kbd>state</kbd>，或者用于表示这类状态，表示一类二值序列状态（向量<kbd>vector</kbd>）。
$$\mathrm{mask}: \{b_{1, 1}, b_{1, 2}\} \times \{b_{2, 1}, b_{2, 2}\} \times ... \times \{b_{n, 1}, b_{n, 2}\} \leftrightarrow \{0,1\}^n$$


## 一个特殊的例子Flag
**标志**<kbd>flag</kbd>，是只有1位的掩码。
**标志**<kbd>flag</kbd>，是一个取值0-1二值量。

**Note** 标志也是一类特殊的**状态**<kbd>state</kbd>，或者用于表示这类状态，表示一类二值状态（标量<kbd>scalar</kbd>）。
$$\mathrm{flag}: \{b_1, b_2\} \leftrightarrow \{0,1\}$$

<!--more-->

```python
class Flag:

    def __init__(self):
        self._value = 0

    @property
    def flag(self) -> int:
        return self._value

    def set(self):
        'set the flag to 1'
        self._value = 1

    def clear(self):
        'set the flag to 0'
        self._value = 0

    def reverse(self):
        'reverse/invert the flag, 0 to 1, 1 to 0'
        self._value ^= 1

    def isset(self) -> bool:
        'Is flag = 1?'
        return self._value is 1
```


## 代码实现

**Note** 因为在<kbd>Python</kbd>中，<kbd>int</kbd>是不可变类型，操作的结果往往会产生新的<kbd>int</kbd>对象，这是低效的。建议使用<kbd>C</kbd>扩展或者某些可改动内存的结构，至少保证位操作可以复用原来的内存区域。

```python mask.py
from typing import TypeVar, Generic, Union


T = TypeVar('T')


class MaskOpMixin(Generic[T]):

    def __eq__(self, o: Union[int, T]) -> bool:
        if isinstance(o, type(self)):
            o = o._value
        return self._value == o

    def __invert__(self):
        return type(self)(~self._value)

    def __and__(self, o: Union[int, T]) -> T:
        cls = type(self)
        if isinstance(o, cls):
            o = o._value
        return cls(self._value & o)

    def __or__(self, o: Union[int, T]) -> T:
        cls = type(self)
        if isinstance(o, cls):
            o = o._value
        return cls(self._value | o)

    def __xor__(self, o: Union[int, T]) -> T:
        cls = type(self)
        if isinstance(o, cls):
            o = o._value
        return cls(self._value ^ o)

    __rand__ = __and__
    __ror__ = __or__
    __rxor__ = __xor__

    def __iand__(self, o: Union[int, T]) -> T:
        if isinstance(o, type(self)):
            o = o._value
        self._value &= o
        return self

    def __ior__(self, o: Union[int, T]) -> T:
        if isinstance(o, type(self)):
            o = o._value
        self._value |= o
        return self

    def __ixor__(self, o: Union[int, T]) -> T:
        if isinstance(o, type(self)):
            o = o._value
        self._value ^= o
        return self

    def __sub__(self, o: Union[int, T]) -> T:
        cls = type(self)
        if isinstance(o, cls):
            o = o._value
        # 🤔 l & ~r == (l | r) ^ r
        return cls(self._value & ~o)

    def __isub__(self, o: Union[int, T]) -> T:
        if isinstance(o, type(self)):
            o = o._value
        self._value &= ~o
        return self


class MaskBase:

    def __init__(self, mask: int = 0) -> None:
        self._value = mask

    @property
    def mask(self) -> int:
        return self._value

    @property
    def binmask(self) -> str:
        return bin(self._value)

    def __repr__(self) -> str:
        return f'{type(self).__name__}({bin(self._value)})'

    def set(self, m: int) -> None:
        'set mask by offset m'
        self._value |= 1 << m

    def clear(self, m: int) -> None:
        'clear mask by offset m'
        self._value &= ~(1 << m)

    def reverse(self, m: Union[int, MaskBase]) -> None:
        'reverse/invert mask by offset m'
        self._value ^= 1 << m

    def test(self, m: int) -> bool:
        'test mask by offset m'
        return bool(self._value & 1 << m)


class Mask(MaskBase, MaskOpMixin[MaskBase]):

    def set(self, m: Union[int, MaskBase]) -> None:
        'set mask by mask m'
        self |= m

    def clear(self, m: Union[int, MaskBase]) -> None:
        'clear mask by mask m'
        self -= m

    def reverse(self, m: Union[int, MaskBase]) -> None:
        'reverse/invert mask by mask m'
        self ^= m

    def test(self, m: Union[int, MaskBase]) -> bool:
        'clear mask by mask m'
        return self & m == m
```

## 简单测试

```python
>>> from mask import *
>>> EVENT0 = 1 << 0
>>> EVENT1 = 1 << 1
>>> EVENT2 = 1 << 2
>>> EVENT3 = 1 << 3
>>> m = Mask(EVENT1)
>>> m.set(EVENT0 | EVENT3)
>>> m
Mask(0b1011)
>>> m.test(EVENT0 | EVENT1)
True
>>> m.clear(EVENT0 | EVENT1)
>>> m
Mask(0b1000)
>>> m.reverse((1 << 4) - 1)
>>> m
Mask(0b111)
```

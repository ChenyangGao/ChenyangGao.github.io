---
title: Python脚本作为配置文件加载
date: 2022-08-27 19:37:00
author: ChenyangGao
categories: 脚本工具
tags: Python,Script
thumbnail: https://cdn.pixabay.com/photo/2015/05/17/20/39/snake-771541_640.jpg

---

## 背景

在使用 `Python` 开发时，如果要把对象 `obj` 当作字典使用，可以直接操作 `obj.__dict__`（如果有的话），具体而言：

|     | 对象操作 | 字典操作
---| --- | --- 
查  | `obj.foo` | `obj.__dict__["foo"]`
改  | `obj.foo = "bar"` | `obj.__dict__["foo"] = "bar"`
删  | `del obj.foo` | `del obj.__dict__["foo"]`

对象操作和字典（这里指的是对象的命名空间）操作并不等价，字典操作并不执行复杂的**方法查找**和**描述符**，步骤更少，一般而言更快。

而要把字典 `d` 视为对象操作，则要专门包装一下

 ```python
 class DictAsObject:
 
     def __init__(self, d):
         self.__dict__ = d

    def __repr__(self):
        return f"{type(self).__qualname__}({self.__dict__!r})"

dict_as_obj = DictAsObject
 ```
 
也可以直接用标准库中的包装，例如

```python
from argparse import Namespace

def dict_as_obj(d):
    ns = Namespace()
    ns.__dict__ = d
    return ns
```

令 `ns = dict_as_obj(d)`，具体而言：

|     | 对象操作 | 字典操作
---| --- | --- 
查  | `ns.foo` | `d["foo"]`
改  | `ns.foo = "bar"` | `d["foo"] = "bar"`
删  | `del ns.foo` | `del d["foo"]`

更一般地说，就是要把下列操作建立对应

|     | 对象操作 | 字典操作
---| --- | --- 
查  | `__getattr__` | `__getitem__`
改  | `__setattr__` | `__setitem__`
删  | `__delattr__` | `__delitem__`

有些不同的是，在抛异常时，对象操作抛出 `AttributeError`，字典操作抛出 `KeyError`。

有一些第三方模块实现了上述对应关系的可相互替代，例如

- https://pypi.org/project/attrdict/
- https://pypi.org/search/?q=attrdict
- https://github.com/mewwts/addict
- https://github.com/sanand0/orderedattrdict

我有一个现实的需求，希望能直接以 `Python` 脚本作为配置文件：读取一个脚本，并以 `exec` 函数运行之，然后把脚本运行时的全局命名空间作为配置，如果里面用到了不存在的名字，应该自动把它设为空字典 `{}`。操作策略借鉴了 [addict](https://pypi.org/project/addict/)，操作如下

```shell
>>> from addict import Dict
>>> mapping = Dict()
>>> mapping.a.b.c.d.e = 2
>>> mapping
{'a': {'b': {'c': {'d': {'e': 2}}}}}
```

为此，我专门写了一个模块，来适配我的需求，造轮子并非没有意义，它能加深我的思考，并且享受 DIY 的快乐。

<!--more-->

## 代码实现

> **注意**：使用下面的代码，请用 `Python` <kdb>3.9</kdb> 或者以上版本。

文件名称是 `dictattr.py`，`Python` 实现代码如下：

```python dictattr
#!/usr/bin/env python3
# coding: utf-8

__author__ = "ChenyangGao <https://chenyanggao.github.io/>"
__version__ = (0, 1)
__all__ = ["AttrDict", "DictAttr", "Properties"]

import builtins

from pathlib import Path
from types import CodeType
from typing import MutableMapping


class AttrDict(dict):
    "扩展的 dict 类型，它的实例的 __dict__ 属性（命名空间）就是该字典本身，因此执行 __getattr__、__setattr__、__delattr__ 操作的是该字典本身"
    def __init__(self, *args, **kwds):
        super().__init__(*args, **kwds)
        self.__dict__ = self


@MutableMapping.register
class DictAttr:
    """这个类型实现了 collections.abc.MutableMapping 的接口，作为其结构子类型(structural subtyping)，即鸭子类型(duck typing)，而非名义子类型(nominal subtyping)。
    :param init_dict: 如果不为 None，会用于替换实例的 __dict__ 属性
  
    :: doctest
    >>> d = dict(foo={})
    >>> da = DictAttr(d)
    >>> da
    DictAttr({'foo': {}})
    >>> da.foo
    DictAttr({})
    >>> da.foo.bar = 1
    >>> da
    DictAttr({'foo': {'bar': 1}})
    >>> d
    {'foo': {'bar': 1}}
    >>> da.__dict__ is d
    True
    """
    def __init__(self, init_dict: None | dict = None):
        if init_dict is not None:
            self.__dict__ = init_dict

    def __contains__(self, key):
        return key in self.__dict__

    def __iter__(self):
        return iter(self.__dict__)

    def __len__(self):
        return len(self.__dict__)

    def __repr__(self):
        return f"{type(self).__qualname__}({self.__dict__!r})"

    def __getattribute__(self, attr):
        "如果 attr 是字符串且前后都附缀两个下划线 __，则执行原始行为。否则行为相当于 __getitem__，但在取不到值时，会再执行原始行为。可能抛出 Attribute Error 异常。"
        if type(attr) is str and attr[:2] == attr[-2:] == "__":
            return super().__getattribute__(attr)
        try:
            return self[attr]
        except KeyError:
            return super().__getattribute__(attr)

    def __getitem__(self, key):
        "从 __dict__ 中取值，当值是 dict 类型时，会被 type(self) 对应的类包装"
        val = self.__dict__[key]
        if type(val) is dict:
            return type(self)(val)
        return val

    def __setitem__(self, key, val):
        "向 __dict__ 中设键值"
        self.__dict__[key] = val

    def __delitem__(self, key):
        "从 __dict__ 中删键"
        del self.__dict__[key]


class Properties(DictAttr):
    """这个类型实现了 collections.abc.MutableMapping 的接口，作为其结构子类型(structural subtyping)，即鸭子类型(duck typing)，而非名义子类型(nominal subtyping)。
    :param init_dict: 如果不为 None，会用于替换实例的 __dict__ 属性
  
    :: doctest
    >>> d = dict(foo={})
    >>> props = Properties(d)
    >>> props
    Properties({'foo': {}})
    >>> props.foo
    Properties({})
    >>> props.foo.bar = 1
    >>> props
    Properties({'foo': {'bar': 1}})
    >>> props.bar
    Properties({})
    >>> props
    Properties({'foo': {'bar': 1}, 'bar': {}})
    >>> props.baz.bay.baz = 1
    >>> props
    Properties({'foo': {'bar': 1}, 'bar': {}, 'baz': {'bay': {'baz': 1}}})
    >>> d
    {'foo': {'bar': 1}, 'bar': {}, 'baz': {'bay': {'baz': 1}}}
    >>> props.__dict__ is d
    True
    """
    def __getitem__(self, key):
        "从 __dict__ 中取值。首先执行基类 DictAttr 的原始行为；如果取不到值，再从 builtins.__dict__ 中取值；如果取不到，则对于非下划线前缀的字符串属性，把值设为 {}，再执行一次基类的原始行为，否则抛出 KeyError。"
        try:
            return super().__getitem__(key)
        except KeyError:
            try:
                return builtins.__dict__[key]
            except KeyError:
                pass
            if type(key) is not str or key.startswith("_"):
                raise
            self[key] = {}
            return super().__getitem__(key)

    def __abs__(self) -> dict:
        "创建一个 dict 副本，如果有 __all__ 字段，则筛选出所有在 __all__ 中的键，否则只筛选出键是字符串类型且非下划线前缀的键值对"
        d = self.__dict__
        if "__all__" in d:
            return {
                k: abs(v) if isinstance(v, Properties) else v
                for k, v in ((k, d[k]) for k in d["__all__"] if k in d)
            }
        return {
            k: abs(v) if isinstance(v, Properties) else v
            for k, v in d.items()
            if type(k) is str and not k.startswith("_")
        }

    def __call__(self, source: str | bytes | CodeType | Path):
        """执行一段 Python 代码，并更新 __dict__ 属性（命名空间）
        :param source: Python 代码或者代码文件的路径
        :return: 返回实例本身

        :: tips
        - 请将变量名提前注入 __dict__，否则缺失时自动设为 {}
        - 如果代码中有 import 命令，请确保把需要的路径加到 sys.path 中，避免找不到模块
        - 属性名有前缀下划线 _，用于说明想要过滤掉

        :: doctest
        >>> # 构造一段 Python 代码
        >>> code = 'from math import nan as _nan, inf as _inf\\nz = _inf\\ny.z = _nan\\nx.y.z = y.z\\nfoo = sum([1,2, 3])\\nbar = abs(1+1j)'
        >>> print(code)
        from math import nan as _nan, inf as _inf
        z = _inf
        y.z = _nan
        x.y.z = y.z
        foo = sum([1,2, 3])
        bar = abs(1+1j)
        >>> props = Properties()
        >>> props
        Properties({})
        >>> props(code)
        Properties({'_nan': nan, '_inf': inf, 'z': inf, 'y': {'z': nan}, 'x': {'y': {'z': nan}}, 'foo': 6, 'bar': 1.4142135623730951})
        >>> print(abs(props))
        {'z': inf, 'y': {'z': nan}, 'x': {'y': {'z': nan}}, 'foo': 6, 'bar': 1.4142135623730951}
        """
        code: str | bytes | CodeType
        if isinstance(source, Path):
            code = source.open(encoding="utf_8").read()
        else:
            code = source
        exec(code, None, self) # type: ignore
        return self


if __name__ == "__main__":
    import doctest
    doctest.testmod(verbose=True)


```

## 使用说明

代码中有文档测试 [doctest](https://docs.python.org/3/library/doctest.html)，不妨运行一下

```shell
$ python dictattr.py
Trying:
    d = dict(foo={})
Expecting nothing
ok
Trying:
    da = DictAttr(d)
Expecting nothing
ok
Trying:
    da
Expecting:
    DictAttr({'foo': {}})
ok
Trying:
    da.foo
Expecting:
    DictAttr({})
ok
Trying:
    da.foo.bar = 1
Expecting nothing
ok
Trying:
    da
Expecting:
    DictAttr({'foo': {'bar': 1}})
ok
Trying:
    d
Expecting:
    {'foo': {'bar': 1}}
ok
Trying:
    da.__dict__ is d
Expecting:
    True
ok
Trying:
    d = dict(foo={})
Expecting nothing
ok
Trying:
    props = Properties(d)
Expecting nothing
ok
Trying:
    props
Expecting:
    Properties({'foo': {}})
ok
Trying:
    props.foo
Expecting:
    Properties({})
ok
Trying:
    props.foo.bar = 1
Expecting nothing
ok
Trying:
    props
Expecting:
    Properties({'foo': {'bar': 1}})
ok
Trying:
    props.bar
Expecting:
    Properties({})
ok
Trying:
    props
Expecting:
    Properties({'foo': {'bar': 1}, 'bar': {}})
ok
Trying:
    props.baz.bay.baz = 1
Expecting nothing
ok
Trying:
    props
Expecting:
    Properties({'foo': {'bar': 1}, 'bar': {}, 'baz': {'bay': {'baz': 1}}})
ok
Trying:
    d
Expecting:
    {'foo': {'bar': 1}, 'bar': {}, 'baz': {'bay': {'baz': 1}}}
ok
Trying:
    props.__dict__ is d
Expecting:
    True
ok
Trying:
    code = 'from math import nan as _nan, inf as _inf\nz = _inf\ny.z = _nan\nx.y.z = y.z\nfoo = sum([1,2, 3])\nbar = abs(1+1j)'
Expecting nothing
ok
Trying:
    print(code)
Expecting:
    from math import nan as _nan, inf as _inf
    z = _inf
    y.z = _nan
    x.y.z = y.z
    foo = sum([1,2, 3])
    bar = abs(1+1j)
ok
Trying:
    props = Properties()
Expecting nothing
ok
Trying:
    props
Expecting:
    Properties({})
ok
Trying:
    props(code)
Expecting:
    Properties({'_nan': nan, '_inf': inf, 'z': inf, 'y': {'z': nan}, 'x': {'y': {'z': nan}}, 'foo': 6, 'bar': 1.4142135623730951})
ok
Trying:
    print(abs(props))
Expecting:
    {'z': inf, 'y': {'z': nan}, 'x': {'y': {'z': nan}}, 'foo': 6, 'bar': 1.4142135623730951}
ok
14 items had no tests:
    __main__
    __main__.AttrDict
    __main__.AttrDict.__init__
    __main__.DictAttr.__contains__
    __main__.DictAttr.__delitem__
    __main__.DictAttr.__getattribute__
    __main__.DictAttr.__getitem__
    __main__.DictAttr.__init__
    __main__.DictAttr.__iter__
    __main__.DictAttr.__len__
    __main__.DictAttr.__repr__
    __main__.DictAttr.__setitem__
    __main__.Properties.__abs__
    __main__.Properties.__getitem__
3 items passed all tests:
   8 tests in __main__.DictAttr
  12 tests in __main__.Properties
   6 tests in __main__.Properties.__call__
26 tests in 17 items.
26 passed and 0 failed.
```

---

假设现在有个 `Python` 脚本 `config.py` 作为配置文件，内容如下

```python config.py
## 以下是数据库连接信息

# 数据库连接函数
db.connect = __import__("sqlite3").connect
# 主机
db.host = None
# 端口
db.port = None
# 用户名
db.username = None
# 密码
db.password = None
# 数据库
db.database = ":memory:"

## 下面是日志的相关配置
 
# 日志输出的最低级别
log.level=__import__("logging").INFO
# 日志的输出格式
log.format="[\x1b[1m%(asctime)-15s\x1b[0m] \x1b[36;1m%(name)s\x1b[0m(\x1b[31;1m%(levelname)s\x1b[0m) ➜ %(message)s"

## json 的相关配置

# json 序列化函数
json.serialize = __import__("json").dumps
# json 反序列化函数
json.deserialize = __import__("json").loads

## 一些需要的 64 位浮点数常量

from math import nan as _nan, inf as _inf

Number.SQRT_2 = abs(1+1j)
Number.EPSILON = 2.0 ** -52
Number.MIN_VALUE = 2.0 ** -1074
Number.MAX_VALUE = sum(2.0 ** (1023 - i) for i in range(53))
Number.MAX_SAFE_INTEGER = 2.0 ** 53 - 1
Number.MIN_SAFE_INTEGER = -Number.MAX_SAFE_INTEGER
Number.NaN = _nan
Number.POSITIVE_INFINITY = _inf
Number.NEGATIVE_INFINITY = -_inf
```

可以运行下面代码进行加载

```shell
>>> from dictattr import Properties
>>> props = Properties()
>>> props("config.py")
Properties({'db': {'connect': <built-in function connect>, 'host': None, 'port': None, 'username': None, 'password': None, 'database': ':memory:'}, 'log': {'level': 20, 'format': '[\x1b[1m%(asctime)-15s\x1b[0m] \x1b[36;1m%(name)s\x1b[0m(\x1b[31;1m%(levelname)s\x1b[0m) ➜ %(message)s'}, 'json': {'serialize': <function dumps at 0x76829bcf70>, 'deserialize': <function loads at 0x76829bd480>}, '_nan': nan, '_inf': inf, 'Number': {'SQRT_2': 1.4142135623730951, 'EPSILON': 2.220446049250313e-16, 'MIN_VALUE': 5e-324, 'MAX_VALUE': 1.7976931348623157e+308, 'MAX_SAFE_INTEGER': 9007199254740991.0, 'MIN_SAFE_INTEGER': -9007199254740991.0, 'NaN': Properties({}), 'POSITIVE_INFINITY': inf, 'NEGATIVE_INFINITY': -inf}, 'nan': {}})
```

然后就可以根据需要取用属性了

```shell
>>> props.Number
Properties({'SQRT_2': 1.4142135623730951, 'EPSILON': 2.220446049250313e-16, 'MIN_VALUE': 5e-324, 'MAX_VALUE': 1.7976931348623157e+308, 'MAX_SAFE_INTEGER': 9007199254740991.0, 'MIN_SAFE_INTEGER': -9007199254740991.0, 'NaN': Properties({}), 'POSITIVE_INFINITY': inf, 'NEGATIVE_INFINITY': -inf})
>>> props["Number"].SQRT_2
1.4142135623730951
>>> props.db
Properties({'connect': <built-in function connect>, 'host': None, 'port': None, 'username': None, 'password': None, 'database': ':memory:'})
>>> props["json"]
Properties({'serialize': <function dumps at 0x76829bcf70>, 'deserialize': <function loads at 0x76829bd480>})
```


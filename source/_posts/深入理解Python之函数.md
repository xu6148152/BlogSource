---
title: 深入理解Python之函数
tags: Python
date: 2017-04-22 17:28:44
---


函数在`Python`是首要类对象。
首要对象： 

* 运行时创建 
* 能赋值给变量或者数据结构中的元素
* 接收参数
* 返回结果

### 把函数当做对象

```python

def factorial(n):
    '''returns n!'''
    return 1 if n < 2 else n * factorial(n - 1)


def test_factorial():
    print(factorial(42))
    print(factorial.__doc__)
    print(type(factorial))

    fact = factorial
    print(fact)
    print(fact(5))
    print(map(factorial, range(11)))
    print(list(map(fact, range(11))))

```

* 创建函数，读取`__doc__`属性显示函数注释，使用`type`显示函数类型，其是个`function`类

* 能够将函数赋值给变量

### 高阶函数

函数接收一个函数作为参数，并返回一个函数作为结果，称为*高阶函数*

```python

fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
print(sorted(fruits, key=len))
print(sorted(fruits, key=reverse))

def reverse(word):
    return word[::-1]    
```

#### map, filter, reduce的现代替代品

##### map, filter

`Python3`中`map`,`filter`返回生成器，在`pythn2`中返回序列

```python
print(list(map(factorial, range(6))))
print(list(map(factorial, filter(lambda n: n % 2, range(6)))))
print([factorial(n) for n in range(6) if n % 2])

```

##### reduce

`python3`中，`reduce`被移到`functools`中
```python
from functools import reduce
from operator import add
print(reduce(add, range(100)))

```

* 这里`reduce`的作用相当于`sum`做累加

### 匿名函数

`lambda`关键字创建一个匿名函数

```python
print(sorted(fruits, key=lambda word: word[::-1]))
```

### 可调用对象的七种风格

#### *用户定义函数*
使用`def`或者`lambda`创建的表达式

#### *内建的函数*
`C`语言实现的函数

#### *内建方法*
`C`语言实现的方法

#### *方法*
在类体中定义的函数

#### *类*
当触发时，类运行`__new__`方法创建对象，然后在`__init__`中初始化。最后返回对象给调用者。因为`Python`中没有`new`操作符，因此调用类就像调用函数
#### *类实例*
if类定义了`__call__`方法，它的对象可能像函数一样触发
#### *生成器函数*
函数或者方法使用`yield`关键字。当调用时，返回生成器对象


* 可以使用`callable`来检查对象是否可以调用

```python
[callable(obj) for obj in (abs, str, 13)]
```

### 用户定义的可调用类型
只要实现`__call__`就能让对象表现得像函数

```python
class BingoCage:
    def __init__(self, items):
        self._items = list(items)
        random.shuffle(self._items)

    def pick(self):
        try:
            return self._items.pop()
        except IndexError:
            raise LookupError('pick from empty BingoCage')

    def __call__(self):
        return self.pick()

bingo = BingoCage(range(3))
print(bingo.pick())
print(bingo())
print(callable(bingo))
```

### 函数自省

```python
class C: pass

obj = C()

def func(): pass

print(sorted(set(dir(func)) - set(dir(obj))))

result: ['__annotations__', '__call__', '__closure__', '__code__', '__defaults__', '__get__', '__globals__', '__kwdefaults__', '__name__', '__qualname__']
```

### 函数关键字参数
这是`Python3`的新特性， `*`表示序列， `**`表示字典，关键字参数是可选参数

```python
def tag(name, *content, cls=None, **attrs):
    """Generate one or more HTML tag"""
    if cls is not None:
        attrs['class'] = cls
    if attrs:
        attr_str = ''.join(' %s="%s"' % (attr, value) for attr, value in sorted(attrs.items()))

    else:
        attr_str = ''
    if content:
        return '\n'.join('<%s%s>%s</%s>' % (name, attr_str, c, name) for c in content)

    else:
        return '<%s%s />' % (name, attr_str)
```

### 函数参数信息

函数对象中，`__defaults__`持有位置和关键字参数的默认值，是`tuple`类型。关键字参数的默认值在`__kwdefaults__`

```Python
def clip(text, max_len=80):
    """Return text clipped at the last space before or after max_len"""
    end = None
    if len(text) > max_len:
        space_before = text.rfind(' ', 0, max_len)
        if space_before >= 0:
            end = space_before
        else:
            space_after = text.rfind(' ', max_len)
            if space_after >= 0:
                end = space_after
    if end is None:
        end = len(text)
    return text[:end].rstrip()

print(clip.__defaults__)
print(clip.__code__)
print(clip.__code__.co_argcount)

from inspect import signature
sig = signature(clip)
print(sig)
print(str(sig))
for name, param in sig.parameters.items():
    print(param.kind, ':', name, '=', param.default)
```

使用`inspect/Signature`可以实现类似查看函数元数据的功能

### 函数注解

`Python3`能够添加元数据到函数声明的参数和返回值上

```python
def clip(text: str, max_len: 'int > 0'=80) -> str:
```

### 函数编程的包
#### operator模块

```python
from functools import reduce
from operator import mul

def fact(n):
    return reduce(lambda a, b: a * b, range(1, n + 1))

def fact_op(n):
    return reduce(mul, range(1, n + 1))
```

```python
from operator import methodcaller
s = 'The time has come'
upcase = methodcaller('upper')
print(upcase(s))
hiphenate = methodcaller('replace', ' ', '-')
print(hiphenate(s))
```
### 使用functools.partial冻结参数
`functools.partial`是一种高阶函数，允许应用函数功能的一部分。给定一个函数，部分功能会产生一个新的函数

```python
from operator import mul
from functools import partial
triple = partial(mul, 3)
print(triple(7))
print(list(map(triple, range(1, 10))))
```

### 总结

本文主要描述了`Python`中的函数。主要思想是函数能够赋值给变量，传给其他函数，将他们存在数据结构中，访问函数属性，允许框架和工具操作相关信息。高阶函数的使用。调用的七种形式
